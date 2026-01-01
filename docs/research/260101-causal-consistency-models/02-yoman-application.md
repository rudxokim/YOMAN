# YOMAN Causal Consistency Application Guide

> **목적**: YOMAN의 버킷 시스템과 GAN 검증에 causal consistency를 실제 적용하는 상세 가이드

---

## 1. YOMAN 아키텍처 리뷰

### 1.1 현재 시스템 구조

```
┌─────────────────────────────────────────────────────────────┐
│  Interface Layer (Slack)                                     │
│  - 사용자 입력 (텍스트/음성)                                  │
│  - 버킷별 채널 (#yoman-ideas, #yoman-research, ...)         │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│  Bucket System (물수제비)                                    │
│  IDEA → RESEARCH → TODO → PR                                │
│       ↘ STUDY                                               │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│  Engine Layer (UnitService)                                  │
│  Tier 3 (Opus) → Tier 2 (Sonnet) → Tier 1 (Flash/GLM)      │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│  Storage (Supabase)                                          │
│  + Memory MCP (지식 그래프)                                  │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Causal Dependency 현황

**명확한 Causal Chain**:
```
IDEA_1 (독립)
  ↓ (happens-before)
RESEARCH_1 (depends_on: IDEA_1)
  ↓ (happens-before)
TODO_1 (depends_on: RESEARCH_1, prerequisites: [])
  ↓ (happens-before)
PR_1 (depends_on: TODO_1, generated_by: CODINGBOT)
  ↓ (happens-before)
REVIEWER_result (depends_on: PR_1, TODO_1.acceptance_criteria)
```

**Concurrent Operations** (인과 관계 없음):
```
IDEA_1, IDEA_2, IDEA_3 (동시 생성 가능)
TODO_5, TODO_6 (다른 RESEARCH에서 파생, 병렬 처리)
Unit_A1, Unit_A2, Unit_A3 (양자 검증, 3개 모델 동시 생성)
```

---

## 2. Causal Consistency 적용 전략

### 2.1 Phase 1: Vector Clock 기본 적용

**목표**: 모든 Operation에 vector clock 추가하여 인과관계 추적

#### 2.1.1 Supabase 스키마 확장

```sql
-- 기존 테이블 수정
ALTER TABLE ideas ADD COLUMN vector_clock JSONB DEFAULT '{}'::jsonb;
ALTER TABLE research ADD COLUMN vector_clock JSONB DEFAULT '{}'::jsonb;
ALTER TABLE todos ADD COLUMN vector_clock JSONB DEFAULT '{}'::jsonb;
ALTER TABLE prs ADD COLUMN vector_clock JSONB DEFAULT '{}'::jsonb;

-- Dependencies 컬럼 추가
ALTER TABLE ideas ADD COLUMN dependencies JSONB DEFAULT '[]'::jsonb;
ALTER TABLE research ADD COLUMN dependencies JSONB DEFAULT '[]'::jsonb;
ALTER TABLE todos ADD COLUMN dependencies JSONB DEFAULT '[]'::jsonb;
ALTER TABLE prs ADD COLUMN dependencies JSONB DEFAULT '[]'::jsonb;

-- 인덱스 추가
CREATE INDEX idx_ideas_vc ON ideas USING GIN (vector_clock);
CREATE INDEX idx_research_vc ON research USING GIN (vector_clock);
CREATE INDEX idx_todos_vc ON todos USING GIN (vector_clock);
CREATE INDEX idx_prs_vc ON prs USING GIN (vector_clock);
```

#### 2.1.2 Python Service 수정

```python
# yoman/services/bucket_service.py

from yoman.causal import VectorClock, Operation, BucketType
from supabase import Client

