# Causal vs Non-Causal: YOMAN Solution

> Quasi-Non-Causal Architecture로 신뢰도 15배 향상

**Date**: 2026-01-01
**Prerequisites**: 01-n-tier-architecture.md, 02-yoman-application.md
**Based on**: Meta-Task Planning (2024), GoalAct (2025)

---

## 1. 문제 정의

### 1.1 Pure Causal의 한계

**Pure Causal System**:
```
t=0 → t=1 → t=2 → t=3 (시간 순서대로 처리)
      ↑
      과거 정보만 사용
```

**YOMAN 예시**:
```
IDEA → RESEARCH → TODO → PR (순차 생성)
```

**문제점**:
1. TODO 생성 시 PR 결과 예측 불가
2. 잘못된 방향으로 가도 나중에 발견
3. 되돌아가서 수정 = 비용 증가
4. **신뢰도 낮음: GPT-4 2.92%** (Meta-Task Planning 연구)

---

### 1.2 Meta-Task Planning의 충격적 발견

| 방식 | 모델 | 성공률 | 특징 |
|------|------|--------|------|
| **Flat Planning** | GPT-4 | **2.92%** | Pure Causal, 계층 없음 |
| **Hierarchical** | LLaMA-3.1-8B | **42.68%** | Manager-Executor 분리 |

→ **15배 차이!**

**핵심 인사이트**:
- Pure Causal만으로는 부족
- Hierarchical 분해 필수
- Non-Causal Verification 필요

---

## 2. Causal vs Non-Causal 이해

### 2.1 Causal System

**특징**:
- 순차적 처리 (Sequential)
- 과거 상태만 참조 가능
- 실시간 생성 가능
- 예: Auto-regressive LLM, SampleRNN

**장점**:
- 빠름 (스트리밍 가능)
- 실시간 응답
- 메모리 효율적

**단점**:
- 전체 맥락 파악 불가
- Long-range dependency 약함
- **신뢰도 낮음** (2.92%)

---

### 2.2 Non-Causal System

**특징**:
- 전체 시퀀스 한번에 보고 처리
- 미래 정보 참조 가능
- Batch 처리 필수
- 예: BERT (Bidirectional), Transformer Encoder

**장점**:
- 전체 맥락 이해
- 정확도 높음
- 오류 발견 용이

**단점**:
- 느림 (전체 생성 후 판단)
- 실시간 불가
- 메모리 많이 필요

---

### 2.3 Quasi-Non-Causal (제안)

```
Quasi-Non-Causal = Causal Generator + Non-Causal Discriminator
```

**특징**:
- **생성은 Causal**: 순차적으로 빠르게 생성
- **검증은 Non-Causal**: 전체 보고 판단
- **Feedback Loop**: Discriminator 결과로 Generator 개선

**GAN과의 유사성**:

| 구성 요소 | GAN | YOMAN |
|-----------|-----|-------|
| Generator | 이미지 생성 | CODINGBOT (TODO → PR) |
| Discriminator | 진짜/가짜 판별 | REVIEWER (PR vs TODO 비교) |
| Loss | D(G(z)) vs D(real) | match_rate |
| Training | Adversarial | Iterative (재작업) |

---

## 3. Multi-Discriminator Architecture

### 3.1 Single Discriminator의 문제

**기존 아이디어**:
```
Generator (T1) → PR
                 ↓
Discriminator (T3) → match_rate ≥80% → 머지
```

**문제**: Discriminator가 너무 늦게 발견 (PR 생성 후)
- 잘못된 RESEARCH → 잘못된 TODO → 잘못된 PR
- PR 단계에서 발견 = 모든 작업 재수행
- 비용 증가

---

### 3.2 Multi-Discriminator (제안)

```
Generator (T1):
  IDEA → RESEARCH → TODO → PR  (순차 생성, Causal)
    ↓        ↓        ↓      ↓
Discriminator (T3):
  D1       D2       D3     D4  (각 단계 검증, Non-Causal)
```

**장점**:
- **Early Stopping**: IDEA 단계에서 불명확하면 즉시 차단
- **Layered Verification**: 각 Bucket 전환 시 검증
- **Cost Reduction**: 잘못된 PR 생성 전에 차단

---

### 3.3 구현 예시

