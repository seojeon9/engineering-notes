---
created: 2026-07-21
updated: 2026-07-21

tags:
  - airflow
  - kubernetes
  - coredns
  - nfs
  - monitoring
  - postmortem

difficulty: Advanced
---

# Airflow 배치 플랫폼 Kubernetes 클러스터 안정화·확장

- 대상 시스템: RKE2 기반 Kubernetes 클러스터 상의 Airflow 배치 플랫폼
- 클러스터 구성: On-Prem 노드(Control Plane / DNS / 스토리지) + AWS Worker 혼합(하이브리드)

---

## Case 1 (2026-07-10) — Cross-Site DNS 장애 대응부터 Worker 수평 확장까지

> 인증서 만료로 촉발된 장애를 계기로 표면화된 Kubernetes DNS 구조 문제를 걷어내고,
> Local Storage 종속성을 제거해 확장 가능한 Airflow 배치 플랫폼으로 전환한 운영 개선 기록.

### Background — 배경 (Why)

앞선 장애는 Worker 노드의 인증서 만료가 트리거였지만, 그 과정에서 클러스터에
**잠재되어 있던 DNS 구조 문제**가 함께 드러났다. 당시에는 서비스 복구를 최우선으로 두고
`hostAlias` 추가, `dnsPolicy` 변경 등 **우회 조치**로 서비스를 정상화했다.

우회 조치는 "지금 당장 돌아가게" 만들 뿐, 원인을 남겨둔다. 같은 문제가 재발하고,
플랫폼을 확장하려 할 때마다 발목을 잡는다. 그래서 이 작업의 목적은 단순 복구가 아니라,
**임시 조치를 제거하고 재발을 막으면서, 확장 가능한 플랫폼 기반을 만드는 것**이었다.

기존 환경의 구조적 한계는 두 가지였다.

- **스토리지 종속성**: Local Storage 기반이라 Task Pod가 특정 노드에 고정된다.
  Worker를 늘려도 작업이 분산되지 않고, 동시 실행 시 한 노드에 부하가 몰려 대용량 작업이 실패했다.
- **DNS 불안정성**: On-Prem과 AWS Worker 간 직접 통신이 불가한 상태에서
  Cross-Site DNS 조회가 발생해 간헐적 Timeout이 나타났다.

### Problem — 문제

- Airflow Task가 간헐적으로 DNS Timeout으로 실패한다. "DNS가 살아났다 죽었다" 하는 것처럼 보인다.
- Worker를 추가해도 대용량/동시 실행 작업이 특정 노드에 몰려 실패한다.
- 인증서 만료가 사전 감지 없이 장애로 이어졌다 (관측·알림 부재).

### Hypothesis — 가설 (DNS 간헐적 Timeout)

간헐적 실패는 "완전히 죽은 것"보다 원인 파악이 어렵다. 가능한 원인을 나눠서 세웠다.

- **가설 1.** CoreDNS 자체 오류/과부하 → 단순 리소스 문제
- **가설 2.** 네트워크 분리 구조와 Service의 endpoint 선택 방식이 결합된 문제
- **가설 3.** Pod DNS 설정(hostAlias, dnsPolicy) 자체의 문제

### Investigation — 조사 과정

네트워크 도달성과 Service endpoint 구성을 함께 놓고 확인했다.

- CoreDNS는 **On-Prem DNS 노드**와 **AWS 노드**에 각각 1개씩 존재하고,
  **Control Plane 노드에는 없었다** (replica 2).
- 네트워크 구조상 `Control Plane ↔ On-Prem DNS`, `Control Plane ↔ AWS`는 통신 가능하나,
  **`On-Prem DNS ↔ AWS`는 직접 통신 불가**였다.
- Kubernetes Service는 endpoint 간 **실제 통신 가능 여부를 판단하지 않고**,
  `Ready` 상태의 endpoint로 요청을 분배한다. 즉 Service가 **장애 경로의 CoreDNS를 랜덤 선택**하면
  그 요청만 Timeout이 났다. 정상 endpoint와 장애 endpoint가 **같은 Service 뒤에 함께 등록**되어 있었던 것.
- → "살아났다 죽었다"처럼 보인 것은 요청이 어느 endpoint로 분배되느냐에 따른 확률적 현상이었다.

**중간 시도 — `internalTrafficPolicy: Local`**

로컬 노드의 endpoint로만 트래픽을 보내면 Cross-Site 조회가 사라질 것으로 보고 시도했다.

- 로컬 CoreDNS endpoint가 있는 On-Prem DNS·AWS 노드는 정상 동작.
- 그러나 **로컬 endpoint가 없는 Control Plane 노드**에서는 모니터링 메트릭 연동(DNS 조회)이 실패.
- replica가 2개인 구조에서는 **노드가 늘어날수록 로컬 endpoint 없는 노드가 계속 생긴다** →
  Local 정책은 확장성 측면에서 근본 해법이 아니라고 판단.

### Root Cause — 원인

