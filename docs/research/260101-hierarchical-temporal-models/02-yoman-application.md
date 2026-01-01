# YOMAN Application Strategy

> N-Tier 아키텍처를 YOMAN Bucket System에 적용하는 구체적 방안

**Date**: 2026-01-01
**Prerequisites**: 01-n-tier-architecture.md

---

## 1. 개요

### 1.1 YOMAN 기본 구조 복습

```
Bucket System (인터페이스):
  IDEA → RESEARCH → TODO → PR

UnitService (엔진):
  System (Tier 3) → Module (Tier 2) → Unit (Tier 1)
```

**N-Tier 적용 전략**:
- Bucket 단계마다 다른 Tier 활용
- 복잡도 기반 동적 tier 선택
- Block Type Mixing으로 비용 최적화

---

## 2. Bucket별 Tier 활용 전략

### 2.1 IDEA Bucket

**특성**:
- 러프한 텍스트 (비구조화)
- 빠른 생성 필요
- 정확도 낮아도 됨

**권장 Tier**: **Tier 2 (Module) 또는 Tier 1 (Unit)**

```python
# Simple project (2-tier)
IDEA → Flash (Tier 1)
  - 빠른 러프 생성
  - 비용: $0.01/request

# Medium project (3-tier)
IDEA → Sonnet (Tier 2)
  - 더 나은 구조화
  - 비용: $0.05/request
```

**예시**:
```
User: "AI 에이전트 만들고 싶음"
  ↓ (Flash)
IDEA Output:
  - Title: "AI Agent Framework"
  - Description: "LLM 기반 autonomous agent"
  - Tags: ["AI", "Agent", "LLM"]
```

---

### 2.2 RESEARCH Bucket

**특성**:
- 구조화된 JSON 필요
- 전체 맥락 이해 필요
- 정확도 중요

**권장 Tier**: **Tier 3 (System) 또는 Tier 2 (Module)**

```python
# Simple project (2-tier)
RESEARCH → Sonnet (Tier 1, 최상위)
  - 구조화 능력 필요
  - 비용: $0.10/request

# Medium project (3-tier)
RESEARCH → Opus (Tier 3)
  - 전체 프로젝트 맥락 고려
  - 비용: $0.50/request

# Complex project (4-tier)
RESEARCH → Opus (Tier 3, System)
  - 여러 서비스 간 연관성 파악
  - 비용: $0.50/request
```

**Opus가 필요한 이유**:
- 기존 docs/ 전체 읽기
- 아키텍처 일관성 확인
- 의존성 분석

---

### 2.3 TODO Bucket

**특성**:
- 실행 가능한 스펙 생성
- Prerequisites 체크
- 중간 복잡도

**권장 Tier**: **Tier 2 (Module) + Tier 1 (Unit) 조합**

```python
# 2단계 프로세스
Step 1: Sonnet (Tier 2) - TODO 스펙 생성
  - Prerequisites 나열
  - Acceptance Criteria 정의

Step 2: Flash (Tier 1) - Subtask 분해
  - Unit 단위로 분해 (병렬)
  - 각 Unit은 Flash가 빠르게 처리
```

**예시**:
```
RESEARCH → TODO (Sonnet)
  - Title: "JWT 인증 구현"
  - Prerequisites: ["bcrypt 설치", "JWT library"]
  - Subtasks: 3개

TODO → Subtasks (Flash, 병렬)
  - Unit 1: middleware 구현
  - Unit 2: route 추가
  - Unit 3: test 작성
```

---

### 2.4 PR Bucket

**특성**:
- 코드 생성 (Generator)
- 검증 (Discriminator)
- 신뢰도 최우선

**권장 Tier**: **Multi-Tier 조합 (GAN 방식)**

```python
# Generator (Tier 1-2)
CODINGBOT → Flash/Sonnet
  - 빠른 코드 생성
  - Causal (순차 생성)

# Discriminator (Tier 3)
REVIEWER → Opus
  - 전체 맥락에서 검증
  - Non-Causal (전체 보고 판단)
```

**프로세스**:
```
1. TODO → PR (Flash/Sonnet)
   - 빠르게 코드 생성

2. PR vs TODO (Opus)
   - match_rate 계산
   - 아키텍처 일관성 확인

3. if match_rate < 80%:
     - Feedback 생성 (Opus)
     - 재생성 (Flash/Sonnet)
   else:
     - 머지
```

---

