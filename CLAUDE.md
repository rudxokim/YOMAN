# YOMAN Project Context

## Overview

YOMAN (Your Operation MANager)은 SampleRNN의 계층적 시간 스케일 개념을 LLM 기반 프로젝트 관리에 적용한 차세대 소프트웨어 개발 아키텍처.

**핵심 철학**: "컴퓨터 없이 개발하기" - 폰(Telegram/Slack)만으로 개발 가능한 환경

## Project Structure

```
YOMAN/
├── docs/
│   ├── YOMAN-PROJECT-OVERVIEW.md    # 프로젝트 총 요약
│   ├── archive/                      # 원본 아이디어 아카이브
│   │   └── original-ideas/           # idea.txt, idea2.txt, idea3.txt
│   ├── research/                     # 연구 문서
│   │   ├── 251231-1500-yoman-paper/  # 학술 논문 형식
│   │   ├── 260101-async-system-temporal-gap/
│   │   ├── 260101-causal-consistency-models/
│   │   ├── 260101-causal-vs-non-causal-llm/
│   │   ├── 260101-event-sourcing-cqrs-causality/
│   │   ├── 260101-hierarchical-temporal-models/
│   │   ├── 260101-human-ai-async-interface-patterns/
│   │   └── 260101-temporal-perception-interface-design/
│   └── study/                        # 학습 자료 (4개 폴더)
│       ├── 251228-1815-autocoder-to-skim-stone/
│       ├── 251231-1430-stone-skim-philosophy/
│       ├── 251231-1800-yoman-architecture/
│       └── microservices-for-project-management/
├── sampleRNN.pdf                     # 영감 논문 (ICLR 2017)
└── .env                              # API 키 (Notion, Gemini)
```

## Core Concepts

### Two-Layer Architecture
- **Interface Layer**: 사람 중심 (Bucket System: IDEA → RESEARCH → TODO → PR)
- **Engine Layer**: AI 중심 (UnitService 3계층, Multi-model Orchestration)

### UnitService Hierarchy
```
Tier 3: Opus (System) - 프로젝트 전체 조망
Tier 2: Sonnet (Module) - Operation 관리
Tier 1: Flash/GLM (Unit) - 단일 함수/클래스
```

### Bucket System (물수제비 메타포)
- IDEA (러프) → RESEARCH (구조화) → TODO (실행가능) → PR (검증)
- 정보가 단계별로 정제됨
- 양방향 흐름 가능 (TODO → RESEARCH 등)

### GAN Verification (Quasi-Non-Causal)
- Generator: CODINGBOT (TODO → PR)
- Discriminator: REVIEWER (PR vs TODO 스펙 비교)
- match_rate ≥80% → 머지, <50% → 재작업

## Current Status

**Phase**: 철학 정립 및 문서화 (설계 단계)

**완료**:
- 핵심 개념 정립 (idea.txt, idea2.txt, idea3.txt → archive로 이동)
- 학습 자료 37개 문서 작성
- 학술 논문 형식 문서 (yoman-paper.md)
- 추가 연구 문서 7개 폴더 (260101-* 시리즈)
  - Causal vs Non-Causal LLM 분석
  - Hierarchical Temporal Models
  - Human-AI Async Interface Patterns
  - Event Sourcing + CQRS + Causality

**진행 중**:
- 프로토타입 구현 준비

## Key Files

| 파일 | 설명 |
|------|------|
| `docs/YOMAN-PROJECT-OVERVIEW.md` | 전체 프로젝트 요약 (590줄) |
| `docs/research/251231-1500-yoman-paper/00-yoman-paper.md` | 학술 논문 형식 |
| `docs/study/251231-1800-yoman-architecture/` | 아키텍처 완벽 가이드 |
| `docs/archive/original-ideas/` | 원본 아이디어 (idea.txt, idea2.txt, idea3.txt) |
| `docs/research/260101-causal-vs-non-causal-llm/` | Causal 시스템 연구 |
| `docs/research/260101-hierarchical-temporal-models/` | 계층적 시간 모델 |

## Development Guidelines

### 문서 우선 개발
- 구현 전 docs/ 먼저 작성
- 모든 변경 사항은 CLAUDE.md에 반영

### Memory MCP 활용
```
entities: YOMAN, UnitService, Bucket, CODINGBOT, REVIEWER
relations: uses, creates, validates
observations: 실시간 컨텍스트
```

### 버킷별 AI 모델
| 버킷 | 모델 | 출력 |
|------|------|------|
| IDEA | Gemini Flash | 마크다운 (러프) |
| RESEARCH | Claude | JSON (구조화) |
| TODO | Gemini/Claude | JSON + Prerequisites |

## External Dependencies

- **Notion API**: 데이터베이스 동기화
- **Gemini API**: 빠른 모델 (Unit 레벨)
- **Claude API**: 중상위 모델 (Module/System)

<!-- Updated: 2026-01-01 - Add 7 new research folders, restructure project with archive -->
<!-- Updated: 2026-01-01 - Initial CLAUDE.md creation -->
