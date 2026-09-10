---
type: decision
category: architecture
created: 2026-09-10
modified: 2026-09-10
status: accepted
tags: [naming-convention, monorepo, taxonomy, code-navigation, agent, linter, adr, platform]
aliases: [코드 층 네이밍 개정, code layer naming taxonomy, type taxonomy extension, 이름을 에이전트 인덱스로]
context: "dec-2026-07-31 의 type taxonomy 를 모노레포 내부까지 확장하고, 강제를 Golden Path 에서 린터로 옮긴다"
options_considered: [현행 유지, taxonomy 확장 + 린터 강제, 새 규약 신설, PDP 정책으로 강제, 크로스리포 CI 강제]
decision: "D1 의 type 을 packages/ 내부까지 확장하고, npm 스코프를 @rosmare 로 개정하며, 강제를 platform-data-model 의 린터로 옮긴다"
related_permanent: []
related_project: []
retrospective_date: 2026-10-08
related:
  - "[[dec-2026-09-10-vcs-layer-naming-and-release-tagging]]"
  - "[[dec-2026-07-31-git-repo-topology-naming-convention]]"
  - "[[dec-2026-09-10-data-catalog-ownership-and-staged-rollout]]"
  - "[[fl-2026-07-31-git-repo-naming-convention]]"
  - "[[fl-2026-07-31-git-repo-convention-repo-structure]]"
---

# ADR: 코드 층 네이밍 — taxonomy 확장과 강제의 이전

## 상태

Accepted (2026-09-10). [[dec-2026-07-31-git-repo-topology-naming-convention]]의 **개정**이며 폐기가 아니다. 회고 예정: 2026-10-08.

플랫폼 저장소 쪽 대응 ADR: `platform-data-model` ADR 0012.

**형식에 대한 메모.** [[dec-2026-09-10-data-catalog-ownership-and-staged-rollout]]이 오픈 이슈로 *"다음 결정 노트를 쓰기 전에 정본 형식을 확정할 것"* 을 남겼고 이 노트가 그 "다음 노트"다. 형식 확정은 별도 결정이므로 여기서 하지 않는다. **개정 대상인 `dec-2026-07-31` 과 같은 ADR 형식을 쓰고 frontmatter 는 `tpl-decision.md` 의 키를 채웠다** — 개정문이 원본과 다른 골격이면 두 문서를 나란히 읽을 수 없다.

## 배경 (Context)

[[dec-2026-07-31-git-repo-topology-naming-convention]]의 회고(같은 날 작성)가 네 가지 어긋남을 실측했다. 셋은 강제의 부재이고 하나는 결정 내용의 부족이다.

동기는 문서 정돈이 아니다. **코딩 에이전트는 glob·grep·심볼 검색으로 코드베이스를 항해하므로 이름 패턴이 곧 에이전트가 쓸 수 있는 인덱스다.** 인덱스가 규칙적이지 않으면 에이전트는 트리를 전수 탐색하거나, 못 찾고 새로 만든다. `lint-naming.mjs` 동명이물이 후자의 실물이다.

여기에 원본 결정이 예상하지 못한 요구가 하나 더 있다 — **에이전트가 쓴 것을 다음 세션의 에이전트가 다시 찾아야 한다**(navigation paradox). 읽기 최적화와 쓰기 규범이 같은 규칙이 아니면 세션이 바뀔 때마다 이름이 표류한다. 원본은 사람이 읽는 것만 고려했다.

## 결정 (Decision)

### D1. type taxonomy 를 `packages/` 내부까지 확장한다

원본의 taxonomy(`api`·`web`·`app`·`worker`·`service`·`cli`·`lib`)는 **배포 아티팩트 축**이라 모노레포 안을 `lib` 하나로 뭉친다. `lib` 을 쪼개고 `mcp` 를 더한다.

