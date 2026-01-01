# N-Tier Dynamic Architecture Design

> YOMAN을 위한 동적 계층 확장 아키텍처 설계

**Date**: 2026-01-01
**Based on**: MBLM (2025), Meta-Task Planning (2024), GoalAct (2025)

---

## 1. 개요

### 1.1 문제 정의

**기존 YOMAN 설계의 한계**:
```
System (Tier 3) - Opus    [고정]
Module (Tier 2) - Sonnet  [고정]
Unit (Tier 1)   - Flash   [고정]
```

❌ 간단한 프로젝트에도 Opus 호출 → 비용 낭비
❌ 복잡한 프로젝트에서 Module-Unit gap 너무 큼
❌ 프로젝트 특성에 맞춘 최적화 불가능

### 1.2 해결 방안

**MBLM (2025) 방식 적용**:
```python
MBLMModelConfig(
    hidden_dims=[1024, 1024, 512],  # 배열 길이 = Stage 수 (N개 가능!)
    seq_lens=[4096, 256, 16],       # 각 Stage 시퀀스 길이
    block=[MambaBlock, TransformerBlock, CustomBlock]  # 믹싱 가능
)
```

→ **계층 수는 프로젝트 복잡도의 함수**

---

## 2. N-Tier 설계 원칙

### 2.1 기본 원칙

**원칙 1: Tier 수는 동적**
```
Simple (2-tier): CLI tool, Script
Medium (3-tier): Web app, API server
Complex (4-tier): Microservices
Enterprise (5-tier): Multi-region, Cross-project
```

**원칙 2: 각 Tier는 독립 설정 가능**
```python
Tier(
    name="Module",
    model="Sonnet",
    context=10000,
    temperature=0.3,
    max_complexity=100,  # 이 tier가 처리 가능한 최대 복잡도
)
```

**원칙 3: Tier 간 비율 유지**
- Context 크기: 5-10배씩 감소
- 호출 빈도: 10배씩 증가
- 처리 시간: 5배씩 감소

---

### 2.2 YOMANConfig 스키마

```python
from dataclasses import dataclass
from typing import List, Literal

@dataclass
class TierConfig:
    """단일 Tier 설정"""
    name: str                          # "System", "Module", "Unit" 등
    model: str                         # "Opus", "Sonnet", "Flash", "GLM"
    context: int                       # 최대 context 크기
    temperature: float = 0.3           # 생성 온도
    max_complexity: int = 100          # 처리 가능 최대 복잡도
    causality: Literal["causal", "quasi-causal", "non-causal"] = "causal"

@dataclass
class YOMANConfig:
    """전체 YOMAN 설정"""
    tiers: List[TierConfig]            # N개 tier 정의

    # 복잡도 측정 가중치
    complexity_weights: dict = None

    # 자동 tier 조정 활성화
    auto_adjust: bool = True

    def __post_init__(self):
        if self.complexity_weights is None:
            self.complexity_weights = {
                "files": 0.3,
                "loc": 0.4,
                "dependencies": 0.2,
                "team_size": 0.1,
            }

        # Tier는 complexity 오름차순으로 정렬되어야 함
        self.tiers.sort(key=lambda t: t.max_complexity)
```

---

## 3. Tier 설정 전략

### 3.1 프로젝트 복잡도별 Tier 구성

#### Case 1: Simple Project (2-tier)

**대상**: CLI tool, Script, Single file app

```python
YOMANConfig(
    tiers=[
        TierConfig(
            name="App",
            model="Sonnet",
            context=10000,
            max_complexity=50,
            causality="quasi-causal",
        ),
        TierConfig(
            name="Command",
            model="Flash",
            context=2000,
            max_complexity=20,
            causality="causal",
        ),
    ]
)
```

**특징**:
- Opus 생략 → **70% 비용 절감**
- Flash 위주 → **3배 속도**
- 2-tier로 충분 (복잡도 < 50)

---

#### Case 2: Medium Project (3-tier)

