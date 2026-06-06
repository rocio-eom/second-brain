---
type: permanent
created: 2026-06-06
modified: 2026-06-06
status: draft
domain: ai-ml-llm
moc: "[[moc-rag]]"
tags: [rag, framework, llamaindex, langchain, haystack, dspy, orchestration, ingestion, evaluation]
aliases: [RAG Framework Comparison, LlamaIndex vs LangChain, RAG Framework Selection, LLM Framework Overhead]
promoted_from: fl-2026-06-02-rag-indexing-pipeline
related:
  - "[[rag-ingestion-overview]]"
  - "[[rag-ingestion-production-ops]]"
  - "[[rag-embedding-generation]]"
  - "[[hybrid-search-optimization]]"
---

# RAG Framework Comparison

> 본 노트는 LlamaIndex·LangChain·Haystack·DSPy의 **선택 기준과 overhead 벤치마크**에 집중한다. 파이프라인 5단계 개요는 [[rag-ingestion-overview]] 참조.

## 핵심 요약

RAG 파이프라인 프레임워크는 ingestion·retrieval·generation 각 레이어에서 역할 분담이 다르며, 선택 기준은 데이터 적재 우선인지·orchestration 우선인지·CI 평가 통합인지에 따라 달라진다.

- **2026년 production 경험칙**: RAG 실패의 약 80%가 ingestion·chunking 단계에서 발생하며, LLM 자체보다 retrieval 품질이 답변 품질의 주요 결정 요인
- **흔한 조합**: LlamaIndex로 ingestion/indexing + LangChain(LangGraph)으로 orchestration — 두 프레임워크는 경쟁이 아닌 역할 분담
- **Framework overhead는 예상보다 적음**: ms 단위 차이. 선택 기준은 overhead가 아닌 생태계·추상화·평가 통합

> **버전 민감 항목**: overhead 벤치마크(ms)는 2026년 기준. 각 프레임워크 버전에 따라 변동 가능.

## 시스템 아키텍처

5단계 파이프라인과 운영 layer의 전체 구조.

```mermaid
graph TD
  subgraph "Offline / Indexing"
    SRC[(Source Systems<br/>DBs / S3 / Confluence / Web)] --> ING[Ingestion<br/>connectors / CDC / crawlers]
    ING --> PRS[Parsing<br/>LlamaParse / Unstructured / Docling]
    PRS --> CHK[Chunking<br/>recursive / semantic / late]
    CHK --> EMB[Embedding<br/>OpenAI / Cohere / BGE-M3]
    EMB --> IDX[(Vector + Inverted Index<br/>Weaviate / Qdrant / pgvector)]
  end
  subgraph "Runtime / Retrieval"
    Q[User Query] --> QEMB[Query Embedding]
    QEMB --> RET[Hybrid Retrieval + Reranker]
    IDX --> RET
    RET --> LLM[LLM Generation]
    LLM --> A[Answer + Citations]
  end
  subgraph "Ops Layer"
    REG[Document Registry<br/>content hash] -.-> ING
    EVAL[Evaluation Suite<br/>recall@k / faithfulness] -.-> RET
    OBS[Observability<br/>query log / chunk attribution] -.-> RET
    OBS -.-> LLM
  end
```

## 처리 흐름

신규 문서 도착 → offline 파이프라인이 인덱스 갱신 → runtime이 인덱스를 읽어 응답 생성.

```mermaid
flowchart LR
  A[원본 문서 도착] --> B[Ingestion<br/>변경 감지 + 적재]
  B --> C[Parsing<br/>레이아웃 / 테이블 / 이미지 추출]
  C --> D[Chunking<br/>토큰 단위 분할 + overlap]
  D --> E[Embedding<br/>vector 생성 + 메타데이터]
  E --> F[(Index Write<br/>alias swap)]
  F -.runtime.-> G[Query 들어옴]
  G --> H[Retrieval<br/>hybrid + rerank]
  H --> I[LLM 응답 + 출처]
```

## 핵심 기능 및 서비스

