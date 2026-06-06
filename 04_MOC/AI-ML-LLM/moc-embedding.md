---
type: moc
created: 2026-06-04
modified: 2026-06-06

domain: ai-ml-llm
tags: [embedding, moc]
related_mocs:
  - "[[moc-rag]]"
  - "[[moc-vector-search]]"
---

# MOC: Embedding

임베딩 모델 선택·생성·운영·평가 관련 Permanent 노트 인덱스.

---

## 모델 선택

- [[rag-embedding-model-selection]] — RAG 임베딩 모델 선정 프레임워크: egress→다국어→품질/비용 결정 트리 + OpenAI·Cohere·Voyage·Titan·BGE-M3·Qwen3·Jina 10개 모델 비교·TCO 산정
- [[cohere-embed-multilingual-v3]] — 100+ 언어 cross-lingual, 1024 dim, 512 token. 한국어 RAG 1차 후보
- [[titan-text-embeddings-v2]] — AWS 1st-party, dim 가변(256/512/1024), 영어 최적·비영어 sub-optimal
- [[aws-bedrock-embedding]] — Bedrock 호스팅 채널 일반론

## Layer 설계 정책

- [[rag-embedding-layer]] — RAG vector index의 embedding layer 책임·정책: 모델 선택축·비대칭 인코딩·L2 정규화·Matryoshka 차원·model_version 태깅·4대 안티패턴

## 생성 운영

- [[embedding-api-client-design]] — production embedding API 클라이언트 설계: model 선택(Voyage/OpenAI/Cohere/Titan 비교) + token bucket·exp backoff·Batch API 호출 안정성 패턴
- [[rag-embedding-generation]] — 대규모 배치 임베딩 처리량·지연·비용 튜닝: adaptive batching·length sorting·rate limit·idempotency 패턴
- [[embedding-incremental-cdc-ingestion]] — 변경 감지·chunk-level CDC·content hash·state store·model upgrade migration(Shadow vs Drift-Adapter)
- [[embedding-model-lock-in]] — 임베딩 모델 간 의미 공간 비호환: 인덱스 단위 lock-in·모델 교체 비용·dual index·Drift-Adapter 완화 전략

## 평가

- [[retrieval-quality-metrics]] — P@K·Recall@K·MRR·nDCG·Hit Rate@K·MTEB·golden set 설계: 임베딩 모델 선정·회귀 감지·CI 게이트의 측정 체계

---

## Cross-Domain Links

- [[moc-rag]] — end-to-end 파이프라인 컨텍스트
- [[moc-vector-search]] — 인덱스·운영
- [[moc-chunking]] — 청크 분포와 임베딩 호환성
