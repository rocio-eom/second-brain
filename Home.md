---
type: index
created: 2026-06-04
modified: 2026-06-06
---

# Home

vault 진입점. 도메인 MOC 또는 lifecycle 폴더에서 시작.

---

## 도메인 MOC

### AI/ML/LLM
- [[moc-rag]] — Retrieval-Augmented Generation
- [[moc-embedding]] — 임베딩 모델·운영
- [[moc-vector-search]] — 벡터 검색·인덱스
- [[moc-chunking]] — 청킹 전략
- [[moc-aws-bedrock]] — AWS Bedrock 생태계
- [[moc-llm-serving]] — LLM 추론·서빙
- [[moc-llm-security]] — LLM 보안

### Backend
- [[moc-backend-architecture]] — 백엔드 아키텍처
- [[moc-saas-api-integration]] — SaaS API 연동

### Frontend
- (TBD)

### CS Fundamentals
- (TBD)

---

## Lifecycle 진입

| 단계     | 폴더                             | 행위                    |
| ------ | ------------------------------ | --------------------- |
| 즉시 캡처  | `00_Inbox/`                    | <30s, 분류 없이           |
| 일일 일지  | `00_Inbox/daily/`              | 빈 파일은 자동 정리           |
| 확장 노트  | `01_Fleeting/`                 | 48h 내 처리              |
| 외부 자료  | `02_Literature/`               | 책/논문/영상/코스 (optional) |
| 핵심 개념  | `03_Permanent/{domain}/{sub}/` | atomic, evergreen 지향  |
| 토픽 인덱스 | `04_MOC/{domain}/`             | 링크만, 콘텐츠 없음           |
| 프로젝트   | `05_Projects/_active/`         | 시한 있는 학습/구현           |
| Area   | `06_Areas/`                    | 지속 관리 영역              |
| 결정 기록  | `07_Decisions/`                | ADR + 회고              |
| 템플릿    | `08_Templates/`                | 노트 타입별                |
| 아카이브   | `09_Archive/`                  | 종료/비활성                |
