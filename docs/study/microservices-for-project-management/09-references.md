# 참고 자료

## 핵심 논문 및 문서

### Microservices 정의

1. **Martin Fowler - Microservices (2014)**
   - URL: https://martinfowler.com/articles/microservices.html
   - 핵심: 마이크로서비스 9가지 특성 정의
   - 인용: "The microservice architectural style is an approach to developing a single application as a suite of small services"

2. **Sam Newman - Building Microservices (2015)**
   - 책: "Building Microservices: Designing Fine-Grained Systems"
   - 핵심: 실전 마이크로서비스 아키텍처 설계
   - 주요 개념: Service Boundaries, Deployment, Testing

3. **Chris Richardson - Microservices Patterns (2018)**
   - URL: https://microservices.io/patterns/
   - 핵심: 44개 마이크로서비스 패턴 정리
   - 주요 패턴: API Gateway, Circuit Breaker, Saga

## 역사적 배경

### SOA vs Microservices

1. **Thomas Erl - SOA Principles (2007)**
   - 책: "SOA: Principles of Service Design"
   - 핵심: Service-Oriented Architecture 원칙
   - SOA vs Microservices 차이점 이해

2. **James Lewis - Microservices (2012)**
   - 발표: "Microservices - Java, the Unix Way"
   - 핵심: "Microservices" 용어 최초 사용
   - Unix Philosophy 적용

## 심리학 연구

### 인지 부하 이론

1. **George A. Miller - The Magical Number Seven (1956)**
   - 논문: "The Magical Number Seven, Plus or Minus Two"
   - 핵심: 작업 기억 용량 7±2개 항목
   - 인용: "The span of absolute judgment and the span of immediate memory impose severe limitations on the amount of information"

2. **John Sweller - Cognitive Load Theory (1988)**
   - 논문: "Cognitive Load During Problem Solving"
   - 핵심: 인지 부하 3가지 유형 (Intrinsic, Extraneous, Germane)

### 사회적 관계 한계

1. **Robin Dunbar - Dunbar's Number (1992)**
   - 논문: "Neocortex size as a constraint on group size in primates"
   - 핵심: 150명 = 안정적 사회적 관계 한계
   - 인용: "There is a cognitive limit to the number of individuals with whom any one person can maintain stable relationships"

### 동기부여 이론

1. **Deci & Ryan - Self-Determination Theory (1985)**
   - 논문: "Intrinsic Motivation and Self-Determination"
   - 핵심: 자율성, 유능감, 관계성
   - 인용: "Autonomy, competence, and relatedness are essential for psychological growth"

2. **Mihaly Csikszentmihalyi - Flow (1975)**
   - 책: "Flow: The Psychology of Optimal Experience"
   - 핵심: 몰입 상태 = 생산성 5배
   - 조건: 명확한 목표, 즉각적 피드백, 도전-능력 균형

## 기업 사례 연구

### Netflix

1. **Adrian Cockcroft - Migrating to Microservices (2014)**
   - 발표: AWS re:Invent 2014
   - URL: https://www.slideshare.net/adrianco/dockercon-state-of-the-art-in-microservices
   - 핵심: 500개 서비스로 분할, Chaos Engineering

2. **Netflix Tech Blog**
   - URL: https://netflixtechblog.com/
   - 주요 글:
     - "Chaos Monkey Released into the Wild" (2012)
     - "Hystrix: Latency and Fault Tolerance" (2012)

### Amazon

1. **Werner Vogels - Eventually Consistent (2008)**
   - 글: "Eventually Consistent - Revisited"
   - 핵심: CAP Theorem, BASE vs ACID
   - 인용: "You build it, you run it"

2. **Amazon Two-Pizza Team**
   - 출처: Jeff Bezos 내부 공지 (2002)
   - 핵심: 5-9명 팀 = 의사소통 최적화
   - 인용: "Communication is terrible" (역설적)

### Spotify

1. **Henrik Kniberg - Spotify Engineering Culture (2014)**
   - 비디오: https://www.youtube.com/watch?v=4GK1NDTWbkY
   - 핵심: Squad, Tribe, Chapter, Guild 모델
   - 주요 개념: Aligned Autonomy

2. **Spotify Engineering Blog**
   - URL: https://engineering.atspotify.com/
   - 주요 글:
     - "Scaling Agile @ Spotify" (2012)
     - "Squad Health Check Model" (2014)

### Uber

1. **Uber Engineering Blog**
   - URL: https://eng.uber.com/
   - 주요 글:
     - "Introducing Domain-Oriented Microservice Architecture" (2020)
     - "Microservice Architecture at Uber" (2016)

## 효율성 연구

### DORA Metrics

1. **Accelerate: State of DevOps Report (2019)**
   - 출처: DevOps Research and Assessment (DORA)
   - URL: https://cloud.google.com/devops/state-of-devops/
   - 핵심 지표:
     - Deployment Frequency
     - Lead Time for Changes
     - Time to Restore Service
     - Change Failure Rate

2. **Nicole Forsgren - Accelerate (2018)**
   - 책: "Accelerate: The Science of Lean Software and DevOps"
   - 핵심: 고성능 팀 특징 분석
   - 데이터: 23,000개 조직 설문