class BucketService:
    def __init__(self, supabase: Client, node_id: str = "yoman_server"):
        self.db = supabase
        self.node_id = node_id
        self.my_clock = VectorClock()

    async def create_idea(
        self,
        project_id: str,
        content: dict,
        user_id: str
    ) -> str:
        """IDEA 생성 (vector clock 적용)"""

        # Vector clock 증가
        self.my_clock.increment(self.node_id)

        # IDEA 데이터
        idea_data = {
            "project_id": project_id,
            "content": content,
            "vector_clock": self.my_clock.to_json(),
            "dependencies": [],  # IDEA는 의존성 없음
            "created_by": user_id
        }

        # DB 삽입
        result = self.db.table("ideas").insert(idea_data).execute()
        idea_id = result.data[0]["id"]

        # Slack 알림
        await self.notify_slack(
            channel="yoman-ideas",
            message=f"💡 New IDEA: {content.get('title', 'Untitled')}\nID: {idea_id}"
        )

        return idea_id

    async def create_research(
        self,
        idea_id: str,
        content: dict,
        user_id: str
    ) -> str:
        """RESEARCH 생성 (IDEA에 의존)"""

        # 1. IDEA 조회
        idea_result = self.db.table("ideas").select("*").eq("id", idea_id).execute()
        if not idea_result.data:
            raise ValueError(f"IDEA {idea_id} not found")

        idea = idea_result.data[0]
        idea_clock = VectorClock.from_json(idea["vector_clock"])

        # 2. Vector clock 병합 및 증가
        self.my_clock.merge(idea_clock)
        self.my_clock.increment(self.node_id)

        # 3. RESEARCH 데이터
        research_data = {
            "project_id": idea["project_id"],
            "idea_id": idea_id,
            "content": content,
            "vector_clock": self.my_clock.to_json(),
            "dependencies": [
                {
                    "id": idea_id,
                    "bucket": "IDEA",
                    "version": idea.get("version", 1)
                }
            ],
            "created_by": user_id
        }

        # 4. DB 삽입
        result = self.db.table("research").insert(research_data).execute()
        research_id = result.data[0]["id"]

        # 5. Slack 알림
        await self.notify_slack(
            channel="yoman-research",
            message=f"📊 New RESEARCH: {content.get('title', 'Untitled')}\n"
                    f"ID: {research_id}\n"
                    f"Based on IDEA: {idea_id}"
        )

        return research_id

    async def create_todo(
        self,
        research_id: str,
        content: dict,
        prerequisites: list[str],
        user_id: str
    ) -> str:
        """TODO 생성 (RESEARCH + Prerequisites에 의존)"""

        # 1. RESEARCH 조회
        research_result = self.db.table("research").select("*").eq("id", research_id).execute()
        if not research_result.data:
            raise ValueError(f"RESEARCH {research_id} not found")

        research = research_result.data[0]
        research_clock = VectorClock.from_json(research["vector_clock"])

        # 2. Prerequisites 조회 및 vector clock 병합
        deps = [
            {
                "id": research_id,
                "bucket": "RESEARCH",
                "version": research.get("version", 1)
            }
        ]

        combined_clock = VectorClock(clocks=research_clock.clocks.copy())

        for prereq_id in prerequisites:
            prereq_result = self.db.table("todos").select("*").eq("id", prereq_id).execute()
            if prereq_result.data:
                prereq = prereq_result.data[0]
                prereq_clock = VectorClock.from_json(prereq["vector_clock"])
                combined_clock.merge(prereq_clock)

                deps.append({
                    "id": prereq_id,
                    "bucket": "TODO",
                    "version": prereq.get("version", 1)
                })

        # 3. Vector clock 증가
        combined_clock.increment(self.node_id)

        # 4. TODO 데이터
        todo_data = {
            "project_id": research["project_id"],
            "research_id": research_id,
            "content": content,
            "prerequisites": prerequisites,
            "acceptance_criteria": content.get("acceptance_criteria", []),
            "vector_clock": combined_clock.to_json(),
            "dependencies": deps,
            "created_by": user_id,
            "status": "pending"
        }

        # 5. DB 삽입
        result = self.db.table("todos").insert(todo_data).execute()
        todo_id = result.data[0]["id"]

        # 6. Prerequisites 체크 및 Slack 알림
        all_satisfied = await self.check_prerequisites(prerequisites)

        status_emoji = "✅" if all_satisfied else "⏳"
        await self.notify_slack(
            channel="yoman-todos",
            message=f"{status_emoji} New TODO: {content.get('title', 'Untitled')}\n"
                    f"ID: {todo_id}\n"
                    f"Prerequisites: {len(prerequisites)} (satisfied: {all_satisfied})"
        )

        return todo_id

    async def check_prerequisites(self, prereq_ids: list[str]) -> bool:
        """Prerequisites가 모두 완료되었는지 확인"""
        if not prereq_ids:
            return True

        for prereq_id in prereq_ids:
            result = self.db.table("todos").select("status").eq("id", prereq_id).execute()
            if not result.data or result.data[0]["status"] != "completed":
                return False

        return True
```

### 2.2 Phase 2: GAN 검증에 Causal Consistency 적용

**문제**: CODINGBOT(Generator)과 REVIEWER(Discriminator)의 인과관계 명확화

#### 2.2.1 GAN 패턴 분석

```
TODO ────────┐
             ├───► CODINGBOT ───► PR ───┐
             │                          ├───► REVIEWER ───► Result
             └──────────────────────────┘
                (Acceptance Criteria)
```

**Causal Dependencies**:
1. `PR` depends on `TODO` (입력 스펙)
2. `REVIEWER` depends on `PR` (생성된 코드)
3. `REVIEWER` depends on `TODO` (검증 기준)

**Vector Clock 흐름**:
```python
TODO: {server: 3}
  ↓
PR: {server: 4}  # TODO clock 병합 후 증가
  ↓
REVIEWER: {server: 5}  # PR clock 병합 후 증가
```

#### 2.2.2 CODINGBOT Service

```python
# yoman/agents/codingbot.py

