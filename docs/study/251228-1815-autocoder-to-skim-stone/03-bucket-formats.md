# 03-bucket-formats: 버킷별 포맷 규칙 상세

## 버킷 = 포맷 변환기 개념

```mermaid
flowchart LR
    subgraph Input
        A[정보<br/>(any format)]
    end

    subgraph "Bucket Trigger (AI 재생산)"
        A -->|next_bucket| B{버킷 선택}

        B -->|IDEA| C1[Gemini 2.5 Flash]
        B -->|RESEARCH| C2[Claude 4.5]
        B -->|TODO| C3[Gemini/Claude]

        C1 -->|재생산| D1[간단한<br/>마크다운]
        C2 -->|재생산| D2[JSON +<br/>Mermaid]
        C3 -->|재생산| D3[JSON +<br/>Prerequisites]
    end

    subgraph Output
        D1 --> E1[IDEA 버킷]
        D2 --> E2[RESEARCH 버킷]
        D3 --> E3[TODO 버킷]
    end

    style E1 fill:#ffeb3b
    style E2 fill:#2196f3
    style E3 fill:#4caf50
```

**핵심**: 같은 정보도 버킷에 따라 AI가 다르게 재생산함

## IDEA 버킷 (간단한 아이디어)

### Format Spec

| 항목 | 값 |
|------|-----|
| AI 모델 | Gemini 2.5 Flash |
| 출력 형식 | 마크다운 |
| Temperature | 기본 (0.7) |
| 길이 | 짧게 (핵심만) |

### Output Example

```markdown
# AI 에이전트로 Notion 자동 정리

## 핵심 아이디어
매일 2시에 Notion DB를 자동으로 정리하는 AI 에이전트

## 주요 기능
- Notion API로 DB 조회
- Gemini 2.5 Flash로 카테고리 분석
- 자동 태그 추가 + 중복 제거

## 다음 단계
RESEARCH로 Notion API 조사 필요
```

### 특징
- 핵심만 간결하게
- 유저 코멘트 최우선 반영
- OPERATION 컨텍스트 자동 포함

### When Generated?
- RESEARCH/TODO → IDEA (쪼개기)
- IDEA → _self (명확화)
- 유저가 IDEA 버킷에 직접 생성

## RESEARCH 버킷 (구조화된 리서치)

### Format Spec

| 항목 | 값 |
|------|-----|
| AI 모델 | AWS Bedrock Claude Sonnet 4.5 |
| 출력 형식 | JSON |
| Temperature | 0.3 (안정적) |
| 필수 키 | main_page, child_pages (최소 3개), tags |

### JSON Schema

```json
{
  "main_page": "## 핵심 리서치 질문\n\n답변...",
  "child_pages": [
    {
      "title": "00-overview",
      "content": "# 00-overview\n\n```mermaid\ngraph TB\n...\n```"
    },
    {
      "title": "01-notion-api",
      "content": "# 01-notion-api\n\n## Database API\n..."
    },
    {
      "title": "02-ai-integration",
      "content": "# 02-ai-integration\n\n## Gemini 2.5 Flash\n..."
    }
  ],
  "tags": ["Notion", "AI", "Automation"]
}
```

### 필수 요구사항

#### Child Pages
- **최소 3개, 최대 10개**
- **Numbering**: 00-, 01-, 02-...
- **No placeholder**: "추후 작성" 금지, 반드시 실제 내용

#### Mermaid Diagrams
- **최소 1개** (00-overview에 필수)
- **종류**: flowchart, graph, sequenceDiagram, classDiagram 등
- **목적**: 구조 시각화

#### Code Examples
- **최소화**: 다이어그램 우선 사용
- **10-20줄 이내**: 핵심 로직만

### Quality Validation (규칙 기반)

```python
# bucket_trigger.py:836-882
def validate_research_quality(content):
    # 1. Placeholder 체크
    if "추후 작성" in content or "TODO" in content:
        return False

    # 2. Mermaid 다이어그램 체크 (최소 1개)
    if "```mermaid" not in content:
        return False

    # 3. Child pages 제목 중복 체크
    titles = [child["title"] for child in child_pages]
    if len(titles) != len(set(titles)):
        return False

    return True
```

### When Generated?
- IDEA → RESEARCH (확장)
- TODO → RESEARCH (추가 조사)
- RESEARCH → _self (child pages 확장)

### _self 특별 규칙

```python
# RESEARCH → _self 시
1. 기존 child pages 조회 → 중복 방지
2. 기존 child 업데이트 → 같은 번호 유지
3. 신규 추가 → 다음 번호 (03-, 04-...)
```

**예시**:
```
v1: 00-overview, 01-api, 02-integration
  ↓ 코멘트: "Vector DB 비교 추가"
  ↓ next_bucket=_self
