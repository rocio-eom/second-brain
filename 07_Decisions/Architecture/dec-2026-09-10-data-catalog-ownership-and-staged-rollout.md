---
type: decision
category: architecture
created: 2026-09-10
modified: 2026-09-10
status: accepted
tags: [data-catalog, metadata, ownership, lineage, governance, architecture, adr, platform]
aliases: [Data Catalog Ownership ADR, 데이터 카탈로그 소유 결정, staged catalog rollout]
retrospective_date: 2026-10-01
related:
  - "[[fl-2026-09-10-data-catalog-overview]]"
  - "[[fl-2026-09-10-data-catalog-metadata-model]]"
  - "[[fl-2026-09-10-data-catalog-ingestion-lineage]]"
  - "[[fl-2026-09-10-data-catalog-governance-adoption]]"
  - "[[fl-2026-08-03-unified-connector-api-central-data-layer]]"
  - "[[fl-2026-08-04-erd-to-code-repo-layout]]"
  - "[[fl-2026-08-15-ontology-engineering-governance]]"
  - "[[fl-2026-09-10-ontology-adoption-considerations]]"
  - "[[dec-2026-07-31-git-repo-topology-naming-convention]]"
---

# ADR: 데이터 카탈로그의 소유와 단계별 도입

## 상태

Accepted (2026-09-10). 회고 예정: 2026-10-01 (약 3주 후, S1 적용 결과 점검).

플랫폼 저장소 쪽 대응 ADR은 같은 날 적용됐다 — `platform-governance` ADR 0007, `platform-data-model` ADR 0011.

## 배경 (Context)

데이터 카탈로그 설계를 조사([[fl-2026-09-10-data-catalog-overview]] 외 3편)한 뒤 실제 플랫폼에 적용하려 했는데, **조사한 것과 지금 이행 가능한 것이 달랐다.**

노트 4편은 **데이터 자산 카탈로그**를 다룬다 — 테이블·컬럼·지표, 그리고 그 사이의 리니지. 그런데 이 플랫폼에는 **데이터스토어가 0개**다. `platform-data-model/data-model/physical/` 은 스키마만 있고 인스턴스가 없으며, `platform-infrastructure/stacks/**` 에는 아직 `.tf` 파일이 없다. **수집할 자산이 없다.**

반면 조사하지 않은 곳에 카탈로그가 이미 있었다. `platform-governance/registry/` 가 스스로 "서비스 카탈로그의 정본이다"라고 선언하고 항목 2건을 갖고 있으며, `platform-orchestration` 의 planner 와 CLI 가 **코드로 그것을 읽는다** — 등록되지 않은 `service.id` 의 계획은 거부된다.

**없는 것은 카탈로그가 아니라 그것을 카탈로그라고 인정하는 결정 기록이었다.** `platform-governance/docs/adr/` 는 0006 에서 끝나 있었고, `platform-orchestration/AGENTS.md` 는 "governance ADR 진행 중"이라고 적고 있었는데 그 ADR 이 존재하지 않았다. 그 사이 `platform-data-model/docs/architecture/overview.md` 는 카탈로그의 소유를 "미정"으로, `identity-resolution.md` 는 "현재 해소는 불가능하다"고 적고 있었다 — 둘 다 이미 거짓이었다.

그리고 `platform-data-model` ADR 0003 이 걸어 둔 재검토 트리거가 **이미 발화한 상태**였다:

> 인스턴스 속성을 필요로 하는 소비자가 **두 곳 이상**이 되어 각자 자기 사본을 들기 시작하면 → 카탈로그 저장소의 소유를 결정한다. 이 저장소로 들이는 것이 아니라 **어디에 둘지를 정한다**

## 결정 (Decision)

### D1. 서비스 카탈로그의 정본은 `platform-governance/registry/repos/` 다

판정기 옆에 둔다. 유일한 소비자가 정책 평가와 planner 라우팅이므로 왕복이 없고, 그 저장소는 이미 인스턴스 데이터(`approvals/roles.yaml` 의 `subjects`·`teams`)를 소유하므로 새 원칙이 아니라 기존 사실의 확장이다. 스키마는 `$ref` 한 방향으로만 참조하므로 온톨로지 복제가 일어나지 않는다.

