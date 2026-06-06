---
type: moc
created: 2026-06-04
modified: 2026-06-06
domain: ai-ml-llm
tags: [aws-bedrock, moc]
related_mocs:
  - "[[moc-rag]]"
  - "[[moc-embedding]]"
  - "[[moc-vector-search]]"
---

# MOC: AWS Bedrock

AWS Bedrock 생태계의 매니지드 LLM/RAG 컴포넌트 인덱스.

---

## Foundation Models

- [[aws-bedrock-overview]] — Amazon Bedrock 플랫폼 개요: 멀티-벤더 Foundation Model 매니지드 서비스, Knowledge Bases·Agents·Guardrails 컴포넌트 지도
- [[bedrock-claude-generation-models]] — Claude 4.X(Sonnet/Haiku/Opus) RAG generation 선택 기준·비용·한국어 주의사항
- [[bedrock-converse-api]] — 모든 messages-기반 FM 통합 추론 API. modelId 교체만으로 모델 전환, Tool Use·ConverseStream·Guardrails 통합
- [[bedrock-model-customization]] — SFT·RFT·Continued Pre-training·Distillation 4가지 매니지드 학습 방법. Provisioned Throughput 배포 필수, 과금 3축(학습+보관+PT)
- [[aws-bedrock-seoul-region-model-support]] — ap-northeast-2 가용 모델·가격 카탈로그(2026-06 snapshot), in-region/Geo/Global CRIS 호출 경로, `kr.*` geo profile 부재 등 한국 잔류 요건 대응

## RAG 컴포넌트

- [[aws-s3-vectors-overview]] — Amazon S3 Vectors 플랫폼 개요: 저비용·매니지드 벡터 스토리지, cold/warm 티어링, 경쟁사 비교, 6개 파생 노트 입구
- [[bedrock-knowledge-bases]] — fully-managed RAG: 4-layer 구조(connector→ingestion→vector store→retrieval API), Retrieve/RetrieveAndGenerate API, GraphRAG·hybrid search 지원
- [[aws-bedrock-embedding]] — Titan V2·Cohere Embed: 서버리스 임베딩 API, IAM 인증, Knowledge Bases 통합
- [[titan-text-embeddings-v2]] — Matryoshka dim 가변(256/512/1024), normalize, v1 대비 5x 저렴, 비영어 sub-optimal 주의
- [[aws-bedrock-flows]] — 노드 기반 비주얼 빌더로 Prompt·KB·Agent·Lambda 체이닝, version·alias·trace 지원. PoC·중간 복잡도 워크플로우 최적
- [[aws-bedrock-agents]] — ReAct 기반 매니지드 에이전트: Action Group·KB·Guardrails 오케스트레이션
- [[aws-s3-vectors-resource-model]] — S3 Vectors 2-tier 리소스 계층(bucket→index), s3vectors 네임스페이스 분리, BPA 강제, IAM·CFN 별도 체계
- [[aws-s3-vectors-api]] — S3 Vectors 데이터 플레인 5개 API(PutVectors/QueryVectors 등): 배치 한도·처리량·float32 강제·인덱스 샤딩 전략
- [[aws-s3-vectors-metadata-schema]] — filterable(≤2 KB, 필터 표현식) vs non-filterable(≤10개 불변 키, 큰 payload) 분류 모델 + 실전 표준 키 스키마
- [[aws-s3-vectors-schema-design-strategy]] — 인덱스 명명·메타데이터 배치·멀티테넌트 패턴·CreateIndex/PutVectors/QueryVectors JSON 샘플: 운영 설계 결정 가이드
- [[bedrock-kb-s3-vectors-integration]] — KB + S3 Vectors 통합 패턴: 자동 프로비저닝·ingestion job·IAM 롤·metadata 사이드카·비용 비교
- [[aws-s3-vectors-cold-tier-rag]] — cold-tier only RAG 아카이브 패턴: idle 95%+ 코퍼스에 운영 인스턴스 0으로 시맨틱 검색 부여, 비용 ≈ Storage만
- [[aws-s3-vectors-hot-cold-tiering]] — Hot OSS + Cold S3 Vectors 티어드 패턴: Export·Engine 2개 통합 모드, promotion/demotion 트리거, 70~90% 비용 절감

## 안전·운영

- [[bedrock-guardrails]] — 6대 안전 정책(Content Filter·PII·Contextual Grounding 등) 양방향 필터. ApplyGuardrail로 비-Bedrock 모델에도 적용 가능
- [[aws-s3-vectors-cfn-privatelink-tagging]] — S3 Vectors GA-time enterprise readiness: CloudFormation 리소스 타입 + Interface VPC Endpoint + tag-on-create. IaC·사설 네트워크·비용 할당 통합
- [[aws-s3-vectors-bucket-governance]] — vector bucket 거버넌스: naming/quota/immutability 규칙 + `s3vectors:*` IAM action + resource-based policy union 평가 + BPA 강제
- [[aws-s3-vectors-cost-model]] — S3 Vectors 비용 3축(PUT/Storage/Query): query data-processed 차원이 비직관, 128KB 최소 PUT, idle 0 — cold·저빈도에 강점
- [[aws-s3-vectors-capacity-planning]] — 인덱스 용량 계획 및 샤딩 전략: 처리량·격리·스키마 불변성 5축 기반 분할 결정, multi-index 쿼리 패턴
- [[aws-s3-vectors-encryption]] — 인덱스별 SSE-KMS + CMK override: 멀티테넌트 암호화 격리·BYOK·crypto-shredding·kill switch 패턴

---

## Cross-Domain Links

- [[moc-rag]] — Bedrock KB가 풀어내는 RAG 단계
- [[moc-vector-search]] — S3 Vectors / OpenSearch 통합