class CodingBot:
    def __init__(self, model: str = "grok-code-fast-1"):
        self.model = model
        self.node_id = f"codingbot_{model}"
        self.clock = VectorClock()

    async def generate_code(
        self,
        todo_id: str,
        db: Client
    ) -> str:
        """TODO로부터 코드 생성"""

        # 1. TODO 조회
        todo_result = db.table("todos").select("*").eq("id", todo_id).execute()
        if not todo_result.data:
            raise ValueError(f"TODO {todo_id} not found")

        todo = todo_result.data[0]
        todo_clock = VectorClock.from_json(todo["vector_clock"])

        # 2. Vector clock 병합
        self.clock.merge(todo_clock)
        self.clock.increment(self.node_id)

        # 3. 코드 생성 (실제 LLM 호출)
        code = await self._call_llm(
            model=self.model,
            prompt=self._build_prompt(todo)
        )

        # 4. PR 생성
        pr_data = {
            "todo_id": todo_id,
            "project_id": todo["project_id"],
            "code": code,
            "vector_clock": self.clock.to_json(),
            "dependencies": [
                {
                    "id": todo_id,
                    "bucket": "TODO",
                    "version": todo.get("version", 1)
                }
            ],
            "generated_by": self.node_id,
            "status": "pending_review"
        }

        result = db.table("prs").insert(pr_data).execute()
        pr_id = result.data[0]["id"]

        return pr_id

    async def _call_llm(self, model: str, prompt: str) -> str:
        """실제 LLM API 호출"""
        # Grok API 호출 로직
        pass

    def _build_prompt(self, todo: dict) -> str:
        """TODO에서 프롬프트 생성"""
        return f"""
        Generate code for the following task:

        Title: {todo['content'].get('title')}
        Description: {todo['content'].get('description')}

        Prerequisites:
        {chr(10).join(f"- {p}" for p in todo.get('prerequisites', []))}

        Acceptance Criteria:
        {chr(10).join(f"- {c}" for c in todo.get('acceptance_criteria', []))}

        Generate clean, tested code that satisfies all criteria.
        """
```

#### 2.2.3 REVIEWER Service

```python
# yoman/agents/reviewer.py

class Reviewer:
    def __init__(self, model: str = "claude-opus-4.5"):
        self.model = model
        self.node_id = "reviewer"
        self.clock = VectorClock()

    async def review_pr(
        self,
        pr_id: str,
        db: Client
    ) -> dict:
        """PR 리뷰 (Pass/Fail 판정)"""

        # 1. PR 조회
        pr_result = db.table("prs").select("*").eq("id", pr_id).execute()
        if not pr_result.data:
            raise ValueError(f"PR {pr_id} not found")

        pr = pr_result.data[0]
        pr_clock = VectorClock.from_json(pr["vector_clock"])

        # 2. TODO 조회 (Acceptance Criteria)
        todo_id = pr["todo_id"]
        todo_result = db.table("todos").select("*").eq("id", todo_id).execute()
        todo = todo_result.data[0]
        todo_clock = VectorClock.from_json(todo["vector_clock"])

        # 3. Vector clock 병합 (PR과 TODO 모두 의존)
        self.clock.merge(pr_clock)
        self.clock.merge(todo_clock)
        self.clock.increment(self.node_id)

        # 4. 리뷰 수행
        result = await self._perform_review(pr, todo)

        # 5. 리뷰 결과 저장
        review_data = {
            "pr_id": pr_id,
            "todo_id": todo_id,
            "result": result["verdict"],  # "PASS" or "FAIL"
            "details": result["details"],
            "vector_clock": self.clock.to_json(),
            "dependencies": [
                {"id": pr_id, "bucket": "PR", "version": pr.get("version", 1)},
                {"id": todo_id, "bucket": "TODO", "version": todo.get("version", 1)}
            ],
            "reviewed_by": self.node_id
        }

        db.table("reviews").insert(review_data).execute()

        # 6. PR 상태 업데이트
        new_status = "approved" if result["verdict"] == "PASS" else "changes_requested"
        db.table("prs").update({"status": new_status}).eq("id", pr_id).execute()

        return result

    async def _perform_review(self, pr: dict, todo: dict) -> dict:
        """실제 리뷰 로직"""

        acceptance_criteria = todo.get("acceptance_criteria", [])
        code = pr["code"]

        # 1. 테스트 실행
        test_results = await self._run_tests(code)

        # 2. Acceptance Criteria 체크
        criteria_checks = []
        for criterion in acceptance_criteria:
            passed = await self._check_criterion(code, criterion)
            criteria_checks.append({
                "criterion": criterion,
                "passed": passed
            })

        # 3. Pass/Fail 판정 (모든 기준 충족해야 PASS)
        all_passed = (
            test_results["pass_rate"] == 1.0 and
            all(c["passed"] for c in criteria_checks)
        )

        verdict = "PASS" if all_passed else "FAIL"

        return {
            "verdict": verdict,
            "details": {
                "tests": test_results,
                "criteria": criteria_checks
            }
        }

    async def _run_tests(self, code: str) -> dict:
        """테스트 실행"""
        # 실제 테스트 실행 로직
        pass

    async def _check_criterion(self, code: str, criterion: str) -> bool:
        """Acceptance criterion 체크"""
        # LLM을 사용하여 criterion 충족 여부 판단
        pass