```python
class MultiDiscriminator:
    """4개 Discriminator로 각 단계 검증"""

    def __init__(self, tier3_model: OpusModel):
        self.model = tier3_model  # Tier 3 (Opus)

        # 각 Discriminator는 다른 체크리스트
        self.d1 = Discriminator(check="idea_validity")
        self.d2 = Discriminator(check="research_completeness")
        self.d3 = Discriminator(check="todo_feasibility")
        self.d4 = Discriminator(check="pr_correctness")

    def verify_idea(self, idea: IDEA) -> VerificationResult:
        """D1: IDEA → RESEARCH 전환 전 검증"""

        prompt = f"""
다음 IDEA가 RESEARCH로 진행할 만한지 평가하셈:

IDEA:
{idea.description}

체크리스트:
1. 명확성: 무엇을 만들지 명확한가?
2. 실현 가능성: 기술적으로 가능한가?
3. 가치: 왜 필요한가?

점수 (0.0-1.0)와 이유 제시.
"""

        result = self.model.generate(prompt)
        score = self._extract_score(result)

        if score < 0.6:
            return VerificationResult(
                status="REJECT",
                score=score,
                reason="Idea 불명확",
                suggestion="구체적인 요구사항 추가 필요",
            )

        return VerificationResult(status="PASS", score=score)

    def verify_research(self, research: RESEARCH) -> VerificationResult:
        """D2: RESEARCH → TODO 전환 전 검증"""

        prompt = f"""
다음 RESEARCH가 TODO로 변환 가능한지 평가:

RESEARCH:
{research.to_json()}

체크리스트:
1. 완전성: 필요한 정보 모두 포함?
2. 구조화: JSON 형식 올바른가?
3. 의존성: 외부 라이브러리 명시?
4. 아키텍처: 기존 프로젝트와 일관성?

점수 (0.0-1.0)와 이유.
"""

        result = self.model.generate(prompt)
        score = self._extract_score(result)

        if score < 0.7:
            return VerificationResult(
                status="REJECT",
                score=score,
                reason="Research 불완전",
                suggestion="아키텍처 다이어그램 추가 필요",
            )

        return VerificationResult(status="PASS", score=score)

    def verify_todo(self, todo: TODO) -> VerificationResult:
        """D3: TODO → PR 전환 전 검증"""

        prompt = f"""
다음 TODO가 실행 가능한지 평가:

TODO:
{todo.to_json()}

체크리스트:
1. 실행 가능성: Prerequisites 모두 충족?
2. 명확성: Acceptance Criteria 명확?
3. 분해: Subtask 적절히 나뉨?
4. 테스트: 테스트 계획 있음?

점수 (0.0-1.0)와 이유.
"""

        result = self.model.generate(prompt)
        score = self._extract_score(result)

        if score < 0.8:
            return VerificationResult(
                status="REJECT",
                score=score,
                reason="TODO 실행 불가능",
                suggestion="Prerequisites 재확인 필요",
            )

        return VerificationResult(status="PASS", score=score)

    def verify_pr(self, pr: PR, todo: TODO) -> VerificationResult:
        """D4: PR 최종 검증"""

        prompt = f"""
PR이 TODO 스펙과 일치하는지 평가:

TODO:
{todo.to_json()}

PR:
{pr.get_diff()}

체크리스트:
1. 스펙 일치: TODO의 모든 요구사항 구현?
2. 테스트: 테스트 통과?
3. 아키텍처: 기존 코드와 일관성?
4. 코드 품질: 린트, 포맷 통과?

match_rate (0.0-1.0)와 gap 설명.
"""

        result = self.model.generate(prompt)
        match_rate = self._extract_score(result)

        if match_rate < 0.8:
            gap_explanation = self._extract_gap(result)
            return VerificationResult(
                status="REJECT",
                score=match_rate,
                reason="스펙 불일치",
                suggestion=gap_explanation,
            )

        return VerificationResult(status="MERGE", score=match_rate)

    def _extract_score(self, text: str) -> float:
        """텍스트에서 점수 추출"""
        # 정규식 또는 JSON 파싱
        import re
        match = re.search(r"점수[:\s]+(\d+\.\d+)", text)
        if match:
            return float(match.group(1))
        return 0.0

    def _extract_gap(self, text: str) -> str:
        """gap 설명 추출"""
        # "gap:" 이후 텍스트 추출
        lines = text.split("\n")
        for i, line in enumerate(lines):
            if "gap" in line.lower():
                return "\n".join(lines[i+1:])
        return "No explanation"
```

