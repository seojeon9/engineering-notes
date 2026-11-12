# Engineering Notes Writing Guide

> **기술을 기록하지 말고, 문제 해결을 기록한다.**

---

# Philosophy

Engineering Notes는 기술을 공부한 내용을 정리하는 공간이 아니다.

실제 개발 및 운영 과정에서 마주한 문제를 분석하고,
해결 과정과 의사결정을 기록하여

- 같은 문제를 다시 만났을 때 빠르게 해결하고,
- 미래의 나와 다른 엔지니어가 참고할 수 있는 지식을 만들며,
- 나의 엔지니어링 사고 과정을 남기는 것을 목표로 한다.

이 Repository는 **Technology Encyclopedia(기술 백과사전)** 가 아니라
**Problem Dictionary(문제 사전)** 이다.

---

# Principles

## 1. 기술보다 문제를 기록한다.

❌ Kubernetes란?

❌ Airflow란?

❌ PostgreSQL Index 종류

✅ Airflow Scheduler Bottleneck

✅ Kubernetes DNS Failure

✅ PostgreSQL Query Plan Analysis

---

## 2. Why부터 시작한다.

기술보다

> 왜 이 문제를 해결하려고 했는가?

를 먼저 설명한다.

---

## 3. 사고 과정을 기록한다.

정답보다

생각의 흐름을 기록한다.

- 어떤 가설을 세웠는가
- 어떻게 검증했는가
- 왜 틀렸는가
- 왜 최종 해결책을 선택했는가

---

## 4. Trade-off를 기록한다.

모든 기술 선택에는 장단점이 있다.

왜 이 방법을 선택했는지 설명할 수 있어야 한다.

---

## 5. Lessons Learned를 남긴다.

문서는 해결로 끝나지 않는다.

다음에는

- 어떻게 더 잘할 것인지
- 같은 문제를 어떻게 예방할 것인지

를 기록한다.

---

## 6. 회사 자산은 기록하지 않는다.

다음 내용은 절대 기록하지 않는다.

- 회사명
- 프로젝트명
- 고객명
- 실제 테이블명
- 실제 SQL
- Source Code
- 내부 화면
- API 주소
- IP
- 로그
- 인증 정보

항상 일반화해서 작성한다.

❌

PRD.ORDER_DETAIL

✅

주문 데이터 테이블

---

# Repository Structure

폴더는 **기술(도메인)** 기준으로 나눈다.

```text
engineering-notes/

data-platform/
    airflow/
    postgres/
    snowflake/
    spark/
    etl/

infrastructure/
    kubernetes/
    docker/
    linux/

ai-platform/
    mcp/
    langgraph/
    rag/

architecture/
```

폴더는

> 어떤 기술 영역인가?

를 표현한다.

---

# File Naming

파일명은

> 어떤 문제를 해결하는 문서인가?

를 표현한다.

## 좋은 예

```text
scheduler-bottleneck.md

dns-troubleshooting.md

query-plan-analysis.md

persistent-volume.md

index-strategy.md

mcp-long-running-tool.md
```

## 좋지 않은 예

```text
airflow.md

postgres.md

study.md

memo.md

test.md

2026-07-21.md
```

파일명만 보고도 내용을 예측할 수 있어야 한다.

---

# One Topic, One Document

같은 주제는 하나의 문서에서 계속 관리한다.

❌

```text
scheduler.md

scheduler-v2.md

scheduler-final.md
```

✅

```text
scheduler-bottleneck.md
```

새로운 경험은 문서를 업데이트한다.

## Case에는 발생일을 함께 적는다.

하나의 문서에 여러 사례가 쌓이므로,
각 Case 제목 옆에 **문제(장애)가 발생한 날짜**를 함께 적는다.

- 날짜는 **문서를 작성한 날이 아니라 문제가 발생한 날**이다.
- 해결한 날(적용일)도 아니다. 고친 날을 쓰면 Case마다 날짜의 의미가 달라진다.
- 오래전부터 있던 **만성 문제**는 시작일이 없다. **문제로 처음 기록한 날**을 쓰고, 제목 바로 아래에 만성임을 한 줄 남긴다.

```text
## Case 1 (2026-07-27) — 일별 지표 배치: 애플리케이션 왕복과 다중행 INSERT
> 만성 — 그 전부터 느렸고, 상류 누락 재적재 때 문제로 기록
```
- 형식은 `Case N (YYYY-MM-DD) — 한 줄 요약` 으로 통일한다.
- 문서 작성일/수정일은 Front Matter(`created` / `updated`)에 따로 기록한다.

```text
## Case 1 (2026-07-10) — Cross-Site DNS 장애

...

## Case 2 (2026-11-03) — Scheduler Bottleneck

...

## Case 3 (2027-02-15) — NFS 도달성 장애

...
```

Git History와 함께 문서도 성장한다.

## Case는 각자 완결된 구성을 갖는다.

Case 2를 Case 1의 본문 중간(예: Case 1의 Result와 Lessons 사이)에 끼워 넣지 않는다.
그러면 위에서부터 읽을 때 Case 2의 Lessons가 Case 1의 Lessons보다 먼저 나오고, 어느 결론이 어느 사례 것인지 섞인다.

**각 Case는 그 자체로 Document Template 전체(Background → … → References)를 갖는다.**

```text
# 문서 제목

> 범위: 이 문서가 다루는 판단 (엔진·도구와 무관한 기준)

## TL;DR                    ← Case별 요약을 나란히
## 빠른 진단 인덱스          ← 반복되는 문제라면 (아래 "증상 인덱스" 참고)

---

## Case 1 (YYYY-MM-DD) — 한 줄 요약
### Background
### Problem
### Hypothesis / Investigation
#### (세부 단계는 한 단계 아래 제목)
### Root Cause
### Solution
### Trade-offs
### Result
### Lessons Learned          ← 1번부터
### References

---

## Case 2 (YYYY-MM-DD) — 한 줄 요약
### Background
...
### Lessons Learned          ← 다시 1번부터
### References
```

