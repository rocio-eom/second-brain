---
type: permanent
created: 2026-06-06
modified: 2026-06-06
status: draft
domain: ai-ml-llm
moc: "[[moc-embedding]]"
tags: [embedding, vector-search, versioning, migration, lock-in, rag]
aliases: [Embedding Model Lock-in, Vector Index Lock-in, Re-embedding Strategy, Embedding Space Incompatibility]
promoted_from: fl-2026-06-02-aws-s3-vectors-embedding-model-lock-in
related:
  - "[[aws-s3-vectors-api]]"
  - "[[aws-s3-vectors-capacity-planning]]"
  - "[[titan-text-embeddings-v2]]"
  - "[[cohere-embed-multilingual-v3]]"
  - "[[aws-bedrock-embedding]]"
---

# Embedding Model Lock-in

## 핵심 요약

서로 다른 임베딩 모델은 **서로 다른 의미 공간**에 벡터를 만든다. Titan v2의 1024차원과 Cohere v3의 1024차원은 차원 수가 같아도 **호환되지 않음** — cosine / dot product가 의미 없다. S3 Vectors는 (1) 인덱스 생성 시 차원·메트릭을 **불변**으로 고정하고, (2) 어떤 모델로 임베딩했는지 메타에 직접 강제하지 않기 때문에, **사실상 인덱스 단위로 모델이 잠긴다**. 모델 교체는 **새 인덱스 + 전 코퍼스 재임베딩**이 정석이고, 그 비용·다운타임이 운영의 핵심 위험.

- **본질적 제약**: 다른 모델 간 vector 비교는 **수학적으로 무의미** (다른 좌표계).
- **S3V의 잠금 메커니즘**: dim/metric 불변 + 모델 메타 강제 부재 → 운영 규약으로 인덱스 1개 = 모델 1개.
- **모델 교체 비용**: 임베딩 API 비용 + 인덱스 재생성 + 운영 시간 + 다운타임 리스크.
- **완화 전략**: dual indexes, lazy re-embedding, adapter layer (Drift-Adapter).

## 형식적 정의

모델 $M_A: \text{text} \to \mathbb{R}^{d_A}$, $M_B: \text{text} \to \mathbb{R}^{d_B}$.

같은 텍스트 $t$에 대해:

$$
M_A(t) \in \text{Space}_A \neq \text{Space}_B \ni M_B(t)
$$

$d_A = d_B$이라도 일반적으로:

$$
\cos(M_A(t_1), M_B(t_2)) \quad \text{has no semantic meaning}
$$

따라서 인덱스 $I$가 $M_A$로 빌드되면, 쿼리 임베딩도 $M_A$로만 의미 있다. 다른 모델로 빌드한 벡터를 같은 인덱스에 섞으면 검색 결과가 **임의에 가깝다**.

S3 Vectors 인덱스는 생성 시 $(d, \text{metric})$이 고정되므로:

- $M_A \to M_B$ with $d_A \neq d_B$ → **새 인덱스 필수**.
- $d_A = d_B$여도 의미 호환 X → **별도 인덱스가 안전**.

## 멘탈 모델 및 비유

- **모델 = 언어**: Titan은 한국어, Cohere는 일본어. 같은 알파벳 수(차원)를 가져도 단어가 다름.
- **인덱스 = 사전**: 한 언어로 정리. 다른 언어 단어를 넣으면 검색이 망가진다.
- **모델 교체 = 새 언어로 사전 재작성**: 기존 항목을 모두 새 언어로 다시 임베딩.
- **Adapter = 번역기**: 새 쿼리를 옛 언어로 번역해 옛 사전에서 검색 (단, 정밀도 손실).

## 핵심 기능 및 서비스

벡터 스토어별 lock-in 메커니즘 비교.

| 측면 | S3 Vectors | OpenSearch | Pinecone | pgvector |
|---|---|---|---|---|
| 인덱스 dim 불변 | **Yes** | mapping 변경 제한 | Yes | 컬럼 변경 가능 |
| 인덱스 metric 불변 | **Yes** | 일부 가능 | Yes | 가능 |
| 모델 메타 강제 | 없음 (운영 규약) | 사용자 매핑 | namespace 분리 | 컬럼/태그 |
| 모델 교체 절차 | 새 인덱스 + 재임베딩 | 새 인덱스 또는 매핑 변경 | 새 namespace | ALTER + 재INSERT |
| Adapter 지원 | 없음 (사용자 구현) | 없음 | 일부 연구 | 없음 |

## 유사 기술 비교

모델 변경 시나리오별 비용.

| 모델 변경 시나리오 | 비용 / 절차 |
|---|---|
| 동일 모델 minor 버전 | 보통 호환 (모델 카드 확인 후 무재임베딩 가능) — 검증 필수 |
| 같은 vendor 메이저 (Titan v1→v2) | 거의 항상 호환 X → 재임베딩 |
| 다른 vendor (Titan→Cohere) | 항상 재임베딩 + 인덱스 재생성 |
| 같은 모델·다른 차원 변형 | 새 인덱스 (차원 불변) |
| 같은 모델·다른 normalization | 메트릭 변경 효과 → 재임베딩 권장 |

