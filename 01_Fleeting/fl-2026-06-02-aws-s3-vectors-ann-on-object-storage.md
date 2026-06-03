---
type: fleeting
created: 2026-06-02
modified: 2026-06-02
status: draft
tags: [vector-search, ann, object-storage, concept, latency, architecture]
domain:
  - ai-ml-llm
aliases: [ANN on Object Storage, Approximate Nearest Neighbor on S3, Storage-First Vector Search]
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
  - [[fl-2026-06-02-aws-s3-vectors-vector-index-capacity-planning]]
  - [[fl-2026-06-02-aws-s3-vectors-recall-vs-filter-tradeoff]]
  - [[fl-2026-06-02-aws-s3-vectors-vector-store-decision-matrix]]
  - [[fl-2026-06-02-aws-s3-vectors-embedding-model-lock-in]]
  - [[fl-2026-06-02-aws-s3-vectors-twelvelabs-video-intelligence-case]]
suggested_category: AI-ML-LLM/RAG
---

# ANN on Object Storage — 객체 스토리지 위의 근사 최근접 이웃 검색

## 핵심 요약

전통적으로 ANN(Approximate Nearest Neighbor) 검색은 **메모리 / 로컬 디스크**에 인덱스를 적재하는 것을 전제로 설계됐다(HNSW 그래프, IVF 클러스터 등). 객체 스토리지(S3 등) 위에서 ANN을 실현하는 시도는 **읽기 레이턴시·랜덤 액세스 패널티·작은 객체 비용** 때문에 오래 어려운 문제였고, S3 Vectors(2025-12 GA)는 이 영역의 첫 매니지드 솔루션이다. 핵심 트레이드오프는 "**메모리/로컬 ms 레이턴시 vs 객체 스토리지 sub-second 레이턴시**"를 **수 자릿수 저렴한 비용**으로 교환하는 것.

- **객체 스토리지의 제약**: 높은 read latency, 작은 객체 비효율, 그래프 기반 인덱스의 순차 read 불리.
- **해결 전략 패밀리**: (1) 파티션 기반 인덱스(IVF) + 비동기 I/O, (2) 메모리에 압축 벡터(PQ) + 스토리지에 full precision (DiskANN 계열), (3) 매니지드 추상화(S3 Vectors 처럼 구현 세부를 숨김).
- **AWS의 선택**: 구체 알고리즘 노출 없음. "storage-first RAG" 카테고리 정의.
- **성능 영역**: cold sub-second, warm ~100ms — 메모리 기반 대비 10~50배 느리지만 비용은 1/10 수준.

## 형식적 정의

벡터 집합 $V = \{v_1, ..., v_N\} \subset \mathbb{R}^d$ 와 거리 함수 $D$(cosine, L2 등)에 대해, 쿼리 $q$의 **정확한** $k$-NN은:

$$
\text{kNN}(q) = \arg\!\min_{S \subseteq V, |S|=k} \sum_{v \in S} D(q, v)
$$

ANN은 이를 **recall**(정답률) $\rho \in [0,1]$로 근사:

$$
\rho = \frac{|\hat{S}_k \cap S_k|}{k}, \quad \hat{S}_k = \text{ANN}(q,k)
$$

**Storage ANN의 추가 제약**:
- 읽기 단위 비용: $C_\text{read}(B)$ — 한 번의 read에 일정 오버헤드(예: S3 GetObject HTTP RTT).
- 읽기 양 비용: $\text{cost} \propto (\text{vectors processed}) \times \text{avg vec size}$.
- 따라서 알고리즘은 **방문 벡터 수 최소화**(IVF의 nprobe 작게, PQ로 압축 비교)를 최적화 목표로 한다.

## 멘탈 모델 및 비유

- **메모리 ANN = 동네 도서관 책장**: O(ms)로 책 한 권을 꺼낸다. 그래프(HNSW)·트리 자유.
- **객체 스토리지 ANN = 외부 창고**: 책 1권 꺼내는 데 O(100ms)~. 한 번 보낼 때 박스로 묶어서(batch / partition) 보내야 한다.
- **PQ(Product Quantization) = 책 표지 사진**: 메모리에 작은 사진만 두고 후보 좁힌 뒤 원본은 창고에서 꺼낸다.
- **S3 Vectors의 추상화 = 매니지드 창고**: 사용자는 알고리즘(IVF/HNSW/PQ)을 모르고, 박스 단위 묶음·캐시·복제는 서비스가 처리.

## 핵심 기능 및 서비스

| 차원 | 메모리/디스크 ANN | 객체 스토리지 ANN |
|---|---|---|
| 인덱스 위치 | RAM (HNSW) / SSD (DiskANN) | 객체 스토리지 (S3 등) |
| 단일 쿼리 레이턴시 | 1~수십 ms | 100ms~수 초 |
| 단일 쿼리 비용 | 인스턴스 비용 흡수 | 호출 + 처리 데이터량 |
| 초기 적재 비용 | 인덱스 빌드 (오프라인) | logical GB upload |
| Idle 비용 | 인스턴스 항상 켜져 있음 | 사실상 0 (Storage만) |
| 적합 워크로드 | 고QPS, 대화형 | 저빈도, 대용량, archive |
| 튜닝 노브 | 풍부 (M, ef, nprobe…) | 거의 없음 (매니지드) |