**대상**: Web app, API server, Mobile app

```python
YOMANConfig(
    tiers=[
        TierConfig(
            name="Project",
            model="Opus",
            context=50000,
            max_complexity=200,
            causality="non-causal",
        ),
        TierConfig(
            name="Module",
            model="Sonnet",
            context=10000,
            max_complexity=100,
            causality="quasi-causal",
        ),
        TierConfig(
            name="Unit",
            model="Flash",
            context=2000,
            max_complexity=20,
            causality="causal",
        ),
    ]
)
```

**특징**:
- 기존 YOMAN 설계와 동일
- Opus는 전체 조망 (하루 1-2회)
- Sonnet은 Operation 관리 (시간당 1회)
- Flash는 빠른 구현 (분당 수회)

---

#### Case 3: Complex Project (4-tier)

**대상**: Microservices, Distributed system

```python
YOMANConfig(
    tiers=[
        TierConfig(
            name="System",
            model="Opus",
            context=100000,
            max_complexity=500,
            causality="non-causal",
        ),
        TierConfig(
            name="Service",
            model="Sonnet",
            context=50000,
            max_complexity=200,
            causality="quasi-causal",
        ),
        TierConfig(
            name="API",
            model="Sonnet",
            context=10000,
            max_complexity=100,
            causality="quasi-causal",
        ),
        TierConfig(
            name="Logic",
            model="Flash",
            context=5000,
            max_complexity=50,
            causality="causal",
        ),
    ]
)
```

**특징**:
- Service tier 추가 (서비스 간 조율)
- API tier (Endpoint 관리)
- 하위 tier (90% 트래픽)는 Flash/GLM

---

#### Case 4: Enterprise (5-tier)

**대상**: Multi-region, Cross-project

```python
YOMANConfig(
    tiers=[
        TierConfig(
            name="Ecosystem",
            model="Opus",
            context=200000,
            max_complexity=1000,
            causality="non-causal",
        ),
        TierConfig(
            name="Project",
            model="Opus",
            context=100000,
            max_complexity=500,
            causality="non-causal",
        ),
        TierConfig(
            name="Service",
            model="Sonnet",
            context=50000,
            max_complexity=200,
            causality="quasi-causal",
        ),
        TierConfig(
            name="Module",
            model="Sonnet",
            context=10000,
            max_complexity=100,
            causality="quasi-causal",
        ),
        TierConfig(
            name="Unit",
            model="Flash",
            context=2000,
            max_complexity=20,
            causality="causal",
        ),
    ]
)
```

**특징**:
- Ecosystem tier (멀티 프로젝트 조율)
- 주 1회 호출
- Cross-project dependency tracking

---

### 3.2 Block Type Mixing

**MBLM 아이디어**: 각 Stage마다 다른 Block 타입

```python
# MBLM 예시
block=[
    MambaBlock(d_state=128),        # 빠른 처리
    TransformerBlock(heads=16),     # 정확한 이해
]
```

**YOMAN 적용**:

| Tier | Block Type | 특성 | 예시 모델 |
|------|-----------|------|-----------|
| 최상위 | Transformer | 정확성, 추론 | Opus, GPT-4 |
| 중간 | Hybrid | 균형 | Sonnet, Gemini Pro |
| 하위 | Mamba/SSM | 속도, 효율 | Flash, GLM-4 |
| 최하위 | Custom | 특화 | Fine-tuned SLM |

**구현 예시**:
```python
@dataclass
class TierConfig:
    # ... (기존 필드)
    block_type: Literal["transformer", "hybrid", "mamba", "custom"] = "transformer"

    # Custom block 설정
    custom_block_class: Optional[Type] = None
    custom_block_params: dict = None
```

---

## 4. Dynamic Tier Adjustment

### 4.1 복잡도 측정

**측정 지표**:
- `files`: 파일 수
- `loc`: 코드 라인 수 (Lines of Code)
- `dependencies`: 의존성 개수 (package.json, requirements.txt 등)
- `team_size`: 팀 크기