| 단계 | 핵심 기능 | 대표 도구 / 서비스 |
|---|---|---|
| Ingestion | source connector, schedule/event 트리거, CDC 또는 content-hash 기반 증분 동기화 | Airbyte, Fivetran, LangChain DocumentLoader, 자체 crawler |
| Parsing | 레이아웃 인식, 테이블·이미지·코드 추출, Markdown / JSON 산출 | LlamaParse(90+ 포맷), Unstructured(30+ 포맷), Docling(셀프호스팅), AWS Textract |
| Chunking | semantically complete 분할, overlap, metadata 부착 | LangChain RecursiveCharacterTextSplitter, LlamaIndex SemanticSplitter |
| Embedding | dense vector 생성, batching, dimensionality 조정 | OpenAI `text-embedding-3-small/large`, Cohere `embed-v4`, BAAI `BGE-M3`, Voyage AI |
| Retrieval | ANN + sparse 결합, reranking, metadata filter | Weaviate, Qdrant, Pinecone, Elasticsearch, pgvector + Cohere/BGE reranker |
| Ops layer | 증분 인덱싱, alias-based zero-downtime 배포, recall@k·faithfulness 모니터링 | LangSmith, Arize Phoenix, Ragas, TruLens |

## 유사 기술 비교

| 항목 | LlamaIndex | LangChain | Haystack | DSPy |
|---|---|---|---|---|
| 특징 | 데이터 적재·인덱싱 특화 | 범용 LLM orchestration | 엔터프라이즈 modular pipeline | 프로그램형 LLM 최적화 |
| Framework overhead (ms) | ~6 | ~10 | ~5.9 | ~3.53 |
| 강점 | 92% retrieval accuracy 벤치 / 인덱스 추상화 다양 | 가장 큰 생태계, 에이전트·툴 호출 | 평가 도구 강함, 감사 가능 | overhead 최소, 자동 prompt 튜닝 |
| 약점 | agent / 워크플로 약함 | 인덱싱 추상화는 LlamaIndex보다 약함 | 시작 학습 곡선 | 생태계·문서 상대적으로 얕음 |
| 적합 케이스 | 데이터 중심 RAG | RAG + agentic, 멀티툴 | 규제 산업, 감사 요건 | retrieval/prompt 자동 최적화 연구·실험 |

> 실무 흔한 조합: **LlamaIndex로 ingestion/indexing, LangChain(또는 LangGraph)으로 orchestration**. 두 프레임워크는 역할 분담.

## 실제 사례

### LlamaIndex / LlamaParse 생태계
ingestion·인덱싱 추상화에 집중해 90+ 포맷 파싱(LlamaParse), 다양한 chunking 전략(SemanticSplitter, HierarchicalNodeParser), 다종 vector store 통합을 단일 SDK로 제공. retrieval 정확도 벤치마크에서 LangChain 대비 우위(약 92% vs 85%) 보고가 다수.

### Haystack (deepset)
Modular pipeline + 강한 evaluation 도구를 강점으로 규제 산업(금융·헬스케어) RAG 배포에 채택. CI/CD에 retrieval accuracy / faithfulness 게이트를 자연스럽게 결합 가능.

### Databricks / 엔터프라이즈 데이터레이크 RAG
Delta Lake + Spark + Mosaic AI Vector Search 조합으로 batch 증분 인덱싱 + 대규모 corpus 운영. 사내 BI 자산을 RAG에 그대로 노출하는 패턴이 2026년 보편화.

## 활용 시나리오

### 시나리오 1: 사내 위키 / Confluence RAG 챗봇
다양한 포맷, 빈번한 수정, 권한·메타데이터 필요 → Confluence/Notion connector(증분 sync) → LlamaParse(첨부 PDF 레이아웃 보존) → recursive 512/overlap 64 → BGE-M3(다국어) → Weaviate hybrid + Cohere Rerank. 권한은 chunk metadata로 필터.

### 시나리오 2: 코드 + 문서 통합 검색 (개발자 포털)
코드는 exact match, 문서는 의미 매칭이 강해야 함 → Repo loader(파일 변경 감지) + Markdown 파서 → **코드는 함수/클래스 단위 청킹**, **문서는 semantic 청킹** → 동일 embedding 모델 + 별도 namespace → hybrid retrieval(BM25 가중치 ↑) + reranker.

