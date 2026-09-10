---
type: decision
category: architecture
created: 2026-09-10
modified: 2026-09-10
status: accepted
tags: [naming-convention, harness, skill, subagent, slash-command, plugin, mcp, workflow, routing, linter, adr, platform]
aliases: [하네스 층 네이밍, harness layer naming, 스킬 이름 규약, 서브에이전트 이름 규약, MCP 툴 이름 예산, 스코프 충돌]
context: "앞의 세 층은 이름이 틀리면 무언가가 실패하지만 하네스 층은 아무것도 실패하지 않는다 — 다른 것이 발동하거나 아무것도 발동하지 않을 뿐이다. 그리고 이 층의 이름만 사람이 타이핑하는 문자열이자 모델의 라우팅 입력이다"
options_considered: [현행 유지, conventions/naming.yaml 4절 추가, platform-orchestration 단독 소유, platform-governance 단독 소유, 맨명사 금지를 에러로 강제, 플러그인 스킬 전수 등재, 전역 즉시 강제]
decision: "형태는 platform-data-model/conventions 가, 실재하는 이름의 목록은 platform-governance/registry/harness 가, 발동 정확도는 platform-orchestration/evals 가 소유하는 3분할"
related_permanent: []
related_project: []
retrospective_date: 2026-10-08
related:
  - "[[dec-2026-09-10-vcs-layer-naming-and-release-tagging]]"
  - "[[dec-2026-09-10-code-layer-naming-taxonomy-and-enforcement]]"
  - "[[dec-2026-09-10-data-catalog-ownership-and-staged-rollout]]"
  - "[[fl-2026-07-26-claude-parallel-skill-harness]]"
---

# ADR: 하네스 층 네이밍 — 타이핑 문자열이자 라우팅 입력

## 상태

Accepted (2026-09-10). [[dec-2026-09-10-vcs-layer-naming-and-release-tagging]]의 **확장**이며 개정이 아니다 — 앞의 세 층이 *일의 결과* 의 이름을 가져왔다면 이것은 **그 일을 하는 도구 자신** 의 이름을 가져온다. 회고 예정: 2026-10-08.

플랫폼 저장소 쪽 대응 ADR: `platform-data-model` ADR 0014.

## 배경 (Context)

앞의 세 층에는 공통점이 있다. **이름이 틀리면 무언가가 실패한다.** 테이블명이 틀리면 쿼리가 에러를 내고, 브랜치명이 틀리면 훅이 막고, 태그가 틀리면 발행이 멈춘다.

**하네스 층에는 그 안전망이 없다.** 스킬 이름이 틀려도 예외가 나지 않는다 — 다른 것이 발동하거나 아무것도 발동하지 않을 뿐이다. 그리고 이 층에만 요구가 하나 더 있다: 이 이름은 **사람이 타이핑하는 문자열이면서 동시에 모델이 라우팅 판단에 쓰는 입력**이다. 짧게 만들면 변별력이 죽고 설명적으로 만들면 타이핑 비용이 된다.

### 실측 (2026-09-10)

| | 실측 | 무엇이 깨지는가 |
|---|---|---|
| 형태 | `run-inspect`(동사-동사) · `platform-request`(명사-명사) · `decisions`·`project`(맨명사) | 이름만 보고 성격을 알 수 없다 |
| 길이 | `platform-request` 16자 — 상한 12자 초과 | `/platform-request` 는 외워서 치는 길이가 아니다 |
| 소유자 | 그 상한이 `deffile/canon-single`("정본이 정확히 하나")에 얹혀 있었다 | 규칙 id 와 메시지가 어긋나 에이전트가 id 로 대상을 특정하지 못한다 |
| 본문 예산 | second-brain 스킬 5건이 281~736줄 (상한 70줄의 4~10배) | 본문은 호출마다 메인 컨텍스트를 소비한다. 상시 비용이며 보이지 않는다 |
| 링크 자산 | `ego-browser` 가 심볼릭 링크라 스캐너가 놓쳤다 | 이름은 호출 공간을 점유하는데 충돌 검사에는 없다 |
| 강제 | `--harness` 0건 · 등록부 0건 | [[dec-2026-07-31-git-repo-topology-naming-convention]]의 회고가 진단한 그 상태 — 규칙도 검사도 없다 |

## 결정 (Decision)

**3분할한다.** 형태는 `platform-data-model/conventions/naming.yaml` 4절이, 실재하는 이름의 목록은 `platform-governance/registry/harness/` 가, 발동 정확도는 `platform-orchestration/evals/harness-routing/` 이 소유한다.

### D1. 형태와 목록을 가른다

`platform-data-model` 은 인스턴스를 거부하고([[dec-2026-09-10-data-catalog-ownership-and-staged-rollout]]) `platform-governance` 는 이미 카탈로그를 소유한다. 그런데 **스코프 충돌 검사는 실재하는 이름의 목록 없이 성립하지 않는다** — 그 둘을 잇는 것이 이 분할이다.

### D2. `platform-orchestration` 을 형태의 정본으로 두지 않았다