## 관리 방법론

### Agile & Scrum

1. **Ken Schwaber - Scrum Guide (2020)**
   - URL: https://scrumguides.org/
   - 핵심: Sprint, Daily Standup, Retrospective
   - Operation과의 연관성: Sprint = Operation 주기

2. **Henrik Kniberg - Kanban vs Scrum (2009)**
   - 글: "Kanban and Scrum - making the most of both"
   - 핵심: WIP 제한, Pull System
   - Operation 적용: Active Operation WIP 제한

### OKR (Objectives and Key Results)

1. **John Doerr - Measure What Matters (2018)**
   - 책: "Measure What Matters"
   - 핵심: 목표(Objective) + 핵심 결과(Key Results)
   - Operation 적용: Goal + Success Criteria

## 기술 도구

### Container & Orchestration

1. **Docker Documentation**
   - URL: https://docs.docker.com/
   - 핵심: 컨테이너 기술로 독립 배포 가능

2. **Kubernetes Documentation**
   - URL: https://kubernetes.io/docs/
   - 핵심: 컨테이너 오케스트레이션, Auto-scaling

### API Gateway

1. **Netflix Zuul**
   - URL: https://github.com/Netflix/zuul
   - 핵심: 중앙 라우팅, Load Balancing

2. **Kong API Gateway**
   - URL: https://konghq.com/
   - 핵심: Plugin 기반 확장 가능

### Service Mesh

1. **Istio Documentation**
   - URL: https://istio.io/
   - 핵심: Service-to-Service 통신 관리

2. **Linkerd**
   - URL: https://linkerd.io/
   - 핵심: 경량 Service Mesh

## 프로젝트 관리 도구

### Notion

1. **Notion API Documentation**
   - URL: https://developers.notion.com/
   - 핵심: Database API, Webhook

2. **Notion Templates**
   - URL: https://www.notion.so/templates
   - 참고: Project Management Templates

### CLI Tools

1. **Click (Python CLI Framework)**
   - URL: https://click.palletsprojects.com/
   - rxk CLI 구현에 사용 가능

2. **Rich (Python Terminal Formatting)**
   - URL: https://rich.readthedocs.io/
   - Operation 목록 시각화

## 관련 블로그 & 커뮤니티

### 기술 블로그

1. **Martin Fowler's Blog**
   - URL: https://martinfowler.com/
   - 주요 주제: Software Architecture, Refactoring

2. **The NewStack**
   - URL: https://thenewstack.io/
   - 주요 주제: Microservices, Cloud Native

3. **InfoQ**
   - URL: https://www.infoq.com/
   - 주요 주제: Software Development, Architecture

### 커뮤니티

1. **Microservices.io**
   - URL: https://microservices.io/
   - Chris Richardson 운영, 패턴 라이브러리

2. **Reddit - r/microservices**
   - URL: https://www.reddit.com/r/microservices/
   - 실무 경험 공유, Q&A

## 추가 도서

### 아키텍처

1. **Sam Newman - Monolith to Microservices (2019)**
   - 핵심: Strangler Fig Pattern, 점진적 마이그레이션

2. **Chris Richardson - Microservices Patterns (2018)**
   - 핵심: 44개 패턴 카탈로그

### 조직 문화

1. **Gene Kim - The Phoenix Project (2013)**
   - 핵심: DevOps 철학, 3 Ways
   - 소설 형식, 읽기 쉬움

2. **Gene Kim - The Unicorn Project (2019)**
   - 핵심: 개발자 생산성, 5 Ideals

### 프로젝트 관리

1. **David Allen - Getting Things Done (2001)**
   - 핵심: GTD 방법론
   - Operation 관리와 유사점

2. **Cal Newport - Deep Work (2016)**
   - 핵심: 집중 작업 = 생산성
   - Operation 단위 집중과 연관

## 학술 논문

### 소프트웨어 공학

1. **Conway's Law (1968)**
   - 논문: "How Do Committees Invent?"
   - 인용: "Organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations"

2. **Brooks' Law (1975)**
   - 책: "The Mythical Man-Month"
   - 인용: "Adding manpower to a late software project makes it later"

### 인지 과학

1. **Sophie Leroy - Attention Residue (2009)**
   - 논문: "Why Is It So Hard to Do My Work?"
   - 핵심: 작업 전환 시 주의력 잔류 20-30분

2. **Arie Kruglanski - Need for Cognitive Closure (1989)**
   - 논문: "Lay Epistemics and Human Knowledge"
   - 핵심: 불확실성 회피 → 명확한 목표 선호

## 요약

이 문서는 마이크로서비스 철학을 프로젝트 관리에 적용하는 연구의 참고 자료임.

**핵심 자료**:
- Martin Fowler의 Microservices 정의 (2014)
- Miller의 7±2 이론 (1956)
- Dunbar's Number (1992)
- Netflix, Amazon, Spotify 사례
- DORA Metrics (2019)

**추가 학습 경로**:
1. Martin Fowler 블로그 → 마이크로서비스 개념
2. Netflix Tech Blog → 실전 사례
3. Accelerate 책 → 효율성 데이터
4. Spotify Engineering Culture 영상 → 조직 구조

---

**문서 끝**: 전체 리서치 완료