간헐적 DNS Timeout의 근본 원인은 CoreDNS 오류가 아니라,
**서로 통신 불가한 노드에 흩어진 CoreDNS endpoint가 하나의 Service 뒤에 함께 묶여 있고,
Service가 도달 불가 경로의 endpoint를 확률적으로 선택**한 것이었다.
즉 "DNS 소프트웨어"의 문제가 아니라 **DNS 배치 토폴로지와 네트워크 분리 구조의 불일치** 문제.

### Solution — 해결

#### 1. CoreDNS Central 구조 전환

- `internalTrafficPolicy`를 `Local` → `Cluster`로 되돌림 (모든 노드에서 조회 가능해야 하므로).
- CoreDNS를 **모든 노드에서 도달 가능한 On-Prem 노드에 집중 배치(Central)**.
  - `dns-central=true` 노드 레이블 + HelmChartConfig patch로 스케줄링 고정.
- AWS Worker를 포함한 모든 노드가 **도달 가능한 단일 CoreDNS 집합**을 사용하도록 구성.
- 결과: Cross-Site DNS 요청 제거, DNS Timeout 해소, 응답 안정성 향상.

#### 2. NodeLocal DNSCache 구축

CoreDNS를 대체하는 것이 아니라 **각 노드 앞단의 캐시 계층**을 둔다.
캐시에 없는 요청만 중앙 CoreDNS로 넘어간다.

- 모든 노드에 NodeLocal DNSCache를 **DaemonSet**으로 배포 (RKE2 기본 옵션 활용).
- 조회 경로: `Pod → NodeLocal DNSCache → CoreDNS(Central) → Upstream`.
- 결과: 조회 속도 향상, CoreDNS 부하 감소, `hostAlias` 의존도 최소화.

#### 3. Local Storage → NFS(RWX) 공유 스토리지 전환

노드 종속의 근본 원인인 Local Storage를 걷어냈다.

- 스토리지 서버에 **RWX 지원 NFS** 구축.
- Airflow PVC를 RWX NFS 기반으로 변경, DAG·로그를 **모든 Worker가 공유**.
- 결과:
  - CPU 여유 서버를 Worker로 즉시 활용 가능 (Worker 추가 시 별도 스토리지 구성 불필요).
  - Node 장애 시 Worker 재스케줄링(이동) 가능.

#### 4. 부수 정비

- **스토리지 서버 환경 정비**: Root Disk 사용률 87% → 미사용 로그(구 서비스 로그 ~49GB, System/Journal ~15GB) 정리로 **약 64GB 확보**.
- **Pgpool Backend 정비**: 과거 제거된 PostgreSQL이 설정에만 잔존 → 실제 운영 DB만 참조하도록 변경. 반복 Health Check·오류 로그 제거.

#### 5. 관측·알림 체계 구축

장애를 "사후 복구"가 아니라 "사전 감지"로 전환하기 위한 계층.

- **모니터링**: 스토리지 서버에 Node Exporter(systemd 자동 실행) → Prometheus 연동(ServiceMonitor) → Grafana 대시보드. CPU/Memory/Disk/Filesystem/Network 실시간 관측.
- **알림(Grafana Alerting → 메신저 연동)**:
  - 디스크 사용률 임계 초과 (`/data`, Root)
  - CPU 85% 초과가 **10분 지속** 시 경고 (배치 스파이크 오탐 방지)
  - 메모리 90% 초과가 10분 지속 시 경고
  - **인증서 만료 임박 감지** (`apiserver_client_certificate_expiration_seconds`) — 이번 장애의 트리거였던 인증서 만료 재발 방지
  - 서버·Exporter 다운 감지 (`up == 0`)
  - `by (instance)` 기반 구성으로 **신규 노드 편입 시 자동 감시 대상 편입**.

#### 6. NFS 자동 보존 정책

- 데이터를 영구 / 준영구(로그) / 임시로 분류.
- Task 간 볼륨 공유용 임시 데이터는 **30일 보존** 지정 (증가 추이 보며 조정 예정).
- `systemd-tmpfiles` 규칙 파일로 30일 경과 파일 자동 삭제, 기존 `systemd-tmpfiles-clean.timer`(일 1회)로 무인 정리.
- CentOS 7 특성상 `--dry-run` 미지원 → `find` 기반 삭제 대상 **사전 검증 절차**를 별도로 정립.

#### 7. 신규 Worker 수평 확장

- 대상 클러스터가 Rancher 본체가 구동되는 `local`(RKE2) 클러스터임을 확인 → **UI 노드 등록 미지원**, `rke2-agent` 수동 조인 방식 채택.
- 사전 준비: hostname 규칙 정렬, chrony 시간 동기화, swap 비활성화, 방화벽 포트 개방(9345/6443/10250/tcp, 8472/udp).
- Control Plane의 node-token 확보 → **RKE2 버전 일치** 확인 후 Worker 전용 조인.
- 조인 후 NodeLocal DNSCache·CNI·Node Exporter 등 DaemonSet 자동 배포 확인, NFS 클라이언트 구비 및 도달성 확인.

