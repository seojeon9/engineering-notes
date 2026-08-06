---
created: 2026-08-06
updated: 2026-08-06

tags:
  - kubernetes
  - persistent-volume
  - nfs
  - firewall
  - exports
  - troubleshooting

difficulty: Intermediate
---

# PersistentVolume 마운트 실패 진단 (NFS 중심)

> **"PV 마운트가 안 될 때, 무엇을 어떤 순서로 보는가"** 를 위한 문서.
> "마운트 실패"는 하나의 증상이지 하나의 원인이 아니다.
> 스케줄링 → 노드 상태 → 네트워크 → 서버 권한이 서로 독립적으로 실패하며,
> 하나를 풀어야 다음 증상이 드러난다.

---

## 빠른 진단 인덱스

Pod가 볼륨 때문에 안 뜰 때, `kubectl describe pod`의 Events 문구가 다음 조사 지점을 가리킨다.
증상을 찾아 해당 계층으로 점프한다.

| 증상 (Events / 에러 문구) | 원인 계층 | 확인 방법 | 조치 |
|--------------------------|----------|----------|------|
| `volume node affinity conflict` | ① 스케줄링 | `kubectl get pv <pv> -o jsonpath='{.spec.nodeAffinity}'` | NFS PV의 nodeAffinity 제거 (PV 재생성) |
| `ContainerCreating`에서 정지 + `Failed to garbage collect ... 0 bytes eligible` | ② 노드 상태 | 노드에서 `df -h /` | 미사용 데이터·이미지 정리 or 디스크 증설 |
| `mount.nfs: Connection timed out` | ③ 네트워크 | `nc -zv -w 3 <nfs> 2049` / `111` | 노드↔NFS 방화벽 개방 (TCP 2049 우선) |
| `mount.nfs: access denied by server` | ④ 서버 권한 | NFS 서버 `exportfs -v` | exports에 노드 IP/대역 추가 → `exportfs -ra` |
| `showmount`만 timeout (nc는 됨) | v3 UDP 경로 | — | 무시하고 NFSv4로 마운트(2049 단일) |

**핵심 구분**: `timeout` = 포트/네트워크 차단, `access denied` = 서버 exports 권한.
증상이 timeout → access denied로 바뀌면 네트워크는 뚫린 것 — 후퇴가 아니라 전진이다.

### 진단 플로우

```text
Pod Pending / 마운트 실패
  └─ kubectl describe pod → Events 확인 (제일 먼저)
       ├─ affinity conflict     → ① PV nodeAffinity 확인
       ├─ ContainerCreating 지속 → ② 노드 df -h (디스크)
       ├─ mount timed out        → ③ nc로 포트 확인 (방화벽)
       └─ access denied          → ④ 서버 exports 확인 (권한)
```

