# 08. References (내부 문서 및 용어 정리)

## 8.1 프로젝트 내부 문서

### 핵심 문서

| 문서 | 경로 | 내용 |
|------|------|------|
| **아이디어 원본** | `idea.txt` | UnitService Architecture 초기 아이디어 |
| **인터페이스 비전** | `idea2.txt` | 멀티모달 소통 채널 구상 |
| **Causal 고민** | `idea3.txt` | Non-causal system 질문 |
| **프로젝트 개요** | `docs/YOMAN-PROJECT-OVERVIEW.md` | 전체 아키텍처 정리 |
| **논문 초안** | `docs/research/251231-1500-yoman-paper/` | 학술 논문 형식 |

### 관련 Study 문서

| 문서 | 경로 | 내용 |
|------|------|------|
| **skim-stone 철학** | `docs/study/251228-1815-autocoder-to-skim-stone/` | 물수제비 메타포 상세 |
| **컴퓨터 없애기** | `docs/study/251231-1430-stone-skim-philosophy/` | 사람-로봇 분리 철학 |
| **마이크로서비스** | `docs/study/microservices-for-project-management/` | 심리학적 근거 |

### 영감 자료

| 자료 | 경로 | 내용 |
|------|------|------|
| **SampleRNN 논문** | `sampleRNN.pdf` | 원본 PDF |

---

## 8.2 용어 정리 (Glossary)

### A

**Architecture (아키텍처)**
시스템의 전체 구조와 컴포넌트 간 관계. YOMAN에서는 Two-Layer Architecture (Interface + Engine).

### B

**Bucket (버킷)**
정보 정제 파이프라인의 각 단계. IDEA, RESEARCH, TODO, PR의 4가지 버킷이 존재.
각 버킷은 특정 포맷으로 정보를 변환하는 "Format Transformer" 역할.

### C

**Causal System (인과 시스템)**
출력이 현재와 과거 입력에만 의존하는 시스템. LLM은 정의상 Causal System.
$y_t = f(x_{t-k}, \ldots, x_t)$

**Clock Rate**
각 계층이 동작하는 빈도. SampleRNN에서 차용한 개념.
- System: Operation당 1회
- Module: Operation당 ~10회
- Unit: Operation당 ~100회

**CODINGBOT**
Generator 역할을 하는 AI 에이전트. TODO 스펙을 받아 실제 코드(PR)를 생성.

**Conditioning**
상위 계층이 하위 계층에 컨텍스트를 제공하는 메커니즘.
System → Module → Unit 방향으로 conditioning 발생.

**Context Window**
LLM이 한 번에 처리할 수 있는 토큰 수. YOMAN은 유닛 단위로 쪼개 이 한계 극복.

### D

**Discriminator**
GAN에서 진짜/가짜를 판별하는 네트워크. YOMAN에서는 REVIEWER가 이 역할.

### E

**Engine Layer**
YOMAN의 내부 처리 레이어. UnitService Architecture로 구현.
사람에게 숨겨지며, AI 능력에 최적화됨.

### F

**Format Transformer**
버킷의 역할. 입력을 특정 출력 포맷으로 변환.
예: IDEA (마크다운) → RESEARCH (JSON + Mermaid)

### G

**GAN (Generative Adversarial Network)**
Generator와 Discriminator가 경쟁하며 학습하는 아키텍처.
YOMAN은 이 패턴을 코드 검증에 적용 (적대적 학습은 미사용).

**Generator**
GAN에서 데이터를 생성하는 네트워크. YOMAN에서는 CODINGBOT이 이 역할.

### I

**IDEA**
첫 번째 버킷. 러프한 아이디어를 간단한 마크다운으로 정리.

**Interface Layer**
YOMAN의 사용자 대면 레이어. Bucket System으로 구현.
사람의 인지에 최적화됨.

### K

**Knowledge Graph**
엔티티와 관계로 구성된 지식 표현. Memory MCP로 구현.
코드를 직접 읽는 대신 이를 조회해 96% 토큰 절감.

### M

**match_rate**
REVIEWER가 계산하는 품질 점수. 0~1 범위.
$\text{match\_rate} = 0.3 \cdot F_{\text{files}} + 0.2 \cdot F_{\text{prereq}} + 0.5 \cdot F_{\text{AI}}$

**Memory MCP**
Model Context Protocol 기반 지식 그래프 저장소.
entities/, relations/ 폴더에 JSON으로 저장.

**Miller's Law**
단기 기억의 7±2 항목 한계. YOMAN의 태스크 분할 근거.

**Module Level**
UnitService의 중간 계층. Claude Sonnet이 담당.
컴포넌트 API 설계, 모듈 경계 정의.

### N

**Non-Causal System (비인과 시스템)**
출력이 미래 입력에도 의존할 수 있는 시스템.
$y_t = f(x_{t-k}, \ldots, x_t, \ldots, x_{t+k})$
LLM으로는 직접 구현 불가능.

### O

