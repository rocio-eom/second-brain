---
type: moc
created: 2026-06-04
modified: 2026-06-06
domain: ai-ml-llm
tags: [vector-search, moc]
related_mocs:
  - "[[moc-rag]]"
  - "[[moc-embedding]]"
---

# MOC: Vector Search

벡터 검색·인덱스 운영 관련 Permanent 노트 인덱스.

---

## 아키텍처 · 설계

- [[vector-search-architecture]] — RAG retrieval 설계 결정 트리: chunking·embedding·index·hybrid·rerank 종합. Ingest/Query 2-path 컴포넌트 다이어그램 + shadow→A/B rollout

## 인덱스 알고리즘

- [[ann-index-parameters]] — HNSW(M·efConstruction·ef_search) / IVF(nlist·nprobe) 4축 trade-off, 빌드 비대칭, grid search 절차
- [[]] — Quantization 전략 (SQ · PQ · Binary)

## 시스템 비교

- [[opensearch-knn-vector-search]] — OpenSearch k-NN으로 BM25+vector 단일 클러스터 운영: Lucene/Faiss/Nmslib 3엔진·Neural Search RRF 하이브리드·양자화/disk search
- [[vector-database-comparison]] — Pinecone/Milvus/Qdrant/Weaviate/Chroma/OpenSearch/pgvector 7종 비교: 운영 모델·성능·기능·비용 4축, 전용 DB vs 검색 엔진 확장 양분
- [[vector-store-decision-matrix]] — RAG용 벡터 저장소 선택 결정 매트릭스: latency SLA·idle비용·검색풍부도·RDB통합 4축 기반 S3V·OSS·pgvector·Pinecone 비교

## 운영

- [[recall-vs-filter-tradeoff]] — ANN+필터 보편 한계: pre/post/in-algorithm 3전략 비교, S3V 등 매니지드는 노브 부재로 거친 필터만 권장. oversampling·multi-stage·hybrid로 보완
- [[vector-search-performance-tuning]] — 4-layer 튜닝 메타 프레임워크: 인덱스 → 양자화 → 시스템 → 데이터 순서, SLO 5축 동시 trade-off 측정 루프
- [[vector-search-operations]] — Index lifecycle: 모델 버전=인덱스 버전, shadow+alias swap, dual-write, eval harness, 7–14일 검증 후 구 인덱스 제거
- [[vector-search-system-query-tuning]] — Hybrid·필터·rerank·batching·replica·cache: pre/post filter selectivity 기반 분기, dense+sparse+cross-encoder 3단 표준

---

## Cross-Domain Links

- [[moc-rag]] — Retrieval 컴포넌트로서 vector search
- [[moc-embedding]] — 임베딩 차원·분포가 인덱스 파라미터에 미치는 영향
- [[moc-aws-bedrock]] — Bedrock Knowledge Bases 관리형 인덱스