> 포트 점검은 항상 `-w 3` 타임아웃을 건다 (안 그러면 showmount처럼 무한 대기에 갇힘. 탈출은 `Ctrl+\`).

### NFS 버전 — 처음부터 v4 기준으로

| 방법 | 장점 | 단점 | 선택 |
|------|------|------|------|
| NFSv3 (기본 협상) | 서버 호환성 넓음 | portmapper(111 TCP+UDP) + mountd 랜덤 포트까지 필요 → 방화벽 범위 큼 | ✗ |
| NFSv4.1 명시 | **TCP 2049 단일 포트**, rpcbind/mountd/UDP 불필요 | 서버의 v4 지원 필요 | ✅ |

기본 마운트는 v3부터 협상하며 지연될 수 있으므로, PV `mountOptions`에 `nfsvers=4.1`을 명시해
v3 fallback 경로를 배제한다. 방화벽 신청도 v4 기준(2049 단일)으로 하면 개방 범위와 재신청이 준다.

---

## Case 1 (2026-07-31) — 노드 고정 Task를 NFS로 전환할 때의 도달성·권한 장애

> 관련: [[cluster-dns-and-storage-hardening]] Case 1에서 Local Storage → RWX NFS로 전환했고,
> 그 Next Step("NFS 실적용 시 노드 도달성을 실배치로 검증")이 현실에서 드러난 기록.

### Background — 배경 (Why)

Local Storage 종속을 걷어내고 RWX NFS로 전환한 뒤, 특정 Task를 실제 NFS 볼륨(PVC)으로 옮기는 작업이었다.

조건이 하나 있었다. 이 Task는 **외부 통신(프록시) 제약 때문에 특정 Worker 노드에 `nodeSelector`로 고정**되어 있다.
다른 노드로 재스케줄링할 수 없고, 반드시 그 노드에서 NFS 서버에 접근할 수 있어야 한다.
NFS 서버는 이 노드와 다른 별개의 서버다.

"NFS는 네트워크 스토리지라 어느 노드에서든 붙는다"는 전제는 **모든 노드가 NFS 서버에 도달 가능할 때**의 이야기다.
노드 고정 제약이 있는 Task에서는 "그 노드"의 도달성과 권한이 개별적으로 보장되어야 한다.

### Problem — 문제

- Task Pod가 `Pending`에 머물다 종료되기를 반복한다.
- 표면 증상은 단순히 "NFS 볼륨 마운트 실패"로 보인다.

### Hypothesis — 가설

단일 원인이 아닐 수 있다고 보고, `describe pod`의 Events가 단계마다 가리키는 에러 문구를 따라
원인을 하나씩 벗겨내기로 했다. 에러 메시지가 그 자체로 다음 조사 지점을 지정해 주므로,
가설을 미리 다 세우기보다 **증상 → 다음 증상**으로 좁히는 방식을 택했다.

### Investigation & Root Cause — 조사와 원인 (양파 구조)

#### ① 스케줄링 — volume node affinity conflict

- 증상: `0/N nodes are available: 1 node had volume node affinity conflict` + 나머지 노드는 selector 불일치.
- 조사: NFS PV의 `nodeAffinity.required.nodeSelectorTerms`에 **빈 term(`[{}]`)** 이 박혀 있었다.
- 원인: 빈 nodeSelectorTerm은 스케줄러가 **"어떤 노드에도 매칭되지 않음"** 으로 평가한다.
  Local/hostPath PV 템플릿을 복사해 NFS로 바꾸는 과정에서 남은 잔재였다.
  **NFS PV에는 nodeAffinity가 애초에 없어야 한다** (네트워크 스토리지는 노드에 묶이지 않으므로).
- 해결: PV의 nodeAffinity는 immutable이라 수정 불가 → PVC → PV 순으로 삭제 후 nodeAffinity 없이 재생성.
  reclaimPolicy가 `Retain`이라 NFS 실데이터는 보존됨. 임시 Pod로 마운트·쓰기 검증.

#### ② 노드 상태 — 디스크 포화로 이미지 Pull 불가

- 증상: 스케줄링은 통과했으나 Pod가 `ContainerCreating`에서 넘어가지 못하고 종료.
  노드 이벤트에 `Failed to garbage collect required amount of images ... 0 bytes eligible to free`.
- 조사: 고정 노드의 Root Disk 사용률이 약 89%. 이미지 GC가 회수할 미사용 이미지가 없어 실패 →
  새 이미지(대형)를 Pull할 공간이 없어 컨테이너 생성이 멈춤.
- 원인: 과거 Local Storage 시절 **Task 간 통신용으로 쌓고 정리하지 않은 데이터** + 미사용 테스트 스택/이미지가 누적.
- 해결: 미사용 컨테이너 스택 제거, dangling 이미지 정리, 정리 안 된 옛 Local Storage 데이터 삭제.
  (containerd 런타임은 별도 소켓 지정으로 이미지 확인·정리. 스냅샷 디렉토리는 손으로 지우지 않고 런타임 명령으로만.)

#### ③ 네트워크 — 특정 노드에서 NFS 포트 차단

- 증상: `mount.nfs: Connection timed out`.
- 조사: 고정 노드에서 NFS 서버로 향하는 NFS 포트(TCP 2049, 111)를 타임아웃 걸어 점검하니 **둘 다 차단**.
- 함정: 앞서 ①에서 임시 검증 Pod는 마운트에 성공했었다. 그러나 그 Pod는 `nodeSelector`가 없어
  **NFS에 도달 가능한 다른 노드에 스케줄**되었던 것. 정작 Task가 고정된 노드만 막혀 있었다.
  → "테스트가 됐으니 스토리지는 문제없다"는 판단이 노드 고정 Task에서는 함정이 된다.
- 원인: 고정 노드 ↔ NFS 서버 구간 방화벽 미개방.
- 해결: 해당 구간 방화벽 개방 신청·반영.

#### ④ 서버 권한 — exports 허용 목록 누락

- 증상: 방화벽 개방 후 재시도하니 이번엔 `mount.nfs: access denied by server`.
- 조사: 포트 점검에서 TCP 2049·111은 연결되나 `showmount`는 여전히 timeout.
  rpcbind/showmount가 UDP도 사용하기 때문(NFSv3 경로). 다만 **NFSv4는 portmapper·mountd·UDP를
  쓰지 않고 TCP 2049 단일 포트만** 쓰므로, v4로 마운트를 시도해 서버까지 도달시킴 → `access denied` 확인.
- 원인: NFS 서버 exports 허용 목록에 이 노드가 등록되지 않음.
- 해결: exports에 노드 IP(대역 단위) 추가 후 재적용. 이후 마운트 성공.

### Solution — 요약

증상이 가리키는 순서대로 네 개의 독립 원인을 차례로 제거했다.

1. NFS PV에서 잘못된 nodeAffinity 제거 (PV 재생성, Retain으로 데이터 보존)
2. 고정 노드 디스크 확보 (미사용 데이터·이미지 정리)
3. 노드 ↔ NFS 서버 방화벽 개방 (TCP 2049, 111)
4. NFS 서버 exports에 노드 대역 허용 추가

부가로 PV `mountOptions`에 `nfsvers=4.1`을 명시해 v3 fallback 지연을 배제하는 것을 권장.

### Result — 결과 (Before / After)

| 항목 | Before | After |
|------|--------|-------|
| Task Pod | Pending→종료 반복 (마운트 실패) | 정상 스케줄·마운트·기동 |
| PV nodeAffinity | 빈 term으로 전 노드 conflict | 제거 (NFS에 불필요) |
| 고정 노드 디스크 | 사용률 약 89%, 이미지 Pull 불가 | 미사용 데이터 정리로 여유 확보 |
| 노드↔NFS 네트워크 | 2049·111 차단 | 방화벽 개방 |
| NFS 권한 | exports 미등록 (access denied) | 노드 대역 허용 |
| NFS 버전 | v3 기본 협상 (다중 포트) | v4.1 명시 권장 (2049 단일) |

### Lessons Learned

- **"마운트 실패"는 하나의 증상이지 하나의 원인이 아니다.** 스케줄링·노드·네트워크·권한은 독립적이며,
  하나를 풀어야 다음 증상이 드러난다. `describe`의 Events를 다음 조사 지점으로 삼아 순차적으로 벗겨낸다.
- **`timeout`과 `access denied`를 구분한다.** timeout은 네트워크 차단, access denied는 exports 권한.
- **NFS PV에는 nodeAffinity를 넣지 않는다.** Local/hostPath 템플릿 잔재가 빈 term으로 전 노드 conflict를 유발한다.
- **노드 고정(nodeSelector) Task는 스토리지 전환 시 "그 노드"의 도달성·권한을 개별 검증한다.**
  검증 Pod가 다른 노드에서 성공한 것에 속지 말 것.
- **방화벽은 처음부터 v4(2049 단일) 기준으로 신청한다.**
- **Local → 네트워크 스토리지 전환은 "정리 습관"까지 함께 옮겨야 한다.** 로컬에 쌓고 안 지운 임시 데이터가
  디스크 포화의 근본 원인이었다.

### Next Step

- exports·방화벽을 **단일 IP가 아닌 노드 대역 단위**로 사전 개방해, 신규 노드 편입 시 재작업을 제거한다.
- 노드 고정이 필요한 Task는 고정 노드의 NFS 도달성·디스크 여유를 **관측 대상**으로 승격한다.

### References

- Kubernetes: PersistentVolume `nodeAffinity`, `mountOptions`, reclaimPolicy
- NFS: v4 vs v3 포트 구조 (2049 / rpcbind 111 / mountd), `exports`, `exportfs`
- containerd: 이미지 GC, 런타임 소켓 기반 이미지 정리
- `mount.nfs` 에러 구분: `Connection timed out` vs `access denied by server`
