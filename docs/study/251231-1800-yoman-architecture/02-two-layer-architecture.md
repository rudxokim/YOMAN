# 02. Two-Layer Architecture

## 2.1 설계 철학

### 자동차 비유

> "운전자가 엔진을 이해할 필요 없이 효과적으로 운전할 수 있어야 한다."

| Layer | 사용자 가시성 | 목적 | 비유 |
|-------|-------------|------|------|
| Interface (Bucket) | **Visible** | 사람과 소통 | 대시보드, 컨트롤 |
| Engine (UnitService) | **Hidden** | AI 처리 | 엔진, 변속기 |

### 분리의 핵심 원리

```mermaid
graph TB
    subgraph Human["사람 영역"]
        HI[Human Input<br/>자연어, 피드백]
        HO[Human Output<br/>결과 확인, 승인]
    end

    subgraph Interface["Interface Layer"]
        Bucket[Bucket System<br/>IDEA→RESEARCH→TODO→PR]
    end

    subgraph Engine["Engine Layer"]
        Unit[UnitService<br/>System/Module/Unit]
        Verify[Verification<br/>Generator-Discriminator]
    end

    HI --> Bucket
    Bucket --> HO
    Bucket <--> Unit
    Unit <--> Verify

    style Human fill:#e1f5e1
    style Interface fill:#fff3e1
    style Engine fill:#ffe1e1
```

---

## 2.2 Interface Layer (Bucket System)

### 설계 원칙

1. **Progressive Refinement**: 정보가 단계를 거치며 정제됨
2. **Intuitive Stages**: 각 버킷이 명확하고 이해하기 쉬움
3. **Minimal Cognitive Load**: 4개 개념만 알면 됨 (IDEA, RESEARCH, TODO, PR)

### 버킷 정의

| Bucket | 입력 | 출력 | 사람의 행동 |
|--------|------|------|-----------|
| **IDEA** | 자연어 아이디어 | 간단한 마크다운 | 아이디어 작성, Reviewed=true |
| **RESEARCH** | IDEA 페이지 | 구조화된 분석 + 다이어그램 | 리뷰, 코멘트 추가, Reviewed=true |
| **TODO** | RESEARCH 페이지 | 실행 가능한 스펙 + Prerequisites | 범위 확인, 코딩 승인 |
| **PR** | TODO 페이지 | Git Pull Request | match_rate 확인, merge |

### 사람이 하는 일

```
9:00  📱 /idea "사용자 인증 추가"
      ↓ [AI가 알아서 처리 - 30분]
9:30  📱 "RESEARCH 준비됨" → 폰에서 확인, "approve"
      ↓ [AI가 알아서 처리 - 10분]
10:00 📱 "TODO 준비됨" → "approve"
      ↓ [AI가 코드 생성 - 15분]
10:15 📱 "PR #123 준비 (92% match)" → "merge"
10:16 완료!
```

**컴퓨터 사용 시간**: 0분. 전부 폰으로 처리.

---

## 2.3 Engine Layer (UnitService Architecture)

### 숨겨진 복잡성

Engine Layer는 사람에게 보이지 않음. 하지만 내부에서는:

```mermaid
flowchart TB
    subgraph Engine["ENGINE LAYER (Hidden)"]
        direction TB

        subgraph Hierarchy["3-Tier Hierarchy"]
            System["System Level<br/>Claude Opus<br/>아키텍처 결정"]
            Module["Module Level<br/>Claude Sonnet<br/>컴포넌트 설계"]
            Unit["Unit Level<br/>Flash/GLM<br/>함수 구현"]

            System -->|conditioning| Module
            Module -->|conditioning| Unit
        end

        subgraph Memory["Knowledge Graph"]
            KG[(Memory MCP<br/>entities/<br/>relations/)]
        end

        subgraph Parallel["Multi-Model Verification"]
            GLM[GLM-4.7]
            Flash[Gemini Flash]
            Gem3[Gem 3.0]

            GLM --> Test[Unit Test]
            Flash --> Test
            Gem3 --> Test
            Test --> Best[최적 선택]
        end

        KG -->|context 제공| System
        KG -->|context 제공| Module
        Unit --> GLM & Flash & Gem3
    end

    style Engine fill:#ffe1e1
```

### 왜 숨기는가?

| 관점 | Interface 노출 시 | 숨길 때 |
|------|------------------|--------|
| **복잡성** | 사람이 이해해야 함 | AI가 알아서 처리 |
| **유연성** | 변경 시 재교육 필요 | 내부만 수정 |
| **신뢰** | 세부사항에 집착 | 결과에 집중 |

