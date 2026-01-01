# Causal vs Non-Causal LLM 시스템 연구

> LLM의 태생적 한계(Causal, Time-Invariant)를 극복하여 신뢰도를 높이고,
> 사람의 Time-Variant한 삶에 맞추는 시스템 설계

---

## 1. 문제 정의

### 1.1 Causal System의 한계

LLM은 태생적으로 **Autoregressive (Causal)**:

```
토큰 1 → 토큰 2 → 토큰 3 → ... → 토큰 N
   ↓        ↓        ↓              ↓
  과거만   과거만   과거만         과거만
  참조     참조     참조           참조
```

**문제점**:
- 한 번 생성하면 "미래를 보고 수정" 불가
- 실수가 누적됨
- 전체 맥락을 보고 일관성 체크 불가

**비유**:
| 사람 | LLM (Causal) |
|------|--------------|
| 글 쓰고 → 전체 읽고 → 수정 | 글 쓰면서 → 이전만 참고 → 수정 불가 |
| Non-causal 검토 가능 | Causal만 가능 |

### 1.2 Time-Invariant의 한계

| LLM (Time-Invariant) | 사람 (Time-Variant) |
|---------------------|---------------------|
| 시간 개념 없음 | 아침/저녁/주말 |
| 호출할 때만 존재 | 24시간 생활 |
| 항상 동일한 상태 | 에너지 레벨 변화 |
| 마감 개념 없음 | 마감일, 약속 |

**현재 문제**:
```
사람이 LLM의 time-invariant 방식에 맞춰야 함
→ 컴퓨터 앞에 앉아서
→ LLM이 이해하는 방식으로 명령
→ 결과 기다림
→ "컴퓨터 없이 개발" 불가능
```

---

## 2. Causal → Quasi-Non-Causal 해결책

### 2.1 기존 연구 정리

| 접근법 | 논문/방법 | 핵심 아이디어 |
|--------|----------|--------------|
| **Self-Consistency** | Wang et al. (2023) | 다중 샘플링 → 일관된 답변 선택 |
| **RASC** | NAACL 2025 | 샘플 70% 절감 + 정확도 유지 |
| **CoVe** | Meta (2023) | 생성 → 검증 질문 → 독립 답변 → 최종 검증 |
| **GAN 구조** | Generator + Discriminator | 생성 + 판별 분리 |
| **METALM** | MS Research | Bidirectional Encoder + Causal Decoder |

### 2.2 YOMAN 현재 설계: GAN 검증

```
TODO ─────► CODINGBOT (Generator) ─────► PR
                                          │
                                          ▼
                                     REVIEWER (Discriminator)
                                          │
                               ┌──────────┴──────────┐
                               ▼                     ▼
                             PASS                  FAIL
                          (PR 머지)             (재작업)
```

**이게 Quasi-Non-Causal인 이유**:
- CODINGBOT: Causal (순차 생성)
- REVIEWER: "미래"(스펙)를 보고 "과거"(코드)를 평가
- **전체 시스템**: Quasi-Non-Causal

### 2.3 강화 방안

#### 방안 1: Multi-Discriminator

```
PR ────┬────► REVIEWER 1 (스펙 준수)
       │
       ├────► REVIEWER 2 (보안 체크)
       │
       ├────► REVIEWER 3 (성능 체크)
       │
       └────► REVIEWER 4 (테스트 커버리지)
              │
              ▼
         AND 연산 (모두 PASS여야 머지)
```

**장점**:
- 다각도 검증
- 각 Discriminator 전문화
- 단일 실패로 전체 블록

#### 방안 2: Unit 수준 Self-Consistency

```
유닛 스펙 ────┬────► Grok Code ────► 유닛 A1
             │
             ├────► Gemini Flash ──► 유닛 A2
             │
             └────► DeepSeek ──────► 유닛 A3
                    │
                    ▼
              동일한 테스트 실행
                    │
              ┌─────┴─────┐
              ▼           ▼
        다수결 선택    전원 통과 선택
```

**장점**:
- 단일 모델 의존성 제거
- 테스트 통과한 것만 선택
- 다양한 구현 스타일

#### 방안 3: Chain-of-Verification (CoVe) 적용