### D2. URN을 지금 도입하지 않는다 — 대신 불변식 셋을 못 박는다

[[fl-2026-09-10-data-catalog-metadata-model]]이 "URN과 key aspect는 ingestion 전에 확정하라"고 못 박았고, 그것이 "되돌리기 가장 비싼 결정"이라는 것도 맞다. 그런데 **entity type이 한 종류뿐이라 `urn:platform:service:` 접두어가 구분하는 것이 없다.** 지금 도입하면 소비자 없는 정규화 층을 만드는 것이고, `service.id` 는 이미 최소 다섯 곳(정책 match 경로, planner 라우팅 키, `allowedActions` 대상, 파일명 규약, registry 파일명)에 키로 박혀 있다.

대신 세 가지를 확정했다:

1. **key aspect는 `service.id` 단독이다.** `repo` 는 키가 아니라 별칭 축 — 저장소 이름이 바뀌어도 같은 서비스다
2. **entity type은 파일 위치가 결정한다.** `registry/repos/` 아래면 `service`. 파일에 `type:` 필드를 넣지 않는다
3. **`service.id` 의 패턴은 `platform-data-model` 이 소유한다** (`^[a-z][a-z0-9-]*$`). URN의 NSS로 그대로 안전하다

이 셋이 있으면 `urn:platform:service:{id}` 는 **데이터 마이그레이션 0으로** 언제든 유도된다. "되돌리기 비싼 결정"을 실제로 싸게 만드는 방법은 지금 도입하는 것이 아니라 **나중에 순수 함수로 유도되게 만들어 두는 것**이다.

도입 트리거: **두 번째 entity type의 첫 인스턴스** (`dataset` · `dashboard` · `glossaryTerm` 중 하나).

### D3. 단계 게이트 — 지금 확정된 일정은 S1 하나뿐이다

| 단계 | 내용 | 여는 트리거 |
|---|---|---|
| S1 | 서비스 카탈로그 소유 확정 | **이미 발화** — ADR 0003 트리거 ① |
| S2 | `platform-catalog` 분리 | 등록부 21건 이상 · 또는 플랫폼팀 밖 계정 커밋 머지 · 또는 별칭 축 6개 이상 |
| S3 | 데이터 자산 인벤토리 | 첫 데이터스토어 apply 완료. **단서**: 개인정보를 담으면 접속기록·접근통제(P0)가 자산 카탈로그(P1)보다 먼저 |
| S4 | 테이블 수준 리니지 | 데이터스토어 2개 이상 · 또는 첫 변환 job 커밋 |
| S5 | 제품 전환 평가 | 아래 4개 중 2개 이상 동시 관측 |

나머지는 전부 조건문이다. [[fl-2026-09-10-ontology-adoption-considerations]]의 "12–24개월 계획은 실패 신호, 4–8주 반복이 권장 기준선"을 지키는 방법은 일정을 짧게 쓰는 것이 아니라 **일정을 안 쓰는 것**이다. S3는 S2의 완료를 요구하지 않는다 — 자산 수와 자산 종류는 독립 축이다.

**S5 전환 조건** (관측 가능):

1. 자산 수 > 300 — YAML 300개는 git diff가 의미를 잃는다
2. **운영 메타(신선도·사용량)를 커밋해야 하는 압력** — YAML이 분 단위로 커밋되기 시작하면 그것은 git을 DB로 쓰는 것이다. 가장 결정적인 신호
3. `dist/` 번들 로드가 MCP 호출당 p95 500ms 초과, 또는 자유 텍스트·facet 검색 요구 발생
4. 컬럼 리니지가 규제 요구로 필수화

분기 시 **OSS(DataHub 또는 OpenMetadata)가 기본값이고 상용은 고려하지 않는다.** [[fl-2026-09-10-data-catalog-overview]]의 1차 기준은 "자체 확장 계획 유무"인데, 이 플랫폼은 이미 자체 로더·검증기·MCP를 zero-dependency로 운영한다. 확장 역량이 명백하므로 상용의 번들 이점보다 이식성·모델 확장성이 우선한다. **이 판정을 지금 적어 두어 나중에 재논쟁하지 않는다.**