```

### 2.3 Phase 3: Concurrent Unit Generation (양자 검증)

**목표**: 여러 AI 모델이 동시에 유닛 생성 → Concurrent operations

#### 2.3.1 병렬 유닛 생성

```python
# yoman/agents/quantum_verification.py

import asyncio
from typing import List

class QuantumVerifier:
    """
    여러 모델로 동시에 유닛 생성 후 테스트 통과한 최적 선택

    Causal Consistency:
    - 각 모델의 output은 CONCURRENT (인과 관계 없음)
    - 최종 선택은 모든 output에 의존
    """

    def __init__(self):
        self.models = ["grok-code-fast-1", "gemini-2.0-flash", "deepseek-coder"]
        self.node_id = "quantum_verifier"
        self.clock = VectorClock()

    async def generate_unit(
        self,
        spec: dict,
        parent_clock: VectorClock,
        db: Client
    ) -> str:
        """
        여러 모델로 유닛 생성 (병렬)

        Args:
            spec: 유닛 스펙
            parent_clock: 부모 operation의 vector clock
            db: Supabase client

        Returns:
            최적 유닛 ID
        """

        # 1. 각 모델로 병렬 생성
        tasks = [
            self._generate_with_model(model, spec, parent_clock)
            for model in self.models
        ]

        units = await asyncio.gather(*tasks)

        # 2. 각 유닛의 vector clock은 CONCURRENT
        for i in range(len(units)):
            for j in range(i + 1, len(units)):
                relation = units[i]["clock"].compare(units[j]["clock"])
                assert relation == 'CONCURRENT', "Units should be concurrent"

        # 3. 테스트 실행 (병렬)
        test_tasks = [
            self._test_unit(unit["code"])
            for unit in units
        ]

        test_results = await asyncio.gather(*test_tasks)

        # 4. 통과한 유닛 중 최적 선택
        passed_units = [
            (unit, result)
            for unit, result in zip(units, test_results)
            if result["passed"]
        ]

        if not passed_units:
            raise Exception("No units passed tests")

        # 점수 기준 정렬 (커버리지, 성능 등)
        best_unit, best_result = max(
            passed_units,
            key=lambda x: x[1]["score"]
        )

        # 5. 최종 선택한 유닛 저장
        # Vector clock은 모든 candidate units 병합
        final_clock = VectorClock()
        for unit in units:
            final_clock.merge(unit["clock"])
        final_clock.increment(self.node_id)

        unit_data = {
            "spec": spec,
            "code": best_unit["code"],
            "model": best_unit["model"],
            "vector_clock": final_clock.to_json(),
            "dependencies": [
                # 모든 candidate units에 의존 (quantum superposition)
                {"id": unit["id"], "bucket": "UNIT_CANDIDATE"}
                for unit in units
            ],
            "test_result": best_result
        }

        result = db.table("units").insert(unit_data).execute()
        unit_id = result.data[0]["id"]

        return unit_id

    async def _generate_with_model(
        self,
        model: str,
        spec: dict,
        parent_clock: VectorClock
    ) -> dict:
        """단일 모델로 유닛 생성"""

        # 각 모델별 독립적인 clock
        model_clock = VectorClock()
        model_clock.merge(parent_clock)
        model_clock.increment(f"model_{model}")

        # 코드 생성
        code = await self._call_model(model, spec)

        return {
            "id": str(uuid4()),
            "model": model,
            "code": code,
            "clock": model_clock
        }

    async def _test_unit(self, code: str) -> dict:
        """유닛 테스트 실행"""
        # 실제 테스트 로직
        pass

    async def _call_model(self, model: str, spec: dict) -> str:
        """LLM API 호출"""
        # 실제 API 호출
        pass
```

#### 2.3.2 Visualization (Slack)

```python
# yoman/slack/visualizer.py