## 유사 기술 비교

| 항목 | S3 Vectors | DiskANN (SSD) | Faiss IVF-PQ (RAM) | Turbopuffer (S3 기반) |
|---|---|---|---|---|
| 스토리지 매체 | S3 객체 | NVMe SSD | DRAM | S3 + 캐시 |
| 단일 쿼리 latency | sub-sec~100ms | 수~수십 ms | 1~수 ms | p50 200ms~ |
| 인덱스 알고리즘 | 비공개 (매니지드) | 그래프 + PQ | IVF + PQ | 비공개 |
| 운영 부담 | 매니지드 0 | 자체 운영 SSD 클러스터 | 자체 + 라이브러리 | SaaS |
| 비용 모델 | usage-based | 인스턴스 + SSD | 인스턴스 | usage-based |

## 자주 혼동하는 개념

| 본 개념 | 혼동 대상 | 핵심 차이 |
|---|---|---|
| ANN on Object Storage | Disk-based ANN (DiskANN) | DiskANN은 로컬 SSD/NVMe 가정, 객체 스토리지는 네트워크 + 분산 + 작은 객체 비효율 추가 |
| S3 Vectors | S3 + Faiss 자체 구현 | S3 Vectors는 매니지드 서비스. 직접 구현은 인덱스/캐시/복제를 사용자가 책임 |
| Storage-first RAG | Tiered cold storage | Storage-first는 cold-only도 포함하는 패러다임, tiered는 hot/cold 분리 |
| Recall | Precision | Recall = 진짜 정답 중 잡은 비율, Precision = 잡은 것 중 정답 비율 |

## 실제 사례

### S3 Vectors GA (AWS, 2025-12)
"first cloud object storage with native vector support at scale". 객체 스토리지에 native ANN을 매니지드로 제공한 최초 클라우드 서비스로 자리매김. 인덱스당 20억 벡터.

### DiskANN — Microsoft Research (2019)
그래프 기반 ANN을 SSD에서 운영. 메모리에는 압축 벡터(PQ), 스토리지에는 그래프 + full precision. 객체 스토리지가 아닌 로컬 NVMe 가정이지만 "메모리 외부 ANN"의 기반 작업.

### Turbopuffer — S3 기반 commercial vector + text search
S3를 백엔드로 하는 SaaS 벡터 검색. cold writes/queries p50 > 200ms — 객체 스토리지 ANN의 latency 특성을 그대로 노출.

### 학술: "ANN of Large Scale Vectors on Distributed Storage" (arXiv 2025)
비동기 I/O로 computation과 data retrieval을 분리해 storage latency 흡수. 객체 스토리지 ANN의 표준 패턴이 학술적으로도 정립.

## 활용 시나리오

### 시나리오 1: 매니지드 추상화 활용
직접 S3 + Faiss로 구현하지 말고 S3 Vectors / Turbopuffer / Bedrock KB에 위임. 알고리즘·캐시·복제·확장을 모두 추상화 — 운영 인력의 시간을 자체 알고리즘 튜닝이 아닌 RAG 품질에 투입.

### 시나리오 2: 학술·연구 — DiskANN 자체 구현
객체 스토리지 ANN의 latency 특성을 연구. PQ 압축비, partition size, 비동기 I/O를 직접 튜닝해 sub-second SLO 달성. 매니지드 서비스의 한계를 넘어야 할 때.

### 시나리오 3: 비용 최적화 RAG
"메모리 ANN 인프라가 너무 비싸다"는 사실에서 시작 → 1억+ 벡터 코퍼스를 메모리에 두지 말고 객체 스토리지 ANN으로 이동. 비용 1/10, latency 10~50배 감수. 워크로드가 cold-friendly 일 때만 유효.

## 참고 자료

- [On Storage Neural Network Augmented ANN Search — arXiv 2025](https://arxiv.org/html/2501.16375v1) — storage ANN 기법 (High)
- [ANN of Large Scale Vectors on Distributed Storage — arXiv 2025](https://arxiv.org/html/2510.17326v1) — 분산 스토리지 ANN (High)
- [A Developer's Guide to ANN Algorithms — Pinecone](https://www.pinecone.io/learn/a-developers-guide-to-ann-algorithms/) — ANN 알고리즘 개요 (High)
- [Efficient Graph-Based ANN: Low Latency Without Throughput Loss — arXiv 2025](https://arxiv.org/pdf/2504.20461) — 그래프 ANN 한계 (Mid)
- [Amazon S3 Vectors GA — AWS News Blog](https://aws.amazon.com/blogs/aws/amazon-s3-vectors-now-generally-available-with-increased-scale-and-performance/) — 매니지드 storage ANN (High)
- [Amazon S3 Vectors Reaches GA "Storage-First" RAG — InfoQ](https://www.infoq.com/news/2026/01/aws-s3-vectors-ga/) — 카테고리 정의 (Mid)
- [Vector database options — AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/choosing-an-aws-vector-database-for-rag-use-cases/vector-db-options.html) — 스토리지 매체별 선택 (High)