규칙:

- **문서 레벨**에는 Front Matter, 제목, 범위, TL;DR, 빠른 진단 인덱스만 둔다. 서사는 전부 Case 안에 둔다.
- Case 제목은 `##`, Case 안의 템플릿 섹션은 `###`, 그 아래 세부는 `####`.
- 구획선(`---`)은 **Case 사이에만** 둔다. Case 안에서는 쓰지 않는다 — 구획선이 곧 "다른 사례"라는 신호가 되게.
- Lessons 번호는 **Case마다 1번부터** 다시 센다. 다른 Case를 가리킬 때는 `Case 1 Lesson 10` 처럼 Case를 붙인다.
- 새 Case는 **문서 맨 끝**에 추가한다 (발생일 순).
- TL;DR은 Case별 소제목으로 나눠 요약한다. 문서를 열자마자 "몇 번째 사례에서 무엇이 달랐는지"가 보여야 한다.
- References도 Case 안에 둔다. 사례마다 참조한 기술이 다르다(예: Case 1은 PostgreSQL, Case 2는 Snowflake).

### 단일 서사 문서를 Case 구조로 바꿀 때

처음엔 사례가 하나라 Case 없이 쓴 문서에 두 번째 사례가 생기면:

1. 기존 본문 전체를 `## Case 1 (발생일) — 한 줄 요약` 아래로 넣고, 제목을 한 단계씩 내린다 (`##` → `###`, `###` → `####`).
2. 본문 문장은 고치지 않는다. 위치와 제목 레벨만 바꾼다.
3. Case 1의 발생일은 문서 작성일(`created`)과 다를 수 있다 — 작성일은 대개 해결 후다.
   일일 노트·이슈·커밋 기록에서 **문제로 처음 기록한 날**을 찾아 쓴다. 기록이 전혀 없을 때만 `created`를 쓴다.
4. 문서 레벨 TL;DR·범위를 두 사례를 아우르게 다시 쓰고, 증상이 반복되면 진단 인덱스를 추가한다.

이 구조가 적용된 예: `data-platform/etl/bulk-load-performance.md`

---

# Date

날짜는 파일명에 포함하지 않는다.

Git이 이미 변경 이력을 관리한다.

❌

```text
2026-07-airflow.md
```

✅

```text
scheduler-bottleneck.md
```

날짜는 Front Matter에 기록한다.

```yaml
---
created: 2026-07-21
updated: 2026-08-03

tags:
  - airflow
  - scheduler
  - performance

difficulty: Intermediate
---
```

---

# Document Template

```text
Background

Problem

Hypothesis

Investigation

Root Cause

Solution

Trade-offs

Result

Lessons Learned

References
```

사례가 하나뿐인 문서는 이 템플릿을 그대로 쓴다.
사례가 둘 이상이면 이 템플릿 전체가 **각 Case 안에** 들어간다 ("Case는 각자 완결된 구성을 갖는다" 참고).

---

# 반복되는 문제는 "증상 인덱스"를 앞에 둔다

Troubleshooting·운영 문서는 서사(Background → … → Lessons)만으로는
> "그때 어떻게 고쳤더라?"

를 빠르게 되찾기 어렵다.

같은 문제를 다시 만난 미래의 나는 문서를 처음부터 읽고 싶지 않다.

**증상으로 바로 조치를 찾을 수 있어야 한다.**

그래서 증상이 반복되는 문서는 **맨 앞에 빠른 진단 인덱스**를 둔다.

- 증상(에러 문구) → 원인 계층 → 확인 방법 → 조치 를 한 줄로 매핑한 표
- 필요하면 증상 분기를 보여주는 진단 플로우(트리)

서사(Case)는 그 아래에 둔다.

인덱스는 **빠른 재사용**, Case는 **왜 그렇게 판단했는가(사고 과정)** 를 담당한다.

둘은 역할이 다르다.

✅ 문서 상단 — 증상 → 조치 표 + 진단 플로우

✅ 문서 하단 — Case N (발생일) — 상세 서사

> 서사는 이해를 위해, 인덱스는 재사용을 위해.

이 패턴이 적용된 예: `persistent-volume-mount-troubleshooting.md`, `bulk-load-performance.md`

인덱스 표에는 **어느 Case에서 나온 행인지** 열을 하나 둔다. 표에서 조치를 찾고, 이유가 궁금하면 그 Case로 내려간다.

---

# Good Document

좋은 문서는

"기술이 무엇인가"

를 설명하지 않는다.

대신

"같은 문제가 발생했을 때 해결할 수 있는가"

를 목표로 한다.

---

# Do Not Write

- 기술 소개
- 설치 방법
- 공식 문서 번역
- ChatGPT 답변 복사
- 단순 문법 정리
- 코딩 테스트 풀이
- 일회성 메모

---

# Write More

- Postmortem
- Troubleshooting
- Performance Tuning
- Architecture Decision Record (ADR)
- Benchmark
- 운영 경험
- 설계 의사결정
- Best Practice
- 운영 자동화

---

# Repository Goal

이 Repository를 읽고

> "무엇을 공부했는가"

보다

> "어떤 문제를 해결할 수 있는 엔지니어인가"

를 보여줄 수 있어야 한다.

---

Engineering Notes Writing Guide
Version 1.2
Last Updated: 2026-11-12