**공식**:
```python
def calculate_complexity(project: Project) -> float:
    """프로젝트 복잡도 계산"""
    weights = config.complexity_weights

    complexity = (
        project.files * weights["files"] +
        (project.loc / 1000) * weights["loc"] +
        project.dependencies * weights["dependencies"] +
        project.team_size * weights["team_size"]
    )

    return complexity
```

**Tier 매핑**:
```python
if complexity < 50:
    return 2  # 2-tier
elif complexity < 100:
    return 3  # 3-tier
elif complexity < 200:
    return 4  # 4-tier
else:
    return 5  # 5-tier
```

---

### 4.2 런타임 Tier 조정

**시나리오**: 프로젝트 성장 시 자동 tier 추가

```python
class YOMANRuntime:
    def __init__(self, config: YOMANConfig):
        self.config = config
        self.tiers = config.tiers.copy()

    def adjust_tiers(self, project: Project):
        """복잡도 기반 tier 자동 조정"""
        complexity = calculate_complexity(project)
        required_tier_count = self._get_required_tier_count(complexity)

        current_tier_count = len(self.tiers)

        if required_tier_count > current_tier_count:
            # Tier 추가
            self._add_tier()
            logger.info(f"Tier added: {current_tier_count} → {required_tier_count}")

        elif required_tier_count < current_tier_count:
            # Tier 제거
            self._remove_tier()
            logger.info(f"Tier removed: {current_tier_count} → {required_tier_count}")

    def _add_tier(self):
        """새 tier 추가 (최하위에 추가)"""
        new_tier = TierConfig(
            name="Atom",
            model="GLM",
            context=500,
            max_complexity=10,
            causality="causal",
        )
        self.tiers.append(new_tier)

    def _remove_tier(self):
        """tier 제거 (최하위 제거)"""
        if len(self.tiers) > 2:  # 최소 2-tier 유지
            self.tiers.pop()
```

---

### 4.3 Task Routing

**복잡도 기반 tier 선택**:

```python
class UnitService:
    def __init__(self, runtime: YOMANRuntime):
        self.runtime = runtime

    def route_request(self, task: Task) -> TierConfig:
        """Task를 적절한 tier로 라우팅"""
        task_complexity = self._estimate_task_complexity(task)

        # 이진 탐색으로 적절한 tier 찾기
        for tier in self.runtime.tiers:
            if task_complexity <= tier.max_complexity:
                return tier

        # 복잡도 초과 시 최상위 tier
        return self.runtime.tiers[0]

    def _estimate_task_complexity(self, task: Task) -> float:
        """Task 복잡도 추정"""
        # 간단한 휴리스틱:
        # - 파일 수
        # - 의존성
        # - 설명 길이

        complexity = 0

        if task.files:
            complexity += len(task.files) * 5

        if task.description:
            complexity += len(task.description.split()) * 0.1

        if task.dependencies:
            complexity += len(task.dependencies) * 3

        return complexity
```

---

## 5. 실전 적용 예시

### 5.1 간단한 CLI Tool

**프로젝트 정보**:
```
files: 5
loc: 500
dependencies: 3
team_size: 1

complexity = 5*0.3 + 0.5*0.4 + 3*0.2 + 1*0.1 = 2.8
→ 2-tier
```

**선택된 Config**:
```python
YOMANConfig(
    tiers=[
        TierConfig(name="App", model="Sonnet", context=10000),
        TierConfig(name="Command", model="Flash", context=2000),
    ]
)
```

**예상 비용** (월):
- Opus: $0 (사용 안 함)
- Sonnet: $20
- Flash: $5
- **Total: $25** (기존 3-tier: $85)

**절감률**: **70%**

---

### 5.2 Microservices

**프로젝트 정보**:
```
files: 150
dependencies: 45
team_size: 8
loc: 50000

complexity = 150*0.3 + 50*0.4 + 45*0.2 + 8*0.1 = 75.8
→ 4-tier
```

