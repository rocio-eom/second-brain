---
type: fleeting
created: 2026-06-02
modified: 2026-06-02
status: draft
tags: [vector-search, quantization, product-quantization, scalar-quantization, binary-quantization, memory-optimization, ai-ml-llm]
domain:
  - ai-ml-llm
aliases: [Vector Quantization, Product Quantization, Scalar Quantization, Binary Quantization, Vector Compression]
literature_source: []
related:
  - "[[fl-2026-06-02-vector-search-performance-tuning-overview]]"
  - "[[fl-2026-06-02-vector-search-performance-tuning-index-params]]"
  - "[[fl-2026-06-02-vector-search-performance-tuning-system-query]]"
  - "[[fl-2026-06-02-rag-data-ingestion-architecture-schema]]"
suggested_category: AI-ML-LLM/RAG
---

# Vector Search Performance Tuning — 양자화 / 메모리 최적화 (PQ · SQ · Binary)

## 핵심 요약

양자화(Quantization)는 vector를 더 작은 비트 표현으로 압축해 **메모리·디스크·캐시 효율을 4~32배** 개선하는 기법으로, vector search 성능 튜닝에서 가장 큰 메모리·비용 절감을 제공하는 layer다. 핵심은 **압축률 vs recall 손실**의 trade-off를 워크로드에 맞게 선택하고, **rescore(2단계 검색)**로 손실을 보정해 production-grade 품질을 유지하는 것.

- **3대 양자화 기법**: Scalar(SQ, 4× 압축, 손실 최소) / Product(PQ, 8~64× 압축, codebook 학습) / Binary(BQ, 32× 압축, Hamming distance + popcount 가속)
- **압축은 인덱스와 직교**: HNSW·IVF 어떤 인덱스와도 결합 가능, 양자화는 데이터 layer를 줄이고 인덱스는 탐색 구조를 줄임
- **Rescore가 production 표준**: 양자화로 빠르게 top-N(보통 50~200) 후보 추출 → 원본(또는 SQ) vector로 재정렬 → recall 손실 대부분 회복
- **HW 가속 차원이 다르다**: Binary는 popcount(64bit/cycle), SQ는 SIMD int8, PQ는 SIMD codebook lookup — CPU 캐시 효율도 함께 따져야 함

## 컴포넌트 다이어그램

3개 양자화 기법은 같은 입력 vector를 받지만 압축 단위(차원 단위 vs 차원묶음 단위 vs 비트 단위)가 다르다.

```mermaid
graph TD
    V[Original Vector<br/>float32 x dim]

    subgraph SQ[Scalar Quantization]
        S1[차원 단위 양자화<br/>float32 → int8/fp16]
        S2[4x 압축, 손실 최소]
    end

    subgraph PQ[Product Quantization]
        P1[Vector 분할<br/>dim → m subvectors]
        P2[각 subvector를 codebook의<br/>가장 가까운 centroid id로 치환]
        P3[m bytes (보통 8~64x 압축)]
        P4[Codebook 학습 필요<br/>k-means per subvector]
    end

    subgraph BQ[Binary Quantization]
        B1[각 차원을 1bit<br/>(threshold 0 기준)]
        B2[32x 압축]
        B3[Hamming distance + popcount]
    end

    V --> S1 --> S2
    V --> P1 --> P2 --> P3
    V --> B1 --> B2 --> B3

    subgraph Rescore[2-Stage Rescore]
        R1[1단계: 양자화 vector로<br/>top-N (N=50~200) 후보]
        R2[2단계: 원본/SQ vector로<br/>top-k 재정렬]
    end

    S2 -.-> R1
    P3 -.-> R1
    B3 -.-> R1
    R1 --> R2
```

## 적용 단계

양자화 도입은 1) 압축률 목표 설정 → 2) 기법 선택 → 3) rescore 결합 → 4) 측정의 순서로 진행.

```mermaid
flowchart LR
    A[메모리/비용 SLO 정의<br/>"GB → GB 목표"] --> B[Baseline: 원본 fp32 측정<br/>recall@k vs latency]
    B --> C[SQ int8 도입<br/>4x 압축]
    C --> D{recall 손실<br/>허용 범위?}
    D -->|예| E[채택]
    D -->|아니오| F[Rescore 결합<br/>top-N → fp32 재정렬]
    F --> G{더 큰 압축 필요?}
    G -->|예| H[PQ 또는 Binary 시도]
    G -->|아니오| E
    H --> I[Codebook 학습 / threshold 설정]
    I --> J[Rescore 포함 재측정]
    J --> E
```

