# 08-references: 참고 문서 및 링크

## Internal Documentation

### Main Guides (docs/guides/)

| Document | Description | Link |
|----------|-------------|------|
| **00-PROJECT-OVERVIEW.md** | Quick start guide, CLAUDE.md summary | [docs/guides/00-PROJECT-OVERVIEW.md](../../guides/00-PROJECT-OVERVIEW.md) |
| **architecture.md** | System architecture (v12 Graph Structure) | [docs/guides/architecture.md](../../guides/architecture.md) |
| **getting-started.md** | User guide, 사용법 | [docs/guides/getting-started.md](../../guides/getting-started.md) |
| **todo-bucket-design.md** | TODO 버킷 설계 (v12 소작전 분할) | [docs/guides/todo-bucket-design.md](../../guides/todo-bucket-design.md) |
| **daily-log-generator.md** | Daily Log Generator 가이드 | [docs/guides/daily-log-generator.md](../../guides/daily-log-generator.md) |
| **gcp-deployment-guide.md** | GCP 배포 가이드 | [docs/guides/gcp-deployment-guide.md](../../guides/gcp-deployment-guide.md) |
| **templates.md** | Notion DB property templates | [docs/guides/templates.md](../../guides/templates.md) |
| **changelog.md** | Version history (v10-v13) | [docs/guides/changelog.md](../../guides/changelog.md) |

### Operations (docs/ops/)

