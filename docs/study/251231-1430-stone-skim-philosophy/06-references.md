# 06. 참고 자료

## 프로젝트 문서

### 핵심 철학
- [CLAUDE.md](../../../CLAUDE.md): skim-stone v0.1 철학 전체
- [docs/guides/architecture.md](../../guides/architecture.md): v12 시스템 아키텍처
- [docs/guides/workflow-guidelines.md](../../guides/workflow-guidelines.md): 워크플로우 가이드

### Operation 문서
- [docs/ops/251228-bucket-trigger/](../../ops/251228-bucket-trigger/): Bucket Trigger operation
- [docs/ops/251228-codingbot/](../../ops/251228-codingbot/): CODINGBOT operation
- [docs/ops/251228-reviewer/](../../ops/251228-reviewer/): REVIEWER operation
- [docs/ops/251228-daily-log/](../../ops/251228-daily-log/): Daily Log Generator operation

### 가이드 문서
- [docs/guides/mcp-setup-guide.md](../../guides/mcp-setup-guide.md): MCP 서버 설치 가이드
- [docs/guides/coding-standards.md](../../guides/coding-standards.md): 개발 규칙
- [docs/guides/gcp-deployment-guide.md](../../guides/gcp-deployment-guide.md): GCP 배포 가이드

---

## MCP 서버

### Memory MCP
- **기능**: 지식 그래프 (entities + relations + observations)
- **사용법**:
  ```python
  memory.search_nodes("BucketTrigger")
  memory.open_nodes(["CODINGBOT", "REVIEWER"])
  memory.create_entities({...})
  memory.create_relations({...})
  ```
- **공식 문서**: [MCP Memory Server](https://github.com/modelcontextprotocol/servers)

### Filesystem MCP
- **기능**: 파일 메타데이터, 인덱싱
- **사용법**:
  ```python
  filesystem.list_directory("/path")
  filesystem.read_text_file("/path/file.py")
  filesystem.directory_tree("/path")
  ```

### Notion MCP
- **기능**: Notion API 호출
- **사용법**:
  ```python
  notion.query_database(database_id, filter)
  notion.retrieve_page(page_id)
  notion.patch_page(page_id, properties)
  ```
- **제한사항**: [docs/guides/notion-api-fallback.md](../../guides/notion-api-fallback.md)

---

## 외부 자료

### MCP (Model Context Protocol)
- [MCP 공식 사이트](https://modelcontextprotocol.io/)
- [MCP GitHub](https://github.com/modelcontextprotocol)
- [MCP Servers Registry](https://github.com/modelcontextprotocol/servers)

### Slack API
- [Slack API Documentation](https://api.slack.com/)
- [slack_sdk Python](https://github.com/slackapi/python-slack-sdk)
- [Slack MCP (확인 필요)](https://github.com/modelcontextprotocol/servers)

### Telegram Bot API
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [python-telegram-bot](https://github.com/python-telegram-bot/python-telegram-bot)

### Notion API
- [Notion API Reference](https://developers.notion.com/reference)
- [notion-client Python](https://github.com/ramnes/notion-sdk-py)

---

## 기술 블로그 & 논문

### 지식 그래프
- [Knowledge Graphs: An Introduction](https://arxiv.org/abs/2003.02320)
- [Building Knowledge Graphs](https://neo4j.com/developer/knowledge-graph/)

### MCP 활용
- [How to use MCP for context caching](https://modelcontextprotocol.io/docs/guides/context-caching)
- [Memory MCP best practices](https://github.com/modelcontextprotocol/servers/tree/main/src/memory)

### GAN (Generator-Discriminator)
- [GAN 원리](https://arxiv.org/abs/1406.2661): 생성자 + 판별자
- [CODINGBOT (Generator) + REVIEWER (Discriminator)](../../guides/architecture.md#gan-검증-흐름)

---

## 도구 & 라이브러리

### Python
- [google-genai](https://pypi.org/project/google-genai/): Gemini 2.5 Flash SDK
- [boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html): AWS Bedrock Claude 4.5
- [notion-client](https://github.com/ramnes/notion-sdk-py): Notion API Python SDK
- [flask](https://flask.palletsprojects.com/): Bucket Trigger HTTP 서버

### GCP
- [Cloud Run](https://cloud.google.com/run/docs): Bucket Trigger, DLG 배포
- [Cloud Run Jobs](https://cloud.google.com/run/docs/create-jobs): CODINGBOT, REVIEWER
- [Secret Manager](https://cloud.google.com/secret-manager/docs): 환경변수 관리

---

## 관련 연구

### skim-stone 리서치
- [docs/research/251228-2050-stone-commands-global-migration/](../../research/251228-2050-stone-commands-global-migration/): stone 커맨드 광역화
- [docs/research/microservices-for-project-management/](../../research/microservices-for-project-management/): Operation 개념 연구

### 자동화 워크플로우
- [docs/research/codingbot-reviewer-flow/](../../research/codingbot-reviewer-flow/): GAN 워크플로우
- [docs/research/gcp-deployment-complete/](../../research/gcp-deployment-complete/): GCP 배포 연구

---

## 추가 학습 자료

### Level 1 (Quick, ~30분)
- [00-overview.md](./00-overview.md) 읽기
- [01-core-philosophy.md](./01-core-philosophy.md) 핵심만

### Level 2 (Standard, ~2시간) - 현재
- 전체 문서 순서대로 읽기 (01 → 02 → 03 → 04 → 05)
- Mermaid 다이어그램 이해

### Level 3 (Deep, ~5시간)
- 프로토타입 구현 (Phase 1: zorba 폴더)
- Memory MCP 테스트
- sync-from-code.py 작성

---

## 문의 & 피드백

### 프로젝트 관련
- GitHub Issues: [anthropics/claude-code/issues](https://github.com/anthropics/claude-code/issues)
- 프로젝트 README: [README.md](../../../README.md)

### MCP 관련
- MCP Discord: [Join MCP Community](https://discord.gg/modelcontextprotocol)
- MCP GitHub Discussions: [MCP Discussions](https://github.com/modelcontextprotocol/servers/discussions)

---

## 다음 단계

### 즉시 실행
1. [05-implementation-roadmap.md](./05-implementation-roadmap.md) Phase 1 시작
2. zorba 폴더 생성
3. entities/services.json 수동 작성

### 1주 내
- sync-from-code.py 프로토타입
- Memory MCP 테스트

### 1달 내
- Phase 1-4 전체 완료
- 통합 테스트 통과

---

## 버전 히스토리

- **v1.0** (2025-12-31): 초기 작성
  - 핵심 철학 정립 (컴퓨터 없애기)
  - docs vs zorba 영역 분리
  - Memory MCP 컨텍스트 최적화
  - Telegram/Slack 역할 구분
  - 4 Phase 로드맵

---

## 라이선스

이 문서는 [_AUTOCODER](../../../) 프로젝트의 일부이며, 프로젝트와 동일한 라이선스를 따릅니다.