1. **메모리 SLO 정의**: 현재 메모리 사용량과 목표(예: 200GB → 50GB)를 명시 — 압축률(4× / 8× / 32×)로 변환
2. **Baseline 측정**: 원본 fp32에서 `recall@10`과 `p95 latency` 곡선 그리기
3. **SQ int8 도입**: 코드 변경 최소(대부분 vendor가 옵션 하나로 활성화), 4× 압축, recall 손실 보통 < 1%
4. **압축 부족 시 PQ 시도**: subvectors 수(m) 선택 — `dim / m` 비율이 8 이상이면 일반적, 학습 데이터 sample 필요 (보통 100k vectors)
5. **메모리 1순위면 Binary**: dim ≥ 1024에서 효과 큼, threshold는 보통 0 또는 차원별 median
6. **Rescore 결합**: 양자화 단계에서 top-N(보통 k의 10~20배) 가져온 후 원본 또는 SQ로 재정렬 → recall 보정
7. **운영 metric 추가**: 압축률, rescore 평균 N, rescore 후 recall@k, 메모리·캐시 사용량 모니터

## 핵심 기능 및 서비스

| 기법 | 압축률 | Recall 손실(일반) | 학습 필요 | HW 가속 | 적합 케이스 |
|---|---|---|---|---|---|
| Scalar (int8) | 4× | < 1% | 없음 (min-max scan만) | SIMD int8 | 가장 보편적 기본 옵션, 거의 손실 없음 |
| Scalar (fp16) | 2× | 거의 0 | 없음 | SIMD fp16 | recall 보전 최우선 |
| Product (PQ-8) | 8~16× | 3~10% | codebook 학습 | SIMD lookup | 대용량 + 메모리 우선 |
| Product (PQ-64) | 32~64× | 10~20% (rescore 시 회복) | codebook 학습 | SIMD lookup | 극단적 메모리 절감 + rescore |
| Binary | 32× | 10~20% (rescore 시 95%+ 회복) | threshold만 | popcount (64bit/cycle) | dim 충분(≥1024) + rescore 필수 |
| Matryoshka (참고) | dim 단계 절단 | dim별 비례 | 모델이 학습되어 있어야 | 일반 SIMD | OpenAI 3-large 등에서 지원 |

## 유사 기술 비교

| 항목 | Scalar Quantization | Product Quantization | Binary Quantization |
|---|---|---|---|
| 단위 | 차원당 | 차원 묶음(subvector)당 | 차원당 1bit |
| 코드북 | 없음 (min-max scaling) | 있음 (k-means per subvector) | 없음 (threshold) |
| 비교 연산 | int8 dot/L2 | LUT 기반 PQ ADC | Hamming + popcount |
| 메모리 | 1/4 | 1/8 ~ 1/64 | 1/32 |
| Recall 손실 | 거의 없음 | 중간~큼 | 큼(rescore로 보정) |
| 적합 dim | 모든 dim | 대부분 dim, codebook 학습 가능한 분포 | 보통 ≥ 1024에서 효과 큼 |
| 빌드 비용 | 거의 0 | 학습 데이터 + k-means | threshold 계산만 |

## 실제 사례

### Qdrant Binary Quantization (40× 가속 / 32× 메모리 절감)
Qdrant 공식 글이 OpenAI `text-embedding-ada-002`(dim=1536) + dbpedia 데이터셋으로 binary quantization 적용 시 검색 속도 40배 향상, 4× oversampling + rescore로 recall@100 ≥ 0.98 달성을 보고. recall 손실은 oversampling + rescore(원본 vector로 top-N 재정렬)로 회복 — 양자화와 rescore가 paired 운영되는 production 모델의 대표 사례.

### MongoDB Atlas Vector Search Auto-Quantization
MongoDB가 SQ/BQ를 자동 적용하는 Auto-Quantization을 공식 지원. 메모리 60~75% 절감을 보고하면서 recall은 rescore 활성화 시 95%+ 유지. Auto 옵션은 dimension·corpus 크기를 보고 SQ vs BQ를 자동 선택 — 양자화 의사결정의 자동화 흐름.

## 활용 시나리오

### 시나리오 1: "메모리만 줄이고 싶다, 손실 거의 없이"
- **맥락**: 기존 fp32 인덱스가 메모리 70% 차지, 비용 압박. recall은 1% 미만 손실만 허용
- **선택**: SQ int8만 활성화 (코드 1줄), rescore 없이도 OK
- **이유**: 4× 압축으로 메모리 70% → 17.5%, recall 손실 < 1%가 일반적 — 대부분 production이 여기서 멈춤

