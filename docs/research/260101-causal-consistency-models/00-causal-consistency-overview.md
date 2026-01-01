# Causal Consistency in Distributed Systems

> **연구 목적**: YOMAN의 버킷 시스템과 Operation 관리에 분산 시스템의 causal consistency 모델 적용 가능성 탐색

**Research Date**: 2026-01-01
**Related**: [idea3.txt](../../../idea3.txt) - "causal system vs non-causal system"

---

## 1. Executive Summary

### 1.1 핵심 질문 (idea3.txt)

```
지금 얘기하는걸로 보면 이것은 명백히 causal system임.
하지만 리뷰어의 개념이 들어가면 non-causal system이 될 수 있는건지??
non-causal system으로 어떻게 신뢰도를 높일 수 있을지?
```

### 1.2 연구 결과 요약

**Causal Consistency의 핵심**:
- 인과관계 있는 이벤트는 모든 노드에서 같은 순서로 보임
- 인과관계 없는 이벤트는 순서 보장 안 함 (병렬 처리 가능)
- Strong consistency보다 약하지만 Eventual consistency보다 강함

**YOMAN 적용**:
- YOMAN의 버킷 시스템(IDEA→RESEARCH→TODO→PR)은 **causal dependency chain**
- REVIEWER(Discriminator)는 causal chain을 끊지 않음 (여전히 TODO에 의존)
- GAN 검증 패턴은 **quasi-causal**: Generator는 causal, Discriminator는 검증만

**결론**:
> YOMAN은 **Causal+ Consistency** 모델로 설계 가능. REVIEWER가 있어도 causal system임.

---

## 2. Consistency Models 비교

### 2.1 계층 구조

```
Strong Consistency (가장 강함)
    ├── Linearizability (실시간 순서 보장)
    └── Sequential Consistency (프로그램 순서 보장)
         ↓
Causal Consistency (인과관계 순서 보장) ← YOMAN 여기
    ├── Causal+ (읽기 후 쓰기 추가 보장)
    └── PRAM (프로세서별 쓰기 순서)
         ↓
Eventual Consistency (최종 수렴만 보장)
```

### 2.2 Trade-off 분석

| 모델 | 순서 보장 | 가용성 | 지연시간 | Coordination |
|------|----------|--------|---------|--------------|
| **Linearizability** | 전체 순서 | ❌ 낮음 | ❌ 높음 | ✅ 필수 |
| **Sequential** | 프로그램 순서 | △ 중간 | △ 중간 | ✅ 필요 |
| **Causal** | 인과 순서 | ✅ 높음 | ✅ 낮음 | △ 부분적 |
| **Eventual** | 없음 | ✅ 최대 | ✅ 최소 | ❌ 불필요 |

**CAP Theorem 관점**:
- Strong Consistency: CP (일관성 + 분할 허용)
- Causal Consistency: **AP + 부분 C** (가용성 + 분할 허용 + 인과 일관성)
- Eventual Consistency: AP (가용성 + 분할 허용)

---

## 3. Causality Tracking 기법

### 3.1 Lamport Timestamps

**개념**: 논리적 시간으로 이벤트 순서 추적

