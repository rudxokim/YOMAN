# 공통 디자인 패턴 추출

## 개요

Cursor, GitHub Copilot, Replit Agent의 인터페이스를 분석하여 Human-AI 비동기 협업의 공통 패턴을 추출함.

## 패턴 카탈로그

### P1: Session Isolation Pattern

**문제**: AI가 너무 많은 정보를 한 번에 처리하면 품질 저하, 사람이 추적 불가능

**해결책**: 각 작업을 독립적인 세션으로 격리

**구현 방식**:

| 도구 | 격리 단위 | 권장 크기 |
|------|----------|----------|
| Cursor | Chat session | 30~75줄 변경 |
| Copilot | Agent session | Token count 기반 |
| Replit | Project | 단일 앱 |

**구조**:
```
Session 1 (Bug Fix)    → Context A
Session 2 (Feature)    → Context B (독립)
Session 3 (Refactor)   → Context C (독립)
```

**장점**:
- Context 오염 방지
- 사람이 추적 가능한 단위
- 병렬 실행 가능

**단점**:
- Session 간 정보 공유 어려움
- 전체 맥락 파악 필요시 여러 세션 확인

**YOMAN 적용**:
```
Operation 1 (TODO #123) → Context A
Operation 2 (TODO #124) → Context B
Operation 3 (TODO #125) → Context C
```

---

### P2: Context Budget Monitoring Pattern

**문제**: AI의 작업 범위를 사람이 어떻게 가늠하는가?

**해결책**: 실시간 리소스 사용량 표시

**두 가지 접근법**:

#### A. Percentage-based (Cursor)
```
Context Window: [████████░░░░░░░░░░] 17% ✅
Context Window: [███████████████████░] 80% ⚠️
Context Window: [████████████████████] 100% 🚫
```

**장점**: 직관적 (0~100%)
**단점**: 절대적 크기 모름

#### B. Absolute Count (Copilot)
```
Token Usage: 4,521 / 8,000 tokens (56%)
Est. Cost: $0.23
```

**장점**: 비용 예측 가능
**단점**: 기술적 용어 (토큰)

**권장 사항**:
- **전문가**: Absolute count (정확한 제어)
- **비전문가**: Percentage (직관성)

**YOMAN 적용**:
```
UnitService Tier별 Budget:
- Tier 1 (Unit):   500~1,500 tokens (10~30%)
- Tier 2 (Module): 2,000~4,000 tokens (40~60%)
- Tier 3 (System): 6,000~8,000 tokens (70~80%)
```

---

### P3: Live Progress Log Pattern

**문제**: AI가 "무엇을 하고 있는지" 사람이 어떻게 아는가?

**해결책**: 실시간 단계별 진행상황 표시

**공통 구조**:
```
[Timestamp] [Status Icon] [Action] [Target]
[12:34:21]  🔍           Analyzing   repository structure
[12:34:23]  📂           Found       3 authentication files
[12:34:25]  🛠️            Editing     auth/middleware.ts
[12:34:28]  ✅           Passed      tests (12/12)
```

**구성 요소**:

| 요소 | 목적 | 예시 |
|------|------|------|
| Timestamp | 시간 추적 | `[12:34:21]` |
| Status Icon | 시각적 구분 | 🔍 📂 🛠️ ✅ ❌ |
| Action | 동작 유형 | Analyzing, Editing, Testing |
| Target | 대상 객체 | file path, test name |

**변형**:

#### Cursor: Linter/Test Loop
```
Agent 작업 완료
    ↓
Linter 실행 → Error 발견 → 자동 수정
    ↓
Tests 실행 → Fail → 디버깅
    ↓
✅ All checks passed
```

#### Replit: Phase-Based
```
Phase 1: Initial Planning     [████░░░░] 25%
Phase 2: Plan Approval         [████████] 50%
Phase 3: Visual Preview        [████████] 75%
Phase 4: Interactive Prototype [████████] 100%
```