v2: 00-overview, 01-api, 02-integration, 03-vector-db (추가)
```

## TODO 버킷 (소작전 분할 + Prerequisites) - v12

### Format Spec

| 항목 | 값 |
|------|-----|
| AI 모델 | Gemini 2.5 Flash (또는 Claude 4.5?) |
| 출력 형식 | JSON |
| Temperature | 기본 (0.7) |
| 필수 키 | main_page, child_pages (소작전), tags |

### JSON Schema

```json
{
  "main_page": "## 전체 개발 계획\n\n## Prerequisites\n- [ ] PostgreSQL 설치\n- [ ] Notion API key 발급\n- [ ] Gemini API key 발급\n\n이번 사이클에서 구현할 내용들...",
  "child_pages": [
    {
      "title": "00-notion-client",
      "content": "# 00-notion-client\n\n## 구현 내용\n- Notion API 클라이언트\n- Database 조회 함수\n\n## 기술 스택\n- notion-client==2.2.1"
    },
    {
      "title": "01-ai-agent",
      "content": "# 01-ai-agent\n\n## 구현 내용\n- Gemini 2.5 Flash 통합\n- 카테고리 분석 로직\n\n## 기술 스택\n- google-genai==0.1.0"
    }
  ],
  "tags": ["Backend", "AI", "Automation"]
}
```

### Prerequisites (핵심 개념)

#### 위치
- **main_page 상단**
- "## Prerequisites" 헤딩 아래

#### 형식
- **체크박스 리스트** (`- [ ]`)
- Notion에서 체크 가능

#### 내용
- **꼭 필요한 것만**
- 예: API key, 환경 설정, 패키지 설치
- **선택적**: 필요없으면 없을 수도 있음

#### 완료 조건
- **모든 체크박스 체크** (`- [x]`)
- → 아래 서브 페이지들 전부 구현 (CODINGBOT 트리거)

#### Validation

```python
# bucket_trigger.py:can_trigger_dev()
def can_trigger_dev(page_id):
    # 1. Prerequisites 추출
    prerequisites = parse_prerequisites(page)

    # 2. 모두 체크됐는지 확인
    if not all(p.checked for p in prerequisites):
        return False

    # 3. Reviewed=true 확인
    if not page["properties"]["Reviewed"]["checkbox"]:
        return False

    return True
```

### 소작전 분할 원칙 (v12)

#### 크기
- **Cloud Run Job 제한**:
  - 시간 제한: CODINGBOT 25분 + REVIEWER 10분 = 35분 이내
  - 메모리 제한: CODINGBOT 1GiB + REVIEWER 512MiB
  - 코드 규모: ~500줄 이하 권장

#### 순차성
- **00-xxx → 01-yyy → 02-zzz** 순서 실행
- 이전 소작전 완료 후 다음 소작전 시작
- 의존성은 Prerequisites에 명시

#### 독립성
- 하나의 PR로 완결
- 독립적인 테스트 가능
- 이전 소작전 실패해도 다음 소작전은 이해 가능

### When Generated?
- IDEA/RESEARCH → TODO (구현 준비)
- TODO → _self (범위/접근법 조정)

## Format Comparison Table

| 항목 | IDEA | RESEARCH | TODO |
|------|------|----------|------|
| AI 모델 | Gemini 2.5 Flash | Claude 4.5 (AWS) | Gemini/Claude |
| 출력 형식 | 마크다운 | JSON | JSON |
| Temperature | 0.7 | 0.3 | 0.7 |
| 길이 | 짧게 (핵심만) | 중간 (구조화) | 중간 (실행 가능) |
| 필수 요소 | 핵심 아이디어 | main + child (3+) + Mermaid | main + Prerequisites + child |
| 검증 | 없음 | 규칙 기반 | Prerequisites 체크 |
| 다음 단계 | RESEARCH/TODO | TODO/_self | CODINGBOT |

## AI Prompt Pattern (공통)

### 1. 유저 코멘트 최우선
```
"사용자 코멘트 (최우선):
{user_comment}

위 코멘트를 반드시 반영할 것."
```

### 2. OPERATION 컨텍스트
```
"프로젝트 컨텍스트:
{operation_context}

위 컨텍스트를 고려할 것."
```

### 3. NEW_CONTEXT 추출
```
"새로운 기술 결정/제약 사항이 있으면 'NEW_CONTEXT:' 형식으로 출력"

예시:
NEW_CONTEXT: Claude 4.5는 AWS Bedrock credentials 필수
NEW_CONTEXT: TODO 소작전은 Cloud Run Job 크기로 제한
```

### 4. KIM's Notes (시스템 메타정보)
```
KIM's Notes:
- AI 모델: {model}
- Temperature: {temperature}
- 생성 시각: {timestamp}
- Lineage: {lineage}
```

## Summary

**버킷 = 포맷 변환기**:
- IDEA: 간단한 마크다운 (핵심만)
- RESEARCH: JSON + Mermaid (구조화)
- TODO: JSON + Prerequisites (실행 가능)

**AI 역할**:
- 같은 정보도 버킷 스타일로 재생산
- 유저 코멘트 최우선 반영
- OPERATION 컨텍스트 자동 포함
