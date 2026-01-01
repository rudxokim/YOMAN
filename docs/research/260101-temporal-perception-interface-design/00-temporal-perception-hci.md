# Temporal Perception in Interface Design: HCI Research Survey

**연구 날짜**: 2026-01-01
**연구 목적**: YOMAN 프로젝트의 비동기 워크플로우 설계를 위한 시간 지각 이론 및 인터페이스 디자인 패턴 조사
**키워드**: Chronemics, Waiting Time Perception, Asynchronous Interaction, Progress Indicators, Time-based Affordances

---

## Executive Summary

본 연구는 사람의 시간 지각(temporal perception)과 인터페이스 디자인의 관계를 HCI 연구 중심으로 조사함. 핵심 발견:

1. **Response Time Limits** (Nielsen): 0.1초(즉각 반응), 1초(사고 흐름 유지), 10초(주의 집중 한계)
2. **Progress Indicator Types**: Indeterminate(2-10초 작업), Determinate(10초 이상 작업)
3. **Waiting Psychology**: 불확실성 감소와 제어감 제공이 대기 시간 지각을 개선함
4. **Async Pattern**: 비동기 인터랙션은 적절한 피드백과 상태 표시가 필수

**YOMAN 적용**: Bucket System(IDEA→RESEARCH→TODO→PR)의 각 단계별 시간 스케일에 맞는 피드백 메커니즘 설계 필요.

---

## 1. Theoretical Background

### 1.1 Chronemics (시간 사용학)

**정의**: 인간 커뮤니케이션에서 시간 사용과 지각에 관한 연구 분야.

**핵심 개념**:
- **Monochronic Time**: 선형적 시간 인식 (한 번에 하나씩 순차적 처리)
- **Polychronic Time**: 동시다발적 시간 인식 (여러 작업 병렬 처리)

**HCI 적용**:
- Monochronic → Synchronous interfaces (실시간 채팅, 화상회의)
- Polychronic → Asynchronous interfaces (이메일, 티켓 시스템, YOMAN)

### 1.2 Time Perception Psychology

**Duration Perception Factors**:
1. **Uncertainty**: 불확실성이 높을수록 시간이 길게 느껴짐
2. **Idle vs Occupied**: 대기 중 활동이 있으면 시간이 짧게 느껴짐
3. **Expectation**: 예상 시간과 실제 시간의 차이가 만족도 결정
4. **Context**: 작업 긴급도와 사용 맥락이 인내심 영향

**Key Insight**:
> "대기 시간의 절대값보다 **예측 가능성과 피드백**이 사용자 경험에 더 큰 영향을 미침"

---

## 2. HCI Research: Response Time and Feedback

### 2.1 Nielsen's Response Time Guidelines (1993, 2014 재확인)

Jakob Nielsen의 연구 결과는 수십 년간 일관되게 유효함:

| 시간 임계값 | 사용자 지각 | 피드백 요구사항 |
|-------------|-------------|------------------|
| **0.1초** | 즉각 반응 (Direct manipulation 느낌) | 피드백 불필요 |
| **1초** | 사고 흐름 유지 가능 (Flow state 유지) | 약간의 피드백 권장 |
| **10초** | 주의 집중 한계 (Task switching 고려) | 명확한 피드백 필수 + 진행률 표시 |