**Operation**
마이크로서비스 개념을 프로젝트 관리에 적용한 작업 단위.
단일 책임, 독립 완료 가능, 느슨한 결합.

### P

**PR (Pull Request)**
마지막 버킷. 검증된 코드 변경사항. match_rate와 함께 제공됨.

**Prerequisites**
TODO 실행 전 만족해야 하는 조건들.
파일 존재, 패키지 설치, 환경 변수 등.

**Progressive Refinement**
정보가 단계를 거치며 점점 정제되는 원칙.
IDEA(러프) → RESEARCH(구조화) → TODO(실행 가능) → PR(검증됨)

### Q

**Quasi-Non-Causal**
Causal System에서 Non-Causal과 기능적으로 동등한 효과를 내는 접근.
Generator-Discriminator + 반복으로 구현.

**Quantum Verification (양자 검증)**
여러 모델로 동시에 생성 후, 테스트 통과한 최적 버전 선택.
GLM-4.7, Gemini Flash, Gem 3.0 등 사용.

### R

**REPORT**
REVIEWER가 생성하는 검증 결과.
match_rate, breakdown, issues, decision 포함.

**RESEARCH**
두 번째 버킷. 아이디어를 구조화된 분석으로 변환.
JSON + Mermaid 다이어그램 출력.

**REVIEWER**
Discriminator 역할을 하는 AI 에이전트.
PR을 TODO 스펙과 비교해 match_rate 계산.

### S

**SampleRNN**
ICLR 2017 논문. 계층적 시간 스케일 개념의 원천.
Tier 1/2/3 구조가 YOMAN의 Unit/Module/System에 매핑.

**Self-Refinement Loop (_self)**
버킷이 자기 자신으로 다시 처리되는 루프.
v1 → v2 → v3 ... 무한 정제 가능.

**Skipping Stone (물수제비)**
YOMAN의 핵심 메타포. 한 번에 도달하지 않고 여러 번 튀며 전진.

**System Level**
UnitService의 최상위 계층. Claude Opus가 담당.
아키텍처 결정, Cross-module 조율, 문서 구조화.

### T

**TODO**
세 번째 버킷. 실행 가능한 스펙.
Prerequisites + Sub-operations으로 구성.

**Truncated BPTT**
SampleRNN에서 사용. 짧은 시퀀스로도 긴 의존성 학습.
YOMAN에서는 작은 컨텍스트로 작업하는 것에 해당.

**Two-Layer Architecture**
YOMAN의 핵심 구조. Interface Layer + Engine Layer.
자동차의 대시보드 vs 엔진에 비유.

### U

**Unit Level**
UnitService의 최하위 계층. Flash/GLM이 담당.
단일 함수/클래스 구현. 병렬 처리 가능.

**UnitService Architecture**
Engine Layer의 구현. 3계층 구조 (System/Module/Unit).
SampleRNN의 계층적 시간 스케일 개념 차용.

### Y

**YOMAN**
Your Omniscient Manager for Autonomous Navigation.
본 프로젝트의 이름.

### Z

**zorba-the-robot**
AI 전용 영역을 나타내는 폴더명.
`-zorba-the-robot/`에 src, tests, entities 등 위치.
대시(-) 접두사로 "로봇 영역" 표시.

**Zeigarnik Effect**
미완성 작업이 더 잘 기억되는 현상.
작은 유닛으로 빠르게 완료해 스트레스 최소화.

---

## 8.3 약어 정리

| 약어 | 전체 | 설명 |
|------|------|------|
| BPTT | Backpropagation Through Time | RNN 학습 방법 |
| GAN | Generative Adversarial Network | 생성적 적대 신경망 |
| LLM | Large Language Model | 대규모 언어 모델 |
| MCP | Model Context Protocol | 컨텍스트 프로토콜 |
| PR | Pull Request | 코드 병합 요청 |
| STT | Speech-to-Text | 음성→텍스트 |
| TTS | Text-to-Speech | 텍스트→음성 |

---

## 8.4 수식 요약

### match_rate 계산
$$\text{match\_rate} = 0.3 \cdot F_{\text{files}} + 0.2 \cdot F_{\text{prereq}} + 0.5 \cdot F_{\text{AI}}$$

### Causal System
$$p(x_t | x_{<t}) = f_\theta(x_1, x_2, \ldots, x_{t-1})$$

### SampleRNN Conditioning
$$c^{(k)}_{(t-1) \cdot r + j} = W_j h_t, \quad 1 \leq j \leq r$$

### Context 절감률
$$\text{절감률} = 1 - \frac{\text{Knowledge Graph tokens}}{\text{Full code tokens}} = 1 - \frac{350}{8500} \approx 96\%$$

---

## 8.5 버전 이력

| 버전 | 날짜 | 변경 내용 |
|------|------|----------|
| v1.0 | 2025-12-31 | 초기 Study 문서 생성 |

---

*학습 완료! [00-overview.md](00-overview.md)로 돌아가서 학습 목표 체크하기*
