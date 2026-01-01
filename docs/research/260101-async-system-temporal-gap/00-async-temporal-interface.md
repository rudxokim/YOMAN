# 비동기 작업 시스템의 시간 간극 극복 메커니즘

**Research Date**: 2026-01-01
**Tags**: #async #temporal-interface #hci #yoman

---

## Abstract

비동기 협업 시스템(Slack, Notion, Trello, Linear)은 **time-variant한 인간**(순차적 사고, 시간에 민감)과 **time-invariant한 컴퓨터 시스템**(병렬 처리, 시간 독립)이 만나는 인터페이스임.

본 문서는 이들 시스템이 interface layer에서 시간적 간극을 극복하는 세 가지 핵심 메커니즘을 분석함:
1. **순차적 사고 vs 병렬 실행 간극 해결**
2. **인과관계 유지** (notification, threading, status tracking)
3. **시간 지각 기반 UI/UX 패턴**

YOMAN의 Bucket System과 GAN Verification 사이클에 적용 가능한 디자인 원칙 도출이 목표.

---

## 1. Problem Definition: 시간 간극의 본질

### 1.1 Time-Variant Human

**특성**:
- 순차적 사고: 한 번에 하나의 컨텍스트만 처리
- 시간 민감성: "지금", "방금", "오래전"이 의미를 가짐
- 인과적 추론: 원인 → 결과 순서로 이해
- 제한된 working memory: 컨텍스트 전환 시 손실 발생

**예시**:
- Slack 메시지를 읽을 때 "누가 먼저 말했는지"가 중요
- Notion 문서를 열 때 "누가 마지막으로 수정했는지" 궁금함
- Trello 카드를 볼 때 "왜 이 상태가 되었는지" 추적하려 함

### 1.2 Time-Invariant System

**특성**:
- 병렬 처리: 수백 명의 동시 편집을 동시에 처리
- 시간 독립성: 이벤트의 timestamp는 있지만 "지금"의 개념 없음
- 비인과적 처리: 이벤트를 순서 무관하게 처리 가능 (CRDT, OT)
- 무한 메모리: 모든 이벤트 로그 영구 저장

**예시**:
- Notion의 operational transform은 동시 편집을 순서 무관하게 병합
- Linear의 데이터베이스는 모든 상태 변화를 timestamp로 인덱싱
- Trello의 백엔드는 카드 이동을 이벤트 스트림으로 처리

### 1.3 Interface Layer의 역할

**간극 극복 전략**:
- **Sequentialization**: 병렬 이벤트를 사용자에게 순차적으로 제시
- **Causality Reconstruction**: 시스템의 비인과적 이벤트에서 인과관계 복원
- **Temporal Anchoring**: 상대적 시간 표현으로 "지금"의 의미 부여
- **Context Preservation**: 컨텍스트 전환 시 손실 최소화

---

## 2. System-by-System Analysis

### 2.1 Slack: 실시간-비동기 하이브리드

**아키텍처 특성**:
- 실시간 websocket + 비동기 메시지 저장
- 채널 기반 컨텍스트 분리
- threading으로 인과관계 명시화

#### 2.1.1 순차 vs 병렬 간극 해결

**Sequentialization 메커니즘**:

```
[Parallel Events in System]
10:00:01 - User A: "Bug in login"
10:00:02 - User B: "Deploy ready"
10:00:03 - User C: "@A which login?"
10:00:04 - User A: "OAuth flow"

[Sequential Presentation to User]
#bugs 채널:
  10:00:01 A: "Bug in login"
    └─ 10:00:03 C: "@A which login?"
       └─ 10:00:04 A: "OAuth flow"

#deploys 채널:
  10:00:02 B: "Deploy ready"
```

**디자인 결정**:
- **채널 분리**: 병렬 대화를 topic별로 분리해 순차적 이해 가능하게 함
- **threading**: 인과관계 있는 메시지를 tree 구조로 묶음
- **unread markers**: "여기까지 읽음" 기준점 제공

**효과**:
- 사용자는 한 번에 하나의 thread만 추적하면 됨
- 병렬 대화가 있어도 채널별로 순차 처리 가능

#### 2.1.2 인과관계 유지

**Threading 메커니즘**:

```
Parent Message
├─ Reply 1
│  └─ Reply 1.1
├─ Reply 2
└─ Reply 3
```

**디자인 결정**:
- **명시적 reply button**: 인과관계를 사용자가 직접 선언
- **thread view**: 부모 메시지 + 모든 replies를 하나의 컨텍스트로 제시
- **thread summary**: "3 replies" 표시로 숨겨진 인과관계 암시

**인과관계 복원 전략**:
1. **Explicit**: Reply로 명시된 부모-자식 관계
2. **Temporal**: 같은 채널의 시간 순서
3. **Social**: @mention으로 참조된 사용자 관계

#### 2.1.3 시간 지각 UI/UX

**Temporal Indicators**:

| 시간 차이 | 표시 방식 | 이유 |
|----------|----------|------|
| < 1분 | "Just now" | 실시간 느낌 강조 |
| 1-60분 | "15 minutes ago" | 상대적 시간으로 신선도 표현 |
| 1-24시간 | "3 hours ago" | 오늘 안의 이벤트 강조 |
| 1-7일 | "Yesterday", "Monday" | 최근 기억과 연결 |
| > 7일 | "Jan 1" | 절대 시간으로 전환 |

**Presence Indicators**:
- 🟢 Active (< 10분)
- 🟡 Away (10-30분)
- ⚪ Offline (> 30분)

**디자인 결정 이유**:
- 상대 시간은 "신선도"를 전달 (urgent/stale 구분)
- 절대 시간은 "기록"의 의미 (검색, 참조용)
- presence는 "지금 응답 가능한지" 예측 가능하게 함

**Notification 전략**:

```
[Real-time notifications]
- Direct mention (@user)
- Direct message
- Keyword alerts

[Batched notifications]
- Channel activity digest
- Missed messages summary
```

**디자인 결정**:
- 긴급한 것: 즉시 알림 (인터럽트 허용)
- 비긴급: 배치 알림 (컨텍스트 전환 최소화)

---

### 2.2 Notion: 비동기 협업 플랫폼

**아키텍처 특성**:
- Operational Transform 기반 동시 편집
- 블록 단위 버저닝
- 비동기 우선, 실시간은 보조

#### 2.2.1 순차 vs 병렬 간극 해결

**Real-time Cursors의 착각**:

```
[System Reality]
- User A: editing block 5
- User B: editing block 12
- User C: editing block 5
→ 모두 동시에 처리됨

[User Perception]
A와 C의 커서가 겹침 → "순차적으로 편집해야겠네"
```

**디자인 결정**:
- **실시간 커서**: 병렬 편집을 시각화하면서도 충돌 예측 가능하게 함
- **블록 단위 편집**: 문서 전체가 아닌 블록별 독립 편집으로 병렬성 최대화
- **자동 병합**: 충돌 시 시스템이 자동 해결, 사용자는 결과만 봄

**Activity Log의 Sequentialization**:

```
[Parallel Events]
10:00 - A edited "Introduction"
10:01 - B added "Conclusion"
10:02 - C commented on "Introduction"

[Sequential Presentation]
Updates (newest first):
  10:02 C commented on "Introduction"
  10:01 B added "Conclusion"
  10:00 A edited "Introduction"
```

**효과**:
- 사용자는 "최근에 무슨 일이 있었는지" 시간 역순으로 파악
- 병렬 편집이 순차적 스토리로 재구성됨

#### 2.2.2 인과관계 유지

**Version History**:

```
Version 10 (current)
  └─ "Added conclusion" by B
Version 9
  └─ "Fixed typo" by A
Version 8
  └─ "Initial draft" by A
```

**디자인 결정**:
- **스냅샷 저장**: 모든 상태를 완전히 저장 (diff가 아님)
- **사용자 단위**: 누가 변경했는지 명시
- **복원 가능**: 과거 상태로 rollback 가능

**인과관계 복원 전략**:
1. **Version chain**: 변경 순서의 인과관계
2. **Comments**: 논의 → 변경의 인과관계
3. **Links**: 페이지 간 참조 관계

**Comments와 Resolve**:

```
Comment: "이 섹션 애매함" by C
  └─ Reply: "수정했음" by A
    └─ Resolved by A
```

**효과**:
- 논의 → 변경의 인과관계 명시
- "해결됨" 상태로 완료된 논의와 진행 중 구분

#### 2.2.3 시간 지각 UI/UX

**Temporal Indicators**:

| 컨텍스트 | 표시 방식 | 이유 |
|---------|----------|------|
| 페이지 상단 | "Edited 2 hours ago" | 문서 신선도 표시 |
| 블록 hover | "Last edited by A" | 책임자 명시 |
| Activity log | "10:30 AM" | 정확한 시간 기록 |
| Version history | "Jan 1, 10:30" | 복원 위한 절대 시간 |

**Freshness Indicators**:
- **New**: 마지막 방문 이후 새로 생성된 페이지
- **Updated**: 마지막 방문 이후 수정된 페이지
- **Unread comments**: 아직 읽지 않은 댓글

**디자인 결정**:
- 문서 컨텍스트: 상대 시간으로 "얼마나 오래되었는지" 표현
- 기록 컨텍스트: 절대 시간으로 "정확한 순서" 명시
- 사용자별 상태: "내가 마지막으로 본 이후" 기준

**Notification 전략**:

```
[Immediate]
- Direct mention in comment
- Page shared with you

[Digest (daily)]
- Pages updated in workspace
- Comments in subscribed pages
```

---

### 2.3 Trello: 칸반 상태 머신

**아키텍처 특성**:
- 카드 = 상태 머신의 엔티티
- 리스트 = 상태 (state)
- 이동 = 전이 (transition)

#### 2.3.1 순차 vs 병렬 간극 해결

**Kanban Board의 공간적 Sequentialization**:

```
[Parallel Events]
- Card A: TODO → DOING
- Card B: DOING → DONE
- Card C: TODO → DONE (skipped DOING)

[Spatial Presentation]
TODO        DOING        DONE
  (empty)   Card A       Card B
                         Card C
```

**디자인 결정**:
- **공간 = 상태**: 카드의 위치가 현재 상태를 시각적으로 표현
- **리스트 순서**: 왼쪽 → 오른쪽으로 진행 방향 암시
- **카드 순서**: 리스트 내 수직 순서는 우선순위 (top = urgent)

**효과**:
- 병렬로 진행되는 작업들을 공간적으로 분리해 순차 파악 가능
- 한눈에 "무엇이 어디까지 왔는지" 파악

**Activity Feed의 시간적 Sequentialization**:

```
[Feed View]
Today
  10:30 - B moved Card A to DONE
  10:00 - A added Card B to TODO

Yesterday
  15:00 - C commented on Card A
```

**디자인 결정**:
- **시간 역순**: 최신 활동이 맨 위
- **날짜 청킹**: "Today", "Yesterday"로 시간 단위 묶음
- **카드 단위**: 모든 활동을 카드 중심으로 표현

#### 2.3.2 인과관계 유지

**Card History**:

```
Card: "Fix login bug"
  Created: Jan 1, 9:00 by A
  └─ 9:30: A added to "TODO" list
  └─ 10:00: B moved to "DOING"
  └─ 10:15: B added label "urgent"
  └─ 10:30: B moved to "DONE"
  └─ 10:35: C archived card
```

**디자인 결정**:
- **완전한 이벤트 로그**: 모든 상태 변화 기록
- **사용자 명시**: 누가 변경했는지 항상 표시
- **시간 순서**: 인과관계의 기본 축

**인과관계 복원 전략**:
1. **State transitions**: 상태 변화의 순서
2. **Checklists**: 하위 작업의 완료 → 카드 완료 인과관계
3. **Due dates**: 시간 제약 → 긴급도 인과관계

**Checklist와 Progress**:

```
Card: "Deploy v2.0"
□ Write tests
□ Code review
□ Staging deploy
□ Production deploy

Progress: 0/4 (0%)
```

**효과**:
- 순차적 작업 흐름 명시
- 완료 조건 (모든 체크리스트 완료) 인과관계

#### 2.3.3 시간 지각 UI/UX

**Temporal Indicators**:

| 요소 | 표시 방식 | 이유 |
|------|----------|------|
| Due date | "Jan 5 (4 days)" | 미래 시간 앵커 |
| Overdue | 🔴 "Overdue by 2 days" | 긴급도 강조 |
| Activity | "Active 3 hours ago" | 카드 신선도 |
| Created | "Created Jan 1" | 카드 나이 |

**Due Date와 Urgency**:

```
Due Date States:
- 🟢 > 3 days: 여유
- 🟡 1-3 days: 주의
- 🔴 < 1 day: 긴급
- ⚫ Overdue: 초과
```

**디자인 결정**:
- 미래 시간 앵커 (due date)로 긴급도 자동 계산
- 색상 코딩으로 시간 압박 시각화
- "지금부터 얼마나 남았는지" 상대 시간 표현

**Notification 전략**:

```
[Immediate]
- Card assigned to you
- Mentioned in comment
- Due date approaching (< 24h)

[Daily digest]
- Cards moved in subscribed boards
- New cards in subscribed lists
```

---

### 2.4 Linear: 이슈 트래킹

**아키텍처 특성**:
- 이슈 = 상태 그래프의 노드
- 상태 전이 = 명시적 workflow
- Cycles = 시간 청킹 (2주 단위)

#### 2.4.1 순차 vs 병렬 간극 해결

**Cycle 기반 시간 청킹**:

```
[Parallel Issues in System]
- Issue #123: In Progress (Cycle 5)
- Issue #124: Todo (Cycle 5)
- Issue #125: Done (Cycle 4)
- Issue #126: Backlog (no cycle)

[Chunked Presentation]
Cycle 5 (Jan 1 - Jan 14)
  In Progress: #123
  Todo: #124

Cycle 4 (Dec 18 - Dec 31)
  Done: #125

Backlog
  #126
```

**디자인 결정**:
- **Cycle = 시간 단위**: 2주 단위로 작업 묶음
- **명확한 시작/종료**: 각 cycle에 deadline
- **Backlog 분리**: 시간 할당되지 않은 작업 별도 관리

**효과**:
- 병렬 작업들을 시간 단위로 청킹해 순차 파악
- "이번 cycle에 집중" vs "다음 cycle 계획" 명확히 분리

**Updates Stream의 Sequentialization**:

```
[Stream View]
#123: Status changed → In Progress
  └─ Comment by A: "Starting work"
  └─ Priority changed → High
  └─ Assigned to B

#124: Created
```

**디자인 결정**:
- **이슈 단위 그룹핑**: 같은 이슈의 모든 업데이트를 묶음
- **시간 역순**: 최신 업데이트가 맨 위
- **타입별 아이콘**: 상태 변화/댓글/할당 등을 시각적으로 구분