**구현** (from Mixu's Distributed Systems Book):
```javascript
function LamportClock() {
  this.value = 1;
}

LamportClock.prototype.get = function() {
  return this.value;
}

LamportClock.prototype.increment = function() {
  this.value++;
}

LamportClock.prototype.merge = function(other) {
  this.value = Math.max(this.value, other.value) + 1;
}
```

**특징**:
- 단일 카운터 (간단함)
- 이벤트 A → B이면 timestamp(A) < timestamp(B)
- **역은 성립 안 함**: timestamp(A) < timestamp(B)여도 A → B 아닐 수 있음

**YOMAN 적용**:
```python
# 버킷 간 이동 시 타임스탬프 증가
IDEA_1: ts=1
  ↓ (next_bucket = RESEARCH)
RESEARCH_1: ts=2
  ↓ (next_bucket = TODO)
TODO_1: ts=3
```

### 3.2 Vector Clocks

**개념**: 각 노드별 논리 시계 벡터로 인과관계 정확히 추적

**구현**:
```javascript
function VectorClock(value) {
  // { node1: 1, node2: 3, node3: 2 }
  this.value = value || {};
}

VectorClock.prototype.increment = function(nodeId) {
  if(typeof this.value[nodeId] == 'undefined') {
    this.value[nodeId] = 1;
  } else {
    this.value[nodeId]++;
  }
};

VectorClock.prototype.merge = function(other) {
  var result = {};
  var keys = Object.keys(this.value).concat(Object.keys(other.value));
  keys.forEach(function(key) {
    result[key] = Math.max(this.value[key] || 0, other.value[key] || 0);
  });
  this.value = result;
};
```

**비교 연산**:
```python
def compare(vc1, vc2):
    # vc1 < vc2 (vc1이 vc2보다 이전)
    if all(vc1[k] <= vc2[k] for k in all_keys):
        return BEFORE

    # vc1 > vc2 (vc1이 vc2보다 이후)
    if all(vc1[k] >= vc2[k] for k in all_keys):
        return AFTER

    # 동시 발생 (concurrent)
    return CONCURRENT
```

**YOMAN 적용**:
```python
# 3개 AI 모델이 동시에 유닛 생성 (양자 검증)
GLM:    { GLM: 1, Gemini: 0, Grok: 0 }  # 유닛 A1
Gemini: { GLM: 0, Gemini: 1, Grok: 0 }  # 유닛 A2
Grok:   { GLM: 0, Gemini: 0, Grok: 1 }  # 유닛 A3

# 세 유닛은 CONCURRENT (병렬 생성 가능)
# Module tier가 통합할 때 merge:
Sonnet: { GLM: 1, Gemini: 1, Grok: 1 }  # 최적 유닛 선택
```

### 3.3 Dependency Tracking (COPS 방식)

**COPS (Clusters of Order-Preserving Servers)** - SOSP 2011:
- **Causal+ Consistency** 제안 (Causal + Read-Your-Writes)
- 각 operation에 **dependency set** 첨부
- 읽기 전에 의존성 먼저 적용

**Dependency Set 예시**:
```json
{
  "key": "user_profile_123",
  "value": { "name": "Alice", "age": 30 },
  "version": 5,
  "dependencies": [
    { "key": "user_email_123", "version": 3 },
    { "key": "user_settings_123", "version": 2 }
  ]
}
```

**YOMAN 적용**:
```json
{
  "bucket": "TODO",
  "id": "TODO-45",
  "content": "사용자 인증 기능 구현",
  "version": 1,
  "dependencies": [
    { "bucket": "RESEARCH", "id": "RESEARCH-12", "version": 2 },
    { "bucket": "IDEA", "id": "IDEA-8", "version": 1 }
  ],
  "prerequisites": [
    "DB 스키마 설계 완료",
    "API 엔드포인트 정의"
  ]
}
```

---

## 4. CALM Theorem

### 4.1 정의

**CALM**: **C**onsistency **A**s **L**ogical **M**onotonicity

**핵심 정리**:
> "논리적으로 단조로운(monotonic) 프로그램은 조정(coordination) 없이 최종 일관성을 달성할 수 있다."

**출처**: Joseph M. Hellerstein et al., CIDR 2011

### 4.2 Monotonic vs Non-Monotonic

**Monotonic Operations** (coordination-free 가능):
```python
# 추가만 하는 연산 (단조 증가)
set.add(item)           # 집합에 원소 추가
counter.increment()     # 카운터 증가
log.append(entry)       # 로그 추가

# 함수형 연산
map(f, list)           # 각 원소 변환
filter(pred, list)     # 조건 필터링
union(set1, set2)      # 합집합
```

**Non-Monotonic Operations** (coordination 필요):
```python
# 삭제/감소 연산
set.remove(item)        # 집합에서 제거
counter.decrement()     # 카운터 감소

# 부정/배제 연산
set_difference(A, B)    # A - B (B에 없는 것)
NOT condition           # 부정

# 집계 연산
count(*)                # 전체 개수 (중간에 바뀔 수 있음)
avg(column)             # 평균 (누적된 값 필요)
```

### 4.3 YOMAN 버킷 시스템 분석

**Monotonic 흐름** (coordination-free):
```
IDEA → RESEARCH → TODO (누적)
- 각 단계는 이전 정보에 추가만 함
- 되돌아가지 않으면 단조로움
- 병렬 처리 가능
```

**Non-Monotonic 요소**:
```python
# 버킷 재처리 (next_bucket = _self)
TODO v1 → 코멘트 → TODO v2  # 이전 버전 "부정"

# REVIEWER 거부
PR → REVIEWER → FAIL → TODO  # PR 결과 "취소"
```

**해결책**:
- **Versioning**: 삭제 대신 새 버전 생성 (append-only)
- **Tombstone**: 삭제 표시만 (실제 삭제 안 함)
- **Compensation**: 실패 시 보상 트랜잭션

---

## 5. YOMAN에 Causal Consistency 적용

### 5.1 현재 YOMAN의 Dependency Chain

```
┌─────────────────────────────────────────────────────────────┐
│  IDEA (러프한 아이디어)                                      │
│  - 의존성: 없음                                              │
│  - Causal relation: 독립적                                   │
└────────────────────┬────────────────────────────────────────┘
                     │ depends_on(IDEA_ID)
┌────────────────────▼────────────────────────────────────────┐
│  RESEARCH (구조화된 조사)                                     │
│  - 의존성: IDEA_ID                                           │
│  - Causal relation: IDEA → RESEARCH                          │
└────────────────────┬────────────────────────────────────────┘
                     │ depends_on(RESEARCH_ID)
┌────────────────────▼────────────────────────────────────────┐
│  TODO (실행 가능 작업)                                        │
│  - 의존성: RESEARCH_ID, Prerequisites                        │
│  - Causal relation: RESEARCH → TODO                          │
└────────────────────┬────────────────────────────────────────┘
                     │ depends_on(TODO_ID)
┌────────────────────▼────────────────────────────────────────┐
│  PR (코드 생성/검증)                                          │
│  - 의존성: TODO_ID, CODINGBOT_output                         │
│  - Causal relation: TODO → PR                                │
└────────────────────┬────────────────────────────────────────┘
                     │ validates(PR_ID)
┌────────────────────▼────────────────────────────────────────┐
│  REVIEWER (검증)                                             │
│  - 의존성: PR_ID, TODO_ID (Acceptance Criteria)              │
│  - Causal relation: PR → REVIEWER (non-causal 아님!)         │
└─────────────────────────────────────────────────────────────┘
```

**핵심 인사이트**:
> REVIEWER는 새로운 데이터를 생성하지 않고, PR과 TODO를 **비교**만 함.
> 따라서 causal chain을 끊지 않음. 여전히 causal system임.

### 5.2 Causal vs Non-Causal 판단 기준

**Causal System** (YOMAN):
```python
# 데이터 흐름이 명확한 의존성 그래프
IDEA → RESEARCH → TODO → PR → REVIEWER
       ↓           ↓      ↓
     (추가)     (구조화) (생성) → (검증)

# 각 단계는 이전 단계에 의존
# REVIEWER도 PR에 의존 (causal)
```

**Non-Causal System** (반례):
```python
# 독립적인 이벤트들이 임의 순서로 합쳐짐
Event A: 사용자 로그인
Event B: 상품 조회
Event C: 장바구니 추가
   ↓
최종 상태: A, B, C 순서 상관없이 동일 결과
```

### 5.3 GAN 검증 패턴의 Causality

**Generator (CODINGBOT)**:
```
TODO_ID → CODINGBOT → PR
(causal: TODO에 명확히 의존)
```

**Discriminator (REVIEWER)**:
```
(PR, TODO_Acceptance_Criteria) → REVIEWER → Pass/Fail
(causal: 두 입력에 모두 의존)
```

**전체 흐름**:
```
TODO ──────────┐
               ├──► CODINGBOT ──► PR ──┐
               │                       ├──► REVIEWER ──► Result
               └───────────────────────┘
                (Acceptance Criteria)
```

**결론**:
> GAN 패턴도 causal system임. Generator와 Discriminator 모두 명확한 입력 의존성 있음.

### 5.4 신뢰도 향상 방법 (Causal System 유지하며)

#### 방법 1: 멀티 라운드 검증 (Causal Chain 연장)

```
TODO → CODINGBOT → PR_v1 → REVIEWER → FAIL
                     ↓                   ↓
                  (피드백)          (재작업)
                     ↓                   ↓
TODO (수정) → CODINGBOT → PR_v2 → REVIEWER → PASS
```

**장점**:
- 여전히 causal (각 버전이 이전 버전에 의존)
- 신뢰도 향상 (여러 검증 단계)

#### 방법 2: 앙상블 검증 (Parallel Causal Chains)

```
TODO ──────┬──► CODINGBOT_1 ──► PR_1 ──┐
           ├──► CODINGBOT_2 ──► PR_2 ──┤
           └──► CODINGBOT_3 ──► PR_3 ──┴──► REVIEWER (다수결)
                                              ↓
                                           최종 PR
```

**장점**:
- 병렬 처리 (빠름)
- 각 체인은 causal
- 신뢰도 향상 (여러 모델 합의)

#### 방법 3: Staged Verification (Hierarchical Causality)

```
TODO → CODINGBOT → PR
                    ↓
                Unit Tests (자동) ──► PASS
                    ↓
                Lint/Format ──────► PASS
                    ↓
                Integration Tests ─► PASS
                    ↓
                REVIEWER (최종) ───► PASS
```

**장점**:
- 계층적 검증 (빠른 실패)
- 각 단계 causal
- 신뢰도 최대화

---

## 6. 구현 제안

### 6.1 Vector Clock 적용

**Supabase Schema**:
```sql
CREATE TABLE operations (
  id UUID PRIMARY KEY,
  project_id UUID NOT NULL,
  bucket TEXT NOT NULL,  -- IDEA, RESEARCH, TODO, PR
  version INTEGER NOT NULL,
  vector_clock JSONB NOT NULL,  -- { "node_id": timestamp }
  dependencies JSONB,  -- [{ "op_id": "...", "version": 1 }]
  content JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_vector_clock ON operations USING GIN (vector_clock);
```

**Dependency 추적**:
```python
class Operation:
    def __init__(self, bucket, content, deps=None):
        self.id = uuid4()
        self.bucket = bucket
        self.content = content
        self.vector_clock = VectorClock()
        self.dependencies = deps or []

    def depends_on(self, other_op):
        """다른 Operation에 의존성 추가"""
        self.dependencies.append({
            "op_id": other_op.id,
            "version": other_op.version
        })
        self.vector_clock.merge(other_op.vector_clock)
        self.vector_clock.increment(self.node_id)

    def can_apply(self, state):
        """모든 의존성이 충족되었는지 확인"""
        for dep in self.dependencies:
            if not state.has_version(dep["op_id"], dep["version"]):
                return False
        return True
```

### 6.2 CALM-aware Bucket Design

**Monotonic Buckets** (coordination-free):
```python
# IDEA: 추가만 가능
idea_db.add(new_idea)  # 단조 증가

# RESEARCH: IDEA에 정보 추가
research_db.add(idea_id, research_data)

# STUDY: RESEARCH에서 분기 (독립적)
study_db.add(research_id, study_content)
```

**Non-Monotonic Operations** (coordination 필요):
```python
# TODO 재작업 (버전 관리로 해결)
todo_db.update(todo_id, new_version)  # 실제로는 append
todo_db.versions[todo_id].append(new_version)

# REVIEWER 거부 (보상 트랜잭션)
if reviewer.result == FAIL:
    compensation_db.add({
        "pr_id": pr_id,
        "reason": "테스트 실패",
        "next_action": "TODO 수정"
    })
```

### 6.3 Causal+ Consistency 보장

**Read-Your-Writes 구현**:
```python
class CausalStore:
    def __init__(self):
        self.store = {}  # key → versions
        self.my_vector_clock = VectorClock()

    def write(self, key, value, deps):
        """쓰기 작업 (의존성 포함)"""
        self.my_vector_clock.increment(my_node_id)
        version = {
            "value": value,
            "vector_clock": self.my_vector_clock.copy(),
            "dependencies": deps
        }
        self.store[key] = version
        return version

    def read(self, key):
        """읽기 작업 (의존성 먼저 적용)"""
        version = self.store[key]

        # 의존성 먼저 적용 (COPS 방식)
        for dep in version["dependencies"]:
            self._ensure_dependency(dep)

        # 내 클럭 업데이트
        self.my_vector_clock.merge(version["vector_clock"])

        return version["value"]

    def _ensure_dependency(self, dep):
        """의존성 충족 대기"""
        while not self.has_version(dep["key"], dep["version"]):
            time.sleep(0.1)  # 실제로는 async await
```

---

## 7. 관련 논문 및 시스템

### 7.1 핵심 논문

**SOSP/OSDI 급**:

1. **"Don't Settle for Eventual: Scalable Causal Consistency for Wide-Area Storage with COPS"**
   - Venue: SOSP 2011
   - Authors: Wyatt Lloyd et al., Princeton/Facebook
   - Key Contribution: Causal+ Consistency, Dependency Tracking
   - Video: https://www.youtube.com/watch?v=jh9P1moDpAc (30분)

2. **"Bolt-on Causal Consistency"**
   - Venue: SIGMOD 2013
   - Authors: Peter Bailis et al., UC Berkeley
   - Key Contribution: 기존 시스템에 causal consistency 추가 (shim layer)

3. **"Consistency in Non-Transactional Distributed Storage Systems"**
   - Venue: ACM Computing Surveys 2016
   - 종합 서베이 논문

**CALM Theorem**:

4. **"Consistency Analysis in Bloom: a CALM and Collected Approach"**
   - Venue: CIDR 2011
   - Authors: Joseph M. Hellerstein et al., UC Berkeley
   - Key Contribution: CALM theorem 정의

**최신 연구** (arXiv 2024-2025):

5. **"Arbitration-Free Consistency is Available (and Vice Versa)"**
   - arXiv: 2510.21304 (2025년 10월)
   - Causal consistency와 availability 관계 탐구

6. **"A Framework for Consistency Models in Distributed Systems"**
   - arXiv: 2411.16355 (2024년 11월)
   - Axiomatic framework for causal consistency

7. **"A Unified, Practical, and Understandable Model of Non-transactional Consistency"**
   - arXiv: 2409.01576 (2024년 9월)
   - 실용적 consistency model 제안

### 7.2 실제 시스템

**Causal Consistency 사용 시스템**:

| 시스템 | 조직 | 특징 |
|--------|------|------|
| **COPS** | Princeton/Facebook | Causal+ consistency, geo-replication |
| **Cassandra** (옵션) | Apache | Lightweight transactions, causal reads |
| **Riak** | Basho | Vector clocks, eventual + causal |
| **MongoDB** (Causal Consistency) | MongoDB Inc | Session-based causal consistency |
| **AntidoteDB** | EU 연구 프로젝트 | CRDTs + causal consistency |
| **FaunaDB** | Fauna Inc | Calvin-based causal consistency |

**교육 자료**:

- **Mixu's Distributed Systems Book**: http://book.mixu.net/distsys/
  - Vector clocks, Lamport timestamps 구현
  - Consistency models 비교

---

## 8. YOMAN 적용 로드맵

### Phase 1: 기본 Dependency Tracking (1-2주)

- [ ] Supabase schema에 `vector_clock`, `dependencies` 컬럼 추가
- [ ] Python VectorClock 클래스 구현
- [ ] 버킷 간 이동 시 의존성 자동 추적

### Phase 2: Causal+ Consistency (2-3주)

- [ ] Read-Your-Writes 보장
- [ ] 의존성 충족 대기 로직
- [ ] Slack 메시지에 dependency 시각화

### Phase 3: CALM-aware Design (1-2주)

- [ ] Monotonic operations 식별
- [ ] Non-monotonic operations 버전 관리
- [ ] Compensation 트랜잭션 설계

### Phase 4: 검증 및 최적화 (2주)

- [ ] 동시성 테스트 (여러 Operation 병렬 처리)
- [ ] 의존성 그래프 순환 감지
- [ ] 성능 벤치마크

---

## 9. 결론

### 9.1 핵심 답변

**Q1: "리뷰어 개념이 들어가면 non-causal system이 되는가?"**
> **A: 아니다. REVIEWER도 명확한 입력(PR, TODO)에 의존하므로 여전히 causal system이다.**

**Q2: "Non-causal system으로 신뢰도를 높이는 법?"**
> **A: YOMAN은 이미 causal system이므로, causal consistency를 유지하며 신뢰도를 높여야 함.**
>
> 방법:
> - 멀티 라운드 검증 (causal chain 연장)
> - 앙상블 검증 (parallel causal chains)
> - 계층적 검증 (staged verification)

### 9.2 Causal Consistency가 YOMAN에 주는 이점

1. **병렬 처리 안전성**
   - 인과관계 없는 Operation은 동시 실행 가능
   - Vector clock으로 충돌 감지

2. **신뢰도 검증**
   - 의존성 충족 확인 (Prerequisites)
   - Read-Your-Writes 보장

3. **디버깅 용이성**
   - 전체 의존성 그래프 추적
   - 문제 Operation 역추적 가능

4. **확장성**
   - Coordination-free operations (CALM)
   - 여러 AI 모델 병렬 실행

### 9.3 다음 단계

1. **Vector Clock 프로토타입** 구현 (1주)
2. **Dependency Visualization** Slack 통합 (1주)
3. **Causal+ Consistency** 전체 적용 (2-3주)
4. **논문 작성** - YOMAN의 causal consistency 사례 연구

---

## 10. 참고 문헌

### Academic Papers

1. Lloyd, W., et al. (2011). "Don't Settle for Eventual: Scalable Causal Consistency for Wide-Area Storage with COPS." SOSP.
2. Bailis, P., et al. (2013). "Bolt-on Causal Consistency." SIGMOD.
3. Hellerstein, J. M., et al. (2011). "Consistency Analysis in Bloom: a CALM and Collected Approach." CIDR.
4. Viotti, P., & Vukolić, M. (2016). "Consistency in Non-Transactional Distributed Storage Systems." ACM Computing Surveys.
5. Attiya, H., et al. (2025). "Arbitration-Free Consistency is Available (and Vice Versa)." arXiv:2510.21304.

### Books & Educational Resources

6. van Steen, M., & Tanenbaum, A. S. (2017). "Distributed Systems" (3rd ed.).
7. Kleppmann, M. (2017). "Designing Data-Intensive Applications." O'Reilly.
8. Mixu. "Distributed Systems: For Fun and Profit." http://book.mixu.net/distsys/

### System Documentation

9. MongoDB. "Causal Consistency." https://docs.mongodb.com/manual/core/causal-consistency/
10. Apache Cassandra. "Lightweight Transactions." https://cassandra.apache.org/doc/latest/cql/dml.html#lightweight-transactions

---

**문서 작성**: 2026-01-01
**분석 도구**: Claude Sonnet 4.5, Context7 MCP, Sequential Thinking
**관련 파일**: [idea3.txt](../../../idea3.txt), [YOMAN-PROJECT-OVERVIEW.md](../../YOMAN-PROJECT-OVERVIEW.md)
