---
type: fleeting
created: 2026-06-02
modified: 2026-06-02
status: draft
tags: [vector-search, recall, filter, ann, concept, tradeoff]
domain:
  - ai-ml-llm
aliases: [Recall vs Filter Trade-off, Pre-filter vs Post-filter, Filtered ANN]
literature_source: []
related:
  - [[fl-2026-06-02-aws-s3-vectors]]
  - [[fl-2026-06-02-aws-s3-vectors-vector-bucket-and-index-resource-model]]
  - [[fl-2026-06-02-aws-s3-vectors-api-putvectors-queryvectors]]
  - [[fl-2026-06-02-aws-s3-vectors-metadata-filterable-vs-non-filterable]]
  - [[fl-2026-06-02-aws-s3-vectors-cost-model]]
  - [[fl-2026-06-02-aws-s3-vectors-kms-per-index-multitenancy]]
  - [[fl-2026-06-02-aws-s3-vectors-cloudformation-privatelink-tagging]]
  - [[fl-2026-06-02-aws-s3-vectors-hot-cold-vector-tiering]]
  - [[fl-2026-06-02-aws-s3-vectors-cold-tier-rag-archive]]
  - [[fl-2026-06-02-aws-s3-vectors-bedrock-kb-managed-rag-integration]]
  - [[fl-2026-06-02-aws-s3-vectors-ann-on-object-storage]]
  - [[fl-2026-06-02-aws-s3-vectors-vector-index-capacity-planning]]
  - [[fl-2026-06-02-aws-s3-vectors-vector-store-decision-matrix]]
  - [[fl-2026-06-02-aws-s3-vectors-embedding-model-lock-in]]
  - [[fl-2026-06-02-aws-s3-vectors-twelvelabs-video-intelligence-case]]
suggested_category: AI-ML-LLM/RAG
---

# Recall vs Filter Trade-off — Vector Search의 Achilles Heel

## 핵심 요약

ANN 인덱스에 메타데이터 필터를 적용하면 **recall**(정답률)이 떨어지는 보편 현상. 원인은 ANN 그래프/파티션 구조가 **모든 벡터 분포**에 최적화돼 있는데, 필터가 그 그래프의 일부를 끊거나(pre-filter) Top-K를 사후 잘라내(post-filter) 잠재 정답을 놓치기 때문이다. S3 Vectors의 경우 필터 적용 시 **recall이 50% 이하로 떨어지는 사례**가 보고되며, 알고리즘 노브가 없어 튜닝으로 회복하기 어렵다 — 이 한계는 워크로드 설계 자체로 다뤄야 한다.

- **두 기본 전략**: pre-filter(필터 후 ANN) / post-filter(ANN 후 필터). 둘 다 한계 있음.
- **세 번째 길**: in-algorithm filtering — 인덱스 알고리즘이 필터를 인지 (Pinecone, Qdrant, Weaviate ACORN 등). 일부 벤더만 지원.
- **S3 Vectors**: 필터 가능하지만 **튜닝 불가**, 강한 필터는 recall 크게 감소. 따라서 **거친 분리**(tenant·언어 등)에만 사용 권장.
- **대응**: oversampling(Top-K 키우기), LLM re-ranking, hybrid search(BM25 등)로 보완.

## 형식적 정의

벡터 집합 $V$, 필터 술어 $\phi: V \to \{0,1\}$, 쿼리 $q$의 정답 집합:

$$
S_k = \arg\!\min_{S \subseteq V_\phi, |S|=k} \sum_{v \in S} D(q,v), \quad V_\phi = \{v \in V: \phi(v)=1\}
$$

세 전략의 정확률(recall):

