# Event Sourcing & CQRS의 Causal System 처리 메커니즘

**조사일**: 2026-01-01
**조사자**: Claude (YOMAN Project)
**목적**: Event Sourcing과 CQRS가 causality 문제를 해결하는 방식 분석

---

## 【핵심 질문】

1. **순차적 write (command) vs 병렬적 read (query) 분리**가 causality 문제를 어떻게 해결하는가?
2. **Event ordering과 causal consistency** 보장 메커니즘
3. **Temporal queries** - 과거 상태 재구성 패턴

---

## 【조사 결과】

### 1. Command-Query 분리와 Causality

#### Martin Fowler - Event Sourcing 핵심

**출처**: `martinfowler.com/eaaDev/EventSourcing.html`

```
핵심 개념:
- "All changes to application state as a sequence of events"
- Events는 순서대로 처리됨 (sequential processing)
- 각 event는 정확한 state change를 캡처
```

**Causality 해결 방식**:
- **Write Side**: 완전한 순차 처리 (append-only log)
  - 모든 상태 전환을 event로 저장
  - Event는 immutable → 인과관계 보존 보장
  - Sequential write → causality 자동 보장

- **Read Side**: 병렬 처리 가능 (eventual consistency)
  - Event replay로 재구성 → 항상 deterministic
  - Multiple timelines 지원 (VCS branching 유사)
  - "Determine application state at any point in time"

**핵심 통찰**:
> "You can reconstruct past states by replaying events in their original sequence"

즉, **causality는 event ordering 자체가 보존**함.

---

#### Martin Fowler - CQRS 패턴

**출처**: `martinfowler.com/bliki/CQRS.html`

**분리 메커니즘**:
```
Command Side (Write):
- Sequential processing
- Strong consistency 필요
- Domain logic 처리

Query Side (Read):
- Parallel processing 가능
- Eventual consistency 허용
- Screen-oriented projections
```

**Consistency 모델**:
- Command와 Query 사이: **Eventual Consistency**
- Communication: Shared DB 또는 Event mechanisms
- "There needs to be some communication mechanism between the two models"

**Causality 관점**:
- **Write는 fully consistent** → causal order 보장
- **Read는 eventually consistent** → 대부분 시스템에서 허용 가능
  - Greg Young: "99% of queries already operate eventually consistent"
  - 예: DB 읽은 후 HTTP 응답 전에 data 변경됨 → 이미 eventually consistent

**Trade-offs**:
- Fowler 경고: "Use sparingly, only in complex domains"
- 모든 시스템에 적용 불가 → bounded context 내에서만

---

### 2. Event Ordering과 Causal Consistency

#### Greg Young - Event Sourcing 원조 설명

**출처**: YouTube `JHGkaShoyNs` (CQRS and Event Sourcing - Code on the Beach 2014)

**Event Ordering 보장 메커니즘**:

```python
# Conceptual model
def load_aggregate(id):
    events = db.read_events(stream_id)
    entity = Entity()
    for event in events:  # Sequential processing
        entity.when(event)
    return entity
```

**핵심 포인트**:
1. **Event Stream Per Aggregate**
   - 전체 시스템이 아닌 aggregate마다 독립적 stream
   - 각 stream은 순차적 처리 보장
   - DDD Aggregate = Event Stream

2. **Immutability → Causal Consistency**
   - "You can NEVER update an event"
   - "You can NEVER delete an event"
   - Update 대신 **Correction Events** (reversal pattern)
     - Full reversal: 10K 지급 → 10K 회수 → 1K 지급
     - Partial reversal: 10K 지급 → 9K 회수

3. **Audit Trail = Book of Record**
   - Current state는 derived (first-level derivative)
   - Audit log가 진실의 원천 (source of truth)
   - Worm Drive (Write-Once-Read-Many) 사용 가능
     - Physical immutability 보장
     - Super user attack 방어

**Causal Consistency 예제**:
```
Event Stream:
1. Cart Created
2. Item Added (ProductA)
3. Item Added (ProductB)
4. Item Added (ProductC)
5. Item Removed (ProductB)
6. Shipping Info Added

Q: 이게 "Cart Created + 2 items + shipping"과 같은가?
A: Perspective에 따라 다름
   - Final state 관점: Yes
   - Causality 관점: No (removal behavior 정보 손실)
```

---

#### Temporal Queries - 과거 상태 재구성

**Greg Young 사례**:

```markdown
## Use Case: "5분 내 장바구니 제거 아이템" 분석

### Structural Model 접근 (FAIL)
- removed_items 컬럼 추가
- Report 실행 → 빈 결과 (미래 데이터만 수집)

### Event Sourcing 접근 (SUCCESS)
1. Projection 작성:
   ```python
   def projection(event_stream):
       removed_items = []
       for event in event_stream:
           if event.type == "ItemRemoved":
               removed_items.append({
                   "item": event.item_id,
                   "time": event.timestamp,
                   "found_future": False
               })
           elif event.type == "ShippingAdded":
               checkout_time = event.timestamp
               for item in removed_items:
                   if checkout_time - item["time"] < 5min:
                       # Mark as candidate
           elif event.type == "ItemAdded":
               # Check if re-purchased
               for item in removed_items:
                   if item["item"] == event.item_id:
                       item["found_future"] = True
   ```

2. Replay from Event #0 → 모든 과거 데이터 포함
3. Report: "What would this report show on July 28, 1993 at 4:14 PM?"
   - Time travel query 가능
   - Deterministic replay
```

