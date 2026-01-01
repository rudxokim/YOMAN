# Cursor 2.0 분석

## 개요

Cursor 2.0는 "agent 중심 인터페이스"로 재설계된 AI 코딩 도구. 멀티 에이전트 병렬 실행과 실시간 검증 루프가 핵심.

**출처**:
- YouTube: "Cursor 2.0: What Actually Matters in This Release" (Ray Fernando)
- YouTube: "Cursor AI Agents Work Like 10 Developers (Cursor VP Live Demo)" (Greg Isenberg)
- GitHub Copilot docs (Context7)

## 핵심 메커니즘

### 1. Multi-Agent Parallel Execution

```
Agent 1 (Bug Fix)  ──┐
Agent 2 (Feature)  ──┼─→ Git Work Trees
Agent 3 (Testing)  ──┘
```

**특징**:
- 여러 에이전트가 독립적으로 작업
- Git work trees로 격리
- Remote machine 사용 가능 (간섭 없음)
- 같은 프롬프트를 여러 모델에 동시 실행 가능

**실제 사용 사례** (Lee의 증언):
> "Yesterday I fired off about 20 different agents, 20 different conversations over the day."

각 에이전트는 30~75줄 정도의 작은 변경을 담당.

### 2. Context Window Monitoring

**문제**: AI가 너무 많은 정보를 한 번에 처리하면 품질 저하

**해결책**:
```
Context Window: [████████░░░░░░░░░░] 17% ✅
Context Window: [███████████████████░] 80% ⚠️
Context Window: [████████████████████] 100% 🚫
```

- **17%**: 정상 범위 (이 예시의 실제 값)
- **80% 이상**: 품질 저하 시작 (새 채팅 권장)
- **100%**: 자동 요약 (권장하지 않음)

**권장 패턴**:
> "For each new task to get the best quality out of working with the model, you can just start new chats for each thing that you want to do."

### 3. Composer Model (4x Faster)

**특징**:
- Frontier model (최고 성능)
- 대부분의 작업을 **30초 이내** 완료
- Mixture of Experts (MoE) 아키텍처
- 실제 코딩 작업에 특화된 RL 학습

**성능 비교**:
| 모델 | 속도 | 용도 |
|------|------|------|
| Composer | 1x (기준) | Multi-step coding |
| 유사 모델 | 4x 느림 | General purpose |

**예시 작업** (30초 이내):
1. 버그 조사 (event bus panic)
2. 관련 커밋 검색
3. Race condition 추적
4. 코드 수정
5. 테스트 실행

### 4. 자동 검증 루프

```
Agent 작업 완료
    ↓
Linter 실행
    ↓
Error 발견? ──Yes──→ Agent 자동 수정
    ↓ No
Tests 실행
    ↓
Fail? ──Yes──→ Agent 디버깅
    ↓ No
✅ 사람 리뷰 대기
```

**설정 요구사항** (Lee의 추천):
- TypeScript (타입 체킹)
- ESLint (코드 스타일)
- Prettier (포맷팅)
- Tests (기능 검증)

> "The agent can read its own outputs and then just self-correct and fix itself without me having to do anything."

### 5. Code Review Bot (Bugbot)

**비동기 검증 메커니즘**:

```
Developer → PR 생성 → Bugbot 분석 → 댓글 자동 추가
                          ↓
                    Missing translations
                    Security issues
                    Performance concerns
```

**실제 예시**:
```
Bugbot: "Hey, by the way, I think you missed some of the other languages.
So we have our docs internationalized to a bunch of different languages
and I was like, 'You missed you missed a few, bro.'"
```

**통합 방식**:
- GitHub Actions
- CI/CD 파이프라인
- Headless CLI 모드

```bash
# CI script example
cursor-cli audit --security
cursor-cli auto-fix --tests
```

### 6. Slack/Web Integration

**비동기 작업 트리거**:

1. **Slack에서**:
   ```
   @cursor [repo-link] investigate this bug and fix it
   ```

2. **웹에서** (cursor.com/agents):
   - 브라우저에서 프롬프트 입력
   - Secure virtual sandbox에서 실행
   - PR 자동 생성

3. **로컬 에디터**:
   - 일반적인 코딩 워크플로우

**Lee의 사용 사례**:
> "I'm in Slack and I just kind of pasted in what they said and then I said @cursor here's the repo. Investigate this bug and fix it... I didn't have to go open up the editor. I just kicked it off from Slack on my phone."

### 7. Custom Commands & Rules

**예시: Code Review Command**

```markdown
# .cursor/code-review.md

슬래시 커맨드: /code-review

체크 항목:
- 인터넷 없는 상태 처리 확인
- 로딩 스피너 추가 여부
- 테스트 품질 (양보다 질)
- 인증 변경 시 보안 리뷰
- 캐싱 최적화 기회
```

**예시: Writing Style Rules**