### D4. 강제 메커니즘 — 트리거는 사람이 아니라 검사가 발화시킨다

[[dec-2026-07-31-git-repo-topology-naming-convention]]의 D4와 같은 원칙이다. 트리거를 산문에만 두면 발화하지 않는다 — 항목도 축도 **한 번에 하나씩** 늘고, 늘리는 사람은 그때마다 "하나쯤"이라고 판단한다.

| 검사 | 어디 | 무엇 |
|---|---|---|
| G15 | governance `scripts/validate.mjs` | 등록부 16건부터 경고, 20건 초과 시 **실패** |
| G16 | 같은 곳 | `service.repository` 가 `repo` 에서 유도되지 않으면 실패 — 같은 사실이 두 곳에 있고 갈라지는 것을 막는다 |
| 별칭 축 수 | data-model `scripts/validate.mjs` §11 | 축 4개부터 경고, 5개 초과 시 **실패** (ADR 0003 트리거 ②) |

셋 다 **형제 저장소를 읽지 않는다.** `PLATFORM_REPO_TOKEN` 이 아직 없어 크로스리포 검증이 CI에서 실패하는 상태인데, 카탈로그의 새 불변식까지 그 블로커에 인질로 잡히면 안 되기 때문이다.

그리고 별칭 축 검사에는 **축 6개 픽스처 테스트**를 붙였다. 이 검사는 필드 경로가 틀리면 축을 0으로 세고 정본은 문제없이 통과한다 — 검사가 죽어도 아무도 모르는 정확한 경로다. **"조용히 통과하지 않는다"는 검사 자신에게도 적용된다.**

### D5. 온톨로지와 카탈로그의 경계를 한 문장으로 고정한다

> 온톨로지는 "그것이 무엇을 뜻하는가"에 답하고 카탈로그는 "무엇이 실재하는가"에 답한다. 카탈로그 모델은 추론이 아니라 조회·검색·순회를 위한 구조이므로 추론 규칙을 갖지 않으며, 따라서 인스턴스는 `platform-data-model` 로 들어오지 않고 스키마는 `platform-governance` 로 복제되지 않는다 — 참조는 언제나 `$ref` 한 방향이다.

같은 문자열을 두 ADR 양쪽에 넣었다. [[fl-2026-09-10-data-catalog-metadata-model]]의 "OWL 수준의 표현력이 필요한 경우는 좁다"와 [[fl-2026-08-15-ontology-engineering-governance]]의 물리/시맨틱 소유 분리가 여기서 만난다.

지키는 것은 **양방향 검사 두 개이며 둘 다 이미 존재했다** — 인스턴스가 정본으로 새는 것은 data-model `validate.mjs` §7(모든 메타 스키마 객체가 `additionalProperties: false`)이, 스키마가 복제되는 것은 governance의 G10과 §9가 막는다. 새 검사를 만들지 않고, 대신 **어느 검사가 이 불변식을 지탱하는지를 ADR 본문에 적었다** — 누군가 그 검사를 지울 때 무엇이 무너지는지 알게 하려고.

## 고려한 대안 (Considered options)

| 대안 | 기각 사유 |
|---|---|
| DataHub / OpenMetadata 즉시 도입 | 자산 2건. [[fl-2026-09-10-ontology-adoption-considerations]] — "역량 공백은 도입 유예 사유다. 정답은 '작게 시작'이 아니라 '지금은 안 함'" |
| `platform-catalog` 를 지금 신설 | 파일 두 개짜리 저장소는 카탈로그가 아니라 오버헤드다. 20건 트리거를 미리 당기지 않는다 |
| `platform-data-model` 로 들이기 | ADR 0003이 확정 기각. 변경 주기가 다르다 — 개념 리뷰가 서비스 등록 PR에 파묻힌다 |
| `platform-engineering` / `platform-orchestration` | 전자는 자기 ADR 0006으로 거부했고, 후자는 도메인 데이터를 실행기가 가지면 역의존이 생긴다 |
| URN 스킴 즉시 도입 | D2 참조. entity type이 1종일 때 접두어는 의식(ritual)이다 |
| ADR 0003을 `대체됨` 으로 표시 | 그 결정문("인스턴스는 단 한 건도 들어오지 않는다")은 여전히 참이다. 그리고 살아 있는 검사(`validate.mjs` §7)가 0003을 인용하므로 폐기 처리하면 검사가 폐기된 ADR을 근거로 실패시키게 된다 |
| Tech-Selection 결정 노트를 함께 작성 | 시기상조. 아래 후속 참조 |