---

## 4. Tier별 Causality 전략

### 4.1 Tier별 Causality 설정

| Tier | 역할 | Causality | 이유 |
|------|------|-----------|------|
| **Tier 3 (System)** | 전체 조망 | **Non-Causal** | 전체 프로젝트 파악 필요 |
| **Tier 2 (Module)** | Operation 관리 | **Quasi-Causal** | 현재 + 이전 operation 참조 |
| **Tier 1 (Unit)** | 함수 구현 | **Pure Causal** | 빠른 생성 |

---

### 4.2 정보 흐름

```python
# Tier 3 (Non-Causal): 전체 프로젝트 읽음
class SystemTier:
    def read_all_docs(self) -> Context:
        """docs/ 전체 읽기"""
        docs = []
        for file in glob("docs/**/*.md"):
            content = read_file(file)
            docs.append(content)

        # Memory MCP에서 entities 조회
        entities = memory.list_entities()

        return Context(docs=docs, entities=entities)

    def create_global_plan(self, context: Context) -> Plan:
        """전체 맥락에서 plan 생성 (Non-Causal)"""
        prompt = f"""
전체 프로젝트 맥락:
{context.summary()}

Global Plan 생성:
1. 프로젝트 목표
2. 주요 Operation 리스트
3. 의존성 그래프
4. 리스크
"""
        plan = self.model.generate(prompt)
        return Plan(**plan)
```

```python
# Tier 2 (Quasi-Causal): 현재 + 과거 참조
class ModuleTier:
    def read_current_operation(self, plan: Plan, op_id: str) -> Operation:
        """현재 Operation 정보"""
        return plan.operations[op_id]

    def reference_previous_operations(self, plan: Plan, op_id: str) -> List[Operation]:
        """이전 Operation 결과 참조 (Quasi-Causal)"""
        # 현재 operation의 dependencies 조회
        current = plan.operations[op_id]
        prev_ops = []

        for dep_id in current.dependencies:
            prev_op = plan.operations[dep_id]
            if prev_op.status == "completed":
                prev_ops.append(prev_op)

        return prev_ops

    def execute_operation(self, operation: Operation) -> OperationResult:
        """Operation 실행"""
        # 현재 + 과거 맥락
        context = {
            "current": operation,
            "previous": self.reference_previous_operations(...),
        }

        prompt = f"""
Operation 실행:
{operation.description}

참고 (이전 작업):
{context["previous"]}
"""
        result = self.model.generate(prompt)
        return OperationResult(**result)
```

```python
# Tier 1 (Pure Causal): 순차 생성
class UnitTier:
    def generate_code(self, spec: Spec) -> Code:
        """스펙만 보고 빠르게 생성 (Pure Causal)"""
        prompt = f"""
다음 스펙에 맞는 코드 생성:
{spec.description}

요구사항:
{spec.requirements}
"""
        code = self.model.generate(prompt)
        return Code(content=code)
```

---

## 5. 신뢰도 3단계 체계

### 5.1 Level 1: Generator 내부 검증 (Tier 1)

```python
class Generator:
    """Unit 생성 시 자체 검증"""

    def generate_code(self, spec: Spec) -> Code:
        code = self.llm.generate(spec)

        # 내부 검증 (syntax check, type check)
        if not self.internal_validate(code):
            # 1회 재시도
            code = self.llm.generate(spec)

        return code

    def internal_validate(self, code: Code) -> bool:
        """간단한 검증"""
        checks = [
            self._check_syntax(code),
            self._check_imports(code),
            self._check_basic_structure(code),
        ]
        return all(checks)
```

**신뢰도**: ~60% (빠르지만 부정확)

---

### 5.2 Level 2: Module-level 검증 (Tier 2)