### 시나리오 2: "초대용량 + 빠른 검색, recall 약간은 양보 가능"
- **맥락**: 1억+ vector, dim=1536, 메모리 한도 강함, recall@10 ≥ 0.92 가능
- **선택**: PQ-32 (32× 압축) + IVF + rescore (top-200을 원본으로 재정렬)
- **이유**: PQ + IVF는 대용량의 표준 조합. rescore가 recall 손실의 절반 이상을 회복

### 시나리오 3: "극단적 비용 절감 + dim≥2048"
- **맥락**: dim=3072 (OpenAI 3-large), 5억 vector, 비용 1순위
- **선택**: Binary quantization + oversampling(k의 5~10배) + rescore (SQ 또는 원본)
- **이유**: Hamming + popcount로 single-pass 매우 빠름, 큰 dim에서 정보 손실이 적어 rescore 회복률 높음 (95%+)

## 장단점 및 트레이드오프

| 항목 | 내용 |
|---|---|
| 장점 | 메모리·디스크·캐시 비용을 4~32배 절감. 인덱스와 직교라 기존 HNSW/IVF에 추가 적용 가능. Binary는 검색 속도도 함께 개선 |
| 단점 | Recall 손실이 inevitable — rescore 없이는 production 품질 보장 어려움. PQ는 codebook 학습 데이터 sample 필요, 데이터 분포 변화 시 재학습 필요. dim이 작거나(<256) corpus가 작으면(<100만) 효과 미미 |
| 트레이드오프 | **압축률 vs Recall**: SQ → PQ → Binary 순으로 압축↑·손실↑ / **Rescore N vs Latency**: N↑ → recall↑·latency↑ / **Codebook 학습 비용 vs 압축률**: PQ subvectors↑ → 압축↑·학습↑ |

## 함정 및 안티패턴

- **양자화만 활성화하고 rescore 생략**: PQ/Binary에서 recall이 production-grade 아래로 떨어짐 → 항상 oversampling + rescore를 paired로 운영
- **작은 corpus·낮은 dim에 PQ 적용**: < 100만 vector 또는 dim < 256에서는 압축 이득이 적고 codebook 학습 오버헤드만 발생 → SQ로 충분
- **데이터 분포 변화 후 codebook 재학습 생략**: PQ는 학습 시점의 분포에 종속 — 새 도메인 추가 시 recall이 조용히 하락 → 분포 drift를 metric으로 감시
- **Binary threshold를 무조건 0으로**: dim별 distribution이 비대칭이면 정보 손실 큼 → median 또는 학습된 threshold 사용
- **재인제션 없이 양자화 정책 변경**: 양자화 방식 변경은 사실상 인덱스 재빌드 → namespace 분리 후 dual-write 권장
- **압축률만 보고 채택**: rescore가 추가하는 메모리(원본 vector 또는 SQ vector 보관)와 latency를 무시 → 총 메모리·latency를 함께 측정
- **양자화로 recall 문제를 임베딩 문제와 혼동**: 양자화 손실은 보통 단조적(rescore로 회복), 임베딩 모델 적합도 문제는 rescore해도 해결 안 됨 → 분리 분석

## 참고 자료

- [Vector Quantization Methods (Qdrant)](https://qdrant.tech/course/essentials/day-4/what-is-quantization/) — SQ/PQ/BQ 비교 공식 가이드
- [Binary Quantization - Vector Search, 40x Faster (Qdrant)](https://qdrant.tech/articles/binary-quantization/) — Binary + rescore 결합 + 40× 가속 사례
- [Quantization (Qdrant Docs)](https://qdrant.tech/documentation/manage-data/quantization/) — 양자화 설정·rescore 옵션 운영 문서
- [Why Vector Quantization Matters For AI Workloads (MongoDB)](https://www.mongodb.com/company/blog/innovation/why-vector-quantization-matters-for-ai-workloads) — Auto-Quantization · production 적용 사례
- [Compression (Vector Quantization) (Weaviate Documentation)](https://docs.weaviate.io/weaviate/concepts/vector-quantization) — SQ/PQ/BQ 공식 가이드 + 적용 기준
- [Compress Vectors Using Quantization (Azure AI Search, Microsoft Learn)](https://learn.microsoft.com/en-us/azure/search/vector-search-how-to-quantization) — Azure의 양자화 + rescore 통합 가이드
- [Scaling Vector Search: Comparing Quantization and Matryoshka Embeddings (Towards Data Science)](https://towardsdatascience.com/649627-2/) — 양자화 vs Matryoshka 비용 분석
- [Cost Optimized Vector Database (AWS OpenSearch)](https://aws.amazon.com/blogs/big-data/cost-optimized-vector-database-introduction-to-amazon-opensearch-service-quantization-techniques/) — AWS OpenSearch 양자화 기법 소개