```markdown
# .cursor/writing-rules.md

금지 단어:
- "mission-critical"
- "performant"
- "seamless"
- "it's not just X, it's Y" (LLM 패턴)
```

> "My most recent addition here is also trying to catalog some of the LLM patterns like the junk that it spits out."

### 8. Progress Tracking

**에이전트 리스트 뷰**:

```
[Yesterday]
🟢 Add event tracking (30 lines)
🟢 Fix layout shift (45 lines)
🟢 Remove custom component (75 lines)
⏸️  Resolve module declaration error (진행 중)
```

**각 세션 정보**:
- 파일 변경 개수
- 줄 수 변경량
- Context window 사용률
- Linter 결과
- 실행 시간

## 인터페이스 디자인 원칙

### 1. Agent-Centric View

**기존**: File explorer 중심
```
📁 src/
  📁 components/
    📄 Button.tsx
    📄 Header.tsx
```

**Cursor 2.0**: Agent 중심
```
👤 Agent 1: "Add button hover effect" (30 lines)
👤 Agent 2: "Refactor header layout" (45 lines)
👤 Agent 3: "Fix accessibility issues" (75 lines)
```

### 2. Session Isolation

**권장 패턴**:
- 1 Task = 1 Session
- 30~75 줄 정도 변경
- Context < 80%

**안티패턴**:
- Append-only 대화 (계속 이어가기)
- 여러 기능을 한 세션에서
- Context 100% 도달

### 3. Inspect + Approve Workflow

```
Agent 완료
    ↓
Diff 확인 (Multi-file view)
    ↓
Linter/Test 결과 확인
    ↓
Approve ──→ Merge
  or
Request Changes ──→ Agent 재작업
```

### 4. Plan Mode (Background)

**비동기 계획 수립**:

```
Plan 1 (Model A) ──┐
Plan 2 (Model B) ──┼─→ 사람이 선택
Plan 3 (Model C) ──┘
```

- 여러 모델이 동시에 계획 수립
- 백그라운드 실행
- 최적의 접근법 선택

## Q&A에 대한 답

### Q1: 병렬 처리 vs 순차 피드백

**Cursor의 답**:
- **Task Isolation**: 각 에이전트를 독립 세션으로 분리
- **Context Window Limit**: 사람이 추적 가능한 범위로 제한 (80%)
- **Small Changes**: 30~75줄 정도의 작은 변경 단위

### Q2: Progress Tracking, Interruption, Steering

**Cursor의 답**:
- **Live Agent List**: 20개 에이전트 동시 표시
- **Session Log**: 각 에이전트의 단계별 진행상황
- **Auto-correction**: Linter 에러 발견 시 자동 수정 (사람 개입 없이)
- **Steering**: 새 메시지로 방향 전환 (명시적 언급 없지만 일반적 채팅 인터페이스)

### Q3: Causality 표현

**Cursor의 답**:
- **Multi-file Diff**: 모든 변경사항 한 곳에서 확인
- **Linter Output**: "The changes that the agent made are correct"
- **Context Window %**: 작업 범위 가늠
- **@-tagging**: 명시적 파일 참조 (`@filename`)

## YOMAN 적용 시사점

### Bucket System 연결

| YOMAN | Cursor 패턴 |
|-------|-------------|
| IDEA Bucket | Quick prompts (단순 요청) |
| RESEARCH Bucket | Plan Mode (여러 접근법 비교) |
| TODO Bucket | Agent Session (30~75줄 작업) |
| PR Bucket | Code Review Bot (Bugbot) |

### UnitService와 Context Window

```
Tier 3 (System)  → Context: 70~80% (전체 프로젝트)
Tier 2 (Module)  → Context: 40~60% (여러 파일)
Tier 1 (Unit)    → Context: 10~30% (단일 함수/클래스)
```

### GAN Verification

- **Generator**: Cursor Agent (코드 생성)
- **Discriminator**: Bugbot (자동 리뷰)
- **Human**: 최종 Approve

## 핵심 교훈

1. **컨텍스트 관리가 품질의 핵심**
   - 80% 이상 = 품질 저하
   - 새 세션 = 새로운 시작

2. **자동 검증으로 피드백 루프 단축**
   - Linter → 즉시 수정
   - Tests → 자동 디버깅

3. **멀티 인터페이스로 접근성 향상**
   - Editor (정밀 작업)
   - Slack (빠른 버그 수정)
   - Web (어디서나 접근)

4. **Disposable Software 철학**
   - 코드 작성 비용 = 0
   - 디버깅 도구도 AI 생성
   - 일회용 커스텀 인터페이스

---

**참고 자료**:
- [Cursor VP Live Demo - YouTube](https://www.youtube.com/watch?v=8QN23ZThdRY)
- [Cursor 2.0 Release - YouTube](https://www.youtube.com/watch?v=43r9OZ1a8nk)
- GitHub Copilot docs (Context7)
