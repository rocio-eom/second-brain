---
type: decision
category: architecture
created: 2026-07-31
modified: 2026-09-10
status: accepted
tags: [git, repository, naming-convention, monorepo, polyrepo, ci-cd, adr]
aliases: [Git Repo Topology and Naming ADR, repo convention decision]
retrospective_date: 2026-08-21
related:
  - "[[dec-2026-09-10-code-layer-naming-taxonomy-and-enforcement]]"
  - "[[fl-2026-07-31-git-repo-naming-convention]]"
  - "[[fl-2026-07-31-git-repo-convention-repo-structure]]"
  - "[[fl-2026-07-31-git-repo-convention-branch-strategy]]"
  - "[[fl-2026-07-31-git-repo-convention-commit-message]]"
---

# ADR: Git 레포 topology·네이밍·CI 트리거 컨벤션

## 상태

Accepted (2026-07-31). 회고 예정: 2026-08-21 (약 3주 후, 실제 신규 레포 1건 적용 후 점검).

## 배경 (Context)

개인 포트폴리오·사내/개인 프로젝트·디자인 시스템(`core-design-system`)을 GitHub에서 운영 중이며, **"현업 대기업처럼" 지속 가능한 레포 규칙**을 세우고자 한다. 핵심 질문 3가지:

1. 레포 **네이밍**을 어떻게 표준화할 것인가.
2. **topology**(모노레포 / 도메인 레포 / 폴리레포)를 무엇을 기준으로 고를 것인가.
3. **CI/CD 트리거**를 어떻게 잡아 배포가 꼬이지 않게 할 것인가.

대기업 분석 결과: Google·Meta·Uber(모노레포·경로 기반), Netflix·Amazon(폴리레포·서비스=레포), Spotify Backstage(카탈로그 메타데이터). 공통적으로 **네이밍은 prefix+component+type**, 규모가 커지면 **카탈로그 메타데이터 + Golden Path 템플릿**으로 강제한다.

## 결정 (Decision)

### D1. 네이밍 스킴

`{key}-{descriptor}-{surface?}-{type}` — 업계 표준(prefix + component + type)과 정합.

- `{key}`: 프로젝트/제품 KEY 소문자 (예: `paymt`·`umem`·`dsgn`).
- `{descriptor}`: 도메인/기능, **가능한 서술적으로** (component). key가 project-level인 한 **중복 아님 = 필수**.
- `{surface?}`: audience — **분리 배포 시에만** (`admin`·`console`). type과 직교.
- `{type}`: `api`·`web`·`app`·`worker`·`service`·`cli`·`lib` (`ui`는 `web`/`app`으로 폐기).
- **표기**: 소문자 kebab-case, 영숫자+하이픈, ≤50자. **버전·환경 이름 금지**(태그/배포 설정으로).

### D2. Topology — bounded-context마다 설계 단계(RFC)에서 결정

전역 강제 없음. **결합도·팀·릴리스 주기**로 판단:

| 조건 | topology | 예 |
|---|---|---|
| 공유 라이브러리 다수 | **모노레포** | `core-design-system` (현행 유지) |
| front+back 강결합·한 팀·같은 PR로 변경 | **도메인 레포(meso)** | `paymt-checkout` + `api/`·`web/` 하위 |
| 독립 라이프사이클·다른 팀 | **폴리레포** | `paymt-checkout-api` / `-web` 분리 |

- **모노레포 규칙**: 레포명은 **서술적**(`core-design-system`), `{key}`는 **npm 스코프**(`@rocio-ds/*`), `{type}`은 **`packages/`·`apps/` 경로**로 실현. 레포명을 `dsgn`으로 축약하지 않음(암호적·`dsgn-design-system`은 의미 중복).
- **도메인 레포 규칙**: 레포명 = `{key}-{descriptor}`, **배포 아티팩트명**이 `{key}-{descriptor}-{type}`(예: 레포 `paymt-checkout` → 배포 `paymt-checkout-api`·`paymt-checkout-web`). 풀 네임은 "레포"가 아니라 "배포 단위"에서 실현.

### D3. CI/CD 트리거 — git 이력 기반 경로 트리거

- **coarse**: GitHub Actions `on.push.paths`(또는 `dorny/paths-filter`) — 바뀐 하위 디렉터리만 배포.
- **fine**: 공유 코드 존재 시 **Turborepo `--filter` / Nx `affected`**(git diff 기반 의존 그래프)로 정확도 확보.
- **필수 규칙**: 경로 필터에 **공유 코드(`shared/**`)와 워크플로 파일 자신**을 포함(사일런트 배포 누락 방지).
- **AWS 배포**: 서울 리전(`ap-northeast-2`) 기준. AWS 공식 패턴(`aws-samples/monorepo-multi-pipeline-trigger`, CodePipeline file-path 필터) 참조.

### D4. 강제 메커니즘 — 사람 규율이 아니라 도구