#### 2.4.2 인과관계 유지

**Status Transition Graph**:

```
Backlog → Todo → In Progress → In Review → Done
              ↓                      ↓
            Cancelled              Blocked
```

**디자인 결정**:
- **명시적 상태 그래프**: 가능한 전이만 허용
- **상태 의미**: 각 상태가 명확한 의미 (In Review = PR 생성됨)
- **양방향 전이**: Done → In Progress 가능 (재오픈)

**인과관계 복원 전략**:
1. **State chain**: 상태 전이 순서
2. **Blocking relations**: Issue A blocks Issue B
3. **Parent-child**: Epic → Story → Task 계층
4. **Cycle membership**: 같은 cycle의 이슈들은 시간적 연관성

**Issue Relations**:

```
Epic: "User authentication"
├─ Story: "OAuth integration"
│  ├─ Task: "Google OAuth"
│  └─ Task: "GitHub OAuth"
└─ Story: "JWT tokens"

Issue #100 blocks Issue #101
```

**효과**:
- 계층적 인과관계 (큰 목표 → 작은 작업)
- 블로킹 관계로 순서 의존성 명시

#### 2.4.3 시간 지각 UI/UX

**Temporal Indicators**:

| 요소 | 표시 방식 | 이유 |
|------|----------|------|
| Cycle | "Cycle 5 (4 days left)" | Deadline 압박 시각화 |
| Estimate | "2 points" | 작업 시간 예측 |
| Time in status | "In Progress for 3 days" | 정체 감지 |
| SLA | "🔴 Response overdue" | 서비스 수준 위반 |

**Cycle Progress**:

```
Cycle 5: Jan 1 - Jan 14
[▓▓▓▓▓░░░░░] 50% complete
5/10 issues done
```

**디자인 결정**:
- 진행률 시각화로 "얼마나 남았는지" 직관적 표현
- 완료/미완료 이슈 수로 구체적 수치 제공
- 남은 일수로 긴급도 암시

**Notification 전략**:

```
[Immediate]
- Issue assigned to you
- Mentioned in comment
- Status changed (assigned issues)
- Blocking issue resolved

[Daily digest]
- Cycle summary (start/end)
- Overdue issues
- Team updates
```

**Smart Notifications**:
- **Context-aware**: 내가 관련된 이슈만 알림
- **Batching**: 같은 이슈의 여러 업데이트를 하나로 묶음
- **Priority-based**: High priority 이슈는 즉시 알림

---

## 3. Common Patterns Across Systems

### 3.1 순차 vs 병렬 간극 해결 패턴

#### Pattern 1: Sequentialization by Context

**정의**: 병렬 이벤트를 컨텍스트(채널, 페이지, 보드, cycle)별로 분리해 순차적으로 제시

| 시스템 | 컨텍스트 분리 | 효과 |
|--------|-------------|------|
| Slack | 채널, 스레드 | 대화 topic별 분리 |
| Notion | 페이지, 블록 | 문서 단위 편집 독립 |
| Trello | 보드, 리스트 | 프로젝트/상태별 분리 |
| Linear | Cycle, 상태 | 시간/진행도별 분리 |

**적용 원칙**:
- 컨텍스트 간 독립성 최대화
- 사용자는 한 번에 하나의 컨텍스트만 집중
- 컨텍스트 전환은 명시적 (탭, 링크 클릭 등)

#### Pattern 2: Temporal Chunking

**정의**: 시간 단위로 이벤트를 묶어 "청크" 단위로 제시

```
Today
  10:30 - Event A
  10:00 - Event B

Yesterday
  15:00 - Event C

Last week
  Mon - Event D
```

**적용 사례**:
- Slack: "Today", "Yesterday"
- Notion: Activity log의 날짜 구분
- Trello: Activity feed
- Linear: Cycle 단위 묶음

**효과**:
- 시간 순서를 유지하면서도 소화 가능한 단위로 분할
- "최근 것" vs "오래된 것" 구분 용이

#### Pattern 3: Spatial vs Temporal View

**공간적 뷰**: 현재 상태를 공간으로 표현 (Trello board, Linear board view)
**시간적 뷰**: 변화 과정을 시간으로 표현 (Activity feed, Timeline)

| 질문 | 뷰 타입 | 예시 |
|------|--------|------|
| 지금 상태는? | 공간적 | Trello 보드 |
| 어떻게 바뀌었나? | 시간적 | Card history |
| 누가 작업 중? | 공간적 | 칸반 보드 |
| 언제 완료? | 시간적 | Timeline |

**효과**:
- 두 뷰를 제공해 다른 질문에 최적화
- 사용자가 상황에 맞게 선택

---

### 3.2 인과관계 유지 패턴

#### Pattern 4: Threading / Grouping

**정의**: 인과관계 있는 이벤트를 명시적으로 그룹핑

**구현 방식**:

| 시스템 | 메커니즘 | 인과관계 표현 |
|--------|---------|--------------|
| Slack | Reply button | 부모 메시지 → 자식 replies |
| Notion | Comments | 블록 → 댓글들 |
| Trello | Checklists | 카드 → 하위 작업들 |
| Linear | Parent-child | Epic → Stories → Tasks |