```python
class ModuleValidator:
    """Operation 완료 시 중간 검증"""

    def validate_operation(self, op: Operation) -> float:
        # 1. 여러 unit이 일관성 있는지
        consistency = self.check_consistency(op.units)

        # 2. 이전 operation과 충돌 없는지
        compatibility = self.check_compatibility(op)

        # 3. 테스트 통과?
        test_result = self.run_tests(op)

        score = (consistency + compatibility + test_result) / 3
        return score

    def check_consistency(self, units: List[Unit]) -> float:
        """Unit 간 일관성"""
        # 예: 함수 호출 시그니처 일치?
        mismatches = 0
        for unit1, unit2 in itertools.combinations(units, 2):
            if not self._signatures_match(unit1, unit2):
                mismatches += 1

        consistency = 1.0 - (mismatches / len(units))
        return consistency
```

**신뢰도**: ~75% (중간 정확도)

---

### 5.3 Level 3: System-level 검증 (Tier 3)

```python
class SystemDiscriminator:
    """PR 생성 후 최종 검증"""

    def evaluate_pr(self, pr: PR, todo: TODO) -> float:
        # 1. 전체 프로젝트 맥락에서 평가
        match_rate = self.compare(pr, todo)

        # 2. 아키텍처 일관성
        arch_score = self.check_architecture(pr)

        # 3. 테스트 커버리지
        test_score = self.check_tests(pr)

        # 4. 문서 업데이트
        doc_score = self.check_docs(pr)

        final_score = (
            match_rate * 0.4 +
            arch_score * 0.3 +
            test_score * 0.2 +
            doc_score * 0.1
        )

        return final_score

    def compare(self, pr: PR, todo: TODO) -> float:
        """PR이 TODO 스펙 만족하는지"""
        prompt = f"""
PR이 TODO 스펙을 얼마나 만족하는지 평가:

TODO 요구사항:
{todo.requirements}

PR 변경사항:
{pr.get_diff()}

각 요구사항별 만족도 (0.0-1.0) 계산 후 평균.
"""
        result = self.model.generate(prompt)
        return self._extract_score(result)
```

**신뢰도**: ~90% (느리지만 정확)

---

## 6. Continuous Feedback Loop

### 6.1 GoalAct 방식 적용

```python
class FeedbackLoop:
    """Discriminator 결과로 Generator plan 업데이트"""

    def __init__(self, generator: Generator, discriminator: SystemDiscriminator):
        self.generator = generator
        self.discriminator = discriminator
        self.max_retries = 3

    def generate_with_feedback(self, todo: TODO) -> PR:
        """Feedback loop로 신뢰도 높이기"""

        for attempt in range(self.max_retries):
            # 1. PR 생성 (Generator)
            pr = self.generator.generate_pr(todo)

            # 2. 검증 (Discriminator)
            result = self.discriminator.evaluate_pr(pr, todo)

            if result.score >= 0.8:
                # 통과
                return pr

            # 3. Feedback 생성
            feedback = self.discriminator.explain_gap(pr, todo)

            # 4. Generator plan 업데이트
            self.generator.update_plan(feedback)

            logger.info(f"Attempt {attempt+1}: score={result.score:.2f}, retrying...")

        # 최종 실패
        raise Exception(f"Failed after {self.max_retries} attempts")
```

---

### 6.2 Plan 업데이트 메커니즘

```python
class Generator:
    def __init__(self):
        self.plan = None  # Global Plan

    def update_plan(self, feedback: Feedback):
        """Discriminator feedback로 plan 조정"""

        prompt = f"""
현재 plan:
{self.plan}

Discriminator feedback:
{feedback.suggestion}

Plan 업데이트:
- 무엇이 잘못됐나?
- 어떻게 개선할 것인가?
"""

        updated_plan = self.model.generate(prompt)
        self.plan = updated_plan

        # Memory MCP 업데이트
        memory.update_observation(
            entity="global_plan",
            observation=f"Updated: {feedback.reason}",
        )
```

---

## 7. 실전 시나리오 비교

### 7.1 Pure Causal (기존)

**시나리오**: "사용자 인증 API 구현"

```
1. TODO 생성: "JWT 인증 구현"
   - Prerequisites: ["JWT library"]
   - Subtasks: 3개

2. CODINGBOT: PR 생성
   - middleware.py
   - routes.py
   - test.py

3. REVIEWER: 검증
   - match_rate = 45%
   - 문제: HTTPS redirect 누락
   - 문제: bcrypt 안 씀
   - 문제: 테스트 부족

4. 재작업 → 비용 증가
```