```
1. CODINGBOT: PR 생성 (draft)
2. VERIFIER: 검증 질문 5개 생성
   - "이 함수가 null 입력을 처리하는가?"
   - "에러 핸들링이 스펙과 일치하는가?"
   - ...
3. INDEPENDENT_CHECKER: 각 질문에 독립 답변
4. FINAL_REVIEWER: 검증 결과 종합 → PASS/FAIL
```

**장점**:
- 자가 검증 능력 활용
- 독립적 판단으로 bias 감소
- 명시적 검증 프로세스

### 2.4 신뢰도 계산 모델

```python
def calculate_reliability(pr, todo):
    """
    Quasi-Non-Causal 신뢰도 계산
    """
    # 1. Multi-discriminator 결과
    checks = {
        'spec': reviewer_spec.check(pr, todo.spec),
        'security': reviewer_security.check(pr),
        'performance': reviewer_perf.check(pr),
        'test_coverage': reviewer_test.check(pr)
    }

    # 2. Self-consistency (Unit 수준)
    if is_unit_level:
        variants = generate_multiple(todo.spec)
        passing = [v for v in variants if run_tests(v)]
        consistency = len(passing) / len(variants)

    # 3. 최종 신뢰도
    all_passed = all(checks.values())
    reliability_score = sum(checks.values()) / len(checks)

    return {
        'pass': all_passed,
        'reliability': reliability_score,
        'consistency': consistency if is_unit_level else None
    }
```

---

## 3. Time-Invariant → Time-Aware 해결책

### 3.1 기존 연구/솔루션 정리

| 솔루션 | 핵심 기능 | YOMAN 적용점 |
|--------|----------|--------------|
| **Temporal.io** | Durable Workflows, Signal/Query | 비동기 Agent 오케스트레이션 |
| **iWF** | 상대적 시간 표현, 스케줄링 | "내일", "2시간 후" 지원 |
| **Trigger.dev** | Async AI tasks, Human-in-loop | 장기 실행 태스크 관리 |

### 3.2 핵심 개념: "사람의 시간에 맞추는 시스템"

#### 현재 vs 목표

```
현재:
  사람 ──[time-invariant 방식]──► LLM
  "컴퓨터 앞에 앉아서 명령하고 기다림"

목표:
  사람 ──[time-variant 방식]──► LLM
  "아무 때나 폰으로 한마디, 나중에 결과 확인"
```

### 3.3 비동기 명령 큐

```
유저 (아침 7시, 출근길):
  "오늘 저녁까지 인증 기능 PR 만들어줘"
                    │
                    ▼
          ┌─────────────────────┐
          │    명령 큐 (Queue)    │
          │  deadline: 18:00     │
          │  priority: normal    │
          └──────────┬──────────┘
                     │
     ┌───────────────┼───────────────┐
     ▼               ▼               ▼
  즉시 시작?     스케줄링?       자원 확보?
     │               │               │
     └───────────────┼───────────────┘
                     ▼
           자동 실행 (백그라운드)
                     │
                     ▼
유저 (저녁 6시, 퇴근길):
  Slack 알림 "PR #123 준비 완료. 리뷰 대기중"
```

### 3.4 Context-Aware 스케줄링

```python
class TimeAwareScheduler:
    def schedule_notification(self, message, urgency):
        user_context = self.get_user_context()

        if user_context.is_busy():  # 회의 중, 집중 모드
            return self.queue_for_later(message)

        if user_context.is_available():  # 여유 시간
            if urgency == "decision_required":
                return self.notify_now(message)
            else:
                return self.batch_notifications()

        if user_context.is_sleeping():  # 야간
            return self.queue_for_morning(message)
```

### 3.5 상대적 시간 표현 지원

| 입력 | 해석 |
|------|------|
| "다음 주 월요일에 배포해줘" | schedule: 2026-01-06 09:00 |
| "점심 먹고 나서 알려줘" | schedule: 12:30-14:00 range |
| "오늘 안으로 가능하면" | deadline: today EOD, priority: normal |
| "급하지 않아, 이번 주 내로" | deadline: friday, priority: low |
| "지금 당장" | immediate, priority: urgent |

### 3.6 사람의 생활 리듬 프로필