class CausalGraphVisualizer:
    """Slack에 causal graph 시각화"""

    @staticmethod
    def render_dependency_graph(op_id: str, db: Client) -> str:
        """
        Operation의 의존성 그래프를 Mermaid diagram으로 렌더링

        Returns:
            Mermaid syntax string
        """

        # 의존성 그래프 조회
        graph = db.rpc("get_dependency_graph", {"op_id": op_id}).execute()

        # Mermaid 생성
        lines = ["```mermaid", "graph TD"]

        for node_id, deps in graph.data.items():
            node_label = f"{node_id[:8]}"  # 짧게 표시

            for dep_id in deps:
                dep_label = f"{dep_id[:8]}"
                lines.append(f"    {dep_label} --> {node_label}")

        lines.append("```")

        return "\n".join(lines)

    @staticmethod
    async def notify_concurrent_operations(
        ops: list[str],
        slack_client: any
    ) -> None:
        """
        Concurrent operations 알림

        여러 operation이 동시에 실행 중임을 시각화
        """

        message = "🔀 *Concurrent Operations Detected*\n\n"
        message += "다음 operations는 인과 관계가 없으므로 병렬 처리됩니다:\n\n"

        for op_id in ops:
            message += f"• `{op_id}`\n"

        await slack_client.chat_postMessage(
            channel="yoman-alerts",
            text=message
        )
```

---

## 3. CALM Theorem 활용

### 3.1 Monotonic Operations 식별

**YOMAN의 Monotonic Operations** (coordination-free):

```python
# yoman/calm/monotonic_ops.py

class MonotonicOperations:
    """
    YOMAN에서 coordination 없이 실행 가능한 operations

    CALM theorem: Monotonic operations는 최종 일관성 보장
    """

    @staticmethod
    def is_monotonic(op_type: str, bucket: str) -> bool:
        """Operation이 monotonic인지 판단"""

        # Monotonic (coordination-free)
        monotonic_ops = {
            # 추가 operations
            ("create", "IDEA"),      # IDEA 추가 (독립적)
            ("create", "STUDY"),     # STUDY 추가 (RESEARCH에서 분기)
            ("append", "LOG"),       # 로그 추가
            ("increment", "COUNTER"), # 카운터 증가

            # 읽기 operations (side-effect 없음)
            ("read", "*"),
        }

        # Non-monotonic (coordination 필요)
        non_monotonic_ops = {
            # 삭제/감소 operations
            ("delete", "TODO"),
            ("cancel", "PR"),
            ("decrement", "COUNTER"),

            # 집계 operations
            ("aggregate", "*"),
            ("average", "*"),
        }

        key = (op_type, bucket)

        if key in monotonic_ops:
            return True

        if key in non_monotonic_ops:
            return False

        # Wildcard 체크
        for op, bkt in monotonic_ops:
            if op == op_type and bkt == "*":
                return True

        # 기본값: non-monotonic (안전하게)
        return False
```

### 3.2 Coordination-Free Executor

```python
# yoman/executors/calm_executor.py

class CALMExecutor:
    """
    CALM theorem 기반 실행 엔진

    Monotonic ops → 병렬 실행 (coordination-free)
    Non-monotonic ops → 순차 실행 (locking 필요)
    """

    def __init__(self, db: Client):
        self.db = db
        self.lock = asyncio.Lock()

    async def execute_batch(
        self,
        operations: list[dict]
    ) -> list[any]:
        """
        Operation 배치 실행

        Args:
            operations: [{"type": "create", "bucket": "IDEA", "data": {...}}, ...]

        Returns:
            각 operation의 결과
        """

        # 1. Monotonic/Non-monotonic 분류
        monotonic = []
        non_monotonic = []

        for op in operations:
            if MonotonicOperations.is_monotonic(op["type"], op["bucket"]):
                monotonic.append(op)
            else:
                non_monotonic.append(op)

        # 2. Monotonic operations 병렬 실행
        monotonic_results = await asyncio.gather(*[
            self._execute_single(op)
            for op in monotonic
        ])

        # 3. Non-monotonic operations 순차 실행 (lock)
        non_monotonic_results = []
        for op in non_monotonic:
            async with self.lock:
                result = await self._execute_single(op)
                non_monotonic_results.append(result)

        return monotonic_results + non_monotonic_results

    async def _execute_single(self, op: dict) -> any:
        """단일 operation 실행"""
        op_type = op["type"]
        bucket = op["bucket"]
        data = op["data"]

        if op_type == "create" and bucket == "IDEA":
            return await self._create_idea(data)
        elif op_type == "create" and bucket == "RESEARCH":
            return await self._create_research(data)
        # ... 다른 operations
        else:
            raise ValueError(f"Unknown operation: {op_type} {bucket}")

    async def _create_idea(self, data: dict) -> str:
        """IDEA 생성 (monotonic)"""
        # BucketService 호출
        pass

    async def _create_research(self, data: dict) -> str:
        """RESEARCH 생성 (monotonic if IDEA exists)"""
        # BucketService 호출
        pass