### 시나리오 3: 규제 산업의 감사 가능한 RAG
운영 안정성·재현성·감사 로그가 정확도만큼 중요 → Haystack pipeline + Ragas evaluation을 CI/CD 게이트로. 인덱스는 content-hash 기반 증분 + alias swap으로 무중단 배포. 모든 응답에 `index_version`, `chunk_id`, `retrieval_score` 함께 로깅.

## 장단점 및 트레이드오프

| 항목 | 내용 |
|---|---|
| 장점 | 도메인 지식·실시간 자료를 LLM에 안전하게 주입. fine-tuning 대비 비용·갱신 비용 낮음. 출처 인용으로 hallucination 부분 완화 |
| 단점 | 5단계 모두 운영 부담(인덱스 무결성·증분 동기화·embedding 모델 버전). 각 단계 튜닝 파라미터 폭발. 임베딩 모델 교체 시 전체 corpus 재임베딩 필요 |
| 트레이드오프 | semantic chunking은 recall +9% vs token-based 대비 약 14배 느림. 고품질 embedding 모델은 cost·dimensionality ↑. hybrid + reranker는 정확도 ↑ vs latency·infra ↑ |

## 함정 및 안티패턴

- **안티패턴 1: 청킹을 단순 `chunk_size=1000` 글자 단위로 고정** → 문장·문단 중간 절단으로 retrieval recall 급락 → recursive 400-512 토큰 + 10-20% overlap을 기본값으로 시작.
- **안티패턴 2: PDF를 raw OCR text로 평면화** → 테이블·다단·헤더 손실 → 레이아웃 인식 파서(LlamaParse, Docling, Mistral OCR 3) + Markdown 산출.
- **안티패턴 3: 임베딩 모델을 충분한 검토 없이 도입** → 추후 교체 시 대규모 corpus 재임베딩 비용·다운타임 → 결정 전에 자사 도메인 sample로 retrieval 벤치마크 수행.
- **안티패턴 4: 전체 인덱스 재구축 위주 운영** → 다운타임·비용·문서 일관성 결손 → document registry + content-hash 변경 감지로 증분 인덱싱, alias swap으로 zero-downtime 배포.
- **안티패턴 5: retrieval만 모니터링하고 chunk-level attribution 누락** → 품질 회귀 시 단계 책임 추적 불가 → 응답에 `chunk_id`, `index_version`, `retrieval_score` 로깅 + recall@k / faithfulness 자동 평가 CI/CD 게이트.

## 참고 자료

- [RAG Pipeline Challenges: From Data Ingestion to Retrieval — Unstructured](https://unstructured.io/insights/rag-pipeline-challenges-from-data-ingestion-to-retrieval) — 80% 실패 원인 분포
- [Building Production RAG: Architecture, Chunking, Evaluation & Monitoring 2026 — Prem AI](https://blog.premai.io/building-production-rag-architecture-chunking-evaluation-monitoring-2026-guide/) — 2026 종합 가이드
- [Top Document Parsing APIs for 2026 — LlamaIndex](https://www.llamaindex.ai/insights/top-document-parsing-apis) — LlamaParse / Unstructured / Docling / Textract 비교
- [Best RAG Frameworks 2026: LangChain vs LlamaIndex vs DSPy — Iternal](https://iternal.ai/blockify-rag-frameworks) — 프레임워크 overhead 벤치마크와 사용처 매핑
- [LlamaIndex vs LangChain vs Haystack: Which RAG Framework? — Kanerika](https://kanerika.com/blogs/llamaindex-vs-langchain-vs-haystack/) — 프레임워크 강점과 결합 패턴
- [LangChain vs LlamaIndex in 2026 (DEV)](https://dev.to/lycore/langchain-vs-llamaindex-in-2026-what-we-actually-use-and-why-52eb) — 두 프레임워크의 ingestion 책임 분담 패턴

## 관련 노트

- [[rag-ingestion-overview]] — RAG ingestion 파이프라인 5단계 개요 (본 노트의 pipeline 구조 배경)
- [[rag-ingestion-production-ops]] — DLQ·span tracing·cost attribution 운영 평면
- [[rag-embedding-generation]] — 임베딩 단계 배치 처리·throughput 튜닝
- [[hybrid-search-optimization]] — BM25 + Vector hybrid retrieval — 모든 프레임워크에서 추천되는 retrieval 패턴