- **레포 템플릿(Golden Path)**: 새 레포는 구조 + `.github/workflows`(경로 필터) + `CODEOWNERS`를 박은 템플릿에서 생성.
- **필수 파일**: `README`·`.gitignore`·`LICENSE`는 루트, `CODEOWNERS`·`CONTRIBUTING`·워크플로는 `.github/`.
- **커밋/브랜치**: Conventional Commits + commitlint, GitHub Flow(소규모 기본). 상세는 related 노트.
- **규모 확장 시**: 이름에 다 넣지 말고 **서비스 카탈로그 메타데이터**(Backstage Domain→System→Component)로 이관.

## 고려한 대안 (Considered options)

1. **전역 모노레포 (Google 모델)**: 강력하나 Bazel급 툴링·규모 필요. 현 규모엔 과함 → 디자인 시스템 등 공유 자산에 국한.
2. **전역 폴리레포 (서비스=레포)**: 단순 배포. 그러나 강결합 front+back까지 쪼개면 원자적 변경 상실·레포 폭증 → 강결합엔 부적합.
3. **도메인 레포(meso) 기본 + 케이스별 선택 (채택)**: 결합도에 맞춰 topology 선택, 메커니즘(네이밍·경로 트리거·템플릿)만 표준화. 대기업의 실제 운영 방식과 정합.

## 결과 (Consequences)

**좋은 점**
- 레포 역할이 열지 않고 드러남(네이밍), 관련 레포 접두 그룹핑.
- 강결합 도메인은 한 PR로 원자적 변경, 경로 트리거로 배포 5분화(전체 45분 대비 70~80% 절감).
- 템플릿으로 컨벤션이 시간에 붕괴하지 않음.

**나쁜 점 / 비용**
- 도메인 레포는 경로 필터·경로 스코프 릴리스(Changesets/태그 접두) 설정 필요.
- topology를 케이스별로 판단해야 하므로 RFC 단계 부담(대신 유연성).
- 카탈로그(Backstage)는 규모가 커진 뒤 도입 — 지금은 미도입.

## 트레이드오프

- 표현력(4 세그먼트) vs 간결성(≤50자).
- 도메인 레포 명시성 vs 모노레포 내부 패키지 위임.
- topology 유연성(케이스별) vs 규칙 단순성(전역 강제).

## 후속 (Follow-ups)

- [ ] 레포 템플릿(Golden Path) 1개 제작 — 도메인 레포 패턴 + 경로 필터 워크플로.
- [ ] 다음 실제 신규 프로젝트에 D2 판정 규칙 적용해 topology 확정(RFC).
- [ ] `core-design-system` 리네임(→`dsgn`) 제안은 **기각** — 모노레포 레포명은 서술적 유지, key는 `@rocio-ds` 스코프로 이미 실현.
- [ ] 회고(2026-08-21): 첫 적용 후 네이밍·트리거가 실제로 마찰 없이 작동했는지 점검.

## Retrospective (2026-09-10)

예정일은 2026-08-21 이었다. **20일 늦었다.** 늦은 것 자체가 첫 번째 관측이다 — 회고 일정을 산문에만 두면 발화하지 않는다. 이것은 D4 가 네이밍에 대해 말한 것("사람 규율이 아니라 도구")이 **회고 일정 자신에게는 적용되지 않았다**는 뜻이다.

점검 대상은 follow-up 에 적어 둔 것 — *"첫 적용 후 네이밍·트리거가 실제로 마찰 없이 작동했는지"*. 워크스페이스 7개 저장소를 실측했다.

### 무엇이 작동했나

- **D4 의 Golden Path 는 실현됐다.** `platform-engineering/templates/` 에 `service-node`·`astro-static`·`astro-rosmare` 세 템플릿이 있고, CI 가 렌더 결과를 실제로 빌드한다. "컨벤션이 시간에 붕괴하지 않음"의 수단이 존재한다
- **D2 의 topology 판정은 적용됐다.** `tech-blog` 온보딩이 실제로 그 경로를 탔다
- **D1 의 표기 규칙(소문자 kebab·≤50자)은 위반 0건이다.** 저장소·디렉토리·파일 전부

### 무엇이 어긋났나 — 넷

**1. D2 의 `packages/{descriptor}-{type}` 이 적용되지 않았다.**

소유한 저장소 4개의 패키지·앱 23개 중 **9개가 type 접미를 갖지 않는다** (61% 준수).

| 준수 | `model-core` · `pdp-http` · `policy-engine` · `policy-model` · `core` · `planner` · `executor` · `telemetry` · `agent` · `capability-registry` · `apps/{api,cli,worker,mcp}` |
|---|---|
| **미준수** | `generators` · `naming` · `adapters` · `approval` · `packaging` · `scaffold` · `apps/pdp` · `apps/mcp-server` · `apps/scaffold` |

D2 는 *"`{type}` 은 `packages/`·`apps/` 경로로 실현"* 이라고 적었고 관련 노트는 더 구체적으로 *"`{descriptor}-{type}` 이 모노레포에선 `packages/{descriptor}-{type}` 경로로 내려간다"* 고 적었다. **결정은 있었고 적용이 없었다.** 강제 수단이 없었기 때문이다.