| 원본 type | 개정 | 순수성 | 배치 |
|---|---|---|---|
| `api`·`web`·`worker`·`cli` | 그대로 | app | `apps/` |
| `app`·`service` | 그대로 (현재 사용처 0) | app | `apps/` |
| — | **`mcp` 신설** — MCP 서버 진입점 | app | `apps/` |
| `lib` → | `core` 도메인 타입·불변식 | pure | `packages/` |
| | `engine` 순수 판정·계산 | pure | `packages/` |
| | `model` 스키마·타입 선언만 | pure | `packages/` |
| | `gen` 정본 → 파생물 생성기 | pure | `packages/` |
| | `lint` 규칙 검사기 | pure | `packages/` |
| | `registry` 주입받은 목록의 인메모리 색인 | pure | `packages/` |
| | `planner`·`executor` 계획 수립·실행 조율 | pure | `packages/` |
| | `telemetry` 관측 신호 발행 | pure | `packages/` |
| | `adapter`·`client`·`http` 외부 시스템 결합 | impure | `packages/` |
| | `agent` LLM 호출 경계 | impure | `packages/` |

**순수성 열은 발명이 아니라 실측이다.** `platform-orchestration/AGENTS.md` §2 가 손으로 유지하는 결정성 표와 항목별로 일치한다 — `agent`·`adapters` 는 비결정 허용, `core`·`capability-registry`·`planner`·`executor`·`approval`·`telemetry` 는 금지. **그 손으로 쓴 표가 이름에서 유도될 수 있게 되는 것**이 이 확장의 값이다.

`mcp` 를 신설하는 이유는 원본 taxonomy 가 작성된 시점에 이 워크스페이스에 MCP 서버가 없었기 때문이다. 지금은 둘이다.

### D2. 순수성은 이름이 선언하고, 검사는 각 저장소가 한다

접미사는 순수성을 **주장**한다. 코드가 그 주장과 맞는지는 **이미 있는 검사들이 본다** — `platform-engineering` E11, `platform-orchestration` AGENTS.md §2, `platform-governance` ADR 0002.

**네이밍 린터가 I/O 스캔을 다시 구현하지 않는다.** `platform-governance/standards/naming-conventions.md` 의 원칙 그대로다 — *"나머지를 여기서 다시 검사하면 두 곳이 어긋났을 때 어느 쪽이 정본인지 알 수 없게 된다."*

린터가 검사하는 것은 **배치**다: app type 은 `apps/`, 나머지는 `packages/`. 이것은 아래 두 순수성 모델 어느 쪽에서도 참이라 워크스페이스 전체에 하드 에러로 걸 수 있다.

**그리고 "`packages/` 는 순수하다"를 워크스페이스 규칙으로 적지 않는다.** 실측하면 두 모델이 공존한다:

| 저장소 | 모델 |
|---|---|
| engineering · governance · data-model | `packages/` **전체** 순수 |
| orchestration | **패키지별** — `agent`·`adapters/*` 만 비결정 허용 |

규약이 이 차이를 덮으면 orchestration 의 실제 규칙과 충돌하고, 그때 무시되는 것은 규약 쪽이다.

### D3. npm 스코프를 `@rosmare` 로 개정한다

원본 D2 의 `@rocio-ds/*` 는 실재하지 않는다. 조직이 `rosmare` 이고 이미 그 스코프로 GitHub Packages 에 발행 중이다.

```
@rosmare/{repoToken}-{dirName}
```

디렉토리에 저장소 토큰을 반복하지 않는다(`packages/orchestration-planner` 는 말더듬이다). 변환이 기계적이라 `@rosmare/orchestration-planner` ↔ `platform-orchestration/packages/planner` 로 양방향이다.

**`@platform` 을 기각한다.** GitHub Packages 는 스코프가 저장소 소유자와 같기를 요구하므로 이 레지스트리에 발행될 수 없는 이름이다. 지금 8개가 전부 `private: true` 라 동작하지만, `private` 를 푸는 날 import 가 이미 퍼져 있다. **원본이 `{key}` 를 스코프로 실현하기로 한 판단 자체는 유지한다** — 바뀐 것은 key 의 값이 아니라 그것이 조직 이름이어야 한다는 제약이다.

**드러난 선행 조건.** 플랫폼 저장소 5개의 remote 는 `github.com/rocio-eom/*` 이고 design-system 만 `github.com/rosmare/*` 다. **`rocio-eom` 소유 저장소에서는 `@rosmare/*` 를 발행할 수 없다.** 지금 발행 대상이 0개라 아무것도 깨지지 않지만, 첫 발행 시도는 `403` 으로 실패하고 원인은 코드 어디에도 없다. **린터가 경고로 기록한다. 조직 이전은 이 결정의 범위가 아니다.**

이미 발행된 `@rosmare/{react,tokens,brand}` 는 토큰 접두가 없지만 **동결한다** — `tech-blog` 가 설치 중이라 개명은 축소 변경이다.

