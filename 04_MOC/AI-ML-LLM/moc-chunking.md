---
type: moc
created: 2026-06-04
modified: 2026-06-06
domain: ai-ml-llm
tags: [chunking, moc]
related_mocs:
  - "[[moc-rag]]"
  - "[[moc-embedding]]"
---

# MOC: Chunking

청킹 전략 설계·품질 개선 관련 Permanent 노트 인덱스.

---

## 카테고리 개요

- [[context-aware-chunking]] — 의미·구조·문맥 경계 인식 chunking family(structure-aware / semantic / contextual / late chunking)의 카테고리 정의·선택 원칙·하이브리드 스택

## 전략 카탈로그

- [[fixed-size-chunking]] — Fixed-size. 토큰 수 기준 일률 분할. RAG 베이스라인
- [[recursive-chunking]] — Recursive Character. 구분자 hierarchy 기반. 평문 디폴트
- [[semantic-chunking]] — Semantic. 임베딩 코사인 거리로 경계 탐지
- [[]] — Hierarchical (Parent-Child)
- [[]] — Structure-aware
- [[sliding-window-chunking]] — Sliding window / Sentence-window. retrieval·합성 단위 분리
- [[agentic-chunking]] — Agentic. LLM이 문서별 전략·경계 적응 결정
- [[document-structured-chunking]] — Document-structured. 마크다운·HTML·코드의 구조 마커 사용

## 포맷별 Parser

- [[confluence-storage-format-parser]] — Confluence storage format(XHTML + ac:*/ri:* namespace) 파싱 도구 선택 (BS4 / unstructured / markdownify / atlassian-python-api)

## 전처리 / 언어 처리

- [[korean-sentence-boundary-detection]] — 한국어 종결어미·인용 컨텍스트 기반 sentence split: Kss(전용 toolkit)·Kiwi(형태소 기반) 선택 가이드 + 안티패턴

## 평가 / 품질

- [[chunking-strategy-methodology]] — 전략 설계 → 측정 → 개선 6단계 방법론. golden set 기반 P@K·MRR·nDCG 평가, 안티패턴 8선
- [[chunking-failure-modes]] — Mid-context loss·Context starvation·Boundary noise 3대 실패 유형 진단·디버깅·eval-driven 튜닝 루프
- [[]] — Golden set 구축
- [[contextual-retrieval]] — Contextual Retrieval (Anthropic). LLM 생성 50–100 토큰 prefix로 chunk 의미 맥락 보강 + Contextual BM25 결합, retrieval 실패율 최대 67% 감소

## 임베딩 호환성

- [[chunking-embedding-compatibility-matrix]] — 전략 × 임베딩 매트릭스. `max_seq` 잘림 사전 차단, 분포 일치·한국어 트랙·비용 ROI 5개 결정 룰

---

## Cross-Domain Links

- [[moc-rag]] — Ingestion 단계 핵심
- [[moc-embedding]] — 청크 길이 ↔ max_seq 결합