## 결과 (Consequences)

**좋은 점**

- ADR 0003이 "나쁜 것"으로 적어 둔 항목 — *"billing의 tier가 뭔가"에 답할 곳이 없다* — 이 해소됐다
- **동선 문제가 이미 해결되어 있다.** [[fl-2026-09-10-data-catalog-governance-adoption]]이 "실패의 1차 원인은 기술이 아니라 동선"이라고 했는데, 여기서 카탈로그의 소비자는 사람이 아니라 LLM planner이고 planner는 이미 그것을 읽는다. 별도 웹 UI로 존재해 무시당하는 실패 모드가 구조적으로 없다
- 트리거 셋이 검사를 갖게 됐다. ADR이 산문이 아니라 게이트다

**나쁜 점 / 비용**

- ADR이 두 개 늘었다. ADR 0003을 읽는 사람이 0011까지 따라와야 완전한 상태를 안다
- `platform-governance` 가 규칙과 데이터를 함께 갖는다. `policies/` 는 서비스 이름을 조건절에 쓸 수 없는데 `registry/` 는 이름이 키다 — 두 성격이 한 저장소에 있다는 사실 자체가 이관 트리거의 존재 이유다
- **집행이 이미 켜져 있다.** [[fl-2026-09-10-data-catalog-governance-adoption]]은 "집행을 먼저 켜면 저항이 등록 자체를 막는다"고 경고하는데, 이 플랫폼은 순서가 역전된 상태다 — planner가 미등록 서비스의 계획을 이미 거부한다. **지금은 저항할 주체가 없어 무해하지만, 비플랫폼 기여자가 처음 생기는 순간 그가 완전히 켜진 집행을 정면으로 맞는다.** 그 시점이 정확히 ADR 0007의 "팀 밖 쓰기" 트리거와 같다
- 완화 경로를 트리거 옆에 미리 적어 뒀다: 신규 기여자는 `lifecycle: proposed` 로 등록만 먼저 하면 되고 `proposed` 에는 프로젝트 정책이 적용되지 않는다. **등록의 문턱을 높이는 변경을 할 때 이 경로를 함께 없애지 않는다**

**제약**

- `service` 객체의 스키마 적합성은 **CI에서 검증되지 않는다.** `$ref` 해소에 형제 저장소 체크아웃이 필요한데 `PLATFORM_REPO_TOKEN` 이 없다. 검사는 통과하지 않고 사유를 출력하며 실패한다 — 조용히 넘어가지 않는다. 로컬에서는 `REF_BASE=~/workspace` 로 전부 동작한다
- 해소되는 별칭 축은 `repository` 하나뿐이다. `k8sNamespace` · `terraformStackPath` 는 `reserved` 이며 매핑 행이 아직 어디에도 없다

## 트레이드오프

- **지금의 싼 결정 vs 나중의 마이그레이션 비용** — URN을 미루는 대신 불변식 셋으로 유도 가능성을 산다. 불변식이 깨지면 이 거래가 무효가 된다
- **소유의 명확성 vs 저장소 응집도** — 카탈로그를 판정기 옆에 두면 왕복이 없지만, governance가 규칙과 데이터를 함께 갖는다
- **트리거를 실패로 두는 것 vs 경고로 두는 것** — 실패는 21번째 등록을 막는다. 급할 때 우회 유혹이 생기지만, 통과시키면 "트리거가 발화했는데 아무도 결정하지 않는 상태"가 곧 조용한 stale이 된다. **우회 경로는 트리거를 고치는 것뿐이며 그것은 ADR 개정이다**

