# Human-AI 비동기 인터페이스 패턴 조사

## 연구 목적

YOMAN 프로젝트의 핵심 과제인 **Quasi-Non-Causal System**에서 사람의 순차적 피드백과 AI의 병렬 처리 능력을 연결하는 인터페이스 디자인 패턴을 조사함.

## 핵심 질문 3가지

### Q1: AI의 병렬 처리 vs 사람의 순차적 피드백 간극

**문제**: AI는 동시에 여러 작업을 수행할 수 있지만, 사람은 한 번에 하나씩 확인하고 피드백을 제공함. 이 간극을 어떻게 관리하는가?

**발견**:
- **Context Window Management**: AI가 처리할 수 있는 정보량을 사람이 모니터링 (Cursor: 17%~80% 표시)
- **Task Isolation**: 각 작업을 독립적인 세션으로 분리 (Cursor: 새 채팅 권장, GitHub Copilot: 세션별 관리)
- **Quality Metrics**: 사람이 추적 가능한 지표 정의 (Copilot: Acceptance Rate, Retention Rate)

### Q2: Progress Tracking, Interruption, Steering

**문제**: 사람이 어떻게 AI 작업을 추적하고, 중단하며, 방향을 조정하는가?

**발견**:
- **Live Session Log**: 실시간 작업 과정 표시 (Copilot, Cursor, Replit 공통)
- **Steering Mechanism**: 작업 중단 없이 추가 입력 가능 (Copilot: "steering input", Replit: 단계별 피처 추가)
- **Verification Loop**: 자동 검증 + 사람 확인 (Cursor: Linter/Tests 자동 실행, Copilot: Bugbot)

### Q3: Causality 표현 - 입력과 출력의 연결

**문제**: 어떤 사람의 입력이 어떤 AI 출력을 만들었는지 어떻게 추적하는가?

**발견**:
- **Diff + Overview**: 변경사항 요약 + 전체 맥락 (Copilot, Cursor 공통)
- **Step-by-Step Log**: AI가 수행한 단계를 순서대로 나열 (Replit: "data schema → storage → API routes")
- **Token Usage Tracking**: 리소스 사용량으로 작업 범위 가늠 (Copilot)

## 조사 대상

### 1. Cursor 2.0
- Multi-agent parallel execution
- Composer model (4x faster)
- Context window monitoring
- Code review bot (Bugbot)

### 2. GitHub Copilot Agent
- Agents Tab (centralized control)
- Live session log + diff
- Mid-session steering
- Token usage tracking

### 3. Replit Agent
- Phase-based progress (4 stages)
- Real-time file creation display
- Asynchronous feature requests
- "Steps it took" causality list

## 주요 발견

### 공통 패턴

1. **세션 기반 작업 격리**
   - 각 작업을 독립적 세션으로 관리
   - Context 오염 방지
   - 사람이 추적 가능한 단위

2. **실시간 피드백 루프**
   - Live log로 진행상황 표시
   - 자동 검증 (Linter/Tests)
   - 사람 개입 지점 명확화

3. **계층적 정보 표현**
   - Overview (전체 맥락)
   - Diff (구체적 변경)
   - Log (실행 과정)

4. **비동기 방향 조정**
   - 작업 중단 없이 추가 입력
   - "Steering" 개념
   - Feature request queue

### YOMAN 적용 시사점

| YOMAN 개념 | 인터페이스 패턴 |
|-----------|---------------|
| Bucket System | Session-based isolation |
| UnitService Hierarchy | Context window + Task scope |
| GAN Verification | Automated checks + Human review |
| Quasi-Non-Causal | Diff + Step log for causality |

## 문서 구성

- `00-overview.md`: 본 문서 (전체 요약)
- `01-cursor-2.0-analysis.md`: Cursor 상세 분석
- `02-github-copilot-analysis.md`: Copilot 상세 분석
- `03-replit-agent-analysis.md`: Replit 상세 분석
- `04-design-patterns.md`: 공통 패턴 추출
- `05-yoman-implications.md`: YOMAN 프로젝트 적용 방안
- `references.md`: 참고 자료 (논문, 영상, 블로그)

## 다음 단계

1. 각 도구의 인터페이스 스크린샷 분석
2. 공통 패턴 추출 및 분류
3. YOMAN Bucket System에 적용 가능한 메커니즘 설계
4. 프로토타입 인터페이스 스케치

---

**작성일**: 2026-01-01
**조사 범위**: AI 코딩 도구 (Cursor, GitHub Copilot, Replit Agent)
**관련 문서**:
- `docs/research/260101-async-system-temporal-gap/` (이론적 배경)
- `docs/YOMAN-PROJECT-OVERVIEW.md` (프로젝트 총 요약)
- `idea.txt`, `idea2.txt`, `idea3.txt` (핵심 아이디어)