```yaml
user_profile:
  work_hours: "09:00-18:00"
  focus_blocks:
    - "10:00-12:00"  # 오전 집중
    - "14:00-16:00"  # 오후 집중
  light_check:
    - "09:00-10:00"  # 출근 후 정리
    - "12:00-14:00"  # 점심 시간
    - "16:00-18:00"  # 마무리 시간
  decision_window: "10:00-12:00"  # 중요 결정은 이 시간에
  emergency_only_after: "21:00"

notification_rules:
  - type: "pr_ready"
    urgency: "normal"
    notify_at: "light_check"
  - type: "decision_required"
    urgency: "high"
    notify_at: "decision_window"
  - type: "build_failed"
    urgency: "urgent"
    notify_at: "immediate"
```

### 3.7 Slack 기반 구현

```
# 비동기 명령
/idea 인증 기능 추가하고 싶음
/schedule 저녁에 리뷰 결과 알려줘
/remind 퇴근할 때 PR 상태 체크

# 시간 지정
/todo "결제 연동" --deadline 금요일
/code "로그인 기능" --when 여유있을때

# 긴급 처리
/urgent 배포 롤백 필요

# 상태 확인 (비동기)
/status --summary  # 한 줄 요약
/status --detail   # 상세 (나중에 읽을 때)
```

---

## 4. 통합 설계: YOMAN Time-Aware Quasi-Non-Causal System

### 4.1 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                    사람 (Time-Variant)                       │
│  - 출근길에 아이디어                                          │
│  - 회의 중엔 무응답                                          │
│  - 저녁에 결과 확인                                          │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Time-Aware Interface Layer                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Slack Bot   │  │ Scheduler   │  │ User Context│         │
│  │ (비동기)    │  │ (Temporal)  │  │ (Calendar)  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Quasi-Non-Causal Engine Layer                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ CODINGBOT   │  │ REVIEWER    │  │ VERIFIER    │         │
│  │ (Generator) │  │(Discrimina.)│  │  (CoVe)     │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                              │
│  ┌─────────────────────────────────────────────────┐        │
│  │ Self-Consistency (Unit 수준 다중 생성)            │        │
│  │ Multi-Discriminator (다각도 검증)                 │        │
│  └─────────────────────────────────────────────────┘        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              UnitService Architecture (3-Tier)               │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐               │
│  │ System  │ ──► │ Module  │ ──► │  Unit   │               │
│  │ (Opus)  │     │(Sonnet) │     │ (Grok)  │               │
│  └─────────┘     └─────────┘     └─────────┘               │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 핵심 차이점 정리

| 기존 LLM 시스템 | YOMAN |
|----------------|-------|
| Causal only | Quasi-Non-Causal (GAN + CoVe) |
| Time-Invariant | Time-Aware (Scheduler + Context) |
| 동기식 | 비동기식 (Queue + Notification) |
| 사람이 맞춤 | 시스템이 맞춤 |
| 컴퓨터 필수 | 폰만으로 가능 |

---

## 5. 구현 우선순위

### Phase 1: Quasi-Non-Causal 기반 (현재)
- [x] GAN 구조 (CODINGBOT + REVIEWER)
- [ ] Multi-Discriminator 추가
- [ ] Unit 수준 Self-Consistency

### Phase 2: Time-Aware 인터페이스
- [ ] Slack 비동기 명령 큐
- [ ] 상대적 시간 표현 파서
- [ ] User Context 연동 (Calendar API)

### Phase 3: 고급 기능
- [ ] Chain-of-Verification 적용
- [ ] 생활 리듬 학습
- [ ] 자동 우선순위 조정

---

## 6. 참고 문헌

### Causal → Non-Causal
- [Self-Consistency](https://www.semanticscholar.org/paper/Self-Consistency-Improves-Chain-of-Thought-in-Wang-Wei/5f19ae1135a9500940978104ec15a5b8751bc7d2) - Wang et al.
- [RASC](https://aclanthology.org/2025.naacl-long.184/) - NAACL 2025
- [Chain-of-Verification](https://arxiv.org/abs/2309.11495) - Meta 2023
- [METALM](https://arxiv.org/abs/2206.06336) - MS Research

### Time-Aware Systems
- [Temporal.io](https://temporal.io/blog/orchestrating-ambient-agents-with-temporal) - Workflow Orchestration
- [iWF Framework](https://medium.com/@qlong/build-reliable-ai-agents-with-iwf-on-temporal-7f1a101e000b) - Reliable AI Agents
- [Trigger.dev](https://trigger.dev/) - Async AI Tasks

---

*생성일: 2026-01-01*
*관련 문서: idea3.txt, YOMAN-PROJECT-OVERVIEW.md*