**UI 패턴**:
- 들여쓰기: 계층 표현
- 접기/펼치기: 세부사항 숨김/표시
- 카운터: "3 replies", "2/5 done"

#### Pattern 5: State Machine Visualization

**정의**: 가능한 상태 전이를 명시적으로 표현

**Trello/Linear 예시**:

```
[Trello]
List 순서 = 암묵적 상태 그래프
TODO → DOING → DONE

[Linear]
명시적 상태 그래프
Backlog → Todo → In Progress → Done
```

**효과**:
- 사용자가 "다음 단계"를 예측 가능
- 불가능한 전이 방지 (Done → Backlog 등)

#### Pattern 6: Event Log / Audit Trail

**정의**: 모든 상태 변화를 시간순으로 기록

**공통 요소**:
- Timestamp: 정확한 시간
- Actor: 누가 변경했는지
- Action: 무엇을 했는지
- Before/After: 변경 전후 값

**예시**:

```
[Notion Version History]
v10: "Added conclusion" by B at 10:30
v9: "Fixed typo" by A at 10:00

[Trello Card History]
10:30 - B moved to "DONE"
10:15 - B added label "urgent"
10:00 - B moved to "DOING"
```

**효과**:
- 완전한 인과관계 추적 가능
- "왜 이렇게 되었는지" 역추적
- 책임 소재 명확

---

### 3.3 시간 지각 UI/UX 패턴

#### Pattern 7: Relative vs Absolute Time

**원칙**: 컨텍스트에 따라 상대/절대 시간 선택

| 컨텍스트 | 시간 표현 | 이유 |
|---------|----------|------|
| 최근 활동 | "2 hours ago" | 신선도 전달 |
| 기록/검색 | "Jan 1, 10:30" | 정확한 순서 |
| 긴급도 | "3 days left" | 시간 압박 |
| 기간 | "2 weeks" | 지속 시간 |

**전환 규칙** (대부분의 시스템):
- < 1시간: "N minutes ago"
- 1-24시간: "N hours ago"
- 1-7일: "Yesterday", "Monday"
- > 7일: "Jan 1"

#### Pattern 8: Freshness Indicators

**정의**: "새로운 것" vs "오래된 것" 시각적 구분

**구현 방식**:

| 시스템 | 메커니즘 | 표현 |
|--------|---------|------|
| Slack | Unread badge | 🔴 숫자 |
| Notion | Updated dot | 🔵 점 |
| Trello | Activity indicator | "Active" 라벨 |
| Linear | New badge | "New" 텍스트 |

**효과**:
- 사용자가 "어디를 봐야 할지" 우선순위 제공
- "읽음" 상태 추적으로 개인화

#### Pattern 9: Urgency Cues

**정의**: 시간 압박을 시각적으로 강조

**색상 코딩**:
- 🟢 Green: 여유 (> 3일)
- 🟡 Yellow: 주의 (1-3일)
- 🔴 Red: 긴급 (< 1일)
- ⚫ Black: 초과 (overdue)

**적용 사례**:
- Trello due dates
- Linear cycle deadlines
- Notion reminder
- Slack snooze

**효과**:
- 즉각적인 긴급도 인식
- 시간 관리 행동 유도

#### Pattern 10: Presence Indicators

**정의**: "지금" 누가 온라인인지 표시

| 시스템 | 표시 방식 | 의미 |
|--------|----------|------|
| Slack | 🟢 Active | 응답 가능 |
| Notion | 실시간 커서 | 현재 편집 중 |
| Linear | "Viewing" | 같은 이슈 보는 중 |

**효과**:
- "지금 대화 가능한지" 예측
- 실시간 충돌 방지 (동시 편집)

---

### 3.4 Notification 전략 패턴

#### Pattern 11: Immediate vs Batched

**원칙**: 긴급도에 따라 알림 타이밍 차별화

| 긴급도 | 알림 방식 | 예시 |
|--------|---------|------|
| High | 즉시 푸시 | Direct mention, 할당 |
| Medium | 1시간 배치 | 채널 활동 |
| Low | 일일 다이제스트 | 구독 보드 업데이트 |

**효과**:
- 중요한 것만 즉시 알림 (인터럽트 최소화)
- 비긴급은 배치로 컨텍스트 전환 줄임

#### Pattern 12: Smart Grouping

**정의**: 같은 컨텍스트의 알림을 하나로 묶음

**예시**:

```
[개별 알림 (나쁜 예)]
10:00 - A commented on #123
10:05 - B commented on #123
10:10 - A commented on #123

[그룹 알림 (좋은 예)]
10:10 - 3 new comments on #123
  └─ A, B, A
```

**효과**:
- 알림 피로 감소
- 컨텍스트 유지 (같은 이슈)

#### Pattern 13: Context-Aware Filtering

**정의**: 사용자 관련성에 따라 알림 필터링

**필터 기준**:
- 나에게 할당됨
- 내가 작성함
- 내가 댓글 달음
- 내가 구독함
- 나를 멘션함