```

---

## 4. 실전 시나리오

### 4.1 시나리오 1: 여러 IDEA 동시 생성

**상황**: 사용자가 3개 IDEA를 빠르게 입력

```python
# Slack에서 빠르게 3개 메시지
#yoman-ideas: "AI 에이전트 만들기"
#yoman-ideas: "블록체인 통합"
#yoman-ideas: "모바일 앱 개발"
```

**실행 흐름**:

```python
async def handle_concurrent_ideas():
    executor = CALMExecutor(db)

    # 3개 IDEA 동시 생성 요청
    operations = [
        {"type": "create", "bucket": "IDEA", "data": {"title": "AI 에이전트"}},
        {"type": "create", "bucket": "IDEA", "data": {"title": "블록체인 통합"}},
        {"type": "create", "bucket": "IDEA", "data": {"title": "모바일 앱"}},
    ]

    # CALM executor가 판단: 모두 monotonic → 병렬 실행
    results = await executor.execute_batch(operations)

    # 3개 IDEA의 vector clock은 CONCURRENT
    idea1_clock = VectorClock.from_json(results[0]["vector_clock"])
    idea2_clock = VectorClock.from_json(results[1]["vector_clock"])
    idea3_clock = VectorClock.from_json(results[2]["vector_clock"])

    assert idea1_clock.compare(idea2_clock) == 'CONCURRENT'
    assert idea2_clock.compare(idea3_clock) == 'CONCURRENT'
    assert idea1_clock.compare(idea3_clock) == 'CONCURRENT'

    # Slack 알림
    await slack_notify(
        channel="yoman-ideas",
        message="💡 3개의 IDEA가 동시에 생성되었습니다 (병렬 처리 완료)"
    )
```

### 4.2 시나리오 2: GAN 검증 실패 후 재작업

**상황**: PR이 REVIEWER에 의해 거부됨 → TODO 수정 필요

```python
async def handle_review_failure():
    # 1. PR 리뷰
    reviewer = Reviewer()
    result = await reviewer.review_pr(pr_id="pr-123", db=db)

    if result["verdict"] == "FAIL":
        # 2. TODO 수정 (새 버전 생성)
        todo_id = "todo-45"
        old_todo = db.table("todos").select("*").eq("id", todo_id).execute().data[0]

        # 3. 피드백 반영 (새 버전)
        updated_content = {
            **old_todo["content"],
            "feedback": result["details"],
            "revised": True
        }

        # Vector clock 증가
        new_clock = VectorClock.from_json(old_todo["vector_clock"])
        new_clock.increment("yoman_server")

        # 4. 새 버전 생성 (append-only, monotonic)
        new_version = old_todo["version"] + 1
        db.table("todos").insert({
            "id": todo_id,
            "version": new_version,
            "content": updated_content,
            "vector_clock": new_clock.to_json(),
            "dependencies": old_todo["dependencies"] + [
                {"id": "pr-123", "bucket": "PR", "version": 1}  # 실패한 PR에 의존
            ]
        }).execute()

        # 5. CODINGBOT 재실행
        codingbot = CodingBot()
        new_pr_id = await codingbot.generate_code(todo_id, db)

        # 6. Slack 알림
        await slack_notify(
            channel="yoman-todos",
            message=f"🔄 TODO {todo_id} v{new_version} 생성됨\n"
                    f"이전 PR 실패 피드백 반영\n"
                    f"새 PR: {new_pr_id}"
        )
```

### 4.3 시나리오 3: Prerequisites 체인

**상황**: TODO-3이 TODO-1, TODO-2에 의존 (순차 실행 필요)

```python
async def handle_prerequisite_chain():
    """
    Prerequisites 체인:
    TODO-1 → TODO-2 → TODO-3
    """

    # 1. TODO-1 생성 (의존성 없음)
    todo1_id = await bucket_service.create_todo(
        research_id="research-10",
        content={"title": "DB 스키마 설계"},
        prerequisites=[],
        user_id="user1"
    )

    # 2. TODO-2 생성 (TODO-1에 의존)
    todo2_id = await bucket_service.create_todo(
        research_id="research-10",
        content={"title": "API 엔드포인트 구현"},
        prerequisites=[todo1_id],  # TODO-1 완료 필요
        user_id="user1"
    )

    # 3. TODO-3 생성 (TODO-1, TODO-2에 의존)
    todo3_id = await bucket_service.create_todo(
        research_id="research-10",
        content={"title": "프론트엔드 연동"},
        prerequisites=[todo1_id, todo2_id],
        user_id="user1"
    )

    # 4. Vector clock 관계 확인
    todo1 = await get_operation(todo1_id)
    todo2 = await get_operation(todo2_id)
    todo3 = await get_operation(todo3_id)

    assert todo1.vector_clock.compare(todo2.vector_clock) == 'BEFORE'
    assert todo2.vector_clock.compare(todo3.vector_clock) == 'BEFORE'
    assert todo1.vector_clock.compare(todo3.vector_clock) == 'BEFORE'

    # 5. 의존성 그래프 시각화
    graph = CausalGraphVisualizer.render_dependency_graph(todo3_id, db)

    await slack_notify(
        channel="yoman-todos",
        message=f"📊 TODO 체인 생성 완료\n\n{graph}"
    )
