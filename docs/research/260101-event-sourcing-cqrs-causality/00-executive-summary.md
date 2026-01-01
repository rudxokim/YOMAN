# Event Sourcing & CQRS: Causal System 처리 메커니즘 조사

**조사일**: 2026-01-01
**프로젝트**: YOMAN (Your Operation MANager)
**목적**: Causal system 문제 해결을 위한 Event Sourcing/CQRS 패턴 연구

---

## 【핵심 발견】

### 1. Causality 문제 해결의 핵심

Event Sourcing과 CQRS는 **causality를 근본적으로 해결**함:

```
Write (Command):
- Sequential append-only log
- Immutable events
→ Causal order 자동 보장 (by definition)

Read (Query):
- Event replay (left fold)
- Multiple projections
→ Deterministic 재구성 + temporal queries
```

**핵심 원칙**:
> "Current state is a left fold of previous behaviors"
> — Functional programming perspective

---

### 2. 3가지 질문에 대한 답변

#### Q1: 순차적 write vs 병렬적 read 분리가 causality 문제를 어떻게 해결하는가?

**A1**:
- **Write**: Append-only → event order = causal order (자동 보장)
- **Read**: Event replay → deterministic (같은 events = 같은 state)
- **분리 효과**: Write는 strong consistency, Read는 eventual consistency
  - 대부분 시스템에서 read는 이미 eventually consistent (Greg Young)
  - 예: DB 조회 후 HTTP 응답 전 데이터 변경 → 이미 stale read

**실무적 통찰**:
```python
# 99% of queries already operate with eventual consistency
# You just don't realize it
def get_user(id):
    user = db.query(id)  # T0
    # ... processing ...
    return json(user)    # T1 (data might have changed)
```

---

#### Q2: Event ordering과 causal consistency 보장 메커니즘

**A2**: 3계층 보장

**계층 1: Event Immutability**
```
- Never update event
- Never delete event
- Only append new events
→ Past cannot be altered → causality preserved
```

**계층 2: Per-Aggregate Stream**
```
- One event stream per aggregate/document
- Sequential processing within stream
- DDD Aggregate = Consistency boundary
→ Aggregate 내부 causal order 보장
```

**계층 3: Correction Events**
```
Accountant pattern:
- Full reversal: 10K 지급 → 10K 회수 → 1K 지급
- Partial reversal: 10K 지급 → 9K 회수
→ Audit trail 명확 (auditor가 intent 파악 가능)
```

**구현 예시** (Greg Young):
```python
def load_aggregate(id):
    events = db.read_events(stream_id)
    entity = Entity()
    for event in events:  # Sequential!
        entity.when(event)
    return entity
```

---

#### Q3: Temporal queries - 과거 상태 재구성 패턴

**A3**: Time Travel through Event Replay

**패턴 1: Point-in-Time Query**
```python
def state_at(stream_id, timestamp):
    events = read_events(stream_id)
    events_until_t = [e for e in events if e.ts <= timestamp]
    return fold(events_until_t)  # Deterministic replay
```

**패턴 2: Historical Analysis**
```python
def apply_new_report_to_history(report_logic):
    # Replay from Event #0
    for event in all_events_ever:
        report_logic(event)

    # Result: "What would this report show in 2010?"
```

**실제 사례** (Greg Young):
- Business: "5분 내 장바구니 제거 아이템" 분석
- Structural model: 빈 결과 (미래만 수집)
- Event sourcing: 전체 history replay → 모든 과거 데이터 포함

**패턴 3: Snapshot + Incremental**
```python
def load_with_snapshot(stream_id, version):
    snapshot = find_latest_snapshot_before(version)
    events_after = read_events_after(snapshot.version)
    return fold(events_after, snapshot.state)
```

---

## 【핵심 메커니즘 비교】

| 요소 | Structural Model | Event Sourcing |
|------|------------------|----------------|
| **Causality** | 손실 (update/delete) | 보존 (append-only) |
| **Temporal Query** | 불가능 (snapshot only) | Time travel 가능 |
| **Audit Trail** | 불완전 (증명 불가) | 완전 (book of record) |
| **Versioning** | 복잡 (migration script) | 단순 (multiple projections) |
| **Consistency** | Strong (everywhere) | Strong write + Eventual read |

---

## 【YOMAN 적용 전략】

### 1. Bucket System → Event Stream 매핑

```
Bucket     | Events                          | Causality 보존
-----------|--------------------------------|------------------
IDEA       | IdeaCreated, Refined, Rejected | Refinement history
RESEARCH   | Started, FindingAdded, Done    | Research process
TODO       | Created, Assigned, Completed   | Prerequisites
PR         | Created, Reviewed, Merged      | Full review trail
```

**장점**:
- 각 bucket item = event stream
- Temporal query: "이 IDEA는 3개월 전 어떤 상태?"
- Audit: "왜 이 TODO가 생겼지?" → full causal chain

---

### 2. UnitService Hierarchy와 Projection

```
Tier 3 (Opus - System):
- Aggregation of all project events
- Projection: System-wide dashboard
- Temporal: "3 months ago system state"

Tier 2 (Sonnet - Module):
- Per-operation event stream
- Projection: Operation progress
- Snapshot: Per operation milestone

Tier 1 (Flash/GLM - Unit):
- Per-function event stream
- Multiple projections: Code, Tests, Docs
- Fast append-only writes
```

---

### 3. GAN Verification Deterministic 보장

**현재 설계 문제**:
```python
CODINGBOT(TODO) → PR
REVIEWER compares PR vs TODO

문제: How do we audit "why match_rate = 65%?"
```

