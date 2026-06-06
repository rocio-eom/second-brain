---
type: moc
created: 2026-06-04
modified: 2026-06-06
domain: ai-ml-llm
tags: [rag, moc]
related_mocs:
  - "[[moc-embedding]]"
  - "[[moc-vector-search]]"
  - "[[moc-chunking]]"
---

# MOC: RAG (Retrieval-Augmented Generation)

End-to-end RAG 파이프라인의 설계·운영·평가 관련 Permanent 노트 인덱스.

---

## Ingestion

- [[rag-ingestion-overview]] — RAG ingestion 5단계 파이프라인 개요: GIGO 원칙·Load→Parse/Clean→Chunk→Embed→Index/Store 처리 흐름·ETL/검색 대비 비교·6가지 안티패턴
- [[rag-document-loading]] — Document Loading & Parsing: 포맷별 전용 파서·레이아웃 보존·element 정규화. RAG 파이프라인 최상단
- [[rag-preprocessing-cleaning]] — Preprocessing 5단: normalize · boilerplate strip · quality filter · PII redact · exact/near dedup
- [[rag-chunk-metadata-extraction]] — chunk 메타 부착: extrinsic + intrinsic + LLM 추출 5단 + schema-first filter 설계
- [[rag-multi-modal-ingestion]] — 텍스트 외 modality(이미지/표/오디오/비디오): captioning · CLIP · ColPali · multi-vector 4전략
- [[rag-embedding-generation]] — 대규모 배치 임베딩 처리량·지연·비용 튜닝: adaptive batching·length sorting·rate limit·idempotency 패턴
- [[rag-vector-index-lifecycle]] — 인덱스 lifecycle: blue-green namespace · alias swap · schema versioning · index registry로 무중단 reindex
- [[rag-deletion-propagation]] — 삭제·tombstone·일관성: chunk 매핑 store + compaction + cache invalidate, GDPR right-to-erasure 추적
- [[rag-ingestion-production-ops]] — RAG ingestion production 운영 3축: DLQ로 poison chunk 격리·span tracing으로 stage별 root cause·tag 기반 cost attribution + RAGOps span 표준
- [[rag-pii-handling]] — PII 일관 정책: embedding inversion 방어·reversible tokenization + PII vault·3-layer detection·규제 팩(GDPR/HIPAA/PCI-DSS/DPDPA)
- [[confluence-doc-type-classification-heuristic]] — Confluence 페이지를 FAQ/가이드/정책서로 분류하는 4단계 rule-based heuristic chain

## Framework & Orchestration

- [[rag-framework-comparison]] — LlamaIndex·LangChain·Haystack·DSPy 프레임워크 선정 기준과 overhead 벤치마크(~6ms/~10ms/~5.9ms/~3.53ms): ingestion+orchestration 역할 분담 패턴

## Retrieval

- [[embedding-vector-index-coupling]] — RAG retrieval layer 통합 개요: embedding × vector index 강한 결합도·Recall/Latency/Memory/Cost 4축 트레이드오프·blue/green 인덱스 운영 패턴
- [[korean-rag-embedding-selection]] — 한국어 RAG 임베딩 모델 선정 프레임워크: 한국어 우선·code-switching 견고성·운영 비용 3축 + 모델 비교
- [[aws-s3-vectors-cold-tier-rag]] — cold-tier only RAG 아카이브 패턴: idle 95%+ 코퍼스 시맨틱 검색, 운영 인스턴스 0·비용 ≈ Storage만
- [[vector-store-decision-matrix]] — RAG용 벡터 저장소 선택 결정 매트릭스: latency SLA·idle비용·검색풍부도·RDB통합 4축 기반 S3V·OSS·pgvector·Pinecone 비교
- [[hybrid-search-optimization]] — BM25 + Vector hybrid retrieval: RRF·α-weighted fusion·score normalization·reranker 결합으로 recall과 precision 동시 향상
- [[rag-policy-filter]] — 권한 기반 ACL pre-filter: tenant·role·classification metadata로 cross-tenant leakage 차단 + RBAC/ABAC/ReBAC + 외부 PDP(Cerbos/OPA)
- [[rag-multi-turn-session]] — Conversational RAG multi-turn: query rewriting(decontextualization) + memory 4계층 + semantic cache + context budget 분배

## Generation / Post-processing

- [[rag-guardrail]] — RAG runtime guardrail 3지점: input · retrieval · output rail + contextual grounding + Bedrock/NeMo/LlamaGuard/Lakera 비교

## Evaluation & Monitoring

- [[rag-ingestion-quality-monitoring]] — ingestion-side 품질·관측: chunk 분포·embedding drift·source freshness·SLO burn + Ragas/DeepEval/TruLens/Langfuse 4프레임워크 비교

## Anti-patterns

- [[]] —

---

## Cross-Domain Links

- [[moc-embedding]] — 임베딩 모델 선택·생성·운영
- [[moc-vector-search]] — 벡터 검색·인덱스 운영
- [[moc-chunking]] — 청킹 전략 설계·품질 개선
- [[moc-aws-bedrock]] — AWS Bedrock 기반 매니지드 RAG