## 3. UnitService N-Tier 확장

### 3.1 기존 3-Tier UnitService

```python
class UnitService:
    def __init__(self):
        self.system = OpusModel()   # Tier 3
        self.module = SonnetModel()  # Tier 2
        self.unit = FlashModel()     # Tier 1

    def process(self, task: Task):
        # 고정된 3계층
        context = self.system.get_context()
        operation = self.module.plan(context, task)
        result = self.unit.execute(operation)
        return result
```

**한계**:
- 모든 task에 3계층 사용 (비효율)
- 복잡도 고려 안 함

---

### 3.2 N-Tier UnitService

```python
class UnitService:
    def __init__(self, config: YOMANConfig):
        self.config = config
        self.tiers = self._initialize_tiers()

    def _initialize_tiers(self) -> Dict[str, LLMModel]:
        """Config 기반 tier 초기화"""
        tiers = {}
        for tier_config in self.config.tiers:
            model = self._create_model(tier_config.model)
            tiers[tier_config.name] = model
        return tiers

    def process(self, task: Task):
        """복잡도 기반 tier 선택"""
        complexity = self._estimate_complexity(task)
        selected_tier = self._select_tier(complexity)

        # 선택된 tier로 처리
        model = self.tiers[selected_tier.name]
        result = model.process(task)

        return result

    def _select_tier(self, complexity: float) -> TierConfig:
        """복잡도에 맞는 tier 선택"""
        for tier in self.config.tiers:
            if complexity <= tier.max_complexity:
                return tier
        # 복잡도 초과 시 최상위 tier
        return self.config.tiers[0]
```

---

### 3.3 Bucket 처리 흐름

```python
class BucketProcessor:
    def __init__(self, unit_service: UnitService):
        self.unit_service = unit_service

    def process_idea(self, idea: str) -> IDEA:
        """IDEA Bucket 처리"""
        task = Task(
            type="idea_generation",
            input=idea,
            complexity=10,  # 낮은 복잡도
        )
        result = self.unit_service.process(task)
        # → Flash (Tier 1) 선택됨
        return IDEA(**result)

    def process_research(self, idea: IDEA) -> RESEARCH:
        """RESEARCH Bucket 처리"""
        task = Task(
            type="research",
            input=idea,
            complexity=150,  # 높은 복잡도
        )
        result = self.unit_service.process(task)
        # → Opus (Tier 3) 선택됨
        return RESEARCH(**result)

    def process_todo(self, research: RESEARCH) -> TODO:
        """TODO Bucket 처리 (2단계)"""

        # Step 1: TODO 스펙 (Sonnet)
        spec_task = Task(
            type="todo_spec",
            input=research,
            complexity=80,
        )
        spec = self.unit_service.process(spec_task)
        # → Sonnet (Tier 2) 선택됨

        # Step 2: Subtask 분해 (Flash, 병렬)
        subtasks = []
        for sub in spec["subtasks"]:
            sub_task = Task(
                type="subtask",
                input=sub,
                complexity=20,
            )
            result = self.unit_service.process(sub_task)
            # → Flash (Tier 1) 선택됨
            subtasks.append(result)

        return TODO(spec=spec, subtasks=subtasks)
```

---

## 4. Continuous Planning 적용

### 4.1 GoalAct 방식

**기존 접근**:
```
Plan 생성 → Execute → Done (plan 버림)
```

**GoalAct 방식**:
```
Plan 생성 → Execute → Update Plan → Execute → ...
```

**YOMAN 적용**:
```python
class GlobalPlanner:
    def __init__(self):
        self.plan = None  # docs/CLAUDE.md
        self.memory = MemoryMCP()

    def create_plan(self, project: Project):
        """초기 plan 생성 (Opus, Tier 3)"""
        task = Task(
            type="global_planning",
            input=project,
            complexity=200,  # 최고 복잡도
        )
        self.plan = unit_service.process(task)
        self.memory.write("global_plan", self.plan)

    def update_plan(self, operation_result: OperationResult):
        """Operation 완료 후 plan 업데이트"""
        feedback = {
            "operation": operation_result.operation_id,
            "success": operation_result.success,
            "learnings": operation_result.learnings,
        }

        # Opus로 plan 조정
        task = Task(
            type="plan_update",
            input={
                "current_plan": self.plan,
                "feedback": feedback,
            },
            complexity=150,
        )
        updated_plan = unit_service.process(task)

        # Plan 업데이트
        self.plan = updated_plan
        self.memory.update("global_plan", updated_plan)

        # docs/CLAUDE.md 업데이트
        self._write_to_docs(updated_plan)
```

