---
type: moc
created: 2026-06-06
modified: 2026-06-06
domain: ai-ml-llm
tags: [llm-security, prompt-injection, jailbreak, guardrail, moc]
related_mocs:
  - "[[moc-rag]]"
  - "[[moc-aws-bedrock]]"
  - "[[moc-llm-serving]]"
---

# MOC: LLM Security

LLM application의 input guardrail · output filtering · jailbreak·prompt injection 방어 관련 Permanent 노트 인덱스.

---

## Input Guardrail Classifiers

- [[llama-prompt-guard-2]] — Meta open-source classifier (86M·22M, BENIGN/INJECTION/JAILBREAK 3-class). self-hosted 노선, 한국어는 86M multilingual 권장
- [[bedrock-guardrails]] — AWS managed 가드레일, Bedrock 1급 통합. PII redaction·denied topic·content filter 종합

## Indirect Injection (RAG)

- (TBD) — retrieved chunk 검증 패턴 · sandbox prompt
- (TBD) — provenance·source labeling 기반 trust scoring

## Jailbreak 평가·벤치마크

- (TBD) — PINT / HarmBench / JailbreakBench cross-dataset 변동성

---

## Cross-Domain Links

- [[moc-rag]] — RAG pipeline 내 input·indirect-injection 방어 layer로서 가드레일
- [[moc-aws-bedrock]] — Bedrock Guardrails 통합 (managed 노선)
- [[moc-llm-serving]] — 가드레일 추론 latency가 serving budget에 미치는 영향
