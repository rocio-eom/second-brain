---
type: moc
created: 2026-06-06
modified: 2026-06-06
domain: backend
tags: [backend, architecture, moc]
related_mocs:
  - "[[moc-aws-bedrock]]"
  - "[[moc-rag]]"
---

# MOC: Backend Architecture

백엔드 아키텍처 패턴 및 인프라 설계 개념 인덱스.

---

## 스토리지 아키텍처

- [[hot-cold-data-tiering]] — 접근 빈도 기반 스토리지 계층 분리 패턴: 비용·성능 최적화, 승격/강등 정책, 티어링 vs 캐싱·아카이빙 구분

## 성능 / 배포

- [[performance-capacity-deployment]] — 부하 테스트로 처리 한계 측정 → SLO·error budget·canary 정책 연결, 1%→10%→50% ramp 기반 안전 release

---

## Cross-Domain Links

- [[moc-aws-bedrock]] — S3 Vectors Cold tier를 사용하는 Bedrock RAG 아키텍처
- [[moc-rag]] — 벡터 저장소 티어링이 적용되는 RAG 파이프라인