**2. npm 스코프가 결정문과 다르다.**

D2 는 `{key}` 를 npm 스코프로 실현한다며 `@rocio-ds/*` 를 적었다. 실제는:

| 실제 | 어디 |
|---|---|
| `@rosmare/*` | design-system 3개 패키지. **GitHub Packages 에 실제 발행 중** |
| `@platform/*` | orchestration 8개 패키지. 전부 `private: true`, 미발행 |
| 이름 없음 | data-model · governance · engineering 의 패키지 전부 |

`@rocio-ds` 는 어디에도 없다. **그리고 `@platform` 은 발행될 수 없는 이름이다** — GitHub Packages 는 스코프가 저장소 소유자와 같기를 요구하고, 일반명사 스코프는 우리가 소유했다는 증거가 없다. 지금은 워크스페이스 안에서만 해소되어 동작하므로 아무도 모른다. `private` 를 푸는 날 8개를 한꺼번에 개명해야 한다.

**3. 안티패턴 7 이 실제로 발생했다.**

> *"컨벤션을 사람 기억에 의존 → 시간 지나며 붕괴 → 레포 생성 템플릿(Golden Path)으로 이름·메타데이터 자동 강제"*

`lint-naming.mjs` 라는 같은 파일명이 두 저장소에 **서로 다른 뜻으로** 존재한다 — `platform-infrastructure` 것은 클라우드 리소스 이름을, `platform-data-model` 것은 데이터 층 이름을 검사한다. 둘 다 나중에 독립적으로 만들어졌고, 만든 쪽 누구도 상대를 몰랐다.

**Golden Path 가 이것을 막지 못한 이유는 템플릿이 *새 저장소*를 다루고 *기존 저장소 안의 새 파일*은 다루지 않기 때문이다.** D4 의 강제 메커니즘에 빈칸이 있었다.

**4. type taxonomy 가 모노레포 내부에는 부족하다.**

D1 의 `lib` 하나가 `packages/` 전체를 덮는데, 실제 `packages/` 안에는 순수 도메인 코어·순수 판정 엔진·외부 시스템 어댑터·LLM 호출 경계가 섞여 있다. **`core` 와 `adapter` 가 같은 `lib` 이면 순수/비순수를 이름으로 구분할 수 없다.**

이것이 실제 비용을 낳고 있다 — `platform-orchestration/AGENTS.md` §2 가 어느 패키지가 비결정을 허용하는지를 **손으로 쓴 표**로 유지하고 있고, 그 저장소는 그것을 *"가장 지키기 어렵고 가장 비싼 규칙"* 이라고 적었다. 이름이 그 표를 대신할 수 있었다.

### 부수 관측 — 저장소 이름이 조용히 바뀌었다

D2 의 follow-up 은 *"`core-design-system` 리네임(→`dsgn`) 제안은 기각 — 모노레포 레포명은 서술적 유지"* 였다. 그런데 실제 저장소는 `rosmare/design-system` 이다. **`core-` 접두가 사라졌다.** 기각한 것은 `dsgn` 으로의 축약이었으므로 결정 위반은 아니지만, **결정문이 가리키는 이름이 더 이상 존재하지 않는다.** 이름을 바꾸면서 그것을 근거로 삼은 결정문을 갱신하지 않은 것이 여기서도 반복됐다.

### 판정

**D1·D2 의 결정 내용은 유효하다. 실패한 것은 강제다.**

네 어긋남 중 셋(1·2·3)이 같은 원인을 갖는다 — **규칙은 적혔고 검사는 없었다.** D4 가 그 위험을 정확히 예측했는데("사람 규율이 아니라 도구"), Golden Path 하나로는 이미 존재하는 저장소 안을 덮지 못했다.

넷째(taxonomy 부족)만이 결정 내용 자체의 개정 사유다.

따라서 이 결정을 **폐기하지 않고 개정한다** — taxonomy 를 확장하고, 스코프를 실제에 맞추고, 강제를 린터로 옮긴다. → [[dec-2026-09-10-code-layer-naming-taxonomy-and-enforcement]]

### 다음 회고에 대한 교훈

회고 일정도 검사를 가져야 한다. 개정문에 `retrospective_date` 를 넣되, **그것이 지났는지 보는 것은 `/vault-audit` 의 일**이어야 한다. 지금은 사람이 날짜를 기억해야 하고, 이번에 20일 실패했다.

## 관련 자료

- 네이밍 상세: [[fl-2026-07-31-git-repo-naming-convention]]
- 레포 구조·필수 파일: [[fl-2026-07-31-git-repo-convention-repo-structure]]
- 브랜치 전략: [[fl-2026-07-31-git-repo-convention-branch-strategy]]
- 커밋 컨벤션: [[fl-2026-07-31-git-repo-convention-commit-message]]
- 외부: AWS DevOps Blog(GitHub monorepo + CodePipeline project-specific CI/CD), `aws-samples/monorepo-multi-pipeline-trigger`, Backstage Software Catalog descriptor format.