| Operation | Description | Link |
|-----------|-------------|------|
| **00-INDEX.md** | Operation concept overview + full index | [docs/ops/00-INDEX.md](../../ops/00-INDEX.md) |
| **251228-bucket-trigger/** | Notion workflow automation | [docs/ops/251228-bucket-trigger/](../../ops/251228-bucket-trigger/) |
| **251228-codingbot/** | TODO → PR generation | [docs/ops/251228-codingbot/](../../ops/251228-codingbot/) |
| **251228-reviewer/** | PR verification | [docs/ops/251228-reviewer/](../../ops/251228-reviewer/) |
| **251228-daily-log/** | Daily log automation | [docs/ops/251228-daily-log/](../../ops/251228-daily-log/) |
| **251228-notion-integration/** | Notion DB schema management | [docs/ops/251228-notion-integration/](../../ops/251228-notion-integration/) |

### Research Materials (docs/research/)

| Research | Description | Link |
|----------|-------------|------|
| **microservices-for-project-management/** | Microservices 철학 적용 (10 documents) | [docs/research/microservices-for-project-management/](../microservices-for-project-management/) |
| **autocoder-v13/** | v13 development notes | [docs/research/autocoder-v13/](../autocoder-v13/) |
| **autocoder-v10/** | v10 development notes | [docs/research/autocoder-v10/](../autocoder-v10/) |
| **codingbot-reviewer-flow/** | GAN workflow research | [docs/research/codingbot-reviewer-flow/](../codingbot-reviewer-flow/) |
| **gcp-deployment-complete/** | GCP deployment research | [docs/research/gcp-deployment-complete/](../gcp-deployment-complete/) |
| **telegram-bot-architecture/** | Telegram integration research | [docs/research/telegram-bot-architecture/](../telegram-bot-architecture/) |
| **v9-telegram-tracking/** | v9 tracking system | [docs/research/v9-telegram-tracking/](../v9-telegram-tracking/) |
| **workflow-methodologies/** | Workflow comparison study | [docs/research/workflow-methodologies/](../workflow-methodologies/) |

## External References

### Microservices Architecture

| Resource | Description | URL |
|----------|-------------|-----|
| **Martin Fowler - Microservices** | Microservices 개념 정리 | https://martinfowler.com/articles/microservices.html |
| **Netflix Tech Blog** | Microservices at Scale | https://netflixtechblog.com/ |
| **Amazon - Two Pizza Teams** | Small team ownership | https://docs.aws.amazon.com/whitepapers/latest/introduction-devops-aws/two-pizza-teams.html |
| **Spotify Engineering Culture** | Squad model | https://engineering.atspotify.com/2014/03/spotify-engineering-culture-part-1/ |

### AI Models

| Model | Provider | Documentation | Use Case (skim-stone) |
|-------|----------|---------------|------------------------|
| **Gemini 2.5 Flash** | Google | https://ai.google.dev/gemini-api/docs | IDEA, TODO, CODINGBOT, DLG |
| **Claude Sonnet 4.5** | Anthropic (AWS Bedrock) | https://docs.anthropic.com/en/docs/models-overview | RESEARCH, REVIEWER |

### Notion API

| Resource | Description | URL |
|----------|-------------|-----|
| **Notion API Reference** | API 문서 | https://developers.notion.com/reference/intro |
| **Notion SDK (Python)** | `notion-client` 라이브러리 | https://github.com/ramnes/notion-sdk-py |
| **Database Query** | Query database endpoint | https://developers.notion.com/reference/post-database-query |
| **Page Update** | Update page properties | https://developers.notion.com/reference/patch-page |
| **Block Append** | Append block children | https://developers.notion.com/reference/patch-block-children |

### GitHub API

| Resource | Description | URL |
|----------|-------------|-----|
| **GitHub CLI** | `gh` CLI 도구 | https://cli.github.com/ |
| **Create PR** | `gh pr create` | https://cli.github.com/manual/gh_pr_create |
| **GitHub API Reference** | REST API 문서 | https://docs.github.com/en/rest |

### GCP (Google Cloud Platform)

| Resource | Description | URL |
|----------|-------------|-----|
| **Cloud Run** | Serverless container platform | https://cloud.google.com/run/docs |
| **Cloud Scheduler** | Cron job service | https://cloud.google.com/scheduler/docs |
| **Secret Manager** | Secret management | https://cloud.google.com/secret-manager/docs |
| **Artifact Registry** | Container registry | https://cloud.google.com/artifact-registry/docs |

### AWS Bedrock

| Resource | Description | URL |
|----------|-------------|-----|
| **AWS Bedrock** | Managed AI service | https://aws.amazon.com/bedrock/ |
| **Claude Models** | Anthropic Claude on Bedrock | https://docs.aws.amazon.com/bedrock/latest/userguide/model-ids-arns.html |
| **boto3 Bedrock Client** | Python SDK | https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-runtime.html |

## Related Projects

### Telegram Bot

| Resource | Description | Link |
|----------|-------------|------|
| **@zorba_the_bot** | AUTOCODER Telegram bot | https://t.me/zorba_the_bot |
| **Telegram Bot API** | Bot API documentation | https://core.telegram.org/bots/api |

### Git Repositories

| Repository | Description | URL |
|------------|-------------|-----|
| **_AUTOCODER** | Main project repository | (current repo) |
| **KBO-yaguin-DB** | Baseball player DB project | (external repo) |

## Version History References

### AUTOCODER Versions

| Version | Date | Changelog | Link |
|---------|------|-----------|------|
| **v10.0.0** | 2025-12-22 | PLAN 제거, AI 업그레이드 | [changelog.md#v10.0.0](../../guides/changelog.md#v1000---2025-12-22) |
| **v11.0.0** | 2025-12-23 | RESEARCH 안정화 | [changelog.md#v11.0.0](../../guides/changelog.md#v1100---2025-12-23) |
| **v12.0.0** | 2025-12-25 | TODO 소작전 분할 | [changelog.md#v12.0.0](../../guides/changelog.md#v1200---2025-12-25) |
| **v13.0.0** | 2025-12-25 | BT 버전 통일 | [changelog.md#v13.0.0](../../guides/changelog.md#bt-v1300---2025-12-25) |
| **v13.1.0** | 2025-12-25 | DLG 통합 개선 | [changelog.md#v13.1.0](../../guides/changelog.md#dlg-v1310---2025-12-25) |
| **v13.2.0** | 2025-12-25 | DLG LLM 마이그레이션 | [changelog.md#v13.2.0](../../guides/changelog.md#dlg-v1320---llm-migration) |
| **v13.3.0** | 2025-12-25 | DLG 프롬프트 개선 | [changelog.md#v13.3.0](../../guides/changelog.md#dlg-v1330---2025-12-25) |
| **v13.4.0** | 2025-12-28 | 문서 재구조화 | [changelog.md#v13.4.0](../../guides/changelog.md#v1340---2025-12-27) |
| **skim-stone v0.1** | 2025-12-28 | 물수제비 철학 명시 | [CLAUDE.md](../../../CLAUDE.md) |

## Key Concepts & Glossary

| Term | Definition | Reference |
|------|------------|-----------|
| **물수제비 (Skipping Stone)** | 돌멩이가 물 위를 여러 번 튀며 나아가듯, 정보가 버킷 간 자동 이동하며 점점 구체화되는 시스템 | [01-philosophy.md](01-philosophy.md) |
| **버킷 (Bucket)** | 정보 포맷 변환기. IDEA/RESEARCH/TODO 3종류. | [03-bucket-formats.md](03-bucket-formats.md) |
| **그래프 구조 (Graph Structure)** | 방향 없는 그래프. `next_bucket`으로 어디든 이동 가능. | [02-graph-structure.md](02-graph-structure.md) |
| **_self** | 재처리 루프. 유저 피드백 반영하며 버전업. | [01-philosophy.md#4-재귀-루프-_self](01-philosophy.md) |
| **GAN 검증** | CODINGBOT (Generator) + REVIEWER (Discriminator) 이중 검증 | [01-philosophy.md#5-gan-검증](01-philosophy.md) |
| **Operation** | Microservices 철학 적용한 독립적 작업 단위 | [04-microservices-ops.md](04-microservices-ops.md) |
| **OPERATION (DB)** | Notion Database. 프로젝트 컨텍스트 컨테이너. | [architecture.md](../../guides/architecture.md) |
| **소작전 (Small Operation)** | TODO 버킷의 child pages. Cloud Run Job 크기로 분할. | [03-bucket-formats.md#소작전-분할-원칙-v12](03-bucket-formats.md) |
| **Prerequisites** | TODO main page 상단 체크박스. 모두 체크되면 CODINGBOT 트리거. | [03-bucket-formats.md#prerequisites-핵심-개념](03-bucket-formats.md) |
| **Lineage** | 블록체인 추적. 전체 이동 경로 기록. | [02-graph-structure.md#lineage-블록체인-추적](02-graph-structure.md) |

## Contact & Support

| Channel | Purpose | Link |
|---------|---------|------|
| **GitHub Issues** | Bug reports, feature requests | (current repo issues) |
| **Telegram** | Daily notifications, logs | @zorba_the_bot |
| **Notion** | Project management, DB | (OPERATION database) |

---

**Last Updated**: 2025-12-28
**Document Version**: 1.0
**Maintained by**: AI (Claude Code)
