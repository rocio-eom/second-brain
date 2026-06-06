---
type: permanent
created: 2026-06-06
modified: 2026-06-06
status: draft
domain: ai-ml-llm
moc: "[[moc-llm-security]]"
tags: [llm, security, prompt-injection, jailbreak, guardrail, classifier, llama, meta, open-source]
aliases: [Llama Prompt Guard 2, Prompt Guard 2, llama-prompt-guard]
model_version: prompt-guard-2
promoted_from: fl-2026-06-04-llama-prompt-guard-2
related:
  - "[[bedrock-guardrails]]"
---

# Llama Prompt Guard 2

## 핵심 요약

Llama Prompt Guard 2는 Meta가 공개한 open-source classifier로, prompt injection·jailbreak 입력 탐지를 single-pass classification으로 수행한다. self-hosted 노선의 input guardrail 옵션이며, managed 솔루션([[bedrock-guardrails]])과 ADR 9 옵션 비교의 한 축을 구성한다.

- **모델 사이즈**: 86M / 22M parameter 2종. 22M은 86M 대비 latency·compute cost 최대 75% 절감 (edge·CPU inference 가능)
- **언어 범위**: 86M은 영어 + non-English 공격 패턴 학습. 22M은 영어 중심 — **한국어 RAG라면 86M 채택이 안전**
- **분류 라벨**: BENIGN · INJECTION · JAILBREAK (3-class)
- **학습 방법**: open-source benign 데이터(웹·user prompts) + 합성 injection·jailbreak + red-team 데이터. modified energy-based loss(benign에 큰 음수 energy 페널티)
- **사용 단계**: 사용자 query 입력 직후, retrieve 호출 전 1회 추론. 또는 retrieved chunk 검증(indirect injection 방지)
- **라이선스**: Llama Community License — 상업적 사용 가능, 일부 조건

## 형식적 정의

본 노트가 가정하는 두 공격 카테고리:

- **Prompt injection** — 사용자 query 또는 retrieved context에 모델 의도를 변경하는 지시문이 삽입되어, 시스템 prompt 또는 application 정책을 무력화하는 공격. 직접(direct, 사용자 입력) / 간접(indirect, RAG로 검색된 wiki·웹 페이지 내 삽입) 두 경로.
- **Jailbreak** — 모델의 safety alignment(거절 정책)를 우회해 금지된 출력을 유도하는 공격. role-play, hypothetical framing, encoding 변환 등이 일반적.

Llama Prompt Guard 2는 두 카테고리를 single classifier로 분리 탐지한다 (BENIGN / INJECTION / JAILBREAK 3-class output).

## 유사 기술 비교

| 축 | Llama Prompt Guard 2 | Bedrock Guardrails | 자체 rule-based regex | Hybrid |
| --- | --- | --- | --- | --- |
| 한국어 공격 패턴 인식 | 86M multilingual / 22M 영어 중심 — 정성 검증 필요 | 영어 중심 + multilingual 부분 지원 | 룰 직접 작성 (제어 가능) | 다중 단계 |
| 추론 latency | 수십 ms (22M, CPU) | 수십-수백 ms (managed) | ~ms | 누적 |
| cost | infra 자체 부담 | per-request | 0 | 누적 |
| 운영 부담 | 호스팅·업데이트 | 0 | 룰 유지보수 | 모두 |
| Bedrock 통합 | 없음 (자체 호스팅) | 1급 | — | — |
| RAG indirect injection 대응 | retrieved chunk 검증에도 사용 가능 | 일부 지원 | 룰로 제한 대응 | ○ |

## 한국어 RAG에서의 risk

- **86M vs 22M 선택**: 86M은 영어 + non-English 공격 패턴이 학습 데이터에 포함되어 한국어 detection 기대치가 22M보다 높음. **한국어 RAG는 86M 채택 권장**, 22M은 영어 트래픽이 dominant한 경우에만.
- **여전한 한계**: Meta가 non-English로 학습했다고 명시했더라도 학습 데이터의 한국어 비중·도메인 분포는 비공개. 한국어 prompt injection 패턴(예: "지시 무시하고 ...", "관리자 권한으로 ...", role-play 형식의 "선생님이 학생에게 ...")이 학습 분포 내에 포함됐는지 보장 불가 → false-negative 위험 잔존.
- **사전 평가 절차**: golden set의 prompt injection 변형 20-30건을 한국어로 작성 → 86M·22M·Bedrock Guardrails 각각 detection rate 측정 + false-positive rate 측정 (일반 한국어 query 50건). PINT Benchmark는 영어 전용이므로 한국어 변형은 직접 구성 필요.
- **cross-dataset 변동성 인지**: 가드레일 솔루션은 dataset에 따라 recall이 크게 흔들림(arXiv 2502.15427 분석) → 한국어 자체 golden set이 production decision의 truth source.