- **Pre-filter ANN**: $V_\phi$에서 ANN 빌드 / 탐색. 정확하나 인덱스 재사용 불가, 그래프 sparsity로 정확도 하락. 비용 ↑, recall은 $\rho_\text{pre} \leq 1$.
- **Post-filter ANN**: $V$에서 Top-$K'$ 추출 후 $\phi$ 적용. $K'$가 작으면 정답 누락:
  $$
  \rho_\text{post}(K', |V_\phi|/|V|) \to 0 \text{ as filter selectivity } \uparrow
  $$
- **In-algorithm filter**: 인덱스 탐색 중 $\phi$ 평가. 알고리즘 수정 필요.

선택률(selectivity) $s = |V_\phi|/|V|$ 가 작을수록 post-filter recall이 급격히 감소.

## 멘탈 모델 및 비유

- **ANN 인덱스 = 도서관 책꽂이**: 비슷한 책끼리 묶어 정리.
- **Post-filter = "역사 책만"이라고 외친 뒤 찾기 시작**: 비슷한 책 후보군 K개 뽑은 다음 역사 책만 남김. K가 작으면 역사 책이 한 권도 없을 수 있음.
- **Pre-filter = 역사 책만 따로 모은 임시 서가**: 정확하지만 서가 재배치 비용. 작은 서가에서는 "비슷한 책" 관계가 잘 안 보임.
- **In-algorithm filter = 책마다 "역사" 태그가 보이고 탐색 중 무시**: 가장 좋지만 책꽂이가 그렇게 설계돼 있어야 함.

## 핵심 기능 및 서비스

| 전략 | 동작 | recall | latency | 인덱스 |
|---|---|---|---|---|
| Pre-filter (단순) | 필터 → ANN | 부분집합에서 정확 | 매번 재구성 비용 | 동적 |
| Post-filter | ANN K → 필터 | selectivity 낮으면 급락 | 빠름 | 정적 |
| In-algorithm filter | ANN 탐색 중 필터 평가 | 높음 | 비슷 또는 약간 ↑ | 알고리즘 수정 |
| Oversampling | K를 N배 키워 post-filter | 향상 (N만큼) | latency·비용 ↑ | 정적 |
| Multi-stage | 거친 필터 + ANN + LLM 재랭킹 | 높음 (특히 의미 필터) | 추가 단계 | 정적 |

## 유사 기술 비교

| 항목 | S3 Vectors | OpenSearch | Pinecone | Qdrant |
|---|---|---|---|---|
| 필터링 방식 | post-filter 추정, 비공개 | pre/post 선택 | in-algorithm (단일 패스) | in-algorithm + payload index |
| 튜닝 노브 | 없음 | HNSW M, ef, num_candidates | 자동 | payload index, filter |
| 강한 필터 시 recall | **<50% 사례 보고** | num_candidates 조절로 회복 | 안정 | 안정 |
| 권장 사용 | 거친 필터만 | 정밀 필터 가능 | 정밀 필터 가능 | 정밀 필터 가능 |

## 자주 혼동하는 개념

| 본 개념 | 혼동 대상 | 핵심 차이 |
|---|---|---|
| Recall | Precision | Recall = 진짜 정답 중 잡은 비율, Precision = 잡은 것 중 정답 |
| Recall@K | mAP | Recall@K는 K개 결과 안에 정답이 있는 비율, mAP는 정답 위치까지 가중 |
| Selectivity | Cardinality | Selectivity = 필터 통과 비율(작을수록 selective), Cardinality = 고유 값 수 |
| Pre-filter | Pre-processing 필터 | Pre-filter는 탐색 직전, pre-processing은 인덱싱 단계 필터 |
| In-algorithm filter | Post-filter with metadata index | 후자는 별도 인덱스로 후처리, 전자는 ANN 탐색 자체에 통합 |

## 실제 사례

### S3 Vectors — 필터 적용 시 recall 50% 이하 보고
모 노트의 비교 표에서 인용. 강한 필터(tenant + category + date 조합 등) 적용 시 정답 절반 누락 가능 — **튜닝 불가**가 핵심 제약.

### pgvector — pre-filtering 미지원으로 ANN recall 감소
pgvector는 IVF/HNSW에 post-filter만 가능. WHERE 절이 selective하면 후보군에서 다 걸러져 빈 결과 발생. MongoDB 등에서 인용된 사례.

### Pinecone — 통합 메타 + 벡터 인덱스 (ICML 2025)
Pinecone의 in-algorithm filtering 논문. 필터 술어를 인덱스 탐색에 통합해 pre-filter의 정확성 + ANN의 속도를 단일 패스에서 달성. S3 Vectors와의 핵심 차별점.

### Milvus — Filtering Without Killing Recall
필터링 영향을 측정하는 모범 사례 정리. selectivity가 10% 이하면 ANN 알고리즘에 따라 recall이 20~80%까지 분포. 워크로드 측정 후 결정.

## 활용 시나리오

### 시나리오 1: S3 Vectors의 안전한 필터 패턴
**tenant_id**, **language**, **doc_type** 등 high-selectivity·거친 분리만 filterable로. 의미 기반 정밀 필터("법무 분야 중 계약")는 LLM 재랭킹 또는 hybrid search로 후처리.

### 시나리오 2: Oversampling 패턴
Top-K=10이 필요한데 강한 필터 적용 → Top-K=100으로 oversample → 필터 적용 → 상위 10. latency·비용 trade-off지만 recall 회복.

### 시나리오 3: Multi-stage retrieval
(1) 거친 필터 + ANN Top-K=100 → (2) BM25 / hybrid 검색 추가 → (3) LLM 재랭킹 Top-10. S3 Vectors의 약한 필터 표현력 한계를 외부 단계로 보완.

## 참고 자료

- [The Achilles Heel of Vector Search: Filters — Yudhiesh](https://yudhiesh.github.io/2025/05/09/the-achilles-heel-of-vector-search-filters/) — 보편 트레이드오프 (High)
- [Pre-filtering vs Post-filtering in Vector Search — apxml](https://apxml.com/courses/advanced-vector-search-llms/chapter-2-optimizing-vector-search-performance/advanced-filtering-strategies) — 두 전략 비교 (High)
- [Vector Search Filtering: how it works — Elasticsearch Labs](https://www.elastic.co/search-labs/blog/vector-search-filtering) — 실제 구현 (High)
- [Filtering Without Killing Recall — Milvus Blog](https://milvus.io/blog/how-to-filter-efficiently-without-killing-recall.md) — 측정 사례 (High)
- [A Complete Guide to Filtering in Vector Search — Qdrant](https://qdrant.tech/articles/vector-search-filtering/) — in-algorithm filter (High)
- [Vector Query Filters — Azure AI Search](https://learn.microsoft.com/en-us/azure/search/vector-search-filters) — 정의·기법 (High)
- [Accurate and Efficient Metadata Filtering — Pinecone ICML 2025](https://www.pinecone.io/research/ICML_2025.pdf) — in-algorithm filter 논문 (High)
- [No pre-filtering in pgvector reduces ANN recall — Dev.to MongoDB](https://dev.to/mongodb/no-pre-filtering-in-pgvector-means-reduced-ann-recall-1aa1) — pgvector 한계 (Mid)
- [All About Filtered Vector Search — MyScale](https://www.myscale.com/blog/filtered-vector-search-in-myscale/) — 트레이드오프 사례 (Mid)