**Event Sourcing 해결**:
```python
# CODINGBOT events (audit trail)
[
    CodeGenStarted(todo_id, spec),
    FileModified("user.py", diff),
    TestGenerated("test_user.py"),
    PRCreated(pr_id, files)
]

# REVIEWER projection (deterministic)
def review(events):
    spec = events[0].spec
    changes = extract_changes(events)
    match_rate = compare(spec, changes)

    # Full audit trail
    return ReviewReport(
        match_rate=match_rate,
        causal_chain=events  # Why this result?
    )
```

**장점**:
- **Deterministic**: 같은 events → 같은 review
- **Auditable**: "왜 이 PR은 65%?" → events 추적
- **Temporal**: "V2 spec 기준으로 재평가"

---

### 4. 멀티모달 인터페이스 (idea2.txt 비전)

**Event Sourcing 적합성**:
```
Telegram/Slack input → Command (sequential write)
  ↓
Event Store (append-only)
  ↓
Projections (parallel, eventually consistent):
  ├─ Mobile App View
  ├─ Web Dashboard
  ├─ CLI Status
  └─ Notion Sync
```

**Causality 보장**:
- 폰에서 명령 → event append (순차)
- 여러 UI → 각자 projection (병렬)
- **Consumer-driven subscription** (Greg Young):
  - Each UI manages own position in stream
  - No control channel needed
  - Replay 자유롭게 가능

---

## 【권장 구현 방향】

### Phase 1: Core Event Store

```python
class YomanEventStore:
    """Append-only event log"""

    def append(self, stream_id: str, events: List[Event]):
        """Sequential write → causality guaranteed"""
        pass

    def read(self, stream_id: str, from_version: int = 0):
        """Consumer-driven read"""
        pass

    def read_until(self, stream_id: str, timestamp: datetime):
        """Temporal query support"""
        pass
```

### Phase 2: Bucket as Event Stream

```python
class BucketEventStream:
    def __init__(self, bucket_type: str, item_id: str):
        self.stream_id = f"{bucket_type}/{item_id}"

    def append_event(self, event: BucketEvent):
        YomanEventStore.append(self.stream_id, [event])

    def get_current_state(self) -> BucketState:
        events = YomanEventStore.read(self.stream_id)
        return project_state(events)  # Left fold

    def get_state_at(self, timestamp: datetime) -> BucketState:
        events = YomanEventStore.read_until(self.stream_id, timestamp)
        return project_state(events)  # Time travel
```

### Phase 3: Multiple Projections

```python
class NotionProjection:
    """Sync to Notion DB (eventually consistent)"""
    def consume(self, events: List[Event]):
        for event in events:
            self.sync_to_notion(event)

class TelegramProjection:
    """Build telegram-friendly view"""
    def consume(self, events: List[Event]):
        self.update_telegram_state(events)

class DashboardProjection:
    """Real-time dashboard (in-memory)"""
    def consume(self, events: List[Event]):
        self.update_dashboard_state(events)
```

---

## 【주요 참고 자료】

1. **Martin Fowler - Event Sourcing**
   - `martinfowler.com/eaaDev/EventSourcing.html`
   - 핵심: "All changes as sequence of events"

2. **Martin Fowler - CQRS**
   - `martinfowler.com/bliki/CQRS.html`
   - 핵심: Command/Query 분리 → eventual consistency

3. **Greg Young - CQRS and Event Sourcing (2014)**
   - YouTube: `JHGkaShoyNs` (1h 4m)
   - 핵심: Immutability, audit trail, temporal queries

4. **Greg Young - Event Sourcing (GOTO 2014)**
   - YouTube: `8JKjvY4etTY` (54m)
   - 핵심: Consumer-driven subscription, projection patterns

---

## 【핵심 교훈】

### 1. Causality는 공짜
> "Event order IS causal order"

Append-only sequential log → causality 자동 보장. 별도 메커니즘 불필요.

### 2. Immutability는 필수
> "You can NEVER update an event"

Immutable events → 과거 불변 → causality 영구 보존.

### 3. State는 transient
> "Current state is a first-level derivative"

State는 언제든 삭제/재생성 가능. 진실은 events.

### 4. Temporal queries는 기본
> "What would this report show on July 28, 1993?"

Event replay → deterministic time travel. Free feature.

### 5. Eventual consistency는 이미 현실
> "99% of queries already operate eventually consistent"

Read side eventual consistency는 실무적으로 문제없음. 대부분 시스템이 이미 그럼.

---

## 【다음 단계】

1. **Prototype 구현**
   - Simple event store (file-based)
   - Bucket event mapping
   - Single projection (CLI view)

2. **YOMAN Architecture 통합**
   - Event store as core
   - Bucket → event stream
   - UnitService → projection tier

3. **GAN Verification 개선**
   - CODINGBOT event trail
   - REVIEWER deterministic projection
   - Audit dashboard

4. **Temporal Query API**
   - `state_at(timestamp)`
   - `trace_causality(item_id)`
   - `replay_with_new_logic(projection)`

---

**결론**:

Event Sourcing과 CQRS는 **causality 문제의 완전한 해답**임. YOMAN의 "컴퓨터 없이 개발하기" 비전과 완벽히 호환:

- **Append-only log**: 폰에서도 command 가능 (network 끊겨도 local append)
- **Consumer-driven**: 각 UI가 자기 pace로 consume
- **Temporal queries**: "3개월 전 상태" 조회 가능
- **Full audit**: 모든 decision의 causal chain 추적

다음은 **구현**.

<!-- Updated: 2026-01-01 -->