---

## 2.4 정보 흐름 상세

### End-to-End 시퀀스

```mermaid
sequenceDiagram
    participant H as Human
    participant I as Interface (Bucket)
    participant E as Engine (UnitService)
    participant R as REVIEWER
    participant G as GitHub

    H->>I: /idea "인증 추가"
    I->>I: IDEA 페이지 생성

    H->>I: Reviewed = true
    I->>E: IDEA → RESEARCH 처리
    E->>E: System: 범위 분석
    E->>E: Module: 리서치 구조화
    E->>I: RESEARCH 문서 반환

    H->>I: Reviewed = true
    I->>E: RESEARCH → TODO 처리
    E->>E: 실행 가능한 유닛으로 분할
    E->>I: TODO + Prerequisites 반환

    H->>I: /code
    I->>E: TODO 실행
    E->>E: System: 아키텍처 결정
    E->>E: Module: 컴포넌트 조율
    E->>E: Unit: 함수 구현 (병렬)
    E->>G: PR 생성

    G->>R: PR 리뷰 요청
    R->>R: PR vs TODO 스펙 비교
    R->>I: REPORT (match_rate)

    I->>H: "PR #123 준비 (92% match)"
    H->>I: "merge"
    I->>G: PR Merge
```

### 각 Layer의 책임

| Layer | 책임 | NOT 책임 |
|-------|------|---------|
| **Interface** | 상태 추적, 형식 변환, 사용자 소통 | 코드 생성, 품질 검증 |
| **Engine** | 코드 생성, 컨텍스트 관리, 품질 검증 | 사용자 직접 소통 |

---

## 2.5 분리의 이점

### 1. 독립적 진화

```
Interface 개선:                    Engine 개선:
- Telegram → Slack 전환           - Opus 4.5 → Opus 5.0
- 새로운 버킷 추가                 - 새로운 검증 로직
- UI/UX 개선                       - 성능 최적화

→ 서로 영향 없이 각각 개선 가능
```

### 2. 관심사 분리

```python
# Interface Layer - 사람의 인지에 최적화
def create_idea(user_input: str) -> IdeaPage:
    """사람이 이해하기 쉬운 형태로 변환"""
    return IdeaPage(
        title=extract_title(user_input),
        goals=extract_goals(user_input),
        non_goals=[]  # 사람이 채움
    )

# Engine Layer - AI 능력에 최적화
def generate_code(todo: TodoSpec) -> PullRequest:
    """복잡한 다중 모델 오케스트레이션"""
    system_ctx = opus.analyze(knowledge_graph, todo)
    modules = [sonnet.design(m, system_ctx) for m in system_ctx.modules]
    units = [flash.implement(f, m) for m in modules for f in m.functions]
    return create_pr(units)
```

### 3. 테스트 용이성

| Layer | 테스트 방법 |
|-------|-----------|
| Interface | UI/UX 테스트, 사용자 피드백 |
| Engine | 자동화된 품질 메트릭, match_rate |

---

## 2.6 실제 구현: zorba-the-robot

### 프로젝트 구조

```
project/
│
├── docs/                          # 사람 영역 (5%)
│   ├── guides/                    # 가이드 문서
│   ├── research/                  # 리서치 결과
│   ├── study/                     # 학습 자료
│   └── ops/                       # Operation 정의
│
└── -zorba-the-robot/              # 로봇 영역 (95%)
    ├── src/                       # 소스 코드
    │   ├── CODINGBOT/             # Generator
    │   ├── REVIEWER/              # Discriminator
    │   └── Bucket-Trigger/        # 버킷 전환
    ├── tests/                     # 테스트
    ├── entities/                  # Memory MCP
    ├── relations/                 # 지식 그래프
    └── .github/                   # CI/CD
```

### 명명 규칙

- **docs/**: 사람이 읽는 모든 것
- **-zorba-the-robot/**: 대시(-)로 시작 = AI 전용

> "사람은 src/도 안 봄. 코드 자체를 안 봄."

---

## 핵심 요약

```mermaid
mindmap
  root((Two-Layer<br/>Architecture))
    Interface Layer
      Bucket System
        IDEA
        RESEARCH
        TODO
        PR
      사람 최적화
        자연어
        직관적 단계
        최소 인지 부하
    Engine Layer
      UnitService
        System Opus
        Module Sonnet
        Unit Flash
      AI 최적화
        병렬 처리
        컨텍스트 관리
        다중 모델
```

---

*다음: [03-unitservice-hierarchy.md](03-unitservice-hierarchy.md) - SampleRNN 영감의 3계층 구조*
