---
type: reference
created: 2026-06-04
modified: 2026-06-04
---

# Tag Taxonomy

vault 전역 태그 통제 어휘. 신규 노트 작성 시 본 카탈로그 외 태그는 도입 전 본 문서 갱신 필수.

---

## 규칙

1. **언어**: 영어 `kebab-case`만 사용. 한국어 태그 금지 (한국어 식별자는 `aliases` frontmatter로).
2. **단복수**: 단수형으로 통일 (`embedding`, not `embeddings`).
3. **계층**: `domain/sub` 형태 권장 (`rag/ingestion`, `rag/retrieval`).
4. **메타 태그**: `moc`, `draft`, `evergreen`, `anti-pattern`, `case-study` 등 카테고리·라이프사이클 태그는 평면(non-hierarchical).
5. **신규 도입**: 동일 의미 기존 태그가 없을 때만. 추가 시 본 문서 → 해당 섹션에 등록.

---

## Domain 태그 (top-level)

| Tag | 의미 |
|---|---|
| `ai-ml-llm` | AI/ML/LLM 도메인 (가장 광범위) |
| `backend` | 백엔드 시스템 |
| `frontend` | 프론트엔드 |
| `cs-fundamentals` | CS 기반 이론 |

## Sub-domain — AI/ML/LLM

| Tag | 의미 |
|---|---|
| `rag` | RAG 일반 |
| `rag/ingestion` | 적재·수집 단계 |
| `rag/retrieval` | 검색 단계 |
| `rag/generation` | 생성 단계 |
| `rag/evaluation` | 평가 단계 |
| `ingestion` | 데이터 적재 일반 (RAG 외 포함) |
| `data-ingestion` | 동의어 — `ingestion` 권장 |
| `preprocessing` | 전처리·클리닝 |
| `retrieval` | 검색 일반 |
| `chunking` | 청킹 전략 |
| `embedding` | 임베딩 모델·운영 |
| `llm` | LLM 모델 일반 |
| `vector-search` | 벡터 인덱스·검색 |
| `vector-db` | 벡터 DB 시스템 |
| `vector-index` | 벡터 인덱스 자체 |
| `hybrid-search` | sparse+dense 결합 검색 |
| `ann` | Approximate Nearest Neighbor |
| `hnsw` | HNSW 알고리즘 |
| `ivf` | IVF/IVFPQ 알고리즘 |
| `indexing` | 색인 작업 일반 |
| `metadata` | 메타데이터 부착·필터링 |
| `llm-serving` | LLM 서빙·추론 |
| `prompting` | 프롬프트 엔지니어링 |
| `aws-bedrock` | AWS Bedrock 생태계 |
| `aws-s3-vectors` | S3 Vectors 서비스 |
| `opensearch` | OpenSearch (kNN) |
| `pgvector` | Postgres pgvector |
| `pinecone` | Pinecone |
| `cohere` | Cohere 모델/플랫폼 |
| `openai` | OpenAI 모델/플랫폼 |

## Sub-domain — Backend

| Tag | 의미 |
|---|---|
| `distributed-systems` | 분산 시스템 |
| `data-pipeline` | ETL/ELT/streaming |
| `aws` | AWS 일반 |

## Sub-domain — Frontend

> 현재 `03_Permanent/Frontend/`는 flat. 노트 ≥10건 시 본 섹션에 sub-domain 태그 등재 후 폴더 분할.

| Tag | 의미 |
|---|---|
| (TBD) | — |

## Sub-domain — CS-Fundamentals

> 현재 `03_Permanent/CS-Fundamentals/`는 flat. 노트 ≥10건 시 본 섹션에 sub-domain 태그 등재 후 폴더 분할.

| Tag | 의미 |
|---|---|
| (TBD) | — |

## 운영·아키텍처 태그

| Tag | 의미 |
|---|---|
| `operations` | 운영 일반 |
| `observability` | 모니터링·로깅·트레이싱 |
| `performance-tuning` | 성능 튜닝 |
| `capacity-planning` | 용량 계획 |
| `orchestration` | 워크플로 orchestration |
| `architecture-pattern` | 아키텍처 패턴 |

## 메타 태그

| Tag | 의미 |
|---|---|
| `moc` | MOC 노트 표시 |
| `concept` | 단일 개념 노트 |
| `anti-pattern` | 안티패턴 카탈로그 |
| `case-study` | 실제 사례 |
| `methodology` | 방법론·프레임워크 |
| `tool` | 구체 도구·라이브러리 |
| `benchmark` | 벤치마크·평가 데이터 |

---

## Deprecated / 금지 (자동 정규화 매핑)

`/tag-normalize` skill이 사용하는 매핑 규칙. 신규 매핑 추가 시 본 표 갱신.

| Before | After | 사유 |
|---|---|---|
| `청킹`, `청킹-전략` | `chunking` | 한국어 → 영어 |
| `의미-청킹` | `semantic-chunking` | 한국어 → 영어 |
| `구조-청킹` | `structure-chunking` | 한국어 → 영어 |
| `컨텍스추얼-리트리벌` | `contextual-retrieval` | 한국어 → 영어 |
| `벡터-인덱스` | `vector-index` | 한국어 → 영어 |
| `재인덱싱` | `reindexing` | 한국어 → 영어 |
| `임베딩` | `embedding` | 한국어 → 영어 |
| `임베딩-드리프트` | `embedding-drift` | 한국어 → 영어 |
| `임베딩-모델` | `embedding-model` | 한국어 → 영어 |
| `인덱스-운영` | `index-operations` | 한국어 → 영어 |
| `한국어-임베딩` | `korean-embedding` | 한국어 → 영어 |
| `vector-database` | `vector-db` | 동의어 통일 |
| `bedrock` | `aws-bedrock` | prefix 일관성 |
| `s3-vectors` | `aws-s3-vectors` | prefix 일관성 |
| `chunks`, `embeddings` 등 복수형 | 단수형 | 컨벤션 |
| `ml` | `ai-ml-llm` | top-level domain 통일 |

---

## 유지 보수

태그 추가/삭제/병합 시:
1. 본 문서에 변경 반영
2. 기존 노트의 태그 정규화는 `/tag-normalize` skill로 일괄 처리
