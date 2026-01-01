# YOMAN 프로젝트 적용 방안

## 개요

조사한 인터페이스 패턴을 YOMAN의 Quasi-Non-Causal System에 적용하는 구체적 방안.

**목표**: "폰만으로 개발" - Telegram 인터페이스로 전체 워크플로우 제어

## 핵심 아키텍처 매핑

### Bucket System ↔ Session Isolation

**기존 Bucket 개념**:
```
IDEA (러프) → RESEARCH (구조화) → TODO (실행) → PR (검증)
```

**패턴 적용**:
```
각 Bucket = 독립적 Session
- IDEA Session: Gemini Flash (빠른 브레인스토밍)
- RESEARCH Session: Claude (심층 분석)
- TODO Session: UnitService 계층 (3-tier 병렬)
- PR Session: REVIEWER (검증)
```

**Session 격리 규칙**:
- 하나의 IDEA는 여러 RESEARCH 생성 가능 (1:N)
- 하나의 RESEARCH는 여러 TODO 생성 가능 (1:N)
- 하나의 TODO는 하나의 PR 생성 (1:1)
- PR은 TODO로 롤백 가능 (match_rate < 50%)

### UnitService Hierarchy ↔ Context Budget

**3-Tier Token Budget**:

```
Tier 3 (System Level)
├─ Context: 6,000~8,000 tokens (70~80%)
├─ Model: Claude Opus
├─ Scope: 전체 프로젝트 아키텍처
└─ Duration: 5~10분

Tier 2 (Module Level)
├─ Context: 2,000~4,000 tokens (40~60%)
├─ Model: Claude Sonnet
├─ Scope: 여러 파일 (5~10개)
└─ Duration: 2~5분

Tier 1 (Unit Level)
├─ Context: 500~1,500 tokens (10~30%)
├─ Model: Gemini Flash / GLM
├─ Scope: 단일 함수/클래스
└─ Duration: 30초~1분
```

**동적 할당**:
```python
def allocate_tier(todo_complexity):
    if todo_complexity == "simple":
        return Tier1(model="gemini-flash", budget=1000)
    elif todo_complexity == "moderate":
        return Tier2(model="claude-sonnet", budget=3000)
    else:  # complex
        return Tier3(model="claude-opus", budget=7000)
```

### GAN Verification ↔ Automated Loop

**Generator (CODINGBOT)**:
```
TODO 스펙
    ↓
UnitService (Tier 1/2/3)
    ↓
PR 초안 생성
    ↓
Linter 자동 실행 (P6 패턴)
    ↓
Self-correction (에러 발견 시)
```

**Discriminator (REVIEWER)**:
```
PR 초안
    ↓
TODO 스펙과 비교
    ↓
Match Rate 계산 (0~100%)
    ↓
≥80%: Approve
50~79%: Revision Request
<50%: Reject (TODO 재작업)
```

**Feedback Loop**:
```
Reject → TODO 보완 → CODINGBOT 재시도
Revision → PR 수정 → REVIEWER 재검증
Approve → Human Review → Merge
```

## Telegram 인터페이스 설계

### 기본 명령어

#### 1. Bucket 관리

```
/idea [내용]
→ 새 IDEA 생성 (Gemini Flash)

/research [IDEA #N]
→ IDEA를 RESEARCH로 전환

/todo [RESEARCH #N]
→ RESEARCH를 TODO로 전환

/pr [TODO #N]
→ TODO를 PR로 전환 (자동: CODINGBOT)
```