## 자주 혼동하는 개념

| 본 개념 | 혼동 대상 | 핵심 차이 |
|---|---|---|
| Embedding lock-in | Vector store vendor lock-in | 전자는 모델 의존, 후자는 SaaS / 서비스 의존 |
| Model versioning | Index versioning | 모델 버전 = 임베딩 함수, 인덱스 버전 = 적재된 벡터 집합 |
| Re-embedding | Re-indexing (인덱스 알고리즘 변경) | 전자는 입력 임베딩 자체 재생성, 후자는 ANN 그래프/IVF 재구성 |
| Dimension match | Semantic compatibility | 차원 같다고 의미 공간 같지 않음 |
| Data drift | Embedding drift | Data drift는 입력 분포 변화, embedding drift는 모델 업그레이드로 인한 좌표계 변화 |

## 실제 사례

### "I Updated My Embedding Model and My RAG Broke" — Post-mortem
프로덕션 RAG에서 임베딩 모델만 업그레이드했더니 retrieval 품질이 즉시 붕괴. 원인: 인덱스의 옛 벡터는 옛 모델, 쿼리는 새 모델 → 다른 좌표계 비교. 복구: 전 코퍼스 재임베딩.

### Drift-Adapter (arXiv 2025-09)
새 쿼리를 옛 임베딩 공간으로 매핑하는 학습 가능 어댑터. 전 코퍼스 재임베딩을 미루는 **near-zero downtime** 업그레이드 전략. recall 손실 약 5~10% 허용 가능한 워크로드에서 유효.

### Bedrock Titan v2 출시 시 운영 가이드
AWS는 Titan v1 사용자에게 **새 KB / 새 vector index 생성**을 권장 — in-place 변경 불가. 같은 vendor지만 버전이 다르면 재임베딩 강제.

## 활용 시나리오

### 시나리오 1: 인덱스 네이밍 표준
인덱스 이름에 모델·버전·차원 명시: `rag-titan-v2-1024`. 신규 모델 도입 시 `rag-cohere-v3-1024` 인덱스 신규 생성, **혼동 방지**.

### 시나리오 2: Dual Index 점진 마이그레이션
구·신 인덱스 병행 운영 → 신규 입력은 양쪽에 적재 → 검색은 신 인덱스 우선, miss 시 구 인덱스 fallback → 옛 데이터 점진 재임베딩 → 옛 인덱스 폐기. S3V의 인덱스당 추가 기반 비용 없음이 이 패턴을 저렴하게 만든다.

### 시나리오 3: Lazy Re-embedding
검색 빈도 기반 우선순위로 자주 접근되는 문서부터 재임베딩. cold 문서는 점진적으로. archive 워크로드와 자연스럽게 결합.

### 시나리오 4: Adapter Layer (실험적)
Drift-Adapter류로 쿼리만 옛 공간에 매핑 → 인덱스 재생성 미룸. recall 손실 5~10% 허용 가능한 워크로드에서.

## 참고 자료

- [Drift-Adapter — arXiv 2025](https://arxiv.org/pdf/2509.23471) — 어댑터 기반 업그레이드 전략 (High)
- [When Good Models Go Bad — Weaviate](https://weaviate.io/blog/when-good-models-go-bad) — 모델 교체 위험 분석 (High)
- [I Updated My Embedding Model and My RAG Broke — Decompressed.io](https://decompressed.io/learn/rag-observability-postmortem) — 프로덕션 사후 분석 (High)
- [Different Embedding Models, Different Spaces — Gary Stafford / Medium](https://garystafford.medium.com/different-embedding-models-different-spaces-the-hidden-cost-of-model-upgrades-899db24ad233) — 호환성 분석 (Mid)
- [Embedding Models in Production: Versioning and Index Drift — TianPan](https://tianpan.co/blog/2026-04-09-embedding-models-production-versioning-index-drift) — 버전 관리 실무 (Mid)
- [Embedding Portability and Versioning — Mixpeek](https://mixpeek.com/guides/embedding-portability-versioning) — 호환성 가이드 (Mid)
- [Migrating Vector Databases — CallSphere](https://callsphere.ai/blog/migrating-vector-databases-pinecone-pgvector-weaviate-embeddings) — 마이그레이션 패턴 (Mid)

## 관련 노트

- [[aws-s3-vectors-api]] — dim/metric 불변이 API 수준에서 강제되는 S3V 데이터 플레인
- [[aws-s3-vectors-capacity-planning]] — 모델 교체 위험을 인덱스 분할 축으로 고려하는 용량 계획
- [[titan-text-embeddings-v2]] — Titan v1→v2 의미 공간 비호환: 재임베딩 강제 실사례
- [[cohere-embed-multilingual-v3]] — Cohere v3 고유 의미 공간: Titan 혼용 불가
- [[aws-bedrock-embedding]] — Bedrock 임베딩 API: 모델 선택이 lock-in 기점
- [[moc-embedding]] — 본 노트 소속 MOC
