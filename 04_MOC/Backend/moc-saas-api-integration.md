---
type: moc
created: 2026-06-04
modified: 2026-06-06
domain: backend
tags: [api, integration, saas, moc]
related_mocs:
  - "[[moc-rag]]"
---

# MOC: SaaS API Integration

외부 SaaS Cloud REST API의 인증·페이지네이션·rate limit·통합 패턴을 다루는 Permanent 노트 인덱스. Confluence, Jira, Notion, GitHub 등 위키·이슈·코드·문서 호스팅 API를 자동화·동기화하는 시나리오를 정리한다.

---

## Wiki / Docs APIs

- [[confluence-REST-API-v2]] — Atlassian Confluence Cloud의 차세대 REST API. content type별 endpoint 분리·cursor pagination·points 기반 rate limit이 특징
- [[confluence-storage-format-parsing]] — Confluence page body의 storage(XHTML) vs ADF(JSON) 표현 선택·정규화. RAG 인덱싱 전처리 관점

## Issue Tracker APIs

- [[]] —

## Code Hosting APIs

- [[]] —

## Authentication Patterns

- [[]] —

## Anti-patterns

- [[]] —

---

## Cross-Domain Links

- [[moc-rag]] — SaaS API가 RAG 파이프라인의 데이터 소스(Confluence·Notion 등)로 활용되는 상위 토픽