### D4. 강제를 Golden Path 에서 린터로 옮긴다

원본 D4 는 강제 수단을 **레포 생성 템플릿** 하나로 두었다. 그것이 못 덮은 곳이 세 어긋남 전부의 원인이다 — 템플릿은 *새 저장소*를 다루고 *기존 저장소 안의 새 파일*은 다루지 않는다.

정본과 린터의 자리는 `platform-data-model/conventions/` 다. 그 층이 이미 데이터 층 이름을 같은 방식으로 규율하고 있고 린터 엔진이 존재한다.

**경계는 `platform-infrastructure` ADR 0007 이 이미 그어 뒀다:**

> *"두 축은 다루는 것이 다르다 — 그쪽은 자유 문자열의 형태를 규율하고 여기는 닫힌 토큰 집합을 조립한다."*

코드 층 이름은 **형태 규율**이므로 `conventions/` 가 맞는 자리다. 클라우드 리소스 이름은 손대지 않는다.

**CI 로 강제할 수 있는 범위에 제약이 있다.** [[dec-2026-09-10-data-catalog-ownership-and-staged-rollout]]이 못박은 대로 `PLATFORM_REPO_TOKEN` 이 없어 크로스리포 검증이 CI 에서 실패한다. 그래서 린터는 두 모드로 나뉜다:

| 모드 | 무엇 | 어디서 |
|---|---|---|
| `--repo` | 케이스·약어·type·배치·배포명·정의 파일·파일명·심볼 | **CI (기본)** |
| `--workspace` | 저장소 간 중복 이름·파일명 충돌·토큰 등재 | **로컬 전용.** PAT 이 생기면 CI 로 승격 |

승격 전까지 크로스리포 항목은 `conventions/REVIEW-CHECKLIST.md` 가 사람에게 맡긴다. **강제 불가 목록을 명시적으로 적는 것이 이 결정의 일부다** — 적지 않으면 "린터가 통과했으니 이름이 옳다"가 되고, 그것이 이 플랫폼이 이미 겪은 실패다.

### D5. 규약이 실제로 탐색을 돕는지 기계로 측정한다

원본에는 없던 항목이다. 규약이 에이전트 인덱스라면 **그 주장은 검증 가능해야 한다.**

LLM 을 돌려 재지 않는다 — 비결정적이고 네트워크가 필요해 CI 게이트가 될 수 없다. 대신 **규약이 예측한 glob 이 실제로 맞히는가**를 잰다. 질문·예측 glob·정답을 픽스처로 두고 적중률·정밀도·패턴 수 셋을 본다.

**진짜 값은 일회성 측정이 아니라 회귀 방지다.** 규약을 고쳤는데 glob 이 더 이상 맞히지 않으면 CI 가 실패한다. 읽기 최적화와 쓰기 규범이 갈리는 것을 기계가 잡는 지점이다.

기준선은 개명 **전에** 찍는다. 개명 후에만 재면 개선을 증명할 수 없고, 그러면 다음 개정이 근거 없는 취향 조정이 된다.

## 고려한 대안 (Considered options)

| 대안 | 기각 사유 |
|---|---|
| 현행 유지 | 회고가 네 어긋남을 실측했다. `lint-naming.mjs` 동명이물은 이미 발생한 비용이다 |
| **새 규약을 별도로 신설** | taxonomy 가 두 개 생긴다. 원본 D1 은 유효하므로 확장이 맞다 |
| PDP 정책(`POL-NNNN`)으로 강제 | 평가 입력이 `{service, environment, action, actor}` 뿐이라 이름을 판정할 수 없다. `policies/README.md` — *"평가되지 않는 정책 파일을 두는 것이 비워 두는 것보다 나쁘다."* `policies/naming/` 은 계속 비워 둔다 |
| 크로스리포 검사를 CI 에서 강제 | `PLATFORM_REPO_TOKEN` 이 없다. 새 불변식을 그 블로커에 인질로 잡지 않는다 |
| 층별 이름 변환표를 `conventions/` 에 신설 | 온톨로지 `identity.aliases[].axis` 가 이미 그 기능이고 매핑 행은 governance registry 에 있다. 새 표는 두 정본을 만든다 |
| 패키지 디렉토리에도 토큰을 넣는다 | `packages/orchestration-planner` 는 저장소 이름과 중복이다. 대신 D5 의 측정이 이 판단을 나중에 **수치로** 뒤집을 수 있게 해 둔다 |
| 개명을 미루고 신규만 강제 | 발행된 패키지가 0건인 지금이 가장 싸다. 첫 발행 이후에는 축소 변경 절차 대상이 된다 |

