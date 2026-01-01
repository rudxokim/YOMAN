# Causal vs Non-Causal System in LLM-based Code Generation

> YOMAN 프로젝트에서 제기된 "causal system의 한계와 REVIEWER를 통한 quasi non-causal 효과" 연구

---

## 1. 문제 정의

### 1.1 원본 질문 (idea3.txt)

```
지금 얘기하는걸로 보면 이것은 명백히 causal system임.
하지만 리뷰어의 개념이 들어가면 non casual system이 될 수 있는건지??
```

### 1.2 맥락

YOMAN 시스템의 GAN 검증 패턴:
- **Generator (CODINGBOT)**: TODO 스펙을 받아 PR 생성
- **Discriminator (REVIEWER)**: PR과 TODO 스펙 비교 후 match_rate 산출

질문의 핵심: **REVIEWER가 "미래 정보"(최종 스펙)를 알고 검증하면 non-causal 효과가 나는가?**

---

## 2. Causal vs Non-Causal 정의

### 2.1 신호처리/시스템 이론 관점

| 구분 | Causal System | Non-Causal (Acausal) System |
|------|---------------|----------------------------|
| **정의** | 출력이 현재/과거 입력에만 의존 | 출력이 미래 입력에도 의존 |
| **Impulse Response** | h(t) = 0 for t < 0 | h(t) != 0 for some t < 0 |
| **물리적 구현** | 가능 (실시간 처리) | 불가능 (미래 모름) |
| **오프라인 처리** | - | 저장된 데이터면 가능 |