**YOMAN 적용**:
```
[14:23:01] 🔍 RESEARCH: TODO #123 분석 시작
[14:23:15] 📋 RESEARCH → TODO: 실행 계획 수립
[14:23:45] 🛠️  CODINGBOT: PR #45 생성 중
[14:24:20] 🔎 REVIEWER: PR #45 검증 중
[14:24:50] ✅ PR #45 match_rate: 85% (통과)
```

---

### P4: Steering Without Interruption Pattern

**문제**: 작업 중 방향을 바꾸고 싶은데 처음부터 다시 시작하기는 싫음

**해결책**: 작업 중단 없이 추가 입력으로 방향 조정

**구현 방식**:

#### Copilot: Explicit Steering (비용 명확)
```
Agent 작업 중...
    ↓
User: "Use existing ErrorHandler utility"
    (Cost: 1 premium request)
    ↓
Agent 방향 전환
```

#### Replit: Anytime Feature Request
```
Phase 4 완료
    ↓
User: "Add database"
    ↓
Agent가 기존 코드 수정 (새 Phase 시작)
```

#### Cursor: Implicit (Chat 기반)
```
Agent: [30줄 변경 완료]
    ↓
User: "Also add error handling"
    ↓
Agent: [추가 15줄 변경]
```

**공통점**:
- 작업 중단 없음
- Context 유지
- 즉각적 반응

**차이점**:
- **비용 모델**: 명시적 (Copilot) vs 암묵적 (Cursor/Replit)
- **Scope 제어**: 증분 (Cursor) vs 마이그레이션 (Replit)

**YOMAN 적용**:
```
CODINGBOT 작업 중 (TODO #123 → PR #45)
    ↓
User (Telegram): "Database도 PostgreSQL로 바꿔"
    ↓
CODINGBOT: 추가 작업 (PR #45 업데이트)
    (비용: +1 Operation 또는 동일 Operation 내 처리)
```

---

### P5: Diff + Overview Pattern

**문제**: "무엇이 바뀌었는지" 어떻게 빠르게 파악하는가?

**해결책**: 요약(Overview)과 세부사항(Diff)을 계층적으로 제공

**2-Level Hierarchy**:

#### Level 1: Overview (High-level)
```
Summary:
✅ Refactored authentication middleware to use JWT
✅ Added error handling for expired tokens
✅ Updated tests to cover new edge cases

Files changed: 3
Lines added: 45
Lines removed: 23
```

#### Level 2: Diff (Low-level)
```diff
# auth/middleware.ts
- if (token) {
-   const user = verifyToken(token);
- }
+ try {
+   if (token) {
+     const user = await verifyToken(token);
+     if (user.expired) throw new TokenExpiredError();
+   }
+ } catch (error) {
+   ErrorHandler.handle(error);
+ }
```

**탐색 패턴**:
```
User: "뭐가 바뀌었지?" → Overview 확인
    ↓
User: "JWT 부분 어떻게 구현했지?" → Diff 확인
```

**YOMAN 적용**:

#### Telegram 인터페이스
```
User: "PR #45 어떻게 되가?"
    ↓
Bot (Level 1):
"✅ Auth 미들웨어 리팩토링
 ✅ 에러 핸들링 추가
 ✅ 테스트 3개 추가
 [자세히] [Approve] [수정 요청]"
    ↓
User: [자세히] 클릭
    ↓
Bot (Level 2):
"[File 1/3] auth/middleware.ts
 +45 -23 lines
 [Diff 보기] [다음 파일]"
```

---

### P6: Automated Verification Loop Pattern

**문제**: AI 출력의 품질을 누가 검증하는가?

**해결책**: 자동 검증 + 사람 최종 승인

**3-Stage Verification**:

```
Stage 1: Machine (즉시)
    ↓
Linter, Tests, Type Checker
    ↓
Stage 2: AI Self-Correction (자동)
    ↓
Agent가 에러 수정
    ↓
Stage 3: Human (최종)
    ↓
사람이 Approve
```