**핵심 통찰**:
> "Not only can I see everything at that point, I can also tell you at any point in time what this report would have told you if we had this report at that point in time"

---

### 3. Practical Mechanisms

#### Snapshot Pattern

**문제**: 5M events 재생 → 느림

**해결**:
```python
# Rolling Snapshot
def load_with_snapshot(stream_id, version):
    # Read backwards to find snapshot
    snapshot = find_latest_snapshot_before(stream_id, version)
    events = read_events_after(stream_id, snapshot.version)

    state = snapshot.state
    for event in events:
        state = apply(state, event)
    return state
```

**주의사항**:
- Snapshot은 side-effect로 저장 (optimistic concurrency 문제 회피)
- Snapshot at version N은 영구히 valid
- Version pointer로 연결
- "Never edit a projection, only create new projection"

---

#### Projection = Left Fold

**Functional Programming 관점**:
```haskell
-- Current state is a left fold of previous behaviors
currentState :: [Event] -> State
currentState = foldl apply initialState

-- Snapshot is memoization
snapshot :: Int -> State -> Snapshot
snapshot version state = Snapshot { ver = version, st = state }
```

**Multiple Read Models**:
- Document DB (screen-aligned)
- OLAP Cube (reporting)
- Graph DB (relationships)
- Lucene (full-text search)

각각 **독립적 projection** → **서로 다른 perception**

---

### 4. Causal Consistency 보장 전략

#### Greg Young - GOTO 2014

**출처**: YouTube `8JKjvY4etTY`

**Aggregate Boundary = Consistency Boundary**:
```
1. Load aggregate (fully consistent)
2. Process command
3. Append new events (sequential)
4. Projections subscribe (consumer-driven)
```

**Consumer-Driven Subscription**:
```
Blog 모델 (Atom Feed):
- Producer: Event stream 제공
- Consumer: 자신의 position 관리
- Control channel 불필요
- Replay 자유롭게 가능

vs Service Bus (AMQP):
- Producer가 queue 관리
- Control channel 필요 (history 요청)
- Replay 복잡함
```

**핵심**:
> "The consumer remembers where they are in the stream, not the producer"

---

## 【핵심 메커니즘 요약】

### 1. Sequential Write → Causal Order 자동 보장

| 요소 | 메커니즘 | Causality 보장 |
|------|----------|----------------|
| **Write** | Append-only log | Event order = Causal order |
| **Immutability** | No update/delete | 과거 불변 → causality 보존 |
| **Per-Aggregate Stream** | Independent streams | Aggregate 내부 consistency |

### 2. Parallel Read → Eventual Consistency 허용

| 요소 | 메커니즘 | Causality 영향 |
|------|----------|----------------|
| **Projections** | Replay from Event #0 | Deterministic 재구성 |
| **Multiple Models** | Different perceptions | 각 모델 독립적 consistency |
| **Consumer-Driven** | Self-managed position | Replay 제어 가능 |

### 3. Temporal Queries → Time Travel

| 기능 | 구현 | 장점 |
|------|------|------|
| **Past State** | Replay to point T | "What was X at time T?" |
| **Historical Analysis** | New projection on old events | "What would report Y show in 2010?" |
| **Debugging** | Command replay + version | "Reproduce bug at event N" |

---

## 【YOMAN 적용 시사점】

### 1. Bucket System ↔ Event Stream 매핑

```
IDEA Bucket:
- Event: IdeaCreated, IdeaRefined, IdeaRejected
- Causality: Refinement history 보존

RESEARCH Bucket:
- Event: ResearchStarted, FindingAdded, ResearchCompleted
- Projection: Structured JSON output

TODO Bucket:
- Event: TodoCreated, TodoAssigned, TodoCompleted
- Causal dependency: Prerequisites tracking

PR Bucket:
- Event: PRCreated, ReviewRequested, PRMerged
- Audit trail: Full review history
```

### 2. UnitService Hierarchy와 Consistency

**Tier 3 (Opus - System)**:
- Fully consistent view 필요
- Event stream aggregation across projects
- Temporal queries: "프로젝트 상태 @ 3 months ago"

**Tier 2 (Sonnet - Module)**:
- Per-operation event stream
- Snapshot per operation
- Projection: Operation progress dashboard

**Tier 1 (Flash/GLM - Unit)**:
- Per-function event stream
- Fast append-only writes
- Multiple projections: Code, Tests, Docs

### 3. GAN Verification과 Causality