최초 안이 그것이었다. 기각 사유 셋 중 결정적인 것은 **`skillNameLength` 가 이미 `platform-data-model` 에 있었다**는 사실이다 — orchestration 을 골랐다면 하네스 규칙이 첫날부터 두 곳으로 갈렸다. 나머지 둘: 그 저장소 README 가 *"두뇌다. 손발이 아니다"* 로 자기 범위를 그었고, 형태는 손발 쪽 사실이다. 그리고 `platform-data-model` 은 리프라 4층 린터가 한곳에 모인다.

### D3. 자산의 종류가 채번 출처를 정한다

스킬은 **디렉토리가**, 서브에이전트는 **`name` 필드가** 채번한다. 스킬의 `name` 은 생략 가능하고 생략하면 디렉토리명이 쓰이므로, 둘 다 있으면서 다르면 파일을 열지 않고는 무엇으로 호출되는지 알 수 없다.

### D4. 형태 전략은 셋뿐이다

`{대상}-audit`(감사형) · `{동사}-{대상}`(생성형) · `{명사}`(참조형). 셋 밖으로 나가는 이름은 **자산 종류를 잘못 고른 신호다** — `run-inspect`(동사-동사)는 두 일을 하나에 담았다는 뜻이고, 그러면 어느 쪽으로 라우팅할지 이름이 말하지 않는다.

### D5. 예산은 합으로 센다

MCP 노출명 `mcp__{server}__{tool}` 이 그대로 API 의 `tool.name` 이 되고 그쪽 정규식이 `^[a-zA-Z0-9_-]{1,128}$` 다. **점과 콜론을 쓸 수 없어** capability id 의 `namespace.resource.action` 을 여기 가져올 수 없고, 고정 문자 7자를 빼면 **서버명+툴명 = 121자**다.

### D6. 가릴 수 있는 것만 에러다

스코프 교차 동명은 에러(개인 스코프는 어떤 프로젝트와도 함께 로드된다), 같은 스코프·다른 저장소는 note(한 세션이 둘을 함께 열 때만 부딪힌다), 외부 소유 이름의 형태는 검사하지 않는다.

### D7. 저장소별 opt-in

`--harness` 기본 꺼짐. 형상관리 층과 같다.

## 트레이드오프

**얻는 것.** 이름 하나로 성격·스코프·호출 문자열이 정해진다. 충돌이 사후가 아니라 **조립 시점에** 걸린다. 탐색력 측정 13/13 적중·잡음 0·패턴 1 — "이름이 인덱스다"라는 주장이 이 층에서도 성립한다.

**드는 것.** 새 자산마다 8단계. 그 중 둘이 등록부 갱신을 요구하고, 잊으면 다음 실행이 "미등록 자산"으로 잡는다.

**가장 큰 제약 — 호환 별칭이 없다.** 데이터 층은 `v_{옛이름}` 뷰로 개명을 유예할 수 있지만 스킬·에이전트에는 그 장치가 **존재하지 않는다.** 개명은 호출자 전부 + 등록부 + 재시작 확인이 한 변경에 들어가야 한다. 앞 층의 유예 절차를 그대로 쓸 수 있다고 믿는 것이 이 층에서 가장 흔한 오해다.

**개인 스코프에는 CI 가 없다.** `~/.claude/` 는 어느 저장소에도 속하지 않아 강제 경로가 로컬 린터와 `defs-audit` 뿐이다.

## 기각한 대안

**맨명사 금지를 에러로.** 초안이 그랬고 실측에서 `defs` 를 잡았다 — 의도적으로 짧게 지은 고유 약어다. *"일반명사인가"* 를 하이픈 유무로 대신 재면 그런 오탐이 나고, 오탐이 쌓이면 사람들이 규칙을 끈다. 지금은 note 이고, **가림이 등록부로 증명될 때만** 에러다.

**플러그인 스킬 전수 등재.** 40건이 넘고 릴리스마다 낡는다. 그리고 넣을 이유가 없다 — 플러그인 스킬은 `{plugin}:{skill}` 로 불려 가릴 수도 가려질 수도 없다. 반면 플러그인 **에이전트** 는 네임스페이스가 붙지 않아 덮이므로 등재한다. 이 비대칭이 등록부가 무엇을 위한 목록인지 말해 준다.

**링크 자산 건너뛰기.** 스캐너의 최초 동작이 그랬고 `ego-browser` 를 놓쳤다. 링크된 자산은 **이름을 실제로 점유한다.**

## 후속

1. **본문 예산 5건.** 이 저장소(second-brain)의 스킬이 281~736줄이다. `--harness` 를 켜는 순간 불합격하며, 유예를 미리 넣지 않은 것은 **추측한 유예는 만료 때 연장되기 때문이다.** 이 저장소가 시점을 고른다.
2. **eval 기준선.** `evals/harness-routing/` 에 케이스만 있고 첫 실행이 없다. 그 전까지 「라우팅이 옳다」는 측정되지 않은 주장이다.
3. **`platform-request` 개명.** 2026-10-01 유예 만료. `request` 로 줄이면 상한 안에 들고 저장소 이름과의 중복도 사라진다.
4. **재검토 트리거 6개** — ADR 0014 에 수치로 적혀 있다. 미등재 3건 누적 · `/doctor` 동명 신고 · MCP 예산 여유 20자 미만 · eval 정확도 하락 · 유예 연장 · 자산 종류 7종 초과.