### Trade-offs — 대안 비교

#### DNS 배치 전략

| 방법 | 장점 | 단점 | 선택 |
|------|------|------|------|
| `internalTrafficPolicy: Local` | Cross-Site 조회 원천 차단, 지연 최소 | 로컬 endpoint 없는 노드에서 조회 실패, replica < 노드 수일 때 확장성 없음 | ✗ |
| CoreDNS Central(도달 가능 노드 집중) + `Cluster` | 모든 노드에서 도달 가능, 노드 증가에 영향 없음 | 특정 노드군에 DNS 집중 → 해당 노드 장애 영향, 노드 로컬 지연 존재 | ✅ (NodeLocal Cache로 지연 보완) |
| CoreDNS 전 노드 분산(replica 확대) | 지연 최소, 단일점 완화 | On-Prem↔AWS 통신 불가 경로가 여전히 endpoint로 등록되어 근본 원인 미해결 | ✗ |

> Central의 "노드 집중" 단점은 앞단 **NodeLocal DNSCache**가 대부분의 조회를 흡수해 완화된다.

#### 스토리지 전략

| 방법 | 장점 | 단점 | 선택 |
|------|------|------|------|
| Local Storage | 구성 단순, 로컬 I/O 빠름 | Pod가 노드에 고정 → Worker 확장 불가, 부하 집중 | ✗ |
| NFS (RWX) | 모든 Worker가 공유, 재스케줄링·수평 확장 가능 | 네트워크 스토리지 지연, NFS 서버 단일점, 도달성 의존 | ✅ |

### Result — 결과 (Before / After)

| 항목 | Before | After |
|------|--------|-------|
| DNS | Cross-Site 조회로 간헐적 Timeout | Central + NodeLocal Cache로 Timeout 해소 |
| 스토리지 | Local Storage, 노드 종속 | NFS(RWX) 공유, Worker 수평 확장 가능 |
| 스토리지 서버 디스크 | 사용률 87% | 약 64GB 확보 |
| Pgpool | 미사용 Backend로 반복 오류 로그 | 운영 DB만 참조, 오류 로그 제거 |
| 관측 | 부재 | Prometheus/Grafana 실시간 + 메신저 알림 |
| 인증서 만료 | 사전 감지 없이 장애 | 만료 임박 자동 알림 |
| Worker 확장 | 사실상 불가 | 신규 Worker 조인 완료, 연산 여력 확장 |

### Lessons Learned

- **"간헐적"은 대개 확률적 분배의 신호다.** 완전히 죽지 않고 살았다 죽었다 한다면,
  단일 컴포넌트 고장보다 **로드밸런싱/endpoint 선택 뒤에 도달 불가 경로가 섞인 상태**를 먼저 의심한다.
- **Kubernetes Service는 endpoint의 실제 도달성을 보장하지 않는다.** `Ready`는 "프로세스가 떴다"이지
  "이 노드에서 저 노드로 갈 수 있다"가 아니다. 하이브리드(On-Prem+Cloud) 클러스터에서는
  **네트워크 분리 구조를 endpoint 배치에 반드시 반영**해야 한다.
- **우회 조치(hostAlias, dnsPolicy 변경)는 부채다.** 복구 직후 원인 제거 계획을 함께 세워야 재발을 막는다.
- **확장성은 "지금 되는가"가 아니라 "노드가 늘어도 되는가"로 판단한다.**
  `internalTrafficPolicy: Local`은 현재는 동작했지만 노드 증가 시 깨졌다. 미래 상태로 검증해야 한다.
- **장애의 트리거(인증서 만료)는 반드시 관측 대상으로 승격한다.** 같은 장애를 두 번 겪지 않으려면
  트리거 지표에 알림을 건다. `by (instance)`처럼 **신규 대상이 자동 편입되는 구성**이 운영 부담을 줄인다.
- **다음에는**: NFS 실적용 시 다중 노드 분산 실행과 AWS Worker ↔ NFS 도달성을 실제 배치로 검증한다
  (아래 Next Step). NFS 서버 단일점에 대한 이중화/백업 전략도 검토 대상.

### Operating Architecture — 개선 후 운영 구조

```text
[DNS]
Pod → NodeLocal DNSCache → CoreDNS (On-Prem, Central) → Upstream DNS

[Storage]
Airflow Worker → RWX PVC → NFS Server

[Monitoring & Alerting]
Node Exporter → Prometheus ─┬─▶ Grafana (Dashboard)
                            └─▶ Grafana Alerting ─▶ Messenger
                                (디스크 / CPU / 메모리 / 인증서 / 헬스체크)
```

### References

- Kubernetes: Service `internalTrafficPolicy`, EndpointSlice
- Kubernetes: NodeLocal DNSCache
- RKE2: HelmChartConfig, rke2-agent 노드 조인
- CoreDNS 공식 문서
- `systemd-tmpfiles` / `systemd-tmpfiles-clean.timer`
- Prometheus Node Exporter, Grafana Alerting
