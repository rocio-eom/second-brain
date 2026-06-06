---
type: permanent
created: 2026-06-06
modified: 2026-06-06
status: draft
domain: backend
moc: "[[moc-backend-architecture]]"
tags: [storage, tiering, hot-tier, cold-tier, warm-tier, architecture, cost-optimization, data-lifecycle, rag, vector-store, ai-ml-llm]
aliases: [Hot-Cold Data Tiering, Storage Tiering, Hot Tier Cold Tier, Tiered Storage, Hot Warm Cold Architecture]
promoted_from: fl-2026-06-06-hot-cold-data-tiering
related:
  - "[[aws-s3-vectors-api]]"
  - "[[bedrock-kb-s3-vectors-integration]]"
  - "[[bedrock-knowledge-bases]]"
---

# Hot/Cold Data Tiering

## 핵심 요약

**Hot/Cold Data Tiering**은 데이터를 접근 빈도에 따라 서로 다른 비용·성능 특성의 스토리지 계층(tier)에 배치하는 아키텍처 패턴이다. 자주 조회되는 데이터는 고성능·고비용 스토리지(Hot), 드물게 조회되는 데이터는 저비용·고지연 스토리지(Cold)에 두어 성능과 비용을 동시에 최적화한다.

- **비용 절감**: 전체 데이터를 Hot에 두는 대비 40~70% 스토리지 비용 절감이 일반적으로 보고됨.
- **성능 유지**: Hot tier는 자주 쓰는 데이터에만 고성능 자원을 집중, 불필요한 오버프로비저닝 방지.
- **생명주기 자동화**: 접근 패턴을 기반으로 데이터가 자동으로 티어 간 이동(승격/강등)됨.
- **적용 범위**: 검색 인덱스(Elasticsearch ILM), 오브젝트 스토리지(S3 Lifecycle), 벡터 DB(Hot OSS ↔ Cold S3 Vectors) 등 영역 불문.
- **임계값은 도메인 정의**: "자주"·"드물게"에 대한 범용 표준은 없음. 대부분의 시스템은 접근 횟수가 아닌 **마지막 접근 이후 경과 시간(Recency)** 으로 판단 (예: S3 Intelligent-Tiering 30일 미접근 → IA 이동).

## 형식적 정의

티어 집합 $T = \{T_1, T_2, \ldots, T_n\}$ 을 비용 내림차순(성능 내림차순)으로 정렬한다:

$$
\text{cost}(T_1) > \text{cost}(T_2) > \cdots > \text{cost}(T_n)
$$
$$
\text{latency}(T_1) < \text{latency}(T_2) < \cdots < \text{latency}(T_n)
$$

데이터 객체 $d$ 의 접근 빈도(또는 재접근 시간) $f(d, t)$ 를 시간 윈도우 $t$ 기준으로 측정했을 때, 두 임계값 $\theta_{\uparrow}, \theta_{\downarrow}$ ($\theta_{\uparrow} > \theta_{\downarrow}$) 에 따라 티어 배정이 결정된다:

$$
\text{tier}(d) =
\begin{cases}
T_{i-1} & \text{if } f(d) > \theta_{\uparrow} \quad \text{(승격, Promotion)} \\
T_{i+1} & \text{if } f(d) < \theta_{\downarrow} \quad \text{(강등, Demotion)} \\
T_i & \text{otherwise}
\end{cases}
$$

**핵심 불변**: 데이터 객체는 어느 한 시점에 **정확히 하나의 티어에만** 존재한다 (티어링 ≠ 복제).

실무에서는 보통 Hot / Warm / Cold / (Frozen / Archive) 3~4 계층으로 구현된다.

| 계층 | 접근 기준(재접근 시간) | 스토리지 유형 | 지연 시간 | 대략 비용 |
|---|---|---|---|---|
| Hot | ~7–30일 이내 접근 | NVMe SSD / In-memory | < 1ms | ~$0.10–1.00/GB/월 |
| Warm | ~30–90일 이내 접근 | SSD / HDD 혼합 | 10~50ms | ~$0.03–0.10/GB/월 |
| Cold | ~90일–1년 미접근 | 오브젝트 스토리지(S3) | 50~500ms | ~$0.004–0.023/GB/월 |
| Frozen / Archive | 1년 이상 미접근 | S3 Glacier / 테이프 | 수분~수시간 | ~$0.001–0.004/GB/월 |