## 후속 (Follow-ups)

- [ ] **`PLATFORM_REPO_TOKEN` PAT 생성 (사람 작업, 약 5분).** 없으면 "문서는 정확한데 기계는 절반만 지킨다"는 상태로 굳는다. 그런 상태를 방치한 전례가 이미 있다
- [ ] Tech-Selection 노트는 **쓰지 않는다 — 시기상조다.** 선택된 것이 "현 방식 연장 + 전환 트리거"이므로 제품 비교가 실행되지 않았다(PoC·비용 산정 전무). 지금 쓰면 껍데기가 되고, 나중에 진짜 선택할 때 "이미 결정했다"는 착각을 만든다. **S5 조건 2개가 관측되는 시점에 spawn**한다
- [ ] 두 번째 entity type이 들어오면 `dec-YYYY-MM-DD-catalog-metadata-model` 을 Architecture로 분리한다. 메타데이터 모델은 "되돌리기 가장 비싼 결정"이므로 그때는 자기 노트를 가질 값어치가 있다
- [ ] **Fleeting 4편의 Permanent 승격 — `overview` 1건 파일럿부터.** 선결 문제 셋이 있다: (a) `Backend/Architecture` · `Backend/Databases` 는 vault CLAUDE.md에 선언된 서브폴더가 아니다(`Backend/{Distributed-Systems, Data-Pipeline, AWS}` 뿐)이므로 폴더 표를 먼저 갱신해야 하고, (b) Permanent 필수 필드 `moc` 를 채울 수 없다(`04_MOC/` 가 디스크에 없다), (c) 4편 모두 한국어 태그를 달고 있다. **이것이 vault의 첫 승격이라 승격 기계 자체가 미검증**이므로 1건으로 검증한 뒤 나머지를 일괄 처리한다
- [ ] **오픈 이슈 — 결정 노트 형식이 둘로 갈린다.** `.claude/skills/decisions/SKILL.md` 는 Phase 1–4 H1 넷과 고정 H2 열하나(`## 로드밸런서` · `## 레디스` · `## 사용자 데이터베이스` 포함)를 강제하는데, 실제 존재하는 결정 노트([[dec-2026-07-31-git-repo-topology-naming-convention]])는 그 골격을 따르지 않고 ADR 형식이다. 이 노트도 ADR 형식을 따랐다 — 소유·경계 판정에 "레디스" 절을 채우는 것은 길이를 채우려고 쓴 문장이 되기 때문이다. **다음 결정 노트를 쓰기 전에 정본 형식을 확정할 것.** 스킬에 "설계 인터뷰가 아닌 경계·소유 판정" 모드를 추가하는 쪽이 유력하다
- [ ] 회고(2026-10-01): 등록부 항목이 몇 개로 늘었는지, G15/G16이 실제로 무언가를 잡았는지, PAT이 생겼는지 점검

## 관련 자료

- 설계 개관·요구사항·안티패턴: [[fl-2026-09-10-data-catalog-overview]]
- 메타데이터 모델·URN·Entity/Aspect: [[fl-2026-09-10-data-catalog-metadata-model]]
- 수집·리니지·OpenLineage: [[fl-2026-09-10-data-catalog-ingestion-lineage]]
- 거버넌스·채택 전략: [[fl-2026-09-10-data-catalog-governance-adoption]]
- 카탈로그·리니지의 P1 판정: [[fl-2026-08-03-unified-connector-api-central-data-layer]]
- 중앙 표준 / 프로젝트 데이터의 연합 분할: [[fl-2026-08-04-erd-to-code-repo-layout]]
- 물리 소유 vs 시맨틱 스튜어드십: [[fl-2026-08-15-ontology-engineering-governance]]
- 도입 판정 게이트("지금은 안 함"): [[fl-2026-09-10-ontology-adoption-considerations]]
- 강제 메커니즘 원칙(D4): [[dec-2026-07-31-git-repo-topology-naming-convention]]
- 저장소 ADR: `platform-governance/docs/adr/0007-registry-is-the-service-catalog.md` · `platform-data-model/docs/adr/0011-catalog-ownership-resolved.md`