**예시 (Cursor)**:
```
Agent: 코드 생성 완료
    ↓
Linter: ❌ "Missing semicolon"
    ↓
Agent: 자동 수정 (Stage 2)
    ↓
Linter: ✅ "No errors"
    ↓
Human: [Diff 확인] → Approve (Stage 3)
```

**YOMAN GAN 모델과 대응**:

| 단계 | 역할 | YOMAN |
|------|------|-------|
| Stage 1 | Syntax check | Linter |
| Stage 2 | Semantic check | REVIEWER (Discriminator) |
| Stage 3 | Business logic | Human (PM/Architect) |

**Match Rate 계산**:
```
Stage 1 Pass Rate: 100% (자동 수정)
Stage 2 Match Rate: 85% (REVIEWER)
Stage 3 Approval:   100% (Human 최종 결정)
```

---

### P7: Multi-Interface Access Pattern

**문제**: 다양한 상황에서 어떻게 AI에 접근하는가?

**해결책**: 여러 인터페이스 제공, Context 동기화

**인터페이스 스펙트럼**:

```
Low Friction ←───────────────────→ High Control
    │                                  │
    │                                  │
Telegram/Slack ──── Web ──── CLI ──── IDE
(빠른 명령)      (관리)   (자동화)  (정밀 편집)
```

**사용 시나리오**:

| 상황 | 인터페이스 | 예시 |
|------|-----------|------|
| 긴급 버그 | Telegram | "TODO #123 우선순위 올려" |
| 전체 현황 | Web | Agents Tab, Bucket 현황 |
| 배포 자동화 | CLI | `yoman deploy TODO #123` |
| 정밀 수정 | IDE | VS Code에서 직접 편집 |

**Context 동기화**:
```
Telegram에서 작업 시작
    ↓
Web에서 진행상황 확인
    ↓
IDE에서 세부 수정
    ↓
CLI로 배포
```

**YOMAN 적용**:
```
Primary:   Telegram (폰만으로 개발)
Secondary: Notion (Bucket 관리)
Fallback:  VS Code (필요시 정밀 작업)
```

---

### P8: Causality Chain Pattern

**문제**: 어떤 입력이 어떤 출력을 만들었는지 어떻게 추적하는가?

**해결책**: 실행 과정을 순서대로 기록

**형식**:

#### List-Based (Replit)
```
Steps I took:
1. Created data schema for thesis entries
2. Built in-memory storage
3. Implemented API routes
4. Connected frontend to backend
```

#### Graph-Based (Complex)
```
User Input (IDEA)
    ↓
RESEARCH (3 alternatives)
    ├→ Alternative A
    ├→ Alternative B (선택)
    └→ Alternative C
         ↓
TODO (구현 계획)
    ↓
PR (코드 생성)
```

**추적 레벨**:

| 레벨 | 대상 | 예시 |
|------|------|------|
| Micro | Function call | `verifyToken()` 호출 |
| Meso | File change | `auth.ts` 수정 |
| Macro | Feature | "JWT 인증 구현" |

**YOMAN Bucket Chain**:
```
IDEA #45: "JWT 인증 추가"
    ↓ (creates)
RESEARCH #78: "3가지 JWT 라이브러리 비교"
    ↓ (derives)
TODO #123: "jose 라이브러리로 구현"
    ↓ (implements)
PR #89: "middleware/auth.ts 추가"
```

**Causality Query**:
```
User: "PR #89가 왜 만들어졌어?"
    ↓
Bot: "TODO #123 때문 → RESEARCH #78에서 jose 선택
     → IDEA #45에서 시작"
```

---

### P9: Progressive Disclosure Pattern

**문제**: 복잡한 정보를 어떻게 단계적으로 제공하는가?

**해결책**: 정보 수준을 계층화하여 필요에 따라 공개

**3-Level Disclosure**:

#### Level 1: Status (1줄)
```
TODO #123: 진행 중 (60%) ⏳
```

#### Level 2: Summary (5줄)
```
TODO #123: JWT 인증 구현 (60%)
- ✅ Middleware 작성 완료
- ⏳ 테스트 작성 중 (3/5)
- ⏹️  문서화 대기
[자세히] [중단] [재시작]
```

