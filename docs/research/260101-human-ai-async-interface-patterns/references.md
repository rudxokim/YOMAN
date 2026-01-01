# 참고 자료

## 영상 자료

### Cursor 2.0

1. **Cursor AI Agents Work Like 10 Developers (Cursor VP Live Demo)**
   - 채널: Greg Isenberg
   - 길이: 29:42
   - URL: https://www.youtube.com/watch?v=8QN23ZThdRY
   - 핵심 내용:
     - Lee (Cursor VP)의 실제 사용 사례
     - 20개 에이전트 동시 실행 패턴
     - Context window management (17%~80%)
     - Custom commands & rules
     - Bugbot (Code review bot)
     - Slack/Web/CLI 멀티 인터페이스

2. **Cursor 2.0: What Actually Matters in This Release**
   - 채널: WorldofAI
   - 길이: 9:15
   - URL: https://www.youtube.com/watch?v=43r9OZ1a8nk
   - 핵심 내용:
     - Composer model (4x faster)
     - Multi-agent interface 설계
     - Built-in browser testing
     - Plan mode (background execution)
     - Sandbox terminals

### Replit Agent

3. **How to Build AI Apps with Replit Agent (Step-by-Step Tutorial)**
   - 채널: Data with Måns
   - 길이: 8:31
   - URL: https://www.youtube.com/watch?v=WHRWQ-EQbhc
   - 핵심 내용:
     - Phase-based progress (4단계)
     - Real-time file creation display
     - Asynchronous feature requests
     - 15초 법칙 (instant feedback)
     - One-click deployment
     - "Steps it took" causality list

## 문서 자료

### GitHub Copilot

4. **GitHub Copilot Docs (Context7)**
   - Library ID: `/websites/github_en_copilot`
   - 핵심 내용:
     - Agents Tab (central control)
     - Session management
     - Mid-session steering (1 premium request)
     - Live session log
     - Token usage tracking
     - Diff + Overview
     - VS Code integration

5. **How to build an enterprise LLM application: Lessons from GitHub Copilot**
   - 출처: GitHub Engineering Blog
   - URL: https://github.blog/ai-and-ml/github-copilot/how-to-build-an-enterprise-llm-application-lessons-from-github-copilot/
   - 핵심 내용:
     - A/B experimental platform
     - Tight feedback loops
     - Caching for consistency
     - Key metrics (Acceptance Rate, Retention Rate)
     - Security filters
     - "Neighboring tabs" technique

### 학술 연구

6. **Anthropic Research**
   - URL: https://www.anthropic.com/research
   - 관련 논문:
     - "Tracing the thoughts of a large language model" (내부 상태 추적)
     - "Signs of introspection in large language models" (자기 보고)
     - "Persona vectors" (패턴 모니터링)

7. **arXiv Search Results**
   - Query: "human AI collaboration asynchronous"
   - URL: https://arxiv.org/search/?query=human+AI+collaboration+asynchronous
   - 관련 논문:
     - "AsyncVoice Agent" (실시간 설명, 중단 가능)
     - "TherAIssist" (비동기 클라이언트-실무자 협업)
     - "Human-AI Collaboration for UX Evaluation"

## 주요 인용구

### Lee (Cursor VP)

> "Yesterday I fired off about 20 different agents, 20 different conversations over the day."

> "For each new task to get the best quality out of working with the model, you can just start new chats for each thing that you want to do."

> "The agent can read its own outputs and then just self-correct and fix itself without me having to do anything."

> "I didn't have to go open up the editor. I just kicked it off from Slack on my phone. It was amazing."

### GitHub Engineering Blog

> "We have to design apps not only for models whose outputs need evaluation by humans, but also for humans who are learning how to interact with AI."

> "We cached responses to reduce variability and improve performance."

### Linear's Zero-Bug Policy