**출처**: [NN/g - Response Times: The 3 Important Limits](https://www.nngroup.com/articles/response-times-3-important-limits/)

**설계 원칙**:
- 0.1초 이내: 버튼 클릭 반응, 마우스 커서 이동
- 1초 이내: 페이지 전환, 간단한 쿼리 응답
- 10초 이상: 파일 업로드, 복잡한 데이터 처리 → **Progress indicator 필수**

### 2.2 Progress Indicators Research

**출처**: [NN/g - Progress Indicators](https://www.nngroup.com/articles/progress-indicators/)

#### Types of Progress Indicators

1. **Indeterminate (Looped Animation)**
   - **용도**: 2-10초 작업
   - **형태**: 스피너, 원형 애니메이션
   - **효과**: 시스템이 작동 중임을 알림 (불확실성 감소)
   - **예시**: 검색 결과 로딩, API 호출 대기

2. **Determinate (Percent-Done Animation)**
   - **용도**: 10초 이상 작업
   - **형태**: 진행률 바 (선형/원형) + 퍼센트 표시
   - **효과**: 남은 시간 예측 가능 → 사용자 통제감 증가
   - **예시**: 파일 업로드, 대량 데이터 처리

3. **Skeleton Screens** (Modern Pattern)
   - **용도**: 콘텐츠 로딩 시
   - **형태**: 실제 콘텐츠 구조 미리 표시
   - **효과**: 즉각적인 진행감 (전통적 로딩보다 자연스러움)
   - **예시**: Facebook/LinkedIn 피드 로딩

**출처**: [Smashing Magazine - Best Practices for Animated Progress Indicators](https://www.smashingmagazine.com/2016/12/best-practices-for-animated-progress-indicators/)

#### Psychological Effects

**Waiting Time Perception 개선 전략**:
1. **Immediate Visual Feedback**: 사용자 액션 후 즉시 반응 표시
2. **Explanatory Text**: "처리 중입니다..." 대신 "데이터 분석 중..."으로 구체화
3. **Time Estimation**: "약 2분 남음" 같은 예상 시간 제공
4. **Distraction**: 로딩 중 유용한 정보 표시 (팁, 통계 등)

**Nielsen의 핵심 원칙**:
> "Visibility of system status is one of the most important principles in user interface design."

---

## 3. Asynchronous Interaction Patterns

### 3.1 Sync vs Async Communication

| 특성 | Synchronous | Asynchronous |
|------|-------------|--------------|
| **시간 요구** | 실시간 동시 참여 | 각자 편한 시간에 참여 |
| **Cognitive Load** | 높음 (즉시 응답 압박) | 낮음 (숙고 가능) |
| **Feedback Loop** | 즉각적 (<1초) | 지연됨 (분~시간 단위) |
| **예시** | 화상회의, 라이브 채팅 | 이메일, Slack, GitHub Issues |

**YOMAN의 선택**: Asynchronous (폰만으로 개발 가능하려면 비동기 필수)

### 3.2 Async Interface Design Patterns

#### Pattern 1: Notification System

**목적**: 비동기 작업 완료 시 사용자 알림

**구성 요소**:
1. **Push Notification**: 모바일/데스크톱 알림 (Telegram Bot)
2. **In-App Badge**: 읽지 않은 알림 카운터
3. **Notification Center**: 모든 알림 히스토리

**YOMAN 적용**:
- TODO → PR 완료 시 Telegram 알림
- REVIEWER 피드백 시 알림 + match_rate 표시

#### Pattern 2: Status Indicators

**목적**: 현재 작업 상태를 실시간 표시

**상태 타입**:
1. **Idle**: 대기 중
2. **In Progress**: 처리 중 (진행률 표시)
3. **Pending Review**: 검토 대기
4. **Completed**: 완료
5. **Failed**: 실패 (에러 메시지)

**YOMAN 적용**:
- Bucket별 상태 표시 (IDEA: 3개 pending, RESEARCH: 1개 in progress)
- UnitService 계층별 상태 (Tier 3: analyzing, Tier 2: coding, Tier 1: testing)

#### Pattern 3: Timeline/Activity Feed

**목적**: 비동기 작업 흐름의 시간순 기록

**구성 요소**:
1. **Timestamp**: 각 이벤트 발생 시간
2. **Actor**: 누가 했는지 (User/Bot)
3. **Action**: 무엇을 했는지 (created, updated, completed)
4. **Context**: 관련 정보 (코드 diff, 파일명 등)

**YOMAN 적용**:
- 버킷 이동 히스토리 (IDEA → RESEARCH: 2026-01-01 14:30)
- UnitService 작업 로그 (Tier 2 Module Analysis: 5분 소요)

---

## 4. Time-based Affordances Design

### 4.1 Progress Bar Variants

**출처**: Smashing Magazine, NN/g 종합

#### 1. Linear Progress Bar

**사용 시점**: 파일 업로드, 설치 프로세스

**디자인 요소**:
- 진행률 퍼센트 (0-100%)
- 예상 남은 시간 (선택)
- 현재 단계 설명 ("3/5 단계: 의존성 설치 중")

**애니메이션 트릭**:
- 처음에는 느리게 시작 → 나중에 빠르게 (체감 속도 향상)
- 99%에서 멈추지 않기 (불안감 유발)

#### 2. Circular Progress Indicator

**사용 시점**: 작은 영역 내 로딩 표시

**장점**:
- 공간 효율적
- 모바일 친화적
- 명확한 시작/끝 지점

#### 3. Stepped Progress

**사용 시점**: 여러 단계를 거치는 프로세스

**예시**:
```
[완료] 1. 요구사항 분석
[완료] 2. 아키텍처 설계
[진행] 3. 코드 생성      ████████░░ 80%
[대기] 4. 테스트 생성
[대기] 5. 리뷰 요청
```

**YOMAN 적용**: Bucket System 진행 상태

### 4.2 Loading States

#### State 1: Empty State (초기)
```
[ Bucket: IDEA ]
📝 새 아이디어 추가하기
```

#### State 2: Loading State
```
[ Bucket: IDEA → RESEARCH ]
⏳ Gemini Flash로 구조화 중... (2초 예상)
```

#### State 3: Success State
```
[ Bucket: RESEARCH ]
✓ 구조화 완료 (3.2초 소요)
📊 생성된 항목: 5개 섹션, 12개 하위 태스크
```

#### State 4: Error State
```
[ Bucket: RESEARCH ]
❌ 구조화 실패
원인: API 타임아웃
조치: [재시도] [수동 편집]
```

---

## 5. YOMAN Application: Temporal Design Strategy

### 5.1 Bucket System Time Scales

YOMAN의 4단계 버킷 시스템에 시간 지각 이론 적용:

| Bucket | 작업 시간 | Progress Type | Feedback Mechanism |
|--------|-----------|---------------|---------------------|
| **IDEA** | 1-5초 (러프 입력) | Indeterminate Spinner | "저장 중..." |
| **RESEARCH** | 5-30초 (AI 구조화) | Determinate Bar | "Gemini 분석 중... 60%" |
| **TODO** | 10-300초 (실행 가능 변환) | Stepped Progress | "1/3 단계: 의존성 분석" |
| **PR** | 60-600초 (코드 생성+검증) | Multi-Stage Progress | "Tier 2 코딩 완료, Tier 1 테스트 중..." |

### 5.2 UnitService Tier Feedback

3계층 UnitService 구조에 맞는 피드백:

#### Tier 3 (System-level)
- **작업 시간**: 5-30분
- **Feedback**: 단계별 Milestone 표시
- **예시**:
  ```
  [Opus System Analysis]
  ✓ 프로젝트 구조 파악 (2분)
  ✓ 의존성 그래프 생성 (5분)
  ⏳ Operation 분해 중... (8/15 모듈)
  ```

#### Tier 2 (Module-level)
- **작업 시간**: 1-10분
- **Feedback**: 진행률 바 + 현재 파일명
- **예시**:
  ```
  [Sonnet Module Coding]
  ████████░░ 75%
  현재: src/auth/middleware.ts 생성 중
  ```

#### Tier 3 (Unit-level)
- **작업 시간**: 10-60초
- **Feedback**: Spinner + 함수명
- **예시**:
  ```
  [Flash Unit Test]
  ⏳ generateToken() 테스트 케이스 생성 중...
  ```

### 5.3 Async Notification Strategy

**Timeline-based Notification**:

1. **Immediate (<1초)**: 즉시 확인 (버튼 클릭 반응)
2. **Short-term (1-60초)**: In-app status 표시
3. **Medium-term (1-10분)**: Push notification 예약
4. **Long-term (10분+)**: Email/Telegram summary

**YOMAN 예시**:
- TODO 생성 요청 (Telegram) → 즉시 "접수 완료" 응답
- 30초 후 → "RESEARCH 단계 완료, TODO 생성 중" (In-app)
- 5분 후 → "TODO 생성 완료, 12개 태스크 생성됨" (Telegram push)
- 30분 후 CODINGBOT 완료 → "PR #45 생성 완료, 리뷰 대기" (Telegram + Email)

### 5.4 Temporal Affordance Examples

#### Example 1: IDEA → RESEARCH Transition

```
사용자: [Telegram] "JWT 인증 시스템 구현"

즉시 응답 (0.1초):
✓ 아이디어 저장됨 (#IDEA-123)

구조화 진행 (5-30초):
⏳ Gemini Flash 분석 중...
   ├─ 보안 요구사항 추출
   ├─ 아키텍처 패턴 검색
   └─ 하위 태스크 생성

완료 알림 (30초):
✓ RESEARCH 완료
📊 생성 항목:
   - 5개 보안 요구사항
   - 3개 아키텍처 대안
   - 12개 TODO 후보
[RESEARCH 보기] [TODO 생성]
```

#### Example 2: TODO → PR (GAN Verification)

```
사용자: TODO #45 실행 요청

즉시 응답 (0.1초):
✓ CODINGBOT 작업 시작 (#PR-67)

진행 상태 (1-5분):
[████████░░░░░░░░░░░░] 40%
⏳ Tier 2 코딩 중...
   현재: src/auth/jwt.service.ts

완료 알림 (5분):
✓ PR #67 생성 완료
📝 변경 사항:
   - 3개 파일 생성
   - 5개 테스트 추가
   - 의존성: jsonwebtoken@8.5.1

REVIEWER 검증 중 (1-3분):
⏳ Discriminator 분석...
   ├─ TODO 스펙 대조
   ├─ 코드 품질 검사
   └─ 테스트 커버리지 확인

최종 결과 (총 8분):
✓ REVIEWER 승인 (match_rate: 85%)
[머지하기] [변경 요청]
```

---

## 6. Design Recommendations for YOMAN

### 6.1 Core Principles

1. **Always Show System Status**
   - 모든 비동기 작업에 피드백 제공
   - 상태 변화 시 즉시 알림

2. **Match Time Scale to Feedback Type**
   - <10초: Spinner
   - 10-60초: Progress bar
   - 1-10분: Stepped progress + Notification
   - 10분+: Background job + Email summary

3. **Reduce Uncertainty**
   - 예상 시간 표시 (가능한 경우)
   - 현재 단계 설명 ("의존성 분석 중...")
   - 에러 발생 시 명확한 원인과 조치 제시

4. **Support Task Switching**
   - 10초 이상 작업은 백그라운드 처리
   - 완료 시 Notification으로 복귀 유도
   - Activity feed에서 히스토리 확인 가능

### 6.2 Implementation Checklist

**Phase 1: Basic Feedback**
- [ ] Bucket 전환 시 Spinner 표시
- [ ] API 호출 시 "처리 중" 메시지
- [ ] 완료/실패 시 명확한 상태 표시

**Phase 2: Progress Indicators**
- [ ] 10초 이상 작업에 진행률 바 추가
- [ ] UnitService Tier별 진행 상태 표시
- [ ] 예상 시간 계산 및 표시

**Phase 3: Advanced Notifications**
- [ ] Telegram Push 알림 연동
- [ ] In-app Notification center 구현
- [ ] Activity feed 타임라인 구현

**Phase 4: Adaptive Feedback**
- [ ] 사용자별 대기 시간 학습
- [ ] 시간대별 알림 전략 조정 (야간 알림 자제)
- [ ] 긴급도 기반 알림 우선순위

---

## 7. References

### Primary Sources

1. **Nielsen Norman Group (NN/g)**
   - [Response Times: The 3 Important Limits](https://www.nngroup.com/articles/response-times-3-important-limits/)
   - [Progress Indicators](https://www.nngroup.com/articles/progress-indicators/)

2. **Smashing Magazine**
   - [Best Practices for Animated Progress Indicators](https://www.smashingmagazine.com/2016/12/best-practices-for-animated-progress-indicators/)

### Theoretical Background

3. **Chronemics Theory**
   - Hall, E. T. (1983). *The Dance of Life: The Other Dimension of Time*
   - Monochronic vs Polychronic time in communication

4. **Psychology of Waiting**
   - Maister, D. H. (1985). *The Psychology of Waiting Lines*
   - Factors affecting perceived wait time

### HCI Conferences (참고 방향)

5. **CHI (Conference on Human Factors in Computing Systems)**
   - Progress indicator design studies
   - Waiting time perception experiments

6. **UIST (User Interface Software and Technology)**
   - Novel interaction techniques for async systems
   - Temporal affordances in UI

7. **CSCW (Computer-Supported Cooperative Work)**
   - Asynchronous collaboration patterns
   - Temporal coordination in distributed teams

---

## 8. Next Steps for YOMAN

### 8.1 Immediate Actions

1. **프로토타입 구현**
   - Bucket 전환 시 기본 피드백 추가
   - Telegram Bot 응답 시간 측정 및 최적화

2. **User Testing**
   - 대기 시간 지각 실험
   - 피드백 메시지 선호도 조사

3. **성능 벤치마크**
   - 각 버킷별 평균 처리 시간 측정
   - 99 percentile 대기 시간 기준 설정

### 8.2 Research Extensions

1. **Causal vs Non-Causal 시간 모델**
   - SampleRNN의 계층적 시간 스케일 적용
   - 각 Tier별 최적 시간 버퍼 설계

2. **Multimodal Temporal Cues**
   - 시각적 + 청각적 피드백 조합
   - 햅틱 피드백 (모바일 진동)

3. **Adaptive Temporal Design**
   - 사용자별 대기 인내력 학습
   - 컨텍스트 기반 피드백 조정

---

**문서 작성**: 2026-01-01
**다음 업데이트**: 프로토타입 구현 후 사용자 테스트 결과 반영