#### Level 3: Details (전체)
```
TODO #123: JWT 인증 구현
Created: 2026-01-01 14:23:01
Last Update: 14:45:30

Progress:
├─ ✅ Middleware (auth.ts) +120 -0
├─ ⏳ Tests (auth.test.ts) +45 -0 (3/5 passing)
└─ ⏹️  Docs (README.md) 대기

Logs:
[14:23:01] CODINGBOT 시작
[14:35:12] Middleware 완료
[14:40:00] Test 3개 통과
[14:45:30] Test 4 실패 (디버깅 중)

[Full Diff] [Live Log] [Stop]
```

**사용자 선택**:
- 바쁜 경우: Level 1만 확인
- 문제 발생: Level 3으로 드릴다운
- 일반 확인: Level 2가 적당

**YOMAN Telegram 인터페이스**:
```
User: "현황"
    ↓
Bot (Level 1):
"🔴 PR #89 (검증 실패 60%)
 🟡 TODO #124 (진행 중 40%)
 🟢 TODO #125 (완료)
 [전체 보기]"
    ↓
User: "PR #89"
    ↓
Bot (Level 2):
"PR #89: Auth 리팩토링
 Match Rate: 60% (목표 80%)
 문제: Type 에러 3개
 [수정 지시] [전체 로그]"
```

---

### P10: Metrics-Driven Quality Pattern

**문제**: AI 출력의 "품질"을 어떻게 측정하는가?

**해결책**: 사람이 이해 가능한 지표 정의

**핵심 Metrics**:

#### GitHub Copilot
```
1. Acceptance Rate: 65%
   (AI 제안 → 개발자 수락)

2. Retention Rate: 78%
   (수락한 코드 → 최종 유지)
```

#### Cursor
```
1. Context Usage: 17%~80%
   (리소스 사용량)

2. Auto-fix Rate: 95%
   (Linter 에러 자동 수정)
```

#### YOMAN
```
1. Match Rate: 80% 목표
   (TODO 스펙 ↔ PR 구현)

2. Bucket Transition Rate
   IDEA → RESEARCH: 70% (30% 버려짐)
   RESEARCH → TODO: 80%
   TODO → PR: 90%
   PR → Merged: 80%
```

**Metric 사용 패턴**:

```
실시간 모니터링:
Match Rate < 50% → 즉시 중단, 재작업
Match Rate 50~79% → 수정 후 재검증
Match Rate ≥ 80% → Approve

장기 추적:
주간 Acceptance Rate 추이
월간 Retention Rate 개선
```

---

## 패턴 조합 전략

### 전문가용 (Cursor)
```
P1 (Session Isolation) + P2 (Context %) + P3 (Live Log)
+ P4 (Steering) + P5 (Diff) + P6 (Auto-verification)
= 고속 병렬 개발 워크플로우
```

### 비전문가용 (Replit)
```
P3 (Phase Progress) + P8 (Causality List) + P9 (Progressive)
= 이해하기 쉬운 순차적 워크플로우
```

### 기업용 (Copilot)
```
P1 (Session) + P7 (Multi-interface) + P10 (Metrics)
= 관리 가능한 대규모 협업
```

### YOMAN (All)
```
P1 (Operation 격리) + P2 (Tier별 Budget) + P3 (Live Log)
+ P4 (Bucket 이동) + P5 (Notion Diff) + P6 (GAN)
+ P7 (Telegram Primary) + P8 (Bucket Chain) + P9 (Progressive)
+ P10 (Match Rate)
= Quasi-Non-Causal System
```

---

## 다음 단계

1. 각 패턴의 프로토타입 구현
2. Telegram 인터페이스 목업 제작
3. Notion DB 스키마 설계 (Bucket + Operation)
4. CODINGBOT + REVIEWER 에이전트 스펙 작성

**관련 문서**:
- `05-yoman-implications.md` (프로젝트 적용 상세)
- `docs/YOMAN-PROJECT-OVERVIEW.md` (전체 맥락)
