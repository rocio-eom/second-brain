---
type: moc
created: 2026-06-06
modified: 2026-06-06
domain: backend
tags: [aws, backend, moc]
related_mocs:
  - "[[moc-backend-architecture]]"
  - "[[moc-aws-bedrock]]"
---

# MOC: AWS Backend Services

AWS 기반 백엔드 컴퓨트·메시징·오케스트레이션 서비스 인덱스. AI/ML 전용 컴포넌트는 [[moc-aws-bedrock]] 참조.

---

## Compute

- [[aws-lambda]] — 서버리스 FaaS: Firecracker microVM 격리 실행, 200+ AWS 통합, SnapStart·Provisioned Concurrency·Durable Functions

## Container Orchestration

- [[aws-ecs]] — AWS-opinionated container orchestration: Fargate/EC2/External 4가지 launch type, Service Connect, capacity provider 혼합

## Messaging / Event Routing

- [[aws-eventbridge]] — serverless event bus: JSON pattern routing, Pipes·Scheduler·Schema Registry·API Destinations, SaaS partner 통합

---

## Cross-Domain Links

- [[moc-backend-architecture]] — 백엔드 아키텍처 패턴 일반
- [[moc-aws-bedrock]] — AWS 매니지드 LLM/RAG 컴포넌트