**효과**:
- 관련 없는 알림 제거
- Signal-to-noise ratio 향상

---

## 4. Design Principles Summary

### 4.1 간극 극복의 핵심 원칙

**Principle 1: Sequentialize When Possible**
- 병렬 이벤트를 순차적으로 재구성
- 컨텍스트 분리로 동시성 숨김
- 시간 청킹으로 소화 가능한 단위 제공

**Principle 2: Make Causality Explicit**
- Threading/grouping으로 관계 명시
- State machine으로 가능한 전이 표현
- Event log로 완전한 추적 가능

**Principle 3: Use Human Time, Not System Time**
- 상대 시간으로 신선도 전달
- 절대 시간으로 정확한 기록
- 미래 시간 앵커로 긴급도 계산

**Principle 4: Prioritize User Attention**
- Freshness indicators로 새 것 강조
- Urgency cues로 시간 압박 시각화
- Smart notifications로 중요한 것만 알림

**Principle 5: Preserve Context**
- Presence indicators로 실시간성 암시
- Threading으로 대화 흐름 유지
- Spatial view로 현재 상태 직관적 표현

---

## 5. YOMAN Application

### 5.1 Bucket System에 적용

**현재 Bucket 흐름**:

```
IDEA → RESEARCH → TODO → PR
```

**시간 간극 문제**:
- 사용자: 순차적 사고 ("IDEA를 작성했으니 다음은 RESEARCH")
- 시스템: 병렬 처리 (여러 IDEA가 동시에 RESEARCH로 전환)

**적용 패턴**:

#### Pattern: Sequentialization by Bucket

```
[User View]
IDEA Bucket (5 items)
  - "Add OAuth" (2 hours ago)
  - "Fix login bug" (yesterday)
  ...

RESEARCH Bucket (3 items)
  - "OAuth integration study" (in progress)
  ...
```

**디자인 결정**:
- Bucket = 컨텍스트 분리
- Bucket 내 시간 역순 정렬
- "In progress" 상태로 현재 작업 강조

#### Pattern: Causality Chain

```
Item: "Add OAuth"
  Created: IDEA (Jan 1, 9:00)
  └─ Moved to RESEARCH (Jan 1, 10:00)
     └─ Research completed (Jan 1, 12:00)
     └─ Moved to TODO (Jan 1, 12:30)
        └─ PR created (Jan 1, 15:00)
```

**효과**:
- Item의 전체 생애주기 추적
- 각 단계에서 걸린 시간 분석 가능
- "왜 TODO로 갔는지" 인과관계 명확

#### Pattern: Temporal Indicators

| Bucket | 시간 표현 | 의미 |
|--------|----------|------|
| IDEA | "Created 2h ago" | 신선도 |
| RESEARCH | "In progress (3h)" | 소요 시간 |
| TODO | "Ready" | 실행 가능 상태 |
| PR | "Pending review (2 days)" | 대기 시간 |

**Urgency Cues**:
- 🟢 IDEA: 신선 (< 1일)
- 🟡 RESEARCH: 진행 중 (1-3일)
- 🔴 TODO: 긴급 (> 3일 대기)
- ⚫ PR: 정체 (> 1주 머지 안 됨)

---

### 5.2 GAN Verification 사이클에 적용

**현재 흐름**:

```
CODINGBOT (TODO → PR)
    ↓
REVIEWER (PR vs TODO 스펙 비교)
    ↓
match_rate ≥80% → 머지
match_rate <50% → 재작업
```

**시간 간극 문제**:
- 사용자: "PR을 보냈으니 기다려야지" (순차적 기대)
- 시스템: 여러 PR이 동시에 검증됨 (병렬 처리)

**적용 패턴**:

#### Pattern: Verification Pipeline Visualization

```
[Pipeline View]
PR #123: "Add OAuth"
  [▓▓▓▓░░] CODINGBOT (70%)

PR #124: "Fix login"
  [▓▓▓▓▓▓] REVIEWER (100%)
  └─ match_rate: 85% → ✅ Ready to merge

PR #125: "Refactor auth"
  [▓▓▓▓▓▓] REVIEWER (100%)
  └─ match_rate: 45% → ⚠️ Needs rework
```

**효과**:
- 각 PR의 현재 단계 시각화
- 진행률로 "얼마나 남았는지" 표현
- 병렬 검증을 공간적으로 분리

#### Pattern: Feedback Loop Timeline

```
PR #125 Timeline:
  15:00 - Created by CODINGBOT
  15:30 - REVIEWER started (match_rate: 45%)
  15:35 - Feedback: "Missing error handling"
  16:00 - CODINGBOT rework started
  16:30 - Updated PR (match_rate: 82%)
  16:35 - ✅ Approved
```

**효과**:
- GAN 사이클의 반복 과정 시각화
- 각 단계 소요 시간 추적
- 피드백 → 개선의 인과관계 명확

#### Pattern: Quality Metrics Over Time