## 처리 흐름

1. 사이즈 선택: 한국어/multilingual → 86M, 영어 전용·edge → 22M (latency·compute 75% 절감)
2. HuggingFace 또는 PurpleLlama GitHub에서 모델 다운로드 (Llama Community License 동의 필요)
3. ECS task 또는 sidecar에 모델 호스팅 — CPU inference로도 가능 (22M은 특히 적합)
4. FastAPI middleware에서 query 입력 시 추론 호출 → BENIGN 외 모두 reject (threshold 튜닝 필수)
5. retrieved chunks에 대해서도 동일 호출 (indirect injection 방어)
6. detection metric을 CloudWatch에 push → 임계 초과 시 alert
7. 한국어 golden set으로 정기 재평가 (월 1회 권장), false-positive·false-negative 추세 추적

## 함정 및 안티패턴

- **threshold 너무 낮음**: confidence threshold가 너무 낮으면 정상 query까지 차단(false-positive ↑) → 사용자 경험 ↓. 한국어 query 대상으로 threshold 튜닝 필수.
- **단독 사용**: classifier 1개로는 모든 공격 차단 불가. Bedrock Guardrails 또는 rule-based 추가 layer 권장 (ADR 9 hybrid 옵션).
- **retrieved chunk indirect injection 누락**: 사용자 query만 검사하고 wiki chunk는 신뢰하는 패턴 → indirect injection 노출. retrieval 결과도 동일 classifier로 검사 권장.
- **모델 업데이트 자동화 미흡**: open-source 모델은 보안 패치 push 시 수동 재배포 부담. CI/CD에 모델 버전 핀 + 정기 재평가.

## 갱신 정책

- 본 노트는 **Prompt Guard 2 (2025-04 release)** 기준 사양. 86M·22M 두 사이즈, 3-class label, energy-based loss는 v2 가정.
- Meta가 v3 출시 시 (a) 사이즈 라인업 (b) label 체계 (c) 학습 데이터 범위 (d) 라이선스를 재검증 후 본 노트 갱신.
- frontmatter `model_version: prompt-guard-2` 필드로 staleness 추적.

## 참고 자료

- [Llama Prompt Guard 2 (Meta llama.com docs)](https://www.llama.com/docs/model-cards-and-prompt-formats/prompt-guard/) — 공식 model card 페이지
- [Sharing new open source protection tools (Meta AI blog)](https://ai.meta.com/blog/ai-defenders-program-llama-protection-tools/) — v2 release announcement, AI Defenders Program 맥락
- [PurpleLlama / Llama-Prompt-Guard-2 / 86M MODEL_CARD.md (GitHub)](https://github.com/meta-llama/PurpleLlama/blob/main/Llama-Prompt-Guard-2/86M/MODEL_CARD.md) — 공식 model card raw 문서 (학습 데이터·loss function 상세)
- [meta-llama/Llama-Prompt-Guard-2-86M (HuggingFace)](https://huggingface.co/meta-llama/Llama-Prompt-Guard-2-86M) — 86M model card · multilingual
- [meta-llama/Llama-Prompt-Guard-2-22M (HuggingFace)](https://huggingface.co/meta-llama/Llama-Prompt-Guard-2-22M) — 22M model card · English-focused
- [PINT Benchmark (Lakera)](https://www.lakera.ai/product-updates/lakera-pint-benchmark) — 3,007 영어 prompt injection 테스트셋, false-positive·long-document 케이스 포함. 한국어 RAG 사전 평가 시 변형 baseline으로 사용
- [Adversarial Prompt Evaluation: Systematic Benchmarking of Guardrails (arXiv 2502.15427)](https://arxiv.org/html/2502.15427v1) — 가드레일 벤치마크 cross-dataset 변동성 분석 (한 솔루션이 garak·HarmBench·PromptBench에서 recall 차이가 큼)

## 관련 노트

- [[bedrock-guardrails]] — AWS managed 가드레일 옵션, input guardrail 의사결정의 비교 대상