**핵심 인사이트**:
> "All real-world systems must be causal since they do not have access to the future."
> — [Wikipedia: Causal System](https://en.wikipedia.org/wiki/Causal_system)

그러나:
> "If the data are already stored in a computer, it is possible to use future signal values."
> — [ScienceDirect: Noncausal](https://www.sciencedirect.com/topics/engineering/noncausal)

### 2.2 ML/LLM 관점

**Autoregressive (Causal) Language Model**:
- 현재 토큰 예측 시 이전 토큰만 참조
- GPT, Claude 등 대부분의 LLM이 이 방식
- 수학적 표현: P(x_t | x_1, ..., x_{t-1})

**Non-Autoregressive / Bidirectional Model**:
- 현재 토큰 예측 시 전후 문맥 모두 참조
- BERT (Masked LM), Diffusion LM 등
- 수학적 표현: P(x_t | x_1, ..., x_{t-1}, x_{t+1}, ..., x_n)

---

## 3. LLM Code Generation의 Causal 한계

### 3.1 근본적 제약

| 문제 | 설명 |
|------|------|
| **단방향 생성** | 앞에서 뒤로만 생성 → 뒤의 정보가 앞을 개선 불가 |
| **순차적 인과 ≠ 논리적 인과** | 텍스트 순서가 논리적 의존성과 불일치 |
| **긴 시퀀스 일관성** | 길어질수록 초반 목표에서 이탈 |
| **Level-1 Causal Reasoning** | 패턴 기억 위주, 진정한 인과추론(Level-2) 부재 |

> "Autoregressive training makes LLMs memorize common causal knowledge expressions, which limits their ability to reason genuinely rather than simply recall patterns."
> — [NeurIPS 2024: Unveiling Causal Reasoning in LLMs](https://arxiv.org/abs/2506.21215)

### 3.2 코드 생성 시 구체적 문제

```python
# Causal LLM이 생성하는 순서
def process_data(data):    # (1) 함수 시그니처
    result = []            # (2) 초기화
    for item in data:      # (3) 루프
        ...                # (4) 로직 - 여기서 (1)의 파라미터가 부족함을 깨달아도 수정 불가
    return result          # (5) 리턴
```

**문제**: (4)에서 추가 파라미터가 필요함을 깨달아도 (1)로 돌아가 수정 불가

---

## 4. REVIEWER가 "Quasi Non-Causal" 효과를 내는 메커니즘

### 4.1 GAN 구조의 핵심 통찰

```
TODO (스펙) ──► CODINGBOT (Generator) ──► PR (코드)
                                           │
                                           ▼
                    ┌──────────────────────────────────────┐
                    │            REVIEWER                   │
                    │         (Discriminator)               │
                    │                                       │
                    │   입력: PR + TODO (미래 목표)         │
                    │   출력: match_rate (0-100%)           │
                    └───────────────────┬──────────────────┘
                                        │
              ┌─────────────────────────┼─────────────────────────┐
              ▼                         ▼                         ▼
         match ≥80%                  50-80%                     <50%
            PASS                    PARTIAL                      FAIL
         (PR 머지)              (새 TODO 생성)              (IDEA로 회귀)
```

### 4.2 왜 "Quasi Non-Causal"인가?

**전통적 Causal System** (Discriminator 없음):
```
TODO → CODINGBOT → PR → 배포
                        ↑
                        └── 미래 피드백 없음, 그냥 진행
```

**YOMAN GAN Pattern** (Discriminator 있음):
```
TODO → CODINGBOT → PR → REVIEWER → match_rate
 ↑                          │
 └──────────────────────────┘
         피드백 루프 (quasi non-causal)
```

**핵심 차이**:
1. **REVIEWER는 "미래 정보" (TODO 스펙) 를 알고 있음**
2. **생성 후 검증** → 틀렸으면 다시 생성
3. **반복 가능** → 언젠간 스펙에 맞는 코드 생성

### 4.3 이론적 근거

이는 신호처리의 **Block Processing**과 유사:

> "In discrete-time (digital) systems we can create the illusion of non-causality by processing blocks or batches of data... In this extracted block, we are free to look forwards and backwards in time as we wish!"
> — [ScienceDirect](https://www.sciencedirect.com/topics/engineering/noncausal)

YOMAN에 적용:
- **Block** = 하나의 TODO → PR 사이클
- **Forward look** = REVIEWER가 TODO 스펙(목표)을 참조하여 PR 검증
- **Backward correction** = match_rate 낮으면 재생성

---

## 5. 유사 접근법 비교

### 5.1 Multi-Agent Debate

> "Multiple language model instances propose and debate their individual responses... significantly enhances mathematical and strategic reasoning."
> — [ICML 2024: Multiagent Debate](https://arxiv.org/abs/2305.14325)

| 항목 | Multi-Agent Debate | YOMAN GAN |
|------|-------------------|-----------|
| 에이전트 | 동등한 여러 LLM | Generator + Discriminator (역할 분리) |
| 목표 | 합의 도출 | 스펙 충족 검증 |
| 피드백 | 상호 비판 | 단방향 (REVIEWER → CODINGBOT) |
| 장점 | 다양한 관점 | 명확한 성공 기준 (match_rate) |

### 5.2 Self-Verification / Self-Critique

> "Self-correction is an approach to improving responses from LLMs by refining the responses using LLMs during inference."
> — [MIT Press: When Can LLMs Actually Correct Their Own Mistakes?](https://direct.mit.edu/tacl/article/doi/10.1162/tacl_a_00713)

**문제점**:
- LLM이 자기 오류를 자기가 검증 → 같은 blind spot 공유
- "LLMs can be persuaded to accept falsehoods with high-confidence, verbose reasoning"

**YOMAN 장점**:
- Generator와 Discriminator 역할 분리
- REVIEWER는 코드 생성 안 함 → 다른 관점에서 검증

### 5.3 Constitutional AI (CAI)

> "Constitutional AI uses rule-based self-assessment... The model critiques itself using constitution guidelines."
> — [Anthropic: Constitutional AI](https://www.anthropic.com/research/constitutional-ai-harmlessness-from-ai-feedback)

| 항목 | Constitutional AI | YOMAN GAN |
|------|------------------|-----------|
| 규칙 | 헌법 원칙 (추상적) | TODO 스펙 (구체적) |
| 검증 | 자가 비평 | 외부 Discriminator |
| 수치화 | 어려움 | match_rate (명확) |
| 적용 | 안전성/윤리 | 기능 정확성 |

### 5.4 Diffusion Language Models (Bidirectional)

> "Diffusion language models promise bidirectional context and infilling capabilities that autoregressive coders lack."
> — [CoDA: Coding LM via Diffusion Adaptation](https://arxiv.org/abs/2510.03270)

**장점**:
- 진정한 Non-Causal (양방향 컨텍스트)
- 코드 중간 수정 가능 ("jump back and forth through code")

**단점**:
- 아직 초기 연구 단계 (1.7B 파라미터)
- 고정 길이 출력 필요
- KV 캐싱 불가 → 추론 비효율

### 5.5 Test-Time Compute Scaling

> "Scaling LLM Test-Time Compute... can be more effective than scaling parameters for reasoning."
> — [OpenReview: Scaling LLM Test-Time Compute](https://openreview.net/forum?id=4FWAwZtd2n)

YOMAN과의 연결:
- **Test-Time Compute** = 생성 시간에 더 많은 계산 투입
- **REVIEWER 검증 루프** = 일종의 test-time compute
- **재생성 반복** = thinking time 증가 효과

---

## 6. LLMLOOP: 가장 유사한 선행 연구

> "LLMLOOP employs five iterative loops: resolving compilation errors, addressing static analysis issues, fixing test case failures, and improving test quality through mutation analysis."
> — [ICSME 2025: LLMLOOP](https://www.semanticscholar.org/paper/a7da0a5b7331a5e4a97fc944d2a8e84bafc42179)

### LLMLOOP 구조

```
LLM → Code → Compile → Error? → LLM (fix)
              ↓
           Static Analysis → Issue? → LLM (fix)
              ↓
           Test Execution → Fail? → LLM (fix)
              ↓
           Mutation Analysis → Low Quality? → LLM (improve)
```

### YOMAN GAN vs LLMLOOP

| 항목 | LLMLOOP | YOMAN GAN |
|------|---------|-----------|
| 검증 기준 | 컴파일, 테스트, 린트 | TODO 스펙 대비 match_rate |
| 피드백 소스 | 도구 (컴파일러, 테스트) | LLM (REVIEWER) |
| 장점 | 객관적, 자동화 | 의미론적 검증 가능 |
| 단점 | 의미 이해 불가 | LLM 오류 가능성 |

**통합 가능성**: LLMLOOP의 도구 기반 검증 + YOMAN의 의미론적 검증

---

## 7. 결론: YOMAN의 "Quasi Non-Causal" 효과

### 7.1 질문에 대한 답변

**Q**: "리뷰어의 개념이 들어가면 non-causal system이 될 수 있는건지??"

**A**:
- **엄밀한 의미의 non-causal**: 아니오. 시간을 거슬러 생성하지 않음.
- **Quasi Non-Causal (효과적 관점)**: 예! 아래 이유로:
  1. REVIEWER가 "미래 목표"(TODO 스펙)를 알고 검증
  2. 검증 실패 시 재생성 → 결과적으로 목표에 맞는 코드 생성
  3. Block processing 관점에서 "illusion of non-causality" 구현

### 7.2 이론적 위치

```
                    Non-Causal 스펙트럼

Pure Causal          Quasi Non-Causal          True Non-Causal
(단순 생성)          (검증 + 재생성)            (양방향 생성)
    │                      │                        │
    ▼                      ▼                        ▼
 GPT 단발       ─────► YOMAN GAN ◄───── Diffusion LLM
 자동완성                패턴                    (연구중)
                          ↑
                     LLMLOOP
                     CAI
                     Multi-Agent Debate
```

### 7.3 권장 구현 전략

| 전략 | 설명 | 우선순위 |
|------|------|----------|
| **도구 기반 검증 통합** | 컴파일, 테스트, 린트 결과를 match_rate에 반영 | 높음 |
| **다중 모델 검증** | REVIEWER를 다른 모델로 (blind spot 방지) | 높음 |
| **점진적 재생성** | 전체 재생성 대신 부분 수정 | 중간 |
| **Diffusion LLM 주시** | CoDA, LLaDA 등 연구 추적 | 낮음 (미래) |

---

## 8. 참고 문헌

### 8.1 Causal System Theory
- [Causal system - Wikipedia](https://en.wikipedia.org/wiki/Causal_system)
- [Noncausal - ScienceDirect](https://www.sciencedirect.com/topics/engineering/noncausal)
- [TutorialsPoint: Causal and Non-Causal System](https://www.tutorialspoint.com/signals-and-systems-causal-and-non-causal-system)

### 8.2 LLM Causal Limitations
- [NeurIPS 2024: Unveiling Causal Reasoning in LLMs](https://arxiv.org/abs/2506.21215)
- [Causal Reasoning and LLMs - Microsoft Research](https://www.microsoft.com/en-us/research/publication/causal-reasoning-and-large-language-models-opening-a-new-frontier-for-causality/)
- [Understanding Causal LLMs vs Masked LLMs - Medium](https://medium.com/@tom_21755/understanding-causal-llms-masked-llm-s-and-seq2seq-a-guide-to-language-model-training-d4457bbd07fa)

### 8.3 Self-Verification & Multi-Agent
- [MIT Press: When Can LLMs Actually Correct Their Own Mistakes?](https://direct.mit.edu/tacl/article/doi/10.1162/tacl_a_00713)
- [ICML 2024: Multiagent Debate](https://arxiv.org/abs/2305.14325)
- [ChatEval - Multi-Agent Debate Framework](https://openreview.net/forum?id=FQepisCUWu)

### 8.4 Constitutional AI & RLHF
- [Anthropic: Constitutional AI](https://www.anthropic.com/research/constitutional-ai-harmlessness-from-ai-feedback)
- [Safe RLHF - PKU-Alignment](https://github.com/jwliao-ai/Constitutional-AI/)

### 8.5 Diffusion Language Models
- [CoDA: Coding LM via Diffusion Adaptation](https://arxiv.org/abs/2510.03270)
- [LLaDA: Large Language Diffusion Models](https://arxiv.org/html/2502.09992v2)
- [JetBrains: Why Diffusion Models Could Change Developer Workflows](https://blog.jetbrains.com/ai/2025/11/why-diffusion-models-could-change-developer-workflows-in-2026/)

### 8.6 Code Generation & Verification
- [LLMLOOP - ICSME 2025](https://www.semanticscholar.org/paper/a7da0a5b7331a5e4a97fc944d2a8e84bafc42179)
- [WaveCoder - ACL 2024](https://aclanthology.org/2024.acl-long.280.pdf)
- [Verification Limits Code LLM Training](https://arxiv.org/html/2509.20837v1)

### 8.7 Test-Time Compute
- [OpenReview: Scaling LLM Test-Time Compute](https://openreview.net/forum?id=4FWAwZtd2n)
- [AI Formal Verification - Martin Kleppmann](https://martin.kleppmann.com/2025/12/08/ai-formal-verification.html)
- [Karpathy: 2025 LLM Year in Review](https://karpathy.bearblog.dev/year-in-review-2025/)

---

*문서 생성일: 2026-01-01*
*분석 모델: Claude Opus 4.5*
*리서치 방법: 웹 검색 + 프로젝트 문서 분석*