```
[Chart View]
match_rate over attempts:
Attempt 1: 45% ❌
Attempt 2: 82% ✅

Average time to approval: 1.5 hours
```

**효과**:
- GAN 학습 과정의 "개선" 시각화
- 시스템 성능 추적 (시간이 지나면 더 빨라지는지)

---

### 5.3 UnitService 계층에 적용

**현재 계층**:

```
Tier 3: Opus (System) - 프로젝트 전체
Tier 2: Sonnet (Module) - Operation 관리
Tier 1: Flash/GLM (Unit) - 단일 함수
```

**시간 간극 문제**:
- 사용자: Tier 3의 결정을 순차적으로 기다림
- 시스템: 세 계층이 동시에 병렬 작업

**적용 패턴**:

#### Pattern: Hierarchical Activity Feed

```
[System View]
Tier 3 (Opus): Planning next sprint
  └─ Tier 2 (Sonnet): Implementing "auth" operation
     └─ Tier 1 (Flash): Writing unit tests for login()
     └─ Tier 1 (Flash): Writing unit tests for logout()
```

**효과**:
- 계층 간 인과관계 시각화
- 상위 계층 결정 → 하위 계층 실행 흐름

#### Pattern: Delegation Notifications

```
[Notification to User]
Opus: "Planned 3 operations for Sprint 5"
  └─ Sonnet: "Started operation: auth"
     └─ Flash: "Completed unit test: login()"
```

**효과**:
- 사용자는 중요한 결정만 알림 (Opus)
- 세부 진행은 배치 알림 (Sonnet, Flash)
- 컨텍스트 전환 최소화

#### Pattern: Progress Aggregation

```
[Dashboard]
Sprint 5 Progress: 60%
  Operation "auth": 80%
    Unit "login()": 100% ✅
    Unit "logout()": 100% ✅
    Unit "refresh()": 40% 🔄
  Operation "deploy": 40%
    ...
```

**효과**:
- 상위 계층에서 하위 진행률 집계
- 사용자는 전체 진행 상황만 파악하면 됨

---

### 5.4 멀티모달 인터페이스 (폰 기반) 적용

**Vision** (idea2.txt):
- Telegram/Slack으로 개발 지시
- "컴퓨터 없이 개발하기"

**시간 간극 문제**:
- 사용자: 폰에서 짧은 명령 → 빠른 응답 기대
- 시스템: 복잡한 작업 → 긴 처리 시간

**적용 패턴**:

#### Pattern: Async Command with Status Updates

```
[User via Telegram]
10:00 - "/code Add OAuth to login"

[Bot Response]
10:00 - "✅ Received. Creating TODO..."
10:01 - "📝 Research started (Tier 2: Sonnet)"
10:05 - "🔧 Coding started (Tier 1: Flash)"
10:10 - "✅ PR created: #123"
10:15 - "🤖 REVIEWER checking... (match_rate: 85%)"
10:16 - "✅ Approved! Merge?"
```

**효과**:
- 즉시 응답으로 "받았음" 확인
- 중간 상태 업데이트로 안심
- 최종 결과 알림으로 액션 요청

#### Pattern: Smart Notification Batching

```
[Scenario: 여러 작업 동시 진행]
10:00 - User: "/code Task A"
10:05 - User: "/code Task B"
10:10 - User: "/code Task C"

[Bad: 개별 알림]
10:15 - "Task A: PR created"
10:20 - "Task B: PR created"
10:25 - "Task C: PR created"
→ 3번 인터럽트

[Good: 배치 알림]
10:30 - "✅ 3 PRs created (A, B, C). All pending review."
→ 1번 인터럽트
```

**효과**:
- 폰 알림 피로 최소화
- 컨텍스트 전환 줄임
- 배치로 전체 상황 파악

#### Pattern: Voice/Text 하이브리드

```
[Voice Command]
User: "Add OAuth to login page"

[System Parsing]
→ Intent: code generation
→ Scope: login page
→ Feature: OAuth

[Confirmation (Text)]
Bot: "Got it. Creating TODO for OAuth integration in login.js. Proceed?"

[User]
"Yes" (text) or 👍 (reaction)
```

**효과**:
- 음성으로 빠른 입력
- 텍스트로 정확한 확인
- 양방향 확인으로 오해 방지

---

## 6. Design Recommendations for YOMAN

### 6.1 Interface Layer 설계 원칙

**1. Bucket 간 전환을 명시적으로**

```
[Good]
IDEA: "Add OAuth"
  └─ [Move to RESEARCH] button
     └─ Confirmation: "This will trigger Tier 2 analysis. Continue?"

[Bad]
Automatic transition without user control
```

**이유**: 사용자가 상태 전이를 직접 제어해야 인과관계 이해

**2. Notification을 계층별로 차별화**

| 계층 | 알림 방식 | 예시 |
|------|----------|------|
| Tier 3 | 즉시 푸시 | "Sprint plan ready" |
| Tier 2 | 1시간 배치 | "3 operations completed" |
| Tier 1 | 일일 다이제스트 | "42 unit tests passed" |

**이유**: 중요한 결정만 즉시 알림, 세부사항은 배치