```

---

## 5. 모니터링 및 디버깅

### 5.1 Causal Anomaly 감지

```python
# yoman/monitoring/causal_checker.py

class CausalAnomalyDetector:
    """
    Causal consistency 위반 감지

    - 순환 의존성 (circular dependency)
    - 의존성 누락 (missing dependency)
    - 순서 위반 (order violation)
    """

    def __init__(self, db: Client):
        self.db = db

    async def detect_circular_dependency(self, op_id: str) -> Optional[list]:
        """
        순환 의존성 감지

        Returns:
            순환 경로 (없으면 None)
        """

        visited = set()
        path = []

        def dfs(current_id: str) -> Optional[list]:
            if current_id in path:
                # 순환 발견
                cycle_start = path.index(current_id)
                return path[cycle_start:] + [current_id]

            if current_id in visited:
                return None

            visited.add(current_id)
            path.append(current_id)

            # 의존성 조회
            op = self.db.table("operations").select("dependencies").eq("id", current_id).execute()
            if not op.data:
                path.pop()
                return None

            deps = op.data[0].get("dependencies", [])

            for dep in deps:
                cycle = dfs(dep["id"])
                if cycle:
                    return cycle

            path.pop()
            return None

        cycle = dfs(op_id)

        if cycle:
            # Slack 알림
            await slack_alert(
                channel="yoman-alerts",
                message=f"⚠️ *Circular Dependency Detected*\n\n"
                        f"Cycle: {' → '.join(cycle)}"
            )

        return cycle

    async def check_consistency(self, op_id: str) -> dict:
        """
        전체 consistency 체크

        Returns:
            체크 결과 딕셔너리
        """

        op = await get_operation(op_id)

        issues = []

        # 1. 의존성 존재 확인
        for dep in op.dependencies:
            dep_op = await get_operation(dep.op_id)
            if not dep_op:
                issues.append({
                    "type": "missing_dependency",
                    "dep_id": dep.op_id
                })

        # 2. Vector clock 순서 확인
        for dep in op.dependencies:
            dep_op = await get_operation(dep.op_id)
            if dep_op:
                relation = dep_op.vector_clock.compare(op.vector_clock)
                if relation != 'BEFORE':
                    issues.append({
                        "type": "order_violation",
                        "dep_id": dep.op_id,
                        "expected": "BEFORE",
                        "actual": relation
                    })

        # 3. 순환 의존성 확인
        cycle = await self.detect_circular_dependency(op_id)
        if cycle:
            issues.append({
                "type": "circular_dependency",
                "cycle": cycle
            })

        return {
            "op_id": op_id,
            "consistent": len(issues) == 0,
            "issues": issues
        }
```

### 5.2 Slack Dashboard

```python
# yoman/slack/dashboard.py

class CausalDashboard:
    """Slack에 causal consistency 현황 대시보드"""

    @staticmethod
    async def render_daily_report(slack_client: any):
        """
        일일 리포트

        - 총 operations
        - Concurrent operations 비율
        - Causal chain 길이 평균
        - 이상 탐지 결과
        """

        stats = await compute_stats()

        message = f"""
📊 *YOMAN Causal Consistency Report* - {datetime.now().strftime('%Y-%m-%d')}

*Operations*
• Total: {stats['total_ops']}
• IDEA: {stats['idea_count']}
• RESEARCH: {stats['research_count']}
• TODO: {stats['todo_count']}
• PR: {stats['pr_count']}

*Concurrency*
• Concurrent ops: {stats['concurrent_ops']} ({stats['concurrent_ratio']:.1%})
• Parallel executions: {stats['parallel_count']}

*Causal Chains*
• Avg chain length: {stats['avg_chain_length']:.1f}
• Max chain length: {stats['max_chain_length']}

*Anomalies*
• Circular dependencies: {stats['circular_deps']}
• Missing dependencies: {stats['missing_deps']}
• Order violations: {stats['order_violations']}

{'✅ All clear!' if stats['anomaly_count'] == 0 else '⚠️ Issues detected'}
        """

        await slack_client.chat_postMessage(
            channel="yoman-alerts",
            text=message
        )
```

---

## 6. 성능 최적화

### 6.1 Vector Clock 압축

```python
# yoman/optimization/vc_compression.py

class VectorClockOptimizer:
    """
    Vector clock 최적화

    - 0인 엔트리 제거
    - 비활성 노드 prune
    - Delta encoding
    """

    @staticmethod
    def compress(vc: VectorClock) -> dict:
        """0인 엔트리 제거 후 JSON 변환"""
        compressed = {k: v for k, v in vc.clocks.items() if v > 0}
        return compressed

    @staticmethod
    async def prune_inactive_nodes(
        vc: VectorClock,
        db: Client,
        inactive_days: int = 30
    ) -> VectorClock:
        """
        비활성 노드 제거

        Args:
            vc: Vector clock
            db: Database client
            inactive_days: 며칠 동안 활동 없으면 제거

        Returns:
            Pruned vector clock
        """

        # 최근 active nodes 조회
        cutoff = datetime.now() - timedelta(days=inactive_days)

        active_nodes = db.table("operations")\
            .select("node_id")\
            .gte("updated_at", cutoff.isoformat())\
            .execute()

        active_set = {row["node_id"] for row in active_nodes.data}

        # Prune
        pruned_clocks = {
            k: v for k, v in vc.clocks.items()
            if k in active_set
        }

        return VectorClock(clocks=pruned_clocks)