**예시 대화**:
```
User: /idea JWT 인증 추가

Bot: ✅ IDEA #45 생성됨
     "JWT 인증 추가"
     [→ RESEARCH] [삭제] [편집]

User: [→ RESEARCH] 클릭

Bot: 🔍 RESEARCH #78 생성 중...
     (예상 시간: 2~3분)

Bot: ✅ RESEARCH #78 완료
     "3가지 JWT 라이브러리 비교:
      1. jose (추천)
      2. jsonwebtoken
      3. passport-jwt"
     [→ TODO] [IDEA로 돌아가기]

User: [→ TODO] 클릭

Bot: 📋 TODO #123 생성됨
     "jose 라이브러리로 JWT 미들웨어 구현"
     Complexity: Moderate (Tier 2)
     Estimated: 2~5분
     [시작] [우선순위 변경]

User: [시작] 클릭

Bot: 🛠️  CODINGBOT 작업 시작
     TODO #123 → PR #89
     [Live Log] [중단]
```

#### 2. 현황 조회

```
/status
→ 전체 Bucket 현황 (P9: Level 1)

/status [BUCKET]
→ 특정 Bucket 상세 (P9: Level 2)

/log [TODO #N]
→ 실시간 진행 로그 (P3 패턴)

/diff [PR #N]
→ 변경사항 확인 (P5 패턴)
```

**예시 대화**:
```
User: /status

Bot: 📊 YOMAN 현황 (2026-01-01 15:30)

     IDEA: 3개
     RESEARCH: 1개 (진행 중)
     TODO: 2개 (1개 대기, 1개 진행 중)
     PR: 1개 (검증 실패)

     [전체 보기] [우선순위 순]

User: /status TODO

Bot: 📋 TODO Bucket (2개)

     🟡 #123: JWT 인증 (진행 중 60%)
        Tier 2 | 3분 경과 | 2분 남음

     ⏹️  #124: 에러 로깅 (대기)
        Tier 1 | 예상 1분

     [#123 로그] [#124 시작]

User: /log 123

Bot: 📜 TODO #123 Live Log

     [15:20:01] 🛠️  CODINGBOT 시작 (Tier 2)
     [15:21:30] 📄 middleware/auth.ts 생성 (+120)
     [15:22:45] 📄 tests/auth.test.ts 생성 (+45)
     [15:23:10] ✅ Linter pass (0 errors)
     [15:23:20] ⏳ Tests running (3/5 passed)

     [새로고침] [중단] [닫기]
```

#### 3. Steering (방향 조정)

```
/steer [TODO #N] [지시사항]
→ 진행 중 작업에 추가 지시

/revise [PR #N] [수정 요청]
→ PR에 수정 요청 (REVIEWER 피드백)
```

**예시 대화**:
```
User: /steer 123 PostgreSQL 대신 Redis 사용

Bot: 🔄 TODO #123 방향 조정
     "Redis로 세션 저장소 변경"
     (+1 Operation 비용)

     [15:25:00] 🔄 CODINGBOT 방향 전환
     [15:25:30] 📄 config/redis.ts 생성
     [15:26:00] 🛠️  middleware/auth.ts 수정

     [Live Log] [취소]
```

#### 4. 검증 및 승인

```
/review [PR #N]
→ PR 상세 확인 (P5: Diff + Overview)

/approve [PR #N]
→ Human Approve (최종 승인)

/reject [PR #N] [이유]
→ PR 거부, TODO로 롤백
```