**3. Progress를 다층적으로 표시**

```
[Dashboard]
Overall: 60% (3/5 operations)
  Operation "auth": 80% (4/5 units)
    Unit "login()": 100%
    Unit "logout()": 100%
    Unit "refresh()": 40% ← 현재 작업 중
```

**이유**: 사용자가 원하는 세부 수준으로 zoom in/out 가능

---

### 6.2 Telegram/Slack 봇 UX 설계

**Command Syntax**:

```
/idea <text>           → IDEA bucket에 추가
/research <idea_id>    → RESEARCH 시작
/code <todo_id>        → PR 생성
/status                → 현재 진행 상황
/history <item_id>     → 아이템 히스토리
```

**Response Pattern**:

```
[Immediate Ack]
✅ Command received

[Progress Updates]
🔄 Processing... (30s)
🔄 Almost done... (60s)

[Final Result]
✅ Done! [View PR] [Merge]
```

**Notification Settings**:

```
/notify settings
  [x] Sprint decisions (Tier 3)
  [x] Operation updates (Tier 2)
  [ ] Unit completions (Tier 1)
  [x] PR reviews
```

---

### 6.3 GAN Verification UI 설계

**Real-time Feedback**:

```
[PR Review Dashboard]
PR #123: "Add OAuth"
  CODINGBOT: ✅ Implemented (2 files, 150 lines)
  REVIEWER: 🔄 Checking...
    [▓▓▓▓▓░] 85% (spec matching)

  Feedback:
    ✅ All TODO items implemented
    ✅ Error handling present
    ⚠️ Missing unit tests for edge cases

  Decision:
    match_rate: 85% → ✅ Good enough
    [Merge] [Request rework]
```

**Effect**:
- 실시간 진행 상황
- 구체적 피드백
- 명확한 액션

**Iteration History**:

```
PR #124 Iterations:
  Attempt 1: 45% ❌
    Feedback: "Missing error handling"
  Attempt 2: 82% ✅
    Feedback: "All issues resolved"

Time to approval: 1.5 hours
```

**Effect**:
- GAN 학습 과정 시각화
- 개선 추세 확인

---

## 7. Conclusion

### 7.1 핵심 인사이트

**1. 간극 극복의 본질은 Illusion**
- 시스템은 병렬/비인과적이지만, UI는 순차/인과적 "착각" 제공
- Slack threading, Notion cursors: 실제로는 비동기지만 동시성처럼 보임

**2. 시간은 UI의 핵심 차원**
- 공간적 UI (칸반 보드)만으로는 부족
- 시간적 UI (activity feed, timeline) 필수

**3. Notification은 Attention Management**
- 모든 이벤트를 알리면 피로
- 중요도/긴급도에 따라 차별화

**4. Causality는 Context**
- Threading/grouping으로 "왜"를 설명
- 사용자는 원인 없는 결과를 신뢰 안 함

---

### 7.2 YOMAN 적용 로드맵

**Phase 1: 기본 인터페이스**
- Bucket 간 명시적 전환 UI
- 아이템별 히스토리 로그
- 상대/절대 시간 표시

**Phase 2: 계층적 알림**
- Tier별 알림 전략 구현
- 배치 알림 시스템
- 사용자 알림 설정

**Phase 3: GAN Visualization**
- PR 검증 대시보드
- Iteration history
- match_rate 추이 그래프

**Phase 4: 멀티모달 인터페이스**
- Telegram/Slack 봇
- 음성 명령 파싱
- 스마트 배치 알림

---

### 7.3 추가 연구 필요 영역

**1. 시간 지각의 개인차**
- 어떤 사용자는 즉시 알림 선호, 어떤 사용자는 배치 선호
- 개인화된 notification 전략 필요

**2. 멀티모달 인터페이스의 인지 부하**
- 음성 입력 → 텍스트 확인 사이클의 최적화
- 폰 화면 크기 제약 하 정보 밀도 조정

**3. AI 시스템의 불확실성 표현**
- "85% match" 같은 확률적 결과를 사용자가 어떻게 이해하는지
- "거의 완료" vs "아직 멀었음"의 경계선

**4. 장기 기억과 작업 재개**
- 사용자가 1주 후 돌아왔을 때 컨텍스트 복원 전략
- "당신이 마지막으로 한 일" 요약 제공

---

## References

### Academic
- **SampleRNN**: Hierarchical temporal scales (ICLR 2017)
- **HCI Async Collaboration**: (참조 필요)
- **Notification Fatigue**: (참조 필요)

### Product Documentation
- Slack: Threading, Channels
- Notion: Operational Transform, Real-time Collaboration
- Trello: Kanban Workflow
- Linear: Issue State Machine

### Design Resources
- Nielsen Norman Group: Asynchronous Communication
- UX Collective: Temporal UI Patterns

---

**Document Status**: Initial Draft
**Next Steps**:
- YOMAN prototype 구현 시 각 패턴 적용
- 사용자 테스트로 효과 검증
- 개인화 알림 전략 연구

<!-- end of document -->