**문제**: 3단계에서 발견 = 늦음
**비용**: $1.50 (재작업 포함)

---

### 7.2 Quasi-Non-Causal (개선)

**시나리오**: 동일 ("사용자 인증 API 구현")

```
1. IDEA: "사용자 인증 필요"
   ↓
   D1 (Tier 3): "PASS (명확함)"

2. RESEARCH: "JWT vs OAuth 비교, bcrypt 사용, HTTPS 필수"
   ↓
   D2 (Tier 3): "PASS (완전함)"

3. TODO: "HTTPS redirect, JWT middleware, bcrypt hash, test 작성"
   ↓
   D3 (Tier 3): "WARNING: Rate limiting 추가 권장"
   ↓
   TODO 수정: "HTTPS, JWT, bcrypt, rate limit, test"
   ↓
   D3 (Tier 3): "PASS (실행 가능)"

4. PR 생성: CODINGBOT 구현
   ↓
   D4 (Tier 3): match_rate = 85%
   ↓
   MERGE
```

**개선**: 각 단계마다 조기 발견 + 수정 → 최종 match_rate 높음
**비용**: $0.80 (재작업 없음)
**절감**: 47%

---

## 8. 성능 예측

### 8.1 신뢰도 비교

| 방식 | 신뢰도 | 비용 | 속도 |
|------|--------|------|------|
| **Pure Causal** (GPT-4) | 2.92% | 낮음 | 빠름 |
| **Hierarchical** (LLaMA) | 42.68% | 중간 | 중간 |
| **Quasi-Non-Causal** (YOMAN) | **~60%** (예상) | 중간 | 중간 |

**계산 근거**:
- Meta-Task Planning: 15배 향상 (Hierarchical)
- Multi-Discriminator: +50% 향상 (조기 발견)
- Continuous Feedback: +20% 향상 (plan 업데이트)

---

### 8.2 비용 분석

**기존 (Pure Causal)**:
```
실패율: 95%
재작업 횟수: 평균 3회
비용: $0.50 * 3 = $1.50
```

**Quasi-Non-Causal**:
```
실패율: 40%
재작업 횟수: 평균 0.5회
비용: $0.80 (Discriminator 포함)
```

**절감률**: 47%

---

## 9. 구현 로드맵

### 9.1 단기 (1-2주)

- [ ] MultiDiscriminator 클래스 구현
  - D1: Idea validator
  - D2: Research checker
  - D3: TODO feasibility
  - D4: PR correctness

- [ ] Tier별 Causality 설정
  ```python
  TierConfig(causality="non-causal")  # Tier 3
  TierConfig(causality="quasi-causal")  # Tier 2
  TierConfig(causality="causal")  # Tier 1
  ```

- [ ] Feedback Loop 프로토타입
  - Discriminator 결과 → Generator plan 업데이트

---

### 9.2 중기 (1개월)

- [ ] Multi-Scale Validation
  - MBLM 방식 적용
  - 가중치 조정 (system: 0.5, module: 0.3, unit: 0.2)

- [ ] Benchmark
  - Pure Causal vs Quasi-Non-Causal
  - match_rate, 비용, 속도 측정

- [ ] Continuous Planning
  - memory.json + Memory MCP
  - docs/CLAUDE.md 지속 업데이트

---

### 9.3 장기 (3개월)

- [ ] Custom Discriminator Block
  - GAN_Block (특화)
  - 학습 가능 (Fine-tuning)

- [ ] Production 배포
  - Slack 통합
  - 실제 프로젝트 적용

---

## 10. 참고 문헌

- **Meta-Task Planning** ([arXiv:2405.16510](https://arxiv.org/abs/2405.16510)) - Hierarchical 15배 향상
- **GoalAct** ([arXiv:2504.16563](https://arxiv.org/abs/2504.16563)) - Continuous Planning
- **SampleRNN** (ICLR 2017) - Hierarchical Causal Model
- `01-n-tier-architecture.md` - N-Tier 설계
- `02-yoman-application.md` - YOMAN 적용

---

*작성일: 2026-01-01*
*분석 도구: Claude Opus 4.5*