**선택된 Config**:
```python
YOMANConfig(
    tiers=[
        TierConfig(name="System", model="Opus"),
        TierConfig(name="Service", model="Sonnet"),
        TierConfig(name="API", model="Sonnet"),
        TierConfig(name="Logic", model="Flash"),
    ]
)
```

**예상 비용** (월):
- Opus: $50 (System tier만)
- Sonnet: $120 (Service + API)
- Flash: $30 (Logic, 90% 트래픽)
- **Total: $200** (기존 3-tier: $280)

**절감률**: **28%**

---

## 6. 성능 예측

### 6.1 비용 효율

| 프로젝트 복잡도 | Tier 수 | 월 비용 (기존) | 월 비용 (N-Tier) | 절감률 |
|----------------|---------|---------------|-----------------|--------|
| Simple (< 50) | 2-tier | $85 | $25 | **70%** |
| Medium (50-100) | 3-tier | $150 | $150 | 0% |
| Complex (100-200) | 4-tier | $280 | $200 | **28%** |
| Enterprise (200+) | 5-tier | $400 | $350 | **12%** |

**평균 절감률**: ~30%

---

### 6.2 속도 개선

**Block Type Mixing 효과**:
- 하위 tier (90% 트래픽): Flash/GLM → **3배 빠름**
- 상위 tier (10% 트래픽): Opus → 속도 유지
- **전체 평균**: ~2.5배 향상

**병렬 처리**:
- 기존: 순차 (Opus → Sonnet → Flash)
- N-Tier: 하위 tier 병렬 가능
- **동시 처리**: 3-5개 Unit

---

### 6.3 신뢰도 향상

**Meta-Task Planning 비교**:
- Flat planning (GPT-4): **2.92%**
- Hierarchical (LLaMA-3.1-8B): **42.68%**
- **향상**: 15배

**YOMAN N-Tier 예상**:
- 기존 3-tier: ~40%
- N-Tier + Multi-Discriminator: **~60%**
- **향상**: 1.5배

---

## 7. 구현 로드맵

### 7.1 단기 (1-2주)

**Goal**: 2-Tier 프로토타입

- [ ] `YOMANConfig` 스키마 구현
- [ ] `TierConfig` 클래스 구현
- [ ] 복잡도 측정 함수 (`calculate_complexity`)
- [ ] 간단한 CLI tool로 테스트
  - Flash + Sonnet 조합
  - 비용/속도 측정

---

### 7.2 중기 (1개월)

**Goal**: 동적 Tier 조정

- [ ] `YOMANRuntime` 구현
  - `adjust_tiers` (런타임 조정)
  - `route_request` (Task 라우팅)
- [ ] 3-tier → N-tier 마이그레이션
  - 기존 UnitService 리팩토링
- [ ] Benchmark
  - 2-tier vs 3-tier vs 4-tier 비교
  - 비용, 속도, 정확도 측정

---

### 7.3 장기 (3개월)

**Goal**: Custom Block 개발

- [ ] Block 인터페이스 정의
- [ ] BucketBlock (Bucket 특화)
- [ ] MemoryBlock (Memory MCP 통합)
- [ ] GAN_Block (Multi-Discriminator 특화)
- [ ] Production 배포

---

## 8. 참고 문헌

- **MBLM** ([arXiv:2502.14553](https://arxiv.org/abs/2502.14553)) - N-Tier 동적 구조
- **Meta-Task Planning** ([arXiv:2405.16510](https://arxiv.org/abs/2405.16510)) - Zero-shot 분해
- **GoalAct** ([arXiv:2504.16563](https://arxiv.org/abs/2504.16563)) - Continuous Planning
- **Clockwork RNN** ([arXiv:1402.3511](https://arxiv.org/abs/1402.3511)) - 모듈별 clock rate

---

*작성일: 2026-01-01*
*분석 도구: Claude Opus 4.5*