> "PM reports a bug. The AI categorizes it. The engineer who's triaging just tells cursor, 'Hey, investigate this and fix it.' It opens up the PR, they fix it, and they accept the change."

### Peter Levels (on NanoBanano)

> "If you're not building a mini startup with Nano Banano today and launching it tomorrow, you're missing the opportunity of a lifetime."

## 패턴 출처 매핑

| 패턴 | 주요 출처 | 보조 출처 |
|------|----------|----------|
| P1: Session Isolation | Cursor VP Demo | Copilot Docs |
| P2: Context Budget | Cursor VP Demo | Copilot Token Tracking |
| P3: Live Progress Log | 모두 공통 | - |
| P4: Steering | Copilot Docs | Replit Demo |
| P5: Diff + Overview | Copilot Docs | Cursor Demo |
| P6: Auto Verification | Cursor VP Demo | GitHub Blog |
| P7: Multi-Interface | Cursor VP Demo | Copilot Docs |
| P8: Causality Chain | Replit Demo | Copilot Diff |
| P9: Progressive Disclosure | Replit Demo | - |
| P10: Metrics-Driven | GitHub Blog | Cursor Demo |

## 관련 개념

### SampleRNN (영감 논문)
- 파일: `/Users/kyungtaekim/Documents/GitHub/DEV/YOMAN/sampleRNN.pdf`
- 저자: Mehri et al.
- 학회: ICLR 2017
- 핵심: Hierarchical temporal modeling
- YOMAN 연결: UnitService 3-tier hierarchy

### Event Sourcing & CQRS
- 관련 연구: `docs/research/260101-event-sourcing-cqrs-causality/`
- 핵심: 이벤트 기반 인과성 추적
- YOMAN 연결: Bucket 전환을 이벤트로 모델링

### Temporal Perception
- 관련 연구: `docs/research/260101-temporal-perception-interface-design/`
- 핵심: 사람의 시간 인식과 인터페이스 설계
- YOMAN 연결: 15초 법칙, Phase-based progress

## 추가 탐색 필요 자료

### 1. Autonomous Agent 인터페이스
- AutoGPT UI/UX 디자인
- LangChain Agent 모니터링 도구
- CrewAI 멀티 에이전트 협업 패턴

### 2. HCI 학회 논문
- CHI (Human-Computer Interaction)
- CSCW (Computer-Supported Cooperative Work)
- UIST (User Interface Software and Technology)

검색 키워드:
- "asynchronous human-AI collaboration"
- "AI agent monitoring interface"
- "explainable AI progress tracking"
- "causality visualization"

### 3. 프로덕션 사례
- Devin AI (Cognition Labs) - 실제 데모 영상
- GitHub Copilot Workspace - 최신 기능
- v0.dev (Vercel) - Iterative refinement UI
- Bolt.new (StackBlitz) - 실시간 코딩 인터페이스

### 4. 기술 블로그
- Cursor Blog (cursor.com/blog)
- Replit Blog (blog.replit.com)
- OpenAI Platform Updates
- Anthropic Engineering Blog

## 메타 노트

**조사 방법론**:
1. YouTube 검색 (실제 사용 데모)
2. Context7 (공식 문서)
3. WebFetch (블로그, 학술 검색)
4. 트랜스크립트 분석 (정확한 인용구)

**조사 한계**:
- Devin AI: 공개 자료 부족 (waitlist 필요)
- AutoGPT: 오래된 정보 (최근 업데이트 미확인)
- 학술 논문: arXiv 직접 접근 실패 (403/404)

**향후 개선**:
- 각 도구의 실제 사용 (Hands-on)
- 사용자 인터뷰 (Pain points 발굴)
- A/B 테스트 (패턴 검증)

---

**작성일**: 2026-01-01
**조사 기간**: 2026-01-01 (1일)
**조사 도구**: YouTube, Context7, WebFetch, Grep
**총 참고 자료**: 7개 (영상 3 + 문서 4)