---

### 4.2 Memory MCP 통합

**기존**:
```python
# 매번 파일 읽기 (5000 tokens)
Read("/path/to/file.py")
```

**N-Tier + Memory MCP**:
```python
# 1. 초기 로딩 (Tier 3, Opus)
system_tier.load_project()
  → Memory MCP에 entities/relations 저장

# 2. 이후 조회 (200 tokens)
memory.search_nodes("BucketTrigger")
  → 96% 토큰 절약

# 3. 주기적 sync (Tier 3, 하루 1회)
system_tier.sync_memory()
  → 변경사항만 업데이트
```

**구현**:
```python
class TierWithMemory:
    def __init__(self, tier_config: TierConfig, memory: MemoryMCP):
        self.config = tier_config
        self.memory = memory
        self.model = self._create_model()

    def process(self, task: Task):
        # Memory MCP에서 context 조회
        context = self.memory.search_nodes(task.keywords)

        # 모델 실행 (작은 context)
        result = self.model.generate(
            prompt=task.prompt,
            context=context,  # 200 tokens
        )

        # 결과를 Memory MCP에 저장
        self.memory.add_observations(
            entity=task.entity,
            observations=[result.summary],
        )

        return result
```

---

## 5. 비용 최적화 전략

### 5.1 Bucket별 비용 분석

**기존 (고정 3-tier)**:

| Bucket | Model | 호출 횟수 (월) | 비용/호출 | 월 비용 |
|--------|-------|--------------|----------|---------|
| IDEA | Sonnet | 100 | $0.05 | $5 |
| RESEARCH | Opus | 50 | $0.50 | $25 |
| TODO | Sonnet | 80 | $0.10 | $8 |
| PR (Generator) | Sonnet | 60 | $0.10 | $6 |
| PR (Discriminator) | Opus | 60 | $0.50 | $30 |
| **Total** | - | 350 | - | **$74** |

---

**N-Tier 최적화 (2-tier, Simple project)**:

| Bucket | Model | 호출 횟수 (월) | 비용/호출 | 월 비용 |
|--------|-------|--------------|----------|---------|
| IDEA | Flash | 100 | $0.01 | $1 |
| RESEARCH | Sonnet | 50 | $0.10 | $5 |
| TODO | Sonnet + Flash | 80 + 240 | $0.05 + $0.01 | $6.4 |
| PR (Generator) | Flash | 60 | $0.01 | $0.6 |
| PR (Discriminator) | Sonnet | 60 | $0.10 | $6 |
| **Total** | - | 590 | - | **$19** |

**절감률**: **74%** ($74 → $19)

---

### 5.2 Block Type Mixing 전략

**90/10 Rule**:
- 90% 트래픽: Flash (하위 tier)
- 10% 트래픽: Opus (상위 tier)

```python
# 비용 계산
cost_90 = 0.9 * total_requests * flash_cost  # $0.01
cost_10 = 0.1 * total_requests * opus_cost   # $0.50

total_cost = cost_90 + cost_10

# 예: 월 1000 requests
# Old: 1000 * $0.10 (Sonnet 평균) = $100
# New: 900 * $0.01 + 100 * $0.50 = $9 + $50 = $59
# 절감: 41%
```

---

## 6. 실전 시나리오

### 6.1 Scenario: CLI Tool 개발

**프로젝트 정보**:
```
Type: Simple CLI tool
Files: 5
LOC: 500
Complexity: 15
```

**선택된 Config**: 2-tier (Sonnet + Flash)

**작업 흐름**:

```
1. IDEA: "CLI tool for file conversion"
   → Flash (Tier 1, 2초)
   → IDEA object 생성

2. RESEARCH: "지원 포맷, 라이브러리 조사"
   → Sonnet (Tier 1, 최상위, 10초)
   → JSON 구조화

3. TODO: "main.py, converter.py, test 작성"
   → Sonnet (Tier 1, 5초) - 스펙 생성
   → Flash (Tier 1, 병렬 3개, 각 2초) - Subtask

4. PR: "코드 생성 + 검증"
   → Flash (Tier 1, 10초) - Generator
   → Sonnet (Tier 1, 5초) - Discriminator
   → match_rate: 85% → 머지
```