> **버전 민감 항목 flag**: 접근 기준 일수와 단가는 클라우드 공급자·지역·서비스별로 변동합니다. 최신 단가는 [AWS S3 pricing](https://aws.amazon.com/s3/pricing/) 등 공식 페이지에서 확인하세요. 임계값은 비즈니스 요구에 맞게 직접 설정합니다.

## 멘탈 모델 및 비유

**도서관 서가 배치**:
- 사서 책상 위(Hot) — 오늘 대출 요청이 잦은 책. 즉시 집을 수 있음.
- 1층 서가(Warm) — 이번 달 대출 기록이 있는 책. 30초면 찾음.
- 지하 창고(Cold) — 1년 이상 대출 이력 없음. 서고 직원 요청 후 10분 소요.
- 외부 창고(Archive) — 20년 이상 미사용. 반출 요청 후 이틀 소요.

**핵심 직관**: 책이 창고에 있어도 "없어진 게 아님". 필요하면 꺼낼 수 있지만 시간이 더 걸린다. 티어링은 데이터의 존재 자체가 아니라 **접근 속도와 비용을 조절**하는 것이다.

**빈도 vs 재접근 시간**: 현실 구현에서 "자주/드물게"는 거의 항상 **마지막 접근 후 경과 시간(Recency)** 으로 판단한다. 빈도 카운팅은 모든 read 이벤트를 기록·집계해야 하므로 오버헤드가 크다. Recency는 타임스탬프 하나로 판단 가능하다.

## 핵심 기능 및 서비스

| 기능 | 설명 |
|---|---|
| 자동 승격(Promotion) | 접근 빈도가 임계값 초과 시 Cold→Hot으로 데이터 이동 |
| 자동 강등(Demotion) | 일정 기간 미접근 시 Hot→Cold로 이동 |
| 생명주기 정책(Lifecycle Policy) | 시간 기반 자동 규칙 정의 (예: 30일 미접근 후 Cold) |
| 투명한 접근 | 이상적으로 애플리케이션은 계층을 몰라도 되며 단일 인터페이스로 접근 |
| 비용 모니터링 | 티어별 스토리지 사용량 + 조회 비용 추적 |

## 유사 기술 비교

| 항목 | Hot/Cold Tiering | 캐싱(Caching) | 아카이빙(Archiving) | 복제(Replication) |
|---|---|---|---|---|
| 데이터 위치 | 한 시점에 **1개 계층** | 원본 + 캐시 **2곳 동시 존재** | 원본 이동, 복구 비용 큼 | 여러 곳에 **동일 복사본** |
| 목적 | 비용·성능 균형 | 읽기 속도 향상 | 장기 보존, 규정 준수 | 가용성·내결함성 |
| 데이터 이동 방향 | 양방향 (승격↔강등) | 일방향(원본→캐시) + 만료 | 사실상 일방향 (복구는 예외적) | 지속 동기화 |
| 지연 시간 특성 | 계층별 다름 | 캐시 히트 시 최저 | 느림 (복구 수분~수일) | 복제본은 Hot과 동일 |
| 적합 케이스 | 수명주기가 긴 대규모 데이터 | 읽기 집약적 서비스 | 컴플라이언스·감사 보존 | 재해 복구(DR), HA |

## 자주 혼동하는 개념

| 본 개념 | 혼동 대상 | 핵심 차이 |
|---|---|---|
| Hot/Cold Tiering | 캐싱 | 티어링은 데이터를 **이동**(단일 위치); 캐싱은 **복사**(원본+캐시 공존) |
| Hot/Cold Tiering | 아카이빙 | 아카이빙은 사실상 **일방향**·장기 보존; 티어링은 **양방향** 생명주기 관리 |
| Cold Tier | Cold Start | Cold Start는 컴퓨트(함수/컨테이너)의 초기화 지연; Cold Tier는 스토리지 계층 |
| Hot Data | Hot Spot | Hot Spot은 특정 파티션/노드에 부하 집중(분배 문제); Hot Data는 접근 빈도 기반 분류 |
| Tiering | Sharding | Sharding은 데이터를 **분산**(스케일아웃); Tiering은 **비용·성능 계층** 분리 |

## 실제 사례

### Elasticsearch / Elastic Stack — ILM(Index Lifecycle Management)
로그·시계열 인덱스에 Hot(7일) → Warm(30일) → Cold(6개월) → Frozen(1년+) → Delete 정책을 ILM으로 자동화. Hot 노드는 NVMe SSD, Cold/Frozen 노드는 HDD + S3 직접 검색(searchable snapshots). 단일 클러스터 내에서 비용 40~70% 절감.

### AWS OpenSearch Service — UltraWarm + Cold Storage
UltraWarm은 Hot(SSD)과 Cold(S3) 사이의 Warm 계층. 초 단위로 Cold 데이터를 UltraWarm에 attach해 온디맨드 검색 가능. S3 Vectors를 Cold tier로 활용하는 벡터 검색 패턴도 2025년부터 공식 지원.

### Amazon S3 Intelligent-Tiering
오브젝트 접근 패턴을 ML로 모니터링해 자동으로 Standard(30일 이내 접근) → Infrequent Access(30일 미접근) → Glacier Instant(90일 미접근) 간 이동. 사람이 생명주기 규칙을 작성하지 않아도 되는 완전 자동 티어링. S3 Intelligent-Tiering이 구체적인 Recency 기반 임계값의 대표 사례.

### RAG 벡터 파이프라인 — Hot OSS + Cold S3 Vectors
자주 쿼리되는 벡터 → OpenSearch(Hot), 아카이브 코퍼스 → S3 Vectors(Cold). 신규 문서는 Hot 인덱스에 적재 후 60일 미접근 시 S3 Vectors로 이동. 동일 Retrieve API 뒤에 두 인덱스를 라우팅하여 애플리케이션은 계층 구조를 인식하지 않음.

## 활용 시나리오

### 시나리오 1: 로그 분석 플랫폼 비용 최적화
최근 7일 로그만 핫 쿼리됨. 7일 이후 Warm, 30일 이후 Cold로 자동 이동. 6개월 후 삭제. Hot 노드 SSD 용량을 최소화하면서 90일치 데이터를 검색 가능하게 유지. 실제 쿼리가 필요한 Cold 데이터는 Cold→Warm attach로 수 초 내 복구.

### 시나리오 2: RAG 아카이브 — 재접근 시간 기반 벡터 이동
사내 위키 Q&A RAG. 최근 60일 내 접근된 문서 → Hot(OSS), 60일 이상 미접근 문서 → Cold(S3 Vectors). 질의 시 Hot 먼저 검색, miss 시 Cold 폴백. S3 Vectors의 low-idle-cost 특성으로 전체 코퍼스 유지 비용 최소화.

### 시나리오 3: 멀티테넌트 SaaS — 테넌트 활동 기반 티어링
활성 테넌트(MAU > 100): 데이터 Hot tier. 휴면 테넌트(90일 미접속): Cold tier로 자동 이동. 테넌트 재활성화 시 Cold→Hot 승격(수초~수분). 전체 스토리지 비용을 테넌트 활동에 비례하게 유지.

## 참고 자료

- [Elasticsearch data tiers: hot, warm, cold, and frozen — Elastic Docs](https://www.elastic.co/docs/manage-data/lifecycle/data-tiers) — Elasticsearch ILM 공식
- [Introducing Cold Storage for Amazon OpenSearch Service — AWS Big Data Blog](https://aws.amazon.com/blogs/big-data/introducing-cold-storage-for-amazon-opensearch-service/) — OpenSearch 콜드 티어 공식
- [Introducing vector search with UltraWarm in OpenSearch — AWS Big Data Blog](https://aws.amazon.com/blogs/big-data/introducing-vector-search-with-ultrawarm-in-amazon-opensearch-service/) — UltraWarm 벡터 검색
- [What is Storage Tiering and How Does it Differ from Caching? — SystemOverflow](https://www.systemoverflow.com/learn/object-storage/storage-tiering/what-is-storage-tiering-and-how-does-it-differ-from-caching) — 티어링 vs 캐싱 구분
- [Cold vs Hot Storage — QuestDB Glossary](https://questdb.com/glossary/cold-vs-hot-storage/) — 시계열 DB 맥락 정의
- [Data Center Storage Tiers: Hot, Warm, Cold, and Archive — Solved](https://www.solved.scality.com/data-center-storage-tiers/) — 4계층 정의 및 특성 개요
- [Hot vs Cold Data Tiers: Concepts, Tech Choices & Use-Cases — Crow](https://crow.sg/blog/hot-vs-cold-data-tiers/) — 기술 선택 가이드
- [Caching vs Auto-tiering: How Data Storage Acceleration Works — Open-E](https://www.open-e.com/blog/caching-vs-auto-tiering-two-ways-to-accelerate-data-storage/) — 캐싱 vs 티어링 비교

## 관련 노트

- [[aws-s3-vectors-api]] — S3 Vectors가 Cold tier 역할을 하는 벡터 저장소 API
- [[bedrock-kb-s3-vectors-integration]] — Hot OSS + Cold S3 Vectors 패턴을 Bedrock KB에서 구현한 통합 아키텍처
- [[bedrock-knowledge-bases]] — S3 Vectors를 Cold vector store 옵션으로 지원하는 managed RAG 서비스
- [[moc-backend-architecture]] — 본 노트 소속 MOC