**현재 설계**:
```python
# Generator (CODINGBOT)
TODO → [Events] → PR

# Discriminator (REVIEWER)
Compare: PR vs TODO spec
```

**Event Sourcing 적용**:
```python
# CODINGBOT events
events = [
    CodeGenStarted(todo_id, spec),
    FileModified(path, changes),
    TestGenerated(test_file),
    PRCreated(pr_id, files)
]

# REVIEWER projection
def review_projection(events):
    # Reconstruct causality
    spec = events[0].spec
    changes = [e.changes for e in events if e.type == "FileModified"]

    # Deterministic comparison
    match_rate = compare(spec, changes)

    # Audit trail
    return ReviewReport(match_rate, events)
```

**장점**:
- **Full audit trail**: 왜 match_rate가 낮은지 추적 가능
- **Deterministic**: 같은 events → 같은 review 결과
- **Temporal**: "이 PR은 V3 TODO spec 기준으로 80%였음"

### 4. 멀티모달 인터페이스와 Eventual Consistency

**idea2.txt 비전**:
```
Telegram/Slack → Command
└─> YOMAN Backend (Event Sourcing)
    └─> Projections (Eventually Consistent)
        ├─> Mobile App View
        ├─> Web Dashboard
        └─> CLI Status
```

**Causality 보장**:
- 폰에서 명령 → Event append (순차)
- 여러 인터페이스 → 각자 projection consume
- 모든 인터페이스는 eventually consistent하지만 **causally consistent**
  (항상 event order 유지)

---

## 【권장 사항】

### 1. YOMAN Architecture에 통합

```python
# Core Event Store
class YomanEventStore:
    def append(self, stream_id, events):
        # Sequential append → causality 자동 보장
        pass

    def read(self, stream_id, from_version=0):
        # Consumer-driven subscription
        pass

# Bucket as Event Stream
class BucketStream:
    def __init__(self, bucket_type):
        self.stream_id = f"{bucket_type}/{item_id}"

    def append_event(self, event):
        YomanEventStore.append(self.stream_id, [event])

    def get_current_state(self):
        events = YomanEventStore.read(self.stream_id)
        return project(events)  # Left fold

# Projection for each interface
class TelegramProjection:
    def consume(self, events):
        # Build telegram-friendly view
        pass

class NotionProjection:
    def consume(self, events):
        # Sync to Notion DB
        pass
```

### 2. Temporal Query 활용

```python
# Use Case: "3개월 전 프로젝트 상태"
def project_state_at(project_id, timestamp):
    events = YomanEventStore.read(f"project/{project_id}")
    events_until = [e for e in events if e.timestamp <= timestamp]
    return project(events_until)

# Use Case: "이 TODO가 왜 생겼지?"
def trace_causality(todo_id):
    events = YomanEventStore.read(f"todo/{todo_id}")
    # Show full event chain
    return CausalityChain(events)
```

### 3. GAN Verification Deterministic 보장

```python
class ReviewVerifier:
    def verify(self, pr_events):
        # Replay events deterministically
        for event in pr_events:
            state = apply(state, event)

        # Compare against spec
        match_rate = compare(state, original_spec)

        # Store verification as event
        return VerificationCompleted(match_rate, state, pr_events)
```

---

## 【참고 자료】

1. **Martin Fowler - Event Sourcing**
   - URL: `martinfowler.com/eaaDev/EventSourcing.html`
   - 핵심: Event sequence → state reconstruction

2. **Martin Fowler - CQRS**
   - URL: `martinfowler.com/bliki/CQRS.html`
   - 핵심: Command/Query 분리 → eventual consistency

3. **Greg Young - CQRS and Event Sourcing (Code on the Beach 2014)**
   - YouTube: `JHGkaShoyNs` (1h 4m)
   - 핵심: Immutability, audit trail, temporal queries

4. **Greg Young - Event Sourcing (GOTO 2014)**
   - YouTube: `8JKjvY4etTY` (54m)
   - 핵심: Consumer-driven subscription, projection patterns

5. **EventStore (Kurrent.io)**
   - URL: `kurrent.io/blog/what-is-event-sourcing`
   - 핵심: Sequential loading, entity application

---

## 【결론】

Event Sourcing과 CQRS는 **causality 문제를 근본적으로 해결**함:

1. **Write Side**: Append-only sequential log → causal order 자동 보존
2. **Read Side**: Event replay → deterministic 재구성 (temporal queries 가능)
3. **Consistency**: Write는 strong, Read는 eventual → 실용적 트레이드오프

YOMAN에 적용 시:
- **Bucket System**: Event stream per bucket
- **UnitService Hierarchy**: Projection per tier
- **GAN Verification**: Deterministic audit trail
- **멀티모달**: Consumer-driven eventual consistency

**핵심 원칙**:
> "Current state is a left fold of previous behaviors. Snapshots are memoization. Events never change."

---

**다음 단계**:
1. Prototype: Simple event store implementation
2. Bucket → Event mapping 구체화
3. Temporal query API 설계
4. GAN verification deterministic test

<!-- Updated: 2026-01-01 -->