**예시 대화**:
```
User: /review 89

Bot: 🔎 PR #89 검증 결과

     Match Rate: 85% ✅ (목표 80%)

     Overview:
     ✅ JWT 미들웨어 추가 (middleware/auth.ts)
     ✅ 토큰 검증 로직 구현
     ✅ 에러 핸들링 추가
     ✅ 테스트 5개 작성 (모두 통과)

     Files: 2개 변경
     +165 -0 lines

     [Diff 보기] [Approve] [수정 요청]

User: [Diff 보기] 클릭

Bot: 📄 File 1/2: middleware/auth.ts
     +120 -0 lines

     ```diff
     + import jwt from 'jose';
     +
     + export async function authMiddleware(req, res, next) {
     +   try {
     +     const token = req.headers.authorization?.split(' ')[1];
     ...
     ```

     [전체 Diff] [다음 파일] [뒤로]

User: [Approve] 클릭

Bot: ✅ PR #89 승인됨
     TODO #123 완료

     다음 단계:
     - Git merge 실행
     - Notion DB 업데이트
     - 문서 자동 생성

     [Merge 실행] [보류]
```

### Progressive Disclosure 적용

**3-Level 정보 제공** (P9 패턴):

#### Level 1: 한눈에 보기
```
User: /status

Bot: 🟢 2개 | 🟡 1개 | 🔴 1개
     [자세히]
```

#### Level 2: 요약
```
User: [자세히]

Bot: 🟢 TODO #125 완료
     🟢 TODO #126 완료
     🟡 TODO #123 진행 중 (60%)
     🔴 PR #89 검증 실패 (60%)
     [#123] [#89]
```

#### Level 3: 전체 상세
```
User: [#123]

Bot: 📋 TODO #123: JWT 인증
     Status: 진행 중 (60%)
     Created: 2026-01-01 15:20
     Tier: 2 (Module)
     Model: Claude Sonnet

     Progress:
     ✅ middleware/auth.ts (+120)
     ⏳ tests/auth.test.ts (3/5)
     ⏹️  docs/auth.md (대기)

     Context: 3,245 / 4,000 tokens (81%)
     ⚠️  높은 사용량 (품질 주의)

     [Live Log] [Diff] [중단] [재시작]
```

## Notion DB 스키마

### 4개 데이터베이스

#### 1. IDEA Bucket
```
Properties:
- ID (Number): 고유 번호
- Title (Text): 아이디어 제목
- Content (Text): 상세 내용
- Status (Select): New / In Progress / Converted / Archived
- Created (Date): 생성 시각
- → RESEARCH (Relation): 연결된 RESEARCH들
```

#### 2. RESEARCH Bucket
```
Properties:
- ID (Number): 고유 번호
- Title (Text): 연구 주제
- Alternatives (Text): 대안들 (JSON)
- Recommendation (Text): 추천안
- Status (Select): Researching / Completed / Converted
- ← IDEA (Relation): 출처 IDEA
- → TODO (Relation): 연결된 TODO들
- Duration (Number): 소요 시간 (분)
```

#### 3. TODO Bucket
```
Properties:
- ID (Number): 고유 번호
- Title (Text): 작업 제목
- Spec (Text): 상세 스펙 (JSON)
- Complexity (Select): Simple / Moderate / Complex
- Tier (Select): 1 / 2 / 3
- Priority (Number): 우선순위 (1~5)
- Status (Select): Pending / In Progress / Completed / Failed
- ← RESEARCH (Relation): 출처 RESEARCH
- → PR (Relation): 생성된 PR
- Context Budget (Number): 할당된 토큰 수
- Actual Context (Number): 실제 사용 토큰
- Start Time (Date): 시작 시각
- End Time (Date): 완료 시각
- Duration (Number): 실제 소요 시간 (분)
```

#### 4. PR Bucket
```
Properties:
- ID (Number): 고유 번호
- Title (Text): PR 제목
- Files Changed (Number): 변경 파일 수
- Lines Added (Number): 추가된 줄 수
- Lines Removed (Number): 제거된 줄 수
- Match Rate (Number): 일치율 (0~100%)
- Status (Select): Reviewing / Approved / Rejected / Merged
- ← TODO (Relation): 출처 TODO
- Git Branch (URL): 브랜치 링크
- Reviewer Feedback (Text): REVIEWER 의견
- Human Approval (Checkbox): 최종 승인 여부
- Review Time (Date): 검증 시각
- Merge Time (Date): 머지 시각
```

### Causality Chain 추적

**Relation 사용**:
```
IDEA #45
    ↓ (creates)
RESEARCH #78
    ↓ (derives)
TODO #123
    ↓ (implements)
PR #89
```

**Query 예시**:
```
User: /why PR 89

Bot: 🔍 PR #89 생성 이유:

     PR #89: JWT 인증 구현
     ↑ TODO #123 (jose 라이브러리 사용)
     ↑ RESEARCH #78 (3가지 라이브러리 비교)
     ↑ IDEA #45 (JWT 인증 추가)

     전체 소요: 25분
     - IDEA → RESEARCH: 3분
     - RESEARCH → TODO: 1분
     - TODO → PR: 21분

     [IDEA #45] [RESEARCH #78] [TODO #123]
```

## Multi-Interface Strategy (P7)

### Primary: Telegram

**장점**:
- 어디서나 접근 (폰만 있으면 됨)
- 빠른 명령어 입력
- 실시간 알림 (Push notification)
- 대화형 인터페이스

**사용 시나리오**:
- 출퇴근 중: `/status` 확인
- 회의 중: `/steer` 방향 조정
- 긴급 버그: `/idea` → `/research` → `/todo` → `/pr` 빠른 진행
- 승인 대기: Push → `/review` → `/approve`

### Secondary: Notion

**장점**:
- 전체 맥락 파악 (Bucket 시각화)
- 장문 편집 (Spec 작성)
- 관계 탐색 (IDEA → RESEARCH → TODO → PR)
- 히스토리 보존

**사용 시나리오**:
- 주간 리뷰: Bucket별 통계 확인
- 상세 스펙 작성: TODO Spec JSON 편집
- 아키텍처 설계: RESEARCH 대안 비교
- 리포트 생성: 완료된 작업 요약

### Tertiary: VS Code (Fallback)

**장점**:
- 정밀한 코드 편집
- Git 직접 조작
- 디버깅 도구

**사용 시나리오**:
- Match Rate < 80%: 직접 수정
- 복잡한 리팩토링: AI 보조 + 사람 판단
- 긴급 Hot-fix: 빠른 수동 배포

### Context 동기화

**실시간 양방향 동기화**:
```
Telegram 명령어
    ↓
Notion DB 업데이트
    ↓
Git Commit (PR 생성 시)
    ↓
Telegram Push 알림
```

**충돌 해결**:
- Notion에서 직접 편집 → Telegram 알림
- VS Code에서 커밋 → Notion 자동 업데이트
- 마지막 수정 시각 기준 (timestamp)

## 구현 로드맵

### Phase 1: Core Engine (2주)

**목표**: CODINGBOT + REVIEWER 기본 동작

```
Week 1:
- UnitService 3-tier 구현
- Gemini/Claude API 통합
- Context budget 할당 로직

Week 2:
- CODINGBOT 에이전트 (TODO → PR)
- REVIEWER 에이전트 (Match Rate 계산)
- Linter 자동 실행 (P6 패턴)
```

**산출물**:
- `services/unit_service.py` (Tier 1/2/3 클래스)
- `agents/codingbot.py` (Generator)
- `agents/reviewer.py` (Discriminator)

### Phase 2: Notion Integration (1주)

**목표**: 4개 Bucket DB 생성 + CRUD

```
- Notion API 연동
- 4개 DB 스키마 생성 (IDEA/RESEARCH/TODO/PR)
- Relation 설정 (Causality chain)
- Webhook 수신 (변경 감지)
```

**산출물**:
- `integrations/notion_client.py`
- Notion workspace 템플릿

### Phase 3: Telegram Bot (1주)

**목표**: 기본 명령어 구현

```
- Telegram Bot API 연동
- 명령어 파서 (/idea, /status, /log 등)
- Progressive disclosure (버튼 인터페이스)
- Push 알림 (작업 완료, 에러 발생)
```

**산출물**:
- `bot/telegram_handler.py`
- 명령어 문서 (`/help`)

### Phase 4: Live Monitoring (3일)

**목표**: 실시간 진행상황 추적 (P3 패턴)

```
- WebSocket 서버 (실시간 로그 스트리밍)
- Telegram에서 [Live Log] 버튼 구현
- Context budget 실시간 표시
- Phase progress bar (Replit 스타일)
```

**산출물**:
- `monitoring/live_logger.py`
- WebSocket endpoint

### Phase 5: Diff + Review (3일)

**목표**: 변경사항 시각화 (P5 패턴)

```
- Git diff 파싱
- Telegram에서 코드 블록 표시
- Overview 자동 생성 (LLM 요약)
- Approval 워크플로우
```

**산출물**:
- `review/diff_formatter.py`
- `review/overview_generator.py`

### Phase 6: Polishing (3일)

**목표**: UX 개선, 에러 핸들링

```
- 에러 메시지 개선
- 타임아웃 처리 (긴 작업 대기)
- Rate limiting (API 할당량 관리)
- 로깅 및 디버깅 도구
```

**산출물**:
- 테스트 케이스 (pytest)
- 운영 가이드 문서

## 예상 워크플로우 (End-to-End)

### 시나리오: "사용자 인증 기능 추가"

#### 1. IDEA 단계 (1분)
```
[09:00] User (Telegram): /idea 사용자 인증 기능 추가

[09:00] Bot: ✅ IDEA #50 생성됨
            "사용자 인증 기능 추가"
            Model: Gemini Flash
            [→ RESEARCH] [삭제] [편집]

[09:01] User: [→ RESEARCH]
```

#### 2. RESEARCH 단계 (3분)
```
[09:01] Bot: 🔍 RESEARCH #85 생성 중...
            Model: Claude Sonnet
            예상 시간: 2~3분
            [중단]

[09:03] Bot: ✅ RESEARCH #85 완료

            "인증 방식 3가지 비교:

             1. JWT + jose 라이브러리 (추천)
                - Stateless
                - 확장성 높음
                - 토큰 관리 필요

             2. Session + Redis
                - Stateful
                - 간단한 구현
                - Redis 의존성

             3. OAuth2 + passport
                - 표준 프로토콜
                - 복잡한 설정
                - 외부 Provider 필요

             추천: JWT (프로젝트 규모 고려)"

            [→ TODO] [대안 변경] [IDEA로]

[09:04] User: [→ TODO]
```

#### 3. TODO 단계 (0.5분)
```
[09:04] Bot: 📋 TODO #145 생성됨
            "JWT + jose로 인증 미들웨어 구현"

            Complexity: Moderate
            Tier: 2 (Module Level)
            Model: Claude Sonnet
            Context Budget: 3,000 tokens
            예상 시간: 3~5분

            Spec:
            - middleware/auth.ts 생성
            - JWT 토큰 검증 로직
            - Error handling
            - Tests (최소 5개)

            [시작] [우선순위 변경] [Spec 편집]

[09:05] User: [시작]
```

#### 4. CODINGBOT 실행 (5분)
```
[09:05] Bot: 🛠️  CODINGBOT 시작
            TODO #145 → PR 생성 중
            [Live Log] [중단]

[09:05] Bot (Live Log):
            [09:05:01] 🔍 프로젝트 구조 분석
            [09:05:30] 📄 middleware/auth.ts 생성
            [09:06:45] 📄 tests/auth.test.ts 생성
            [09:07:20] ✅ Linter pass (0 errors)
            [09:08:10] ⏳ Tests running...
            [09:09:30] ✅ Tests pass (5/5)
            [09:10:00] 📝 PR #120 생성 완료

            Context used: 2,845 / 3,000 tokens (95%)

            [PR 확인]

[09:10] Bot: ✅ PR #120 생성됨 → REVIEWER 검증 시작
            [Live Review]
```

#### 5. REVIEWER 검증 (1분)
```
[09:10] Bot (Live Review):
            [09:10:05] 🔎 TODO #145 스펙 로드
            [09:10:10] 🔎 PR #120 분석
            [09:10:30] 📊 Match Rate 계산 중...
            [09:10:50] ✅ Match Rate: 88%

            ✅ middleware/auth.ts (100% 일치)
            ✅ Error handling (90% 일치)
            ✅ Tests (80% 일치, 5/5 pass)
            ⚠️  문서 부족 (0%)

            전체: 88% (목표 80% 달성) ✅

[09:11] Bot: 🎉 PR #120 검증 통과!
            Match Rate: 88% ✅

            [Review 상세] [Approve] [수정 요청]

[09:12] User: [Review 상세]
```

#### 6. Human Review (2분)
```
[09:12] Bot: 🔎 PR #120 상세

            📊 Overview:
            ✅ JWT 미들웨어 추가 (middleware/auth.ts)
            ✅ jose 라이브러리 사용
            ✅ 토큰 검증 로직 구현
            ✅ Error handling 추가
            ✅ Tests 5개 작성 (모두 통과)
            ⚠️  README 업데이트 없음 (문서 부족)

            📄 Files: 2개
            +185 -0 lines

            [Diff 보기] [Approve] [문서 추가 요청]

[09:13] User: [문서 추가 요청]

[09:13] Bot: 📝 수정 요청 전달
            "README에 인증 사용법 추가"

            CODINGBOT 재작업 중...

[09:14] Bot: ✅ README 업데이트 완료
            PR #120 업데이트됨

            Match Rate: 95% ✅

            [Review] [Approve]

[09:14] User: [Approve]
```

#### 7. Merge (0.5분)
```
[09:14] Bot: ✅ PR #120 최종 승인

            다음 단계:
            1. Git merge to main
            2. Notion DB 업데이트
            3. 문서 아카이브

            [Merge 실행] [보류]

[09:15] User: [Merge 실행]

[09:15] Bot: 🚀 Merging...
            [09:15:10] ✅ Git merge 완료
            [09:15:20] ✅ Notion 업데이트 완료
            [09:15:30] ✅ 완료!

            📊 Summary:
            전체 소요: 15분
            - IDEA → RESEARCH: 3분
            - RESEARCH → TODO: 1분
            - TODO → PR: 5분
            - REVIEWER: 1분
            - Human Review: 5분

            Causality:
            IDEA #50 → RESEARCH #85 → TODO #145 → PR #120

            [다음 작업 추천] [통계 보기]
```

### 총 소요 시간: 15분

**사람 개입 지점** (5회):
1. IDEA → RESEARCH 전환 (09:01)
2. RESEARCH → TODO 전환 (09:04)
3. TODO 시작 (09:05)
4. 문서 추가 요청 (09:13)
5. 최종 Approve + Merge (09:14~09:15)

**AI 자율 작업** (10분):
- RESEARCH 생성 (3분)
- CODINGBOT 코딩 (5분)
- REVIEWER 검증 (1분)
- 문서 추가 (1분)

**사람 작업** (5분):
- 명령어 입력 (1분)
- Review 확인 (2분)
- 의사 결정 (2분)

## 성공 지표

### 단기 (프로토타입)

1. **Telegram 응답 속도**
   - 명령어 → 봇 응답: < 1초
   - Live Log 업데이트: < 3초

2. **CODINGBOT 품질**
   - Linter pass rate: > 95%
   - Test pass rate: > 90%
   - Match Rate avg: > 75%

3. **전체 워크플로우**
   - IDEA → PR: < 20분 (중간 복잡도)
   - 사람 개입: < 5회
   - 에러율: < 10%

### 장기 (프로덕션)

1. **생산성**
   - 일일 처리 TODO: 10~15개
   - 주간 Merge PR: 50~70개

2. **품질**
   - Match Rate avg: > 85%
   - Human rejection rate: < 15%
   - Rollback rate: < 5%

3. **만족도**
   - "폰만으로 개발" 달성: 80% 작업
   - Telegram 사용률: > 70%
   - Notion 접속: 주 1~2회 (리뷰용)

---

**다음 단계**:
1. Phase 1 구현 시작 (UnitService + CODINGBOT)
2. Notion workspace 템플릿 제작
3. Telegram bot 프로토타입 테스트

**관련 문서**:
- `docs/YOMAN-PROJECT-OVERVIEW.md` (프로젝트 총 요약)
- `docs/research/251231-1500-yoman-paper/` (학술 논문)
- `idea.txt`, `idea2.txt`, `idea3.txt` (핵심 아이디어)