## 결과 (Consequences)

**좋은 점**

- orchestration 이 손으로 유지하던 결정성 표가 이름에서 유도된다. 그 저장소가 *"가장 지키기 어렵고 가장 비싼 규칙"* 이라 적은 것의 유지 비용이 내려간다
- 강제가 템플릿에서 린터로 옮겨져 **기존 저장소 안의 새 파일**도 덮인다. 세 어긋남의 공통 원인이 닫힌다
- `@platform` 이 발행 불가라는 사실이 첫 발행 시도가 아니라 지금 드러난다
- D5 로 규약이 자기 효과를 주장이 아니라 수치로 말한다

**나쁜 점 / 비용**

- **디렉토리 개명 10건 + package.json `name` 8건.** 저장소 4개에 걸치고 orchestration 은 `tsconfig.json` 의 `paths` 10줄이 동반된다
- 개명 기간 동안 기존 이름이 위반 상태다. **만료 있는 유예로 관리하지 않으면 저장소들이 동시에 빨개지고 규칙이 우회된다**
- 크로스리포 검사가 당분간 사람 몫이다. **PAT 이 생기면 체크리스트에서 그 행을 지워야 하는데, 지우지 않으면 사람이 계속 기계 일을 한다**
- ADR 이 또 하나 늘었다. `dec-2026-07-31` 을 읽는 사람이 여기까지 와야 완전한 상태를 안다

**제약**

- `platform-infrastructure` 는 자기 이름 체계를 ADR 0007 로 이미 확정·구현했다. 이 결정은 그 층에 **관여하지 않는다**
- `design-system` 은 `allowedActions: []` 로 에이전트 쓰기가 막혀 있어 개명 대상이 아니다

## 트레이드오프

- **taxonomy 표현력 vs 학습 비용** — type 이 7개에서 18개로 늘었다. 대신 순수성이 이름에서 읽힌다
- **디렉토리 간결성 vs glob 특정성** — 디렉토리에 토큰을 넣지 않으므로 `*/packages/payments-*` 로 한 저장소의 패키지를 찾을 수 없다. 저장소로 한 번 좁혀야 한다. **D5 가 이 비용을 측정한다**
- **지금의 개명 비용 vs 나중의 축소 변경** — 발행 0건인 지금이 가장 싸다

## 후속 (Follow-ups)

- [ ] `PLATFORM_REPO_TOKEN` 이 생기면 `--workspace` 검사를 CI 로 승격하고 `REVIEW-CHECKLIST.md` 에서 해당 행을 지운다
- [ ] 플랫폼 저장소의 `rosmare` 조직 이전 — 발행의 선행 조건. **별도 결정이다**
- [ ] `platform-infrastructure/scripts/lint-naming.mjs` → `lint-resource-naming.mjs` 제안. 근거만 제시하고 그 저장소의 판단에 맡긴다
- [ ] **회고 일정에 검사를 건다.** `/vault-audit` 에 `retrospective_date` 경과 항목을 더한다 — 이번에 20일 밀린 것이 이 항목의 존재 이유다
- [ ] 결정 노트 정본 형식 확정 (이전 노트의 오픈 이슈, 계속 미결)
- [ ] 회고(2026-10-08): 개명이 끝났는지, D5 측정이 개선을 보였는지, PAT 이 생겼는지

## 관련 자료

- 개정 대상: [[dec-2026-07-31-git-repo-topology-naming-convention]]
- 축 예산·트리거·PAT 블로커: [[dec-2026-09-10-data-catalog-ownership-and-staged-rollout]]
- 네이밍 스킴 상세: [[fl-2026-07-31-git-repo-naming-convention]]
- 저장소 구조·필수 파일: [[fl-2026-07-31-git-repo-convention-repo-structure]]
- 저장소 ADR: `platform-data-model/docs/adr/0012-code-layer-naming-taxonomy.md`
- 경계의 출처: `platform-infrastructure/docs/adr/0007-resource-naming-as-parsable-field-sequence.md`
