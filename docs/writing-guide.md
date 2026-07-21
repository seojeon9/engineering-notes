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
Version 1.0
Last Updated: 2026-07-21