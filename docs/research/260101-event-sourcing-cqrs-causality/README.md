# Event Sourcing & CQRS: Causal System 처리 연구

**조사일**: 2026-01-01
**프로젝트**: YOMAN (Your Operation MANager)

---

## 개요

Event Sourcing과 CQRS 패턴이 **causal system 문제를 해결하는 메커니즘**을 조사.

### 조사 배경

YOMAN의 `idea3.txt`에서 제기된 문제:
- Non-causal LLM을 causal system(프로젝트 관리)에 적용 시 어려움
- Event Sourcing/CQRS가 해결책이 될 수 있는지 검증 필요

---

## 문서 구조

1. **[00-executive-summary.md](./00-executive-summary.md)**
   - 핵심 발견 요약
   - YOMAN 적용 전략
   - 구현 방향 제시

2. **[01-causality-analysis.md](./01-causality-analysis.md)**
   - 상세 분석
   - 권위자 자료 인용 (Martin Fowler, Greg Young)
   - 구체적 구현 예시

---

## 핵심 발견

### 1. Causality 자동 보장

```
Append-only sequential log
→ Event order = Causal order (by definition)
```

**별도 메커니즘 불필요**. Event stream 자체가 causality 보존.

### 2. 3가지 핵심 메커니즘

| 메커니즘 | 역할 | 효과 |
|---------|------|------|
| **Immutability** | Never update/delete | 과거 불변 |
| **Sequential Write** | Append-only log | Causal order 보장 |
| **Event Replay** | Left fold | Deterministic 재구성 |

### 3. Temporal Queries

```python
# Time travel 가능
state_at(timestamp)  # "3개월 전 상태?"
trace_causality(id)  # "왜 이렇게 됐지?"
```

**모든 과거 상태 재구성 가능**.

---

## YOMAN 적용

### Bucket System

```
IDEA → IdeaCreated, IdeaRefined, IdeaRejected
RESEARCH → ResearchStarted, FindingAdded
TODO → TodoCreated, TodoCompleted
PR → PRCreated, PRReviewed, PRMerged
```

각 bucket item = **독립적 event stream**.

### GAN Verification

```python
# CODINGBOT events
[CodeGenStarted, FileModified, PRCreated]

# REVIEWER projection (deterministic)
review(events) → match_rate + audit_trail
```

**완전한 audit trail** → "왜 match_rate 65%?" 추적 가능.

### 멀티모달 인터페이스

```
Telegram/Slack → Command (sequential)
  ↓
Event Store
  ↓
Projections (parallel, eventually consistent):
  ├─ Mobile App
  ├─ Web Dashboard
  └─ Notion Sync
```

---

## 주요 참고 자료

1. **Martin Fowler - Event Sourcing**
   - URL: `martinfowler.com/eaaDev/EventSourcing.html`

2. **Martin Fowler - CQRS**
   - URL: `martinfowler.com/bliki/CQRS.html`

3. **Greg Young - CQRS and Event Sourcing (2014)**
   - YouTube: `JHGkaShoyNs` (1h 4m)
   - Code on the Beach 2014

4. **Greg Young - Event Sourcing (GOTO 2014)**
   - YouTube: `8JKjvY4etTY` (54m)

---

## 다음 단계

1. **Prototype 구현**
   - Simple event store
   - Bucket event mapping
   - Basic projection

2. **YOMAN 통합**
   - Event store as core
   - Bucket → event stream
   - UnitService → projection tier

3. **Temporal Query API**
   - `state_at(timestamp)`
   - `trace_causality(item_id)`
   - `replay_with_new_logic(projection)`

---

## 결론

Event Sourcing/CQRS는 **causality 문제의 완전한 해답**:
- Append-only → causal order 자동 보장
- Event replay → deterministic 재구성
- Temporal queries → time travel 가능

YOMAN의 **"컴퓨터 없이 개발하기"** 비전과 완벽 호환.

<!-- Updated: 2026-01-01 -->