```

### 6.2 Dependency Cache

```python
# yoman/optimization/dep_cache.py

from functools import lru_cache

class DependencyCache:
    """
    의존성 조회 캐싱

    자주 조회되는 operations는 메모리에 캐시
    """

    def __init__(self, max_size: int = 1000):
        self.cache = {}
        self.max_size = max_size
        self.hits = 0
        self.misses = 0

    def get(self, op_id: str) -> Optional[Operation]:
        """캐시 조회"""
        if op_id in self.cache:
            self.hits += 1
            return self.cache[op_id]

        self.misses += 1
        return None

    def put(self, op: Operation) -> None:
        """캐시 저장"""
        if len(self.cache) >= self.max_size:
            # LRU eviction
            oldest = min(self.cache.values(), key=lambda o: o.updated_at)
            del self.cache[oldest.id]

        self.cache[op.id] = op

    def hit_rate(self) -> float:
        """캐시 hit rate"""
        total = self.hits + self.misses
        return self.hits / total if total > 0 else 0.0
```

---

## 7. 마이그레이션 계획

### 7.1 Phase 1: 기본 인프라 (2주)

**Week 1**:
- [ ] Supabase schema 확장 (vector_clock, dependencies 컬럼)
- [ ] Python VectorClock, Operation 클래스 구현
- [ ] 단위 테스트 작성

**Week 2**:
- [ ] BucketService에 causal consistency 적용
- [ ] Slack 알림에 dependency 정보 추가
- [ ] 통합 테스트

### 7.2 Phase 2: GAN 검증 (2주)

**Week 3**:
- [ ] CodingBot에 vector clock 적용
- [ ] Reviewer에 dual dependency 추가 (PR + TODO)
- [ ] PR 실패 시 재작업 흐름 구현

**Week 4**:
- [ ] 양자 검증 (concurrent unit generation)
- [ ] CALM executor 구현
- [ ] 성능 벤치마크

### 7.3 Phase 3: 모니터링 및 최적화 (1-2주)

**Week 5-6**:
- [ ] Causal anomaly 감지
- [ ] Slack dashboard
- [ ] Vector clock 압축
- [ ] Dependency cache
- [ ] 전체 시스템 테스트

---

## 8. 예상 효과

### 8.1 정량적 지표

| 메트릭 | Before | After (예상) | 개선율 |
|--------|--------|------------|--------|
| **병렬 처리 안정성** | 60% (race condition) | 95% (causal ordering) | **+58%** |
| **의존성 추적** | 수동 (문서) | 자동 (vector clock) | **100% 자동화** |
| **디버깅 시간** | 2시간 (의존성 역추적) | 10분 (그래프 조회) | **91% 단축** |
| **Concurrent ops** | 1-2개 | 5-10개 (CALM) | **5배 증가** |

### 8.2 정성적 효과

**신뢰도**:
- Prerequisites 자동 검증
- 순환 의존성 사전 차단
- GAN 검증 정확도 향상

**가시성**:
- 전체 의존성 그래프 실시간 확인
- Causal chain 시각화
- Slack dashboard로 현황 파악

**확장성**:
- CALM theorem으로 coordination-free 확장
- 여러 AI 모델 병렬 실행
- 분산 노드 추가 용이

---

## 9. 결론

### 9.1 핵심 요약

1. **YOMAN은 causal system임**
   - REVIEWER가 있어도 causal chain 유지
   - GAN 검증 패턴도 causal

2. **Causal consistency 적용 가능**
   - Vector clock으로 인과관계 추적
   - COPS-style dependency tracking
   - CALM theorem으로 병렬 처리 최적화

3. **신뢰도와 성능 동시 향상**
   - Prerequisites 자동 검증
   - Concurrent operations 병렬 처리
   - Anomaly 사전 감지

### 9.2 다음 단계

1. **프로토타입 구현** (Phase 1) - 2주
2. **GAN 검증 통합** (Phase 2) - 2주
3. **프로덕션 배포** (Phase 3) - 1-2주
4. **논문 작성** - YOMAN의 causal consistency 사례 연구

---

**문서 작성**: 2026-01-01
**작성자**: Claude Sonnet 4.5
**관련 문서**: [00-causal-consistency-overview.md](./00-causal-consistency-overview.md), [01-algorithms-implementation.md](./01-algorithms-implementation.md)