**총 소요 시간**: ~40초
**총 비용**: $0.15

---

### 6.2 Scenario: Microservices

**프로젝트 정보**:
```
Type: Microservices
Files: 150
LOC: 50000
Complexity: 180
```

**선택된 Config**: 4-tier (System, Service, API, Logic)

**작업 흐름**:

```
1. IDEA: "Payment service 추가"
   → Sonnet (Tier 2, Module level)
   → IDEA object

2. RESEARCH: "아키텍처 영향도 분석"
   → Opus (Tier 4, System level)
   → 전체 서비스 의존성 분석
   → 15분 소요

3. TODO: "Service, API, DB migration"
   → Opus (Tier 4, 5분) - 스펙
   → Sonnet (Tier 2, 병렬 5개, 각 2분) - Service subtasks
   → Flash (Tier 1, 병렬 10개, 각 30초) - Unit subtasks

4. PR: "코드 생성 + 검증"
   → Sonnet (Tier 2, 10분) - Generator
   → Opus (Tier 4, 5분) - Discriminator
   → match_rate: 75% → Feedback → 재생성
   → 2nd attempt: 82% → 머지
```

**총 소요 시간**: ~45분
**총 비용**: $8.50

---

## 7. Bucket Trigger 재설계

### 7.1 기존 Bucket Trigger

```python
# 고정된 흐름
IDEA → RESEARCH → TODO → PR
```

**한계**:
- 모든 IDEA가 PR까지 가야 함
- 중간 검증 없음

---

### 7.2 N-Tier Bucket Trigger

```python
class BucketTrigger:
    def __init__(self, unit_service: UnitService):
        self.unit_service = unit_service
        self.discriminator = MultiDiscriminator()

    def trigger_research(self, idea: IDEA):
        """IDEA → RESEARCH 전환 전 검증"""

        # D1: Idea Validator (Tier 3)
        score = self.discriminator.verify_idea(idea)

        if score < 0.6:
            return {"status": "REJECT", "reason": "Idea 불명확"}

        # 통과 시 RESEARCH 시작
        research = self.process_research(idea)
        return {"status": "PASS", "research": research}

    def trigger_todo(self, research: RESEARCH):
        """RESEARCH → TODO 전환 전 검증"""

        # D2: Research Checker (Tier 3)
        score = self.discriminator.verify_research(research)

        if score < 0.7:
            return {"status": "REJECT", "reason": "Research 불완전"}

        # 통과 시 TODO 생성
        todo = self.process_todo(research)
        return {"status": "PASS", "todo": todo}

    def trigger_pr(self, todo: TODO):
        """TODO → PR 전환 전 검증"""

        # D3: TODO Feasibility (Tier 3)
        score = self.discriminator.verify_todo(todo)

        if score < 0.8:
            return {"status": "REJECT", "reason": "TODO 실행 불가능"}

        # 통과 시 PR 생성
        pr = self.process_pr(todo)

        # D4: PR Correctness (Tier 3)
        match_rate = self.discriminator.verify_pr(pr, todo)

        if match_rate < 0.8:
            # Feedback 생성 후 재시도
            feedback = self.discriminator.explain_gap(pr, todo)
            return {"status": "RETRY", "feedback": feedback}

        return {"status": "MERGE", "pr": pr}
```

**장점**:
- Early Stopping (조기 차단)
- Layered Verification
- Feedback Loop

---

## 8. 다음 단계

### 8.1 구현 우선순위

**Phase 1: 2-Tier 프로토타입 (1-2주)**
- [ ] YOMANConfig 스키마
- [ ] 간단한 CLI tool 테스트
- [ ] Flash + Sonnet 조합 검증

**Phase 2: Bucket 연동 (2-3주)**
- [ ] BucketProcessor 구현
- [ ] Multi-Discriminator 추가
- [ ] IDEA → RESEARCH → TODO 흐름

**Phase 3: N-Tier 완성 (1-2개월)**
- [ ] 4-tier, 5-tier 지원
- [ ] Dynamic tier adjustment
- [ ] Memory MCP 통합

---

## 9. 참고 문헌

- `01-n-tier-architecture.md` - N-Tier 설계
- `03-causal-solution.md` - Multi-Discriminator
- **GoalAct** ([arXiv:2504.16563](https://arxiv.org/abs/2504.16563)) - Continuous Planning

---

*작성일: 2026-01-01*
*분석 도구: Claude Opus 4.5*
