# 축구 Temporal KG 스키마 (v3)

선수–클럽–리그 소속을 유효 구간·출처·버전과 함께 담아, 임의 시점의 소속을 질의하고 새 정보를 재구축 없이 반영하기 위한 KG 설계다. 모든 수치는 `dataset/` final_v3 릴리스(2026-10-08)를 2026-10-11에 직접 집계한 값이다. v3는 리뷰 피드백을 반영해 선수별 관측 시점, 추론 규칙 R3, 평가 프로토콜을 추가했다(§15).

- **구현과 함께 읽기:** 실제로 만든 KG(r0)는 [docs/model.md](docs/model.md)가 설명하고, 이 문서보다 정밀하게 정한 부분은 그 문서의 "Clarifications of kg-schema v3"가 우선한다. 갱신 연산의 실제 동작은 [docs/updates.md](docs/updates.md), 수치 재계산 결과는 검증 보고서(`build/r0/reports/validation_report.md`, 릴리스의 Parquet zip에도 있음)에 있다.
  - 이 문서의 다음 표현은 구현에서 바뀌었다: `Club:Unresolved` → 별도 라벨 `UnresolvedClub`, Spell의 `derivation` → `origin`·`end_derivation`·`evidence_end_method`, §8 경고 4의 4,338명 → v3 정의로 4,443명, §12의 "A_max 이하" → "미만", §13의 625명 → 626명.
  - 구현과 다르게 적혀 있던 문장은 2026-10-14에 고쳤다(§15).
- 그림으로 보는 버전: <https://claude.ai/artifact/FfgXGum42D8XLPUD3QHceQ>
  - v2 기준이라 선수별 관측 시점과 R3는 아직 그림에 없다.
  - 비공개 링크라 다른 사람은 공유 설정 후에 열 수 있다.
- 데이터 설명: [dataset-description.md](dataset-description.md) · 컬럼 정의: [dataset/data_dictionary.csv](dataset/data_dictionary.csv)
- 이 문서는 `dataset-description.md` §7의 KG 구성 제안을 대체한다. 그 제안과 다른 점은 세 가지다.
  - Fact 노드 대신 관계를 노드로 만든다.
  - `CURRENTLY_PLAYS_FOR` 대신 `ProfileClaim`을 둔다.
  - 선수별 관측 시점을 둔다.

## 1. 목표와 설계 원칙

| 목표 | 스키마의 대응 |
|---|---|
| 선수–클럽–리그 관계를 유효 구간·출처와 함께 담아 임의 시점의 소속을 질의한다 | 관계를 노드(`Spell`, `LeagueSpell`, `ProfileClaim`)로 만들어 구간과 출처를 직접 붙인다. as-of 규칙(§6)이 답과 확실성 등급을 돌려준다 |
| 무결성 제약·갱신 연산·충돌 해소 규칙으로 새 정보를 재구축 없이 반영한다 | 기간은 이벤트에서 파생하고(§5), 갱신은 덮어쓰지 않고 새 버전을 쌓는다(`Revision`, `SUPERSEDES`). 규칙은 §8–§10 |
| 시점별 QA로 closed-book·문서 RAG 대비 개선을 보인다 | 답의 등급과 선수별 관측 시점으로 정답 셋의 범위를 정하고, `Revision`으로 특정 시점의 KG를 재현한다(§12) |

설계 원칙은 다섯 가지다.

1. **관계를 노드로 만든다.** 선수의 소속(`Spell`), 클럽의 리그 소속(`LeagueSpell`), 프로필상 소속(`ProfileClaim`)을 노드로 둔다. 유효 구간·출처·버전을 관계에 직접 붙일 수 있어서, 출처용 Fact 노드(약 142만 개) 없이 `SUPPORTED_BY {fact_name}` 엣지로 충분하다.
2. **기간은 이벤트에서 나온다.** `Spell`의 시작과 끝은 `TransferEvent`를 가리킨다. 끝의 근거는 `evidence`·`inferred`·`open`으로 구분해 저장한다.
3. **시간 축은 두 개다.** 유효 시간은 실제로 사실이었던 기간이고, 기록 시간은 KG가 그 사실을 알게 된 `Revision`이다.
4. **KG가 아는 시점은 선수마다 다르다.** 선수 이력은 원본 파일마다 다른 시점(2024년 7월~2026년 10월)에 수집됐다. 선수별 관측 시점 `history_observed_at`(H_p) 이후는 KG도 모르는 구간으로 다룬다.
5. **답에는 확실성이 붙는다.** as-of 질의는 클럽과 함께 `CONFIRMED`·`INFERRED`·`OPEN`·`NONE` 중 하나를 돌려주고, `INFERRED`는 근거 규칙(R1·R2·R3)을 함께 밝힌다. planned 이벤트는 spell을 바꾸지 않지만, 예정일이 지나면 답을 `OPEN`으로 낮춘다.

## 2. 전체 구조

```mermaid
flowchart LR
  Player["Player<br/>선수 · history_observed_at"]
  PC["ProfileClaim<br/>프로필상 현재 소속<br/>observed_at 미상"]
  TE["TransferEvent<br/>이적·상태 변화<br/>completed · planned · unknown"]
  SP["Spell<br/>선수의 소속 기간<br/>valid_from → valid_to · end_basis"]
  Club["Club<br/>team_level"]
  CS["CareerStatus<br/>Retired · Without Club · Career break · Ban<br/>+ Unknown = 행선지 미상"]
  LS["LeagueSpell<br/>클럽의 리그 소속<br/>observed_at 미상"]
  League["League"]
  Country["Country"]
  Position["Position"]
  Src["Source<br/>출처 레코드"]
  Rev["Revision<br/>적재·갱신 배치"]

  Player -- HAS_CLAIM --> PC
  PC -- POINTS_TO --> Club
  PC -. POINTS_TO .-> CS
  TE -- FOR_PLAYER --> Player
  TE -- "FROM / TO" --> Club
  TE -. "FROM / TO" .-> CS
  Player -- HAS_SPELL --> SP
  SP -- AT --> Club
  SP -. "AT (status spell)" .-> CS
  SP -- "LOANED_FROM (loan)" --> Club
  SP -- STARTED_BY --> TE
  SP -- "ENDED_BY {basis}" --> TE
  Club -- HAS_SPELL --> LS
  LS -- IN --> League
  Player -- CITIZEN_OF --> Country
  League -- IN_COUNTRY --> Country
  Player -- HAS_POSITION --> Position
  SP -- "SUPPORTED_BY {fact_name}" --> Src
  SP -- RECORDED_IN --> Rev
  SP -- SUPERSEDES --> SP

  classDef entity fill:#FFFFFF,stroke:#A9B5AD,color:#17211B
  classDef fact fill:#DBEFE2,stroke:#1D7A4B,color:#17211B
  classDef snap fill:#DBEFE2,stroke:#1D7A4B,stroke-dasharray:5 4,color:#17211B
  classDef event fill:#FBEFC6,stroke:#A87400,color:#17211B
  classDef prov fill:#E1E9F3,stroke:#3D6189,color:#17211B
  class Player,Club,CS,League,Country,Position entity
  class SP fact
  class PC,LS snap
  class TE event
  class Src,Rev prov
```

선수에서 클럽으로 가는 길이 세 갈래다.

| 경로 | 담는 것 | 답하는 질문 |
|---|---|---|
| `Player → ProfileClaim → Club` | 날짜 없는 프로필 기록 | "지금 어디?"의 약한 근거 |
| `TransferEvent → Player`, `TransferEvent → Club` | 한 시점의 변화 | "언제 무엇이 바뀌었나?" |
| `Player → Spell → Club` | 기간 | "t 시점에 어디서 뛰었나?" |

as-of 질의는 `Spell`로 답하고, 나머지 두 경로는 근거(이벤트)와 보강 증거(프로필)가 된다. `SUPPORTED_BY`·`RECORDED_IN`·`SUPERSEDES`는 그림에서 Spell에만 그렸지만, 버전이 쌓이는 다른 노드(§3.1)에도 똑같이 있다.

## 3. 노드

| 노드 | 의미 | 주요 속성 | 키 | 개수 (final_v3) |
|---|---|---|---|---:|
| `Player` | 선수 | `name`, `birth_date`, `birth_date_precision`(exact·year·unknown), `birth_city`, `birth_country`, `history_coverage`(valid_nonempty·valid_empty·unavailable), `history_source`, `history_observed_at`, `history_complete` | `player_id` | 40,706 |
| `Club` | 실제 클럽 | `name`, `name_scope`, `aliases[]`(observed_names), `team_level`(first_team·reserve·youth, 파생) | `club_id` | 13,599 |
| `Club:Unresolved` | 이름 없는 클럽 ID | 없음 | `club_id` | 9 |
| `CareerStatus` | 클럽이 아닌 상태 | `code`: retired · without_club · career_break · ban · unknown_destination | `club_id` (123, 515, 2113, 2077, 75) | 4 + 1 |
| `League` | 리그 | `name`, `competition_type`, `competition_sub_type`, `metadata_status` | `league_id` | 32 |
| `Country` | 국적·리그 국가 (파생) | `name`, `aliases`(Turkey·Türkiye), `fifa_rank` | `name` | 53 |
| `Position` | 포지션 | `position`, `sub_position` | `position_id` | 13 |
| `ProfileClaim` | 프로필상 현재 소속 (스냅샷) | `relation_kind`, `observed_at`(미상), `stale` | `current_relation_id` + `version` | 40,706 |
| `TransferEvent` | 이적·상태 변화 | `date`, `order_in_day`, `season`, `event_type`, `temporal_status`(completed·planned·unknown·cancelled), `raw_match_status`, `transfer_id`, `fee_raw`, `fee_eur`, `classification_rule` | `transfer_event_id` + `version` | 137,776 |
| `Spell` | 선수의 소속 기간 | `type`(permanent·loan·unknown·status), `valid_from`, `valid_to`, `end_basis`, `derivation`, `last_confirmed_at`, `transit`, `confidence_status` | `spell_id` + `version` | 110,621 + 파생 2,758 |
| `LeagueSpell` | 클럽의 리그 소속 (스냅샷) | `valid_from`·`valid_to`·`observed_at`(모두 미상), `basis = snapshot` | `club_id:league_id` + `version` | 796 |
| `Source` | 출처 레코드 | `source_type`, `url`, `raw_locator`, `retrieved_at`, `content_sha256`, `extraction_method` | `source_id` | 365,128 |
| `Revision` | 적재·갱신 배치 | `recorded_at`, `source_as_of`, `origin`, 입력 해시, 행 수 | `revision_id` | r0 = final_v3 |

- `history_source`, `history_observed_at`, `history_complete`는 §5.2에서 정의한다.
- `CareerStatus`의 Unknown(75)은 경력 상태가 아니다. 이 ID로 가는 completed 이벤트 205건 중 104건이 자유 이적이므로 "행선지 미상"으로 다룬다.
- `Spell.type = unknown`은 데이터의 `membership_type`을 그대로 쓴 것으로, 유료 이적이나 라벨 없는 이동처럼 유형이 확인되지 않은 소속이다. 행선지 미상(`unknown_destination`)과는 다른 개념이다.
- `TransferEvent.order_in_day`는 같은 날 이벤트의 순서다(§5.1).
- `Club.team_level`은 이름 패턴으로 만든 파생 속성이다.
  - 임시 패턴(U19·U21 같은 나이 표기, II, B, Yth, Youth, Juvenil, Primavera, Reserves, Jong, Academy, Sub-20)으로 클럽 ID 5,296개, membership spell의 26.3%가 유스·리저브로 잡힌다.
  - "RM Castilla", "Juve Next Gen", "Barça Atlètic"처럼 패턴에 안 걸리는 리저브가 있다.
  - 걸린 이름 100개와 안 걸린 이름 100개의 정밀도·재현율을 본 뒤 패턴을 확정한다. 채점에 쓰는 1군 클럽 연결(예: Club Brugge U18 → Club Brugge)도 이때 정한다. r0에서는 이 점검을 아직 하지 않았고, 패턴은 임시값(`team_level_basis = provisional_name_pattern_v0`)이다.
- 엔터티(Player, Club, League 등) 속성은 제자리에서 갱신하고, 바뀐 값은 해당 `Revision`의 로그로 남긴다. 속성 이력 질의는 범위 밖이다.

### 3.1 버전이 쌓이는 노드의 공통 속성

`Spell`, `TransferEvent`, `ProfileClaim`, `LeagueSpell`에는 다음이 공통으로 붙는다.

- `version`: 같은 논리 ID 안에서 1부터 증가한다.
- `tx_from`, `tx_to`: 이 버전이 KG에 기록된 구간이다. `tx_to`가 비어 있으면 현재 버전이다. `RECORDED_IN`·`SUPERSEDES` 엣지와 같은 정보를 담은 사본이며, 필터링을 빠르게 하려고 둔다.
- `RECORDED_IN → Revision`: 이 버전을 만든 배치를 가리킨다.
- `SUPERSEDES → 이전 버전`: 첫 버전에는 없다.

## 4. 관계

| 관계 | 시작 → 끝 | 개수 | 속성 | 메모 |
|---|---|---|---|---|
| `HAS_CLAIM` | Player → ProfileClaim | 현재 버전 1 | | |
| `POINTS_TO` | ProfileClaim → Club · CareerStatus · Club:Unresolved | 1 | | Club 40,679 · Unresolved 21 · CareerStatus 6 |
| `FOR_PLAYER` | TransferEvent → Player | 1 | | |
| `FROM`, `TO` | TransferEvent → Club · CareerStatus | 각 1 | | 특수 ID: FROM 2,444건, TO 3,326건 |
| `HAS_SPELL` | Player → Spell | 0..n | | |
| `AT` | Spell → Club (status spell은 CareerStatus) | 1 | | Unresolved는 대상이 될 수 없다 |
| `LOANED_FROM` | Spell(loan) → Club | 0..1 | | 원소속이 특수 ID인 임대 3건은 없다 |
| `STARTED_BY` | Spell → TransferEvent | 1 | | |
| `ENDED_BY` | Spell → TransferEvent | 0..1 | `basis`, `derivation` | open이면 없다. R2로 닫힌 경우 다음 spell의 시작 이벤트를 가리킨다 |
| `HAS_SPELL` | Club → LeagueSpell | 0..n | | Player의 `HAS_SPELL`과 이름이 같으므로 질의할 때 레이블로 구분한다 |
| `IN` | LeagueSpell → League | 1 | | |
| `HAS_POSITION` | Player → Position | 0..1 | | 시간 없음 |
| `CITIZEN_OF` | Player → Country | 1 | | 1차 국적만 |
| `IN_COUNTRY` | League → Country | 1 | | |
| `SUPPORTED_BY` | Country·Revision을 뺀 모든 노드 → Source | 1..n | `fact_name`, `derivation` | §7 |
| `RECORDED_IN` | 버전 노드 → Revision | 1 | | |
| `SUPERSEDES` | 새 버전 → 이전 버전 | 0..1 | | |

## 5. 시간 모델

### 5.1 유효 시간

- 구간은 반개구간 `[valid_from, valid_to)`이다. 이적 당일에는 이전 spell이 아니라 새 spell이 그 시점을 덮는다.
- **하루 안 순서:** 같은 날 이벤트는 원본 응답의 pointer 인덱스(`raw_event_locator`의 `/transfers/N`)로 순서를 정해 `order_in_day`에 둔다.
  - API 응답은 최신순이라 인덱스가 작을수록 나중 이벤트다. 날짜가 다른 연속 이벤트 119,211쌍에서 예외가 없었다.
  - 아래 규칙에서 "이후"는 (날짜, 하루 안 순서)로 비교한다.
- **길이 0 spell:** 같은 날 입단하고 바로 떠난 spell은 `valid_from = valid_to`인 `transit` spell이 된다. 모두 30개(memberships 29, status spell 1)다. 이벤트 사슬을 잇는 데만 쓰고 as-of 답이 되지 않는다.
- `end_basis`는 종료의 근거를 나타낸다.

| `end_basis` | 의미 | 원본 memberships 110,621건 | 파생 status spell 2,758건 |
|---|---|---:|---:|
| `evidence` | 데이터에 기록된 종료 (임대 복귀, 또는 자유 이적·무소속·은퇴·커리어 중단으로 떠남) | 45,403 | 0 |
| `inferred` (R1) | 같은 클럽 이탈로 추론한 종료 | 45,422 | 2,123 |
| `inferred` (R2) | 다음 spell 시작으로 추론한 종료 | 7 | 1 |
| `open` | 종료 근거 없음 | 19,789 | 634 |

추론 규칙 R1·R2는 종료일을 저장한다. 둘 다 해당하면 이른 쪽을 쓰고, 원본 `end_date`는 바꾸지 않으며, `derivation`에 규칙 이름을 남긴다.

- **R1 (같은 클럽 이탈):** 같은 클럽에서 나가는 가장 이른 completed 이벤트의 날짜다. 등록 spell(permanent·unknown)은 임대로 나가는 이벤트를 제외한다. 임대를 보내도 등록은 유지되기 때문이다.
- **R2 (다음 spell 시작):** 임대·status spell은 같은 선수의 다음 spell이 시작되면 끝난 것으로 본다.

나머지 파생 규칙은 다음과 같다.

- **status spell:** `TO`가 특수 ID인 completed 이벤트마다 spell을 하나 만든다(`type = status`, `AT → CareerStatus`). 상태 2,553건과 행선지 미상 205건이다.
- **`last_confirmed_at`:** 등록 spell이 열려 있는 동안 등록이 살아 있었음을 보여 주는 가장 늦은 날짜다. 근거는 두 가지다.
  - 같은 클럽에서 임대를 보낸 날짜: 10,629개 spell.
  - 데이터셋이 임대 spell의 종료로 매칭한(`evidence`) 복귀의 날짜: 9,294개 spell.
  - 매칭되지 않은 임대 복귀 276건은 데이터셋이 이미 문제로 표시한 사례라 근거에서 빼고 검증 표본(§12)으로 돌린다.
  - 복귀가 확인하는 기간은 대부분 임대 spell이 이미 답하는 구간이다. 그래서 뛰는 팀 답의 등급은 시점별로 몇 명 수준만 바뀌고, 주로 등록 소속의 확인 구간이 늘어난다.
- **R3 (관측 창 안 지속):** 열린 spell이 선수의 관측 시점까지 이어졌다고 보는 규칙이다. lag와 A_max에 따라 달라지므로 저장하지 않고 질의할 때 계산한다(§6.3).

### 5.2 관측 시점

선수 이력은 정본 응답 하나에서 오고(`history_source` = 데이터의 `player_transfer_history_status.raw_source_file`), 원본마다 수집 시점이 다르다. completed 이벤트는 수집 시점 이전, planned 이벤트는 수집 시점 이후이므로 이벤트 날짜로 수집 시점의 범위를 정할 수 있다.

| 원본 (`history_source`) | 선수 | 이벤트가 있는 선수 | 마지막 completed | 첫 planned | `history_observed_at` (H_p) |
|---|---:|---:|---|---|---|
| API 2023 파일 (`transfermarkt-api/2023/transfers.json`) | 2,384 | 2,367 | 2024-07-26 | 2024-07-29 | 2024-07-26 |
| API 2024 파일 | 2,122 | 2,118 | 2026-02-27 | 2026-05-03 | 2026-02-27 |
| API 2025 파일 | 11,979 | 11,965 | 2026-06-11 | 2026-06-15 | 2026-06-11 |
| 로컬 캐시 (`data/raw/transfer_history*/<player_id>.json`) | 2,745 | 1,924 | 2026-10-06 | 2026-11-30 | 선수별 `retrieved_at` (2026-10-07~08) |
| 이력 없음 (`unavailable`) | 21,476 | 0 | — | — | 없음 |

- **`history_observed_at` (H_p):** API 파일은 파일별 하한(마지막 completed 날짜)을 쓴다. 상한(첫 planned 날짜)보다 이른 값이라 보수적이다. 로컬 캐시는 선수별 `retrieved_at` 날짜를 쓰고, 값이 없는 1명은 2026-10-07로 둔다.
- **파일 단위 값을 쓰는 이유:** 상·하한이 모두 있는 2,438명 모두 "completed 날짜 ≤ 관측 시점 < planned 예정일"을 만족한다. 파일마다 마지막 completed가 첫 planned보다 앞서므로 각 파일을 한 번에 수집한 것으로 볼 수 있다.
- **`history_complete`:** `valid_nonempty`이고 수록 이벤트 수가 원본 이벤트 수와 같은 선수다(19,190명 중 17,980명). 나머지 1,210명은 기록된 이동 사이에 빠진 이동이 있을 수 있다. 그중 spell이 있는 선수는 396명이다.
- **`Revision.source_as_of` (2026-10-08):** 릴리스 날짜다. 관측 시점이 아니므로 "KG가 언제까지 아는가"에는 쓰지 않는다.

### 5.3 기록 시간

- `r0`은 final_v3 일괄 적재다.
- 갱신은 버전이 쌓이는 노드를 덮어쓰지 않는다. 바뀐 노드는 새 버전을 만들고, 이전 버전의 `tx_to`와 새 버전의 `tx_from`을 배치 시각으로 맞춘다. 엔터티 속성은 §3대로 제자리에서 갱신하고 로그를 남긴다.
- 시작 근거를 잃은 spell은 지우지 않고 `record_status = void`인 새 버전(tombstone)으로 남긴다.
- 엣지도 `[tx_from, tx_to)`를 가진다. 노드에 새 버전이 생기면 이전 버전에 걸린 엣지는 닫고 새 버전에 다시 만든다. 그래서 Revision 시점의 KG에서 엣지는 그 시점의 노드끼리 잇는다. 예외 두 가지(`SUPERSEDES`, 정식 클럽이 된 미해결 클럽 ID)는 [docs/updates.md](docs/updates.md)에 있다.
- 특정 Revision 시점의 KG는 `tx_from ≤ r < tx_to`인 버전과 엣지만 골라 재현한다. 평가를 재현할 때 이 방식을 쓴다.

## 6. as-of 질의

### 6.1 답 고르기

`as_of(player, t)`는 다음 순서로 답을 고른다.

1. t를 덮는 spell을 모은다. 조건은 `valid_from ≤ t`이고, 기록된 종료나 R1·R2 종료가 있으면 `t < 종료일`이어야 한다. transit spell은 뺀다.
2. 임대 spell이 있으면 그 클럽이 답(뛰는 팀)이고, `LOANED_FROM`을 등록 소속으로 함께 돌려준다. 임대가 없으면 등록 spell과 status spell 중에서 고른다.
3. 후보가 여럿이면 근거 등급(`CONFIRMED > INFERRED > OPEN`)이 높은 쪽, 같으면 늦게 시작한 쪽(날짜, 하루 안 순서)을 고른다. 등록 spell이 겹치면 conflict로 기록한다.
4. 고른 spell에 등급을 매긴다.
   - `CONFIRMED`: t가 기록된 시작과 종료 사이에 있거나 `t ≤ last_confirmed_at`이다.
   - `INFERRED`(R1·R2): t가 추론 종료 이전이다.
   - `INFERRED`(R3): 열린 spell이고, 다음 조건을 모두 만족한다.
     - `history_complete`이다.
     - 생일을 안다.
     - `t < H_p − lag`이다.
     - t 시점 나이가 `A_max` 미만이다.
   - `OPEN`: 그 밖의 열린 spell이다. 관측 시점 이후, 이력 불완전, 고령, 생일 미상이 여기에 해당한다.
5. 답이 `OPEN`이고 그 클럽에서 나가는 planned 이벤트의 예정일이 그 spell의 시작 이후이고 t 이전이면(시작일 < 예정일 ≤ t), 프로필 일치 여부와 관계없이 `OPEN`으로 두고 예정 행선지를 함께 돌려준다. 그렇지 않고 ProfileClaim이 같은 클럽을 가리키면 `OPEN · 프로필 일치`다.
6. 답이 status spell이면 클럽 대신 상태(무소속·은퇴·커리어 중단·징계)를 돌려준다. 행선지 미상이면 `NONE`이다.

completed 이벤트는 모두 H_p 이전이므로, `t ≥ H_p − lag`인 시점에서 열린 spell은 R3로 추론되지 않는다. 다만 등록 spell은 그 뒤의 임대 송출이나 매칭된 임대 복귀로 `last_confirmed_at ≥ t`이면 이때도 `CONFIRMED`다(r0에서 `last_confirmed_at`이 H_p − 14일보다 늦은 열린 spell은 164개다). 기록 시점의 상한은 전역 `source_as_of`가 아니라 이 H_p로 정한다.

### 6.2 답의 등급

| 등급 | 조건 |
|---|---|
| `CONFIRMED` | t가 기록된 시작과 종료 사이에 있거나, `last_confirmed_at ≥ t`이다 |
| `INFERRED` (R1·R2) | t가 추론 종료 이전 구간에 있다 |
| `INFERRED` (R3) | 열린 spell이고, 이력이 완전하고 생일을 알며, `t < H_p − lag`이고, t 시점 나이가 `A_max` 미만이다 |
| `OPEN · 프로필 일치` | 위에 해당하지 않는 열린 spell이고, ProfileClaim이 같은 클럽을 가리키며, 그 클럽에서 나가는 planned 이벤트의 예정일이 아직 지나지 않았다 |
| `OPEN` | 위에 해당하지 않는 열린 spell이고, 프로필이 다르거나 없거나, 예정된 변경의 날짜가 이미 지났다 |
| `NONE` | t를 덮는 spell이 없거나, 답이 행선지 미상이다 (이력 미수집 포함) |

### 6.3 R3 매개변수와 민감도

- **lag (기본 14일):** 관측 시점 직전의 보고 지연분이다. H_p 직전 몇 주의 판정만 바꾼다.
  - 2025-01-01과 2026-01-01의 분포는 lag 0·14·28일에서 같다.
  - 2026-06-01에는 R3 비율이 lag 0일 때 67.7%, 14·28일일 때 7.4%다. API 2025 파일의 H_p가 2026-06-11이기 때문이다.
- **A_max (기본 34세):** 은퇴 가드다.
  - completed 은퇴 이벤트가 112건뿐일 만큼 은퇴는 대부분 기록되지 않는다. 그래서 조용히 은퇴한 선수도 R3로는 관측 시점까지 마지막 클럽에 남는다.
  - 기록된 은퇴 나이(생일이 정확한 110건)는 25분위 33.9세, 중앙값 35.8세, 90분위 38.7세다. 기본값은 보수적으로 25분위에 맞춘 34세다.

| A_max | 2026-01-01 R3 비율 | 2026-01-01 OPEN 계열 비율 |
|---|---:|---:|
| 34 (기본) | 64.2% | 19.1% |
| 36 | 66.9% | 16.5% |
| 38 | 68.1% | 15.2% |

- **생일:** 미상 47명은 R3에서 뺀다. 연도만 아는 273명은 1월 1일로 두어 나이를 많게 잡는다. 그만큼 R3가 일찍 끊긴다.
- **R3 적용 대상:** 이력이 완전하고 생일을 알며 spell이 있는 선수 17,972명이다.

### 6.4 예: Wuilker Fariñez (player `373714`)

API 2025 파일 선수이고, H_p는 2026-06-11이며, 이력은 완전하고, 생일은 1998-02-15다.

| 질의 시점 | 답 (뛰는 팀) | 등록 소속 | 등급 | 근거 |
|---|---|---|---|---|
| 2017-01-01 | Caracas FC | (같음) | `INFERRED` (R1) | 2018-01-01 Millonarios 유료 이적에서 종료 추론 |
| 2019-06-01 | Millonarios FC | (같음) | `CONFIRMED` | 2021-06-30 Lens에서 돌아온 임대 복귀(매칭됨)로 그날까지 등록 확인 (`last_confirmed_at`) |
| 2025-06-01 | Rionegro Águilas | Caracas FC | `CONFIRMED` | 2025-03-10 임대 시작과 2025-12-31 복귀가 모두 기록됨 |
| 2026-03-01 | Internacional de Bogotá | Caracas FC | `INFERRED` (R3) | 2026-01-08 임대 이후 관측 시점까지 이탈 기록 없음, 28세 |
| 2026-09-01 | Internacional de Bogotá | Caracas FC | `OPEN · 프로필 일치` | 관측 시점 이후. 2026-12-31 복귀는 planned이고 예정일 전 |

2026-06-01은 H_p − lag(2026-05-28) 이후라 `OPEN · 프로필 일치`가 된다.

### 6.5 전체 선수의 등급 분포

대상은 그 시점 이전에 시작한 spell(status spell 포함)이 있는 선수다. spell이 하나도 없는 22,334명은 모든 시점에서 `NONE`이다. 이력 미수집 21,476명, 빈 이력 40명, 그리고 원천에 이력은 있지만 릴리스에 completed 이적이 없는 818명이다. 기본값(lag 14일, A_max 34세)으로 계산했다.

| 시점 | 대상 | CONFIRMED | INFERRED (R1·R2) | INFERRED (R3) | OPEN · 프로필 일치 | OPEN | NONE |
|---|---:|---:|---:|---:|---:|---:|---:|
| 2020-01-01 | 12,865 | 6,612 (51.4%) | 5,694 (44.3%) | 479 (3.7%) | 5 (0.0%) | 3 (0.0%) | 72 (0.6%) |
| 2023-01-01 | 16,644 | 8,255 (49.6%) | 6,192 (37.2%) | 2,007 (12.1%) | 137 (0.8%) | 10 (0.1%) | 43 (0.3%) |
| 2025-01-01 | 17,993 | 5,742 (31.9%) | 3,889 (21.6%) | 5,453 (30.3%) | 1,214 (6.7%) | 1,659 (9.2%) | 36 (0.2%) |
| 2026-01-01 | 18,291 | 1,852 (10.1%) | 1,173 (6.4%) | 11,751 (64.2%) | 1,693 (9.3%) | 1,799 (9.8%) | 23 (0.1%) |
| 2026-09-01 | 18,372 | 13 (0.1%) | 0 (0.0%) | 1,396 (7.6%) | 11,341 (61.7%) | 5,607 (30.5%) | 15 (0.1%) |

원본별로 나누면 다음과 같다(행 안의 비율, %).

| 시점 | 원본 | 대상 | CONFIRMED | R1·R2 | R3 | OPEN · 프로필 | OPEN | NONE |
|---|---|---:|---:|---:|---:|---:|---:|---:|
| 2020-01-01 | API 2023 | 1,705 | 56.5 | 39.4 | 3.5 | 0.0 | 0.0 | 0.5 |
| 2020-01-01 | API 2024 | 1,331 | 54.7 | 44.0 | 0.8 | 0.0 | 0.0 | 0.5 |
| 2020-01-01 | API 2025 | 8,575 | 49.6 | 45.6 | 4.3 | 0.0 | 0.0 | 0.5 |
| 2020-01-01 | 로컬 캐시 | 1,254 | 53.2 | 41.9 | 3.3 | 0.2 | 0.2 | 1.1 |
| 2023-01-01 | API 2023 | 2,271 | 56.9 | 25.7 | 16.6 | 0.7 | 0.0 | 0.1 |
| 2023-01-01 | API 2024 | 1,868 | 56.8 | 39.9 | 3.1 | 0.1 | 0.0 | 0.2 |
| 2023-01-01 | API 2025 | 10,916 | 47.1 | 39.2 | 12.8 | 0.7 | 0.0 | 0.3 |
| 2023-01-01 | 로컬 캐시 | 1,589 | 47.8 | 37.0 | 11.2 | 3.1 | 0.4 | 0.4 |
| 2025-01-01 | API 2023 | 2,367 | 0.0 | 0.0 | 0.0 | 32.0 | 67.8 | 0.2 |
| 2025-01-01 | API 2024 | 2,093 | 60.5 | 19.9 | 19.3 | 0.2 | 0.1 | 0.0 |
| 2025-01-01 | API 2025 | 11,720 | 33.1 | 25.5 | 38.5 | 2.5 | 0.1 | 0.2 |
| 2025-01-01 | 로컬 캐시 | 1,813 | 32.7 | 26.7 | 29.4 | 8.7 | 2.3 | 0.3 |
| 2026-01-01 | API 2023 | 2,367 | 0.0 | 0.0 | 0.0 | 31.6 | 68.2 | 0.2 |
| 2026-01-01 | API 2024 | 2,114 | 15.0 | 5.5 | 73.6 | 0.4 | 5.3 | 0.1 |
| 2026-01-01 | API 2025 | 11,930 | 10.6 | 7.3 | 76.7 | 5.3 | 0.1 | 0.1 |
| 2026-01-01 | 로컬 캐시 | 1,880 | 14.6 | 10.1 | 55.8 | 16.3 | 3.0 | 0.2 |
| 2026-09-01 | API 2023 | 2,367 | 0.0 | 0.0 | 0.0 | 31.6 | 68.2 | 0.2 |
| 2026-09-01 | API 2024 | 2,117 | 0.0 | 0.0 | 0.0 | 12.4 | 87.6 | 0.0 |
| 2026-09-01 | API 2025 | 11,965 | 0.0 | 0.0 | 0.0 | 82.9 | 17.1 | 0.1 |
| 2026-09-01 | 로컬 캐시 | 1,923 | 0.7 | 0.0 | 72.6 | 21.6 | 5.0 | 0.1 |

- **v2와 비교:** v2에서는 2026-01-01의 OPEN 계열이 83.3%였다. R3를 적용하면 19.1%다. 줄어든 몫은 INFERRED (R3)로 옮겨 가 정답 후보가 된다(§12).
- **API 2023 파일 선수:** 관측 시점(2024-07-26) 이후에는 확정·추론이 0%다. 그 시점들에서 약 68%는 프로필이 다른 클럽을 가리켜, 관측 이후 이적했을 가능성이 크다.
- **관측 시점 이후:** 2026-09-01에는 관측 시점이 그 뒤인 로컬 캐시 선수(2026-10-07)만 R3가 남는다.
- **등록 spell 겹침(§8 경고 1):** 2026-01-01 281명, 2026-09-01 295명이다.
- **예정일이 지난 planned로 `OPEN`이 된 선수:** 2026-01-01 134명, 2026-09-01 2,016명이다.

### 6.6 LLM에 넘기는 형태 (예)

`python -m football_kg context 373714 2026-03-01 --lang ko`의 실제 출력이다. 출처 줄의 `start`는 spell을 시작한 이벤트의 출처, `history_response`는 관측 시점을 정한 선수 이력 응답이다.

```text
[KG as-of 2026-03-01] Wuilker Fariñez (player 373714)
뛰는 팀             Internacional de Bogotá (임대)
등록 소속            Caracas FC
구간               2026-01-08 → 종료 기록 없음
등급               INFERRED (R3)
관측               history observed 2026-06-11 (API 2025); 이후는 모름
예정               2026-12-31 → Caracas FC (planned, 미반영)
출처               start: src_c10955af2882cc9b7fd181f0, history_response: src_b4307d5d5351db95fa4b76eb
```

### 6.7 선수의 리그

선수의 t 시점 리그는 저장하지 않고 질의할 때 구한다: `as_of(player, t)`의 클럽 → 그 클럽의 `LeagueSpell`. 리그 답에는 별도 등급 `SNAPSHOT`을 붙인다. 지금 데이터의 LeagueSpell은 관측 시점을 모르는 스냅샷 796개뿐이라(출처에 수집 시각 없음), 과거 시점의 리그 답이 확정처럼 보이면 안 되기 때문이다. 리그 범위를 넓힐지는 §14에서 정한다.

## 7. 출처

- 출처는 `SUPPORTED_BY {fact_name}` 엣지로 붙인다. `dataset/fact_sources.csv`의 1,462,416행이 그대로 엣지가 된다.
- 파생 사실(R1·R2 종료, status spell, `last_confirmed_at`)은 근거 이벤트의 출처를 `SUPPORTED_BY`로 쓰고, `derivation`(R1·R2·status·last_confirmed_loan_out·last_confirmed_loan_return)을 엣지와 노드에 남긴다. 이 파생 엣지 60,940개를 더하면 r0의 `SUPPORTED_BY`는 1,523,356개다.
- R3 답은 저장하지 않는 대신 근거를 함께 돌려준다. spell 시작의 출처와, 관측 시점을 정한 선수 이력 응답(`history_source`의 레코드)이다.

`fact_sources.entity_type`과 KG 노드는 다음과 같이 대응한다.

| `entity_type` | 붙는 노드 | `fact_name` |
|---|---|---|
| `player`, `player_transfer_history_status` | Player | player_name, date_of_birth, birth_date_precision, city_of_birth, country_of_birth, country_of_citizenship, current_club_id, transfer_history_status |
| `player_position` | Player (`HAS_POSITION`의 근거) | position |
| `position` | Position | position_definition |
| `club` | Club · CareerStatus · Club:Unresolved | club_name, club_name_observation, current_domestic_competition_id, identifier_reference |
| `league` | League | league_name, country_id, country_name, competition_type, competition_sub_type, league_id |
| `club_league_current` | LeagueSpell | latest_league |
| `player_current_club` | ProfileClaim | profile_current_club_reference |
| `transfer_event`, `planned_transfer_event` | TransferEvent | transfer_event, event_temporal_status, event_type, normalized_membership_type, classification_rule, raw_transfer_evidence, planned_transfer_observation |
| `player_club_membership` | Spell | membership_start_and_type, membership_end, parent_club_id |

예를 들어 Fariñez의 Rionegro Águilas 임대 spell에는 출처가 다음과 같이 붙는다.

| 사실 | `fact_name` | 근거 이벤트 | 출처 레코드 |
|---|---|---|---|
| 시작일·유형 | membership_start_and_type | 2025-03-10 loan | `dvc_raw_api` · `transfermarkt-api/2025/transfers.json#line=4548&pointer=/response/transfers/3` |
| 원소속 | parent_club_id | 2025-03-10 loan | 위와 같은 레코드 |
| 종료일 | membership_end | 2025-12-31 loan_return | `dvc_raw_api` · `…/transfers.json#line=4548&pointer=/response/transfers/2` |
| 시작 이벤트 자체 | transfer_event | (이벤트 노드) | `existing_transfer_record` · `transfermarkt.duckdb#table=main.transfers&rowid=19787` |

## 8. 무결성 제약

**거부 (삽입 시 실패)**

1. 엔터티 ID(player, club, league)는 유일하다. 버전이 쌓이는 노드는 (논리 ID, version)이 유일하고, 논리 ID마다 현재 버전(`tx_to`가 비어 있음)은 정확히 1개다.
2. Spell의 시작·종료 이벤트는 completed이고 `raw_match_status = matched`이며, 날짜가 `valid_from`·`valid_to`와 같다. planned·unknown 이벤트는 경계가 될 수 없다.
3. Spell과 경계 이벤트는 같은 선수다. `AT`은 시작 이벤트의 `TO`와 같다. 이탈로 닫힌 경우(evidence, R1) 종료 이벤트의 `FROM`은 `AT` 클럽이고, (날짜, 하루 안 순서)가 시작 이벤트보다 뒤다.
4. `valid_from ≤ valid_to ≤ history_observed_at`이다. 둘이 같으면(길이 0) `transit = true`여야 하고, transit spell은 as-of 답이 될 수 없다.
5. 일반 spell의 `AT`은 Club, status spell은 CareerStatus다. loan spell은 `LOANED_FROM ≠ AT`이다.
6. 모든 사실·이벤트 노드는 `SUPPORTED_BY` 1개 이상과 `RECORDED_IN` 1개를 가진다.
7. 선수마다 completed 이벤트 날짜 ≤ `history_observed_at` < planned 예정일이다. 현재 데이터에서 위반은 0건이다.

**경고 (conflict로 기록하고 진행)**

1. 한 선수·한 시점에 등록 spell과 임대 spell은 각각 최대 1개다. 2026-01-01 기준 281명이 위반 후보다.
2. 임대 구간은 원소속 구간 안에 있다.
3. 한 클럽은 한 시점에 LeagueSpell을 1개만 가진다.
4. 최신 열린 spell과 ProfileClaim이 다르면 충돌이다. 지금 4,338명이다.

## 9. 갱신 연산

모든 연산은 Revision 하나로 묶이고, 같은 입력을 다시 넣어도 결과가 같다(멱등). 한 배치 안에서 여러 연산이 같은 노드를 바꿔도 그 노드의 새 버전은 하나다.

| 연산 | 동작 |
|---|---|
| `refresh_history(p, response)` | 선수 이력을 다시 받았을 때 쓴다. 새 응답으로 `history_source`, `history_observed_at`, `history_complete`를 바꾸고, 응답의 이벤트를 `add_event`로 넣은 뒤 아래 planned 매칭 규칙을 적용한다 |
| `add_event(e)` | 같은 이적인지 먼저 확인한다(아래 1번). 새 이벤트면 응답 안의 순서로 `order_in_day`를 정한다. completed면 §5.1 규칙으로 열린 spell을 닫고 새 spell을 연다. planned면 저장만 한다 |
| `confirm_planned(e)` | planned를 completed 새 버전으로 바꾸고 `add_event` 규칙을 다시 적용한다 |
| `void_planned(e)` | planned의 새 버전을 `cancelled`로 표시한다 |
| `reschedule_planned(e)` | 같은 이적의 새 planned로 이전 planned를 대체한다 |
| `add_snapshot(p, club, observed_at)` | 날짜 있는 프로필 관측을 ProfileClaim 새 버전으로 추가하고, 그 날짜의 as-of 답과 다르면 충돌을 기록한다 |
| `supersede(old, new)` | `old.tx_to`와 `new.tx_from`을 배치 시각으로 맞추고 `SUPERSEDES`로 잇는다. 삭제는 없다 |
| `retract(source)` | 출처를 철회한다. 시간 상태와 원본 증거가 그 출처에만 기댄 이벤트는 `unknown`(`raw_match_status = retracted`)이 되고, 그 선수의 spell을 다시 계산한다(시작 이벤트를 잃은 spell은 `void`). 출처 레코드에는 `retracted_in`을 남긴다. 출처가 하나도 남지 않는 이벤트가 생기면 거부 6에 걸려 배치가 적용되지 않는다 |

**planned와 completed의 매칭 규칙**

1. **같은 이적 판정:** `transfer_id`가 같으면 같은 이적이다. r0의 planned 2,562건과 completed 135,120건은 모두 `transfer_id`를 가진다. `transfer_id`가 없는 출처면 (선수, from, to)가 같고 날짜 차이가 N일 이내일 때 같은 이적으로 본다.
2. **확정:** 같은 이적의 completed가 오면 `confirm_planned`다. 날짜가 다르면 completed 날짜를 쓰고 차이를 기록한다. 확정 판정을 취소보다 먼저 해서, 하루 밀린 확정이 "취소 + 새 이벤트"로 처리되지 않게 한다.
3. **취소:** 같은 이적의 completed는 없고, 같은 from에서 다른 곳으로 가는 completed가 이전 `history_observed_at` 이후 날짜로, 예정일 + N일 이전에 오면 `void_planned`(`cancel_reason = contradicted`)다. 관측 시점 이전의 오래된 이탈은 새 planned를 취소하지 않는다.
4. **재일정:** 같은 `transfer_id`의 planned가 다른 날짜로 오면 `reschedule_planned`로 그 이벤트의 새 버전을 만든다. `transfer_id`가 다른 새 planned가 같은 (선수, from, to)로 다른 날짜에 오면, 이전 planned를 `cancel_reason = rescheduled`로 취소하고 새 이벤트를 추가한다. 임대 연장이 여기에 해당한다.
5. **만료:** 재수집으로 `history_observed_at`이 예정일 + N일을 넘었는데 같은 이적의 completed도 새 planned도 없으면 `void_planned`다.

N의 초기값은 31일이다. r1에서 planned 예정일과 실제 확정 날짜의 차이 분포를 보고 조정한다.

예를 들어 다음 수집(r1)에서 Fariñez의 이력을 다시 받았고, 2026-12-31 복귀가 같은 `transfer_id`의 completed로 왔다고 하자(가정). 그러면 버전이 다음과 같이 쌓인다.

| 노드 버전 | 유효 시간 | 기록 시간 | 바뀐 이유 |
|---|---|---|---|
| Internacional 임대 Spell v1 | 2026-01-08 → 열림 | r0 → r1 | final_v3 적재 |
| Internacional 임대 Spell v2 | 2026-01-08 → 2026-12-31 | r1 → | 복귀 이벤트가 completed가 되어 `ENDED_BY {basis: evidence}` 연결 |
| 2026-12-31 복귀 TransferEvent v1 | 2026-12-31 · planned | r0 → r1 | 원본의 upcoming 플래그 |
| 2026-12-31 복귀 TransferEvent v2 | 2026-12-31 · completed | r1 → | 같은 `transfer_id`로 확정 |

- 같은 배치에서 Fariñez의 `history_observed_at`도 2026-06-11에서 새 수집일로 바뀐다.
- v1은 남아 있으므로 r0 시점의 KG로 질의하면 예전 답이 그대로 나온다.
- Internacional을 가리키던 ProfileClaim도 지우지 않고, 날짜 있는 completed 이벤트와 충돌하므로 `stale`로 표시한다.

## 10. 충돌 해소 순서

1. completed가 planned보다 우선한다. planned는 spell을 바꾸지 않고, 예정일이 지나면 답의 등급만 `OPEN`으로 낮춘다.
2. 기록된 종료가 추론 종료보다, 추론 종료가 날짜 없는 ProfileClaim보다 우선한다.
3. 날짜 정밀도는 exact, year, unknown 순이다. (r0의 이벤트 날짜는 모두 exact라 구현하지 않았다.)
4. 같은 사실을 같은 유형의 출처가 말하면 나중에 기록된 관측이 우선한다.
5. 같은 시점을 덮는 등록 spell이 둘 이상이면 근거 등급이 높은 쪽, 같으면 늦게 시작한 쪽(날짜, 하루 안 순서)을 답으로 하고 conflict로 기록한다.
6. 그래도 같으면 둘 다 보존하고 `conflicting`으로 표시해 확정 답에서 뺀다. (r0에서는 선수마다 하루 안 순서까지 겹치는 이벤트가 없어 이런 경우가 생기지 않는다. `Spell.conflicting`은 모두 false다.)

## 11. 데이터 → KG 매핑

| 데이터 | KG 요소 | 개수 | 메모 |
|---|---|---:|---|
| `players.csv` + `player_transfer_history_status.csv` | Player | 40,706 | 이력 수집 상태를 `history_coverage`로 |
| `player_transfer_history_status.raw_source_file` + 원본 이벤트 날짜 | Player `history_source`·`history_observed_at`·`history_complete` | 19,230 | 이력 응답이 있는 선수(§5.2) |
| `positions.csv`, `player_positions.csv` | Position, HAS_POSITION | 13 / 40,238 | 선수당 최대 1개, 시간 없음 |
| 국적 라벨, `leagues.country_name` | Country (파생) | 53 | Turkey·Türkiye 병합, FIFA 순위를 속성으로 |
| `clubs.csv` | Club / CareerStatus / Club:Unresolved | 13,599 / 4 + 1 / 9 | `entity_kind`로 나눔. Unknown(75)은 행선지 미상. Club에 `team_level` 파생 |
| `leagues.csv` | League | 32 | 31개는 1부 리그로 표시돼 있고, COL1은 리그 유형 정보 없이 식별 정보만 있다 |
| `club_league_current.csv` | LeagueSpell | 796 | `valid_from`·`observed_at` 모두 미상. 리그 답은 `SNAPSHOT` |
| `player_current_clubs.csv` | ProfileClaim | 40,706 | Club 40,679 · Unresolved 21 · CareerStatus 6 |
| `transfer_events.csv` (+ planned) | TransferEvent | 137,776 | completed 135,120 · planned 2,562 · unknown 94. planned 표는 `temporal_status`로 흡수 |
| `player_club_memberships.csv` | Spell | 110,621 | 기록된 종료 45,403 · 추론 종료 45,429 · 열림 19,789 (transit 29) |
| 파생: R1·R2 | Spell `end_basis = inferred` | 45,429 | R1 45,422 · R2 7. 원본 `end_date`는 유지 |
| 파생: 특수 ID로 가는 completed 이벤트 | Spell `type = status` | 2,758 | 상태 2,553 · 행선지 미상 205. 추론 종료 2,124 · 열림 634 |
| 파생: R3 | (저장 안 함) | — | 질의 시 계산. 적용 대상 선수 17,972 |
| `source_documents.csv` | Source | 365,128 | 어디에도 쓰이지 않는 6,947건은 생략 가능 |
| `fact_sources.csv` | SUPPORTED_BY {fact_name} | 1,462,416 | Fact 노드 대신 엣지 |
| `build_manifest.json`, `database_statistics.json` | Revision r0 | 1 | 입력 해시, 검증용 수치 |

## 12. 평가 프로토콜

**정답 셋**

- 정답은 `CONFIRMED`와 `INFERRED`(R1·R2·R3) 답에서 뽑고, derivation별로 따로 보고한다.
- 빼는 것은 세 가지다.
  - `history_complete = false`인 1,210명(spell이 있는 선수 396명): 빠진 이벤트 때문에 확정 구간도 틀릴 수 있다.
  - transit spell.
  - 유스·리저브 spell(`team_level`): 질문에서 빼거나 별도 층으로 둔다.
- 질문 시점은 선수마다 `t < H_p − lag`에서만 뽑는다.

**층 나누기**

- **최근 이적 층:** spell 시작이 모델의 지식 시점 이후인 질문이다. API 2023 파일 선수 2,384명은 2024년 7월 이후로 질문할 수 없으므로 이 층에 들어가지 못한다. 최근 이적 층은 API 2024·2025 파일과 로컬 캐시 선수로만 구성된다는 점을 결과에 밝힌다.

| 시작 시점 기준 | 클럽 spell | 선수 | API 2025 | API 2024 | 로컬 캐시 |
|---|---:|---:|---:|---:|---:|
| 2025-07-01 이후 | 8,976 | 7,922 | 5,926 | 1,396 | 600 |
| 2026-01-01 이후 | 3,279 | 3,212 | 2,450 | 390 | 372 |

(이력이 완전하고 시작일이 H_p − 14일보다 앞선 spell만 셌다.)

- **평가 모집단:** 활동 중인 선수로 정의한다. 기준은 "자기 관측 시점 H_p의 as-of 답이 클럽 spell(등록 또는 임대)이고, 그때 나이가 A_max 이하"다. ("H_p에 클럽 spell이 열려 있음"으로 세면 16,521명이다.)
  - 16,517명이 해당하고, 그중 이력이 완전한 선수는 16,139명이다.
  - 원본별로는 API 2025 11,068명, API 2023 1,851명, API 2024 1,845명, 로컬 캐시 1,753명이다.
  - 인기도와 이력 보유율의 관계도 이 모집단 안에서 잰다. (인기도 지표가 정해진 뒤에 한다, §14.)

**정답 검증 (표본)**

- KG에서 뽑은 정답의 오류는 KG와 공유되므로 KG 증강에 유리하게 작용한다. 그래서 등급·derivation별 정확도를 독립 출처로 표본 검증한다. (r0에서는 표본만 뽑았다. 검증 칸이 빈 `build/r0/eval/verification_sample.csv`가 있고, 검증과 그 결과를 쓰는 아래 결정은 아직 하지 않았다.)
- **표본:** CONFIRMED, INFERRED (R1·R2), INFERRED (R3), OPEN · 프로필 일치, OPEN에서 50건씩, 모두 250건을 뽑는다. R2는 8건뿐이라 R1과 합친다. 정확도가 90% 근처면 95% 신뢰구간은 50건에서 약 ±8%p, 100건에서 약 ±6%p다.
- **출처:**
  - 클럽 공식 발표와 리그 공식 등록·경기 기록을 우선한다.
  - Transfermarkt를 인용하는 위키백과·Wikidata는 독립 출처로 치지 않는다.
  - upstream `appearances`도 같은 Transfermarkt 계열이라 보조 확인용으로만 쓴다.
- **집계:** "확인 불가"는 따로 세고, 표본은 인기도 층별로 뽑는다. (인기도 지표가 없어 r0 표본은 층을 나누지 않았다, §14.)
- **더 많이 뽑을 표본:** 다음 세 가지는 오류 위험이 커서 표본을 늘린다.
  - 유형 없는 이탈로 닫힌 R1 spell.
  - 매칭되지 않은 임대 복귀 276건.
  - 고령 선수의 긴 R3 구간.
- **결과의 쓰임:** 이 표 하나로 세 가지를 정한다. 어떤 등급까지 정답 셋에 넣을지, 정답 셋의 오류율, 그리고 등급이 실제 정확도와 맞는지(calibration)다.

**채점과 재현**

- 채점은 이름 문자열이 아니라 클럽 ID와 `aliases`로 한다.
- 질문은 날짜로 묻는다. "2019/20 시즌"처럼 시즌으로 물으면 단년제 리그(K리그, J리그, MLS, 브라질, 북유럽 등) 때문에 뜻이 모호해진다.
- 평가에 쓴 KG 상태는 Revision으로 고정한다. 갱신 효과는 r0와 r1 결과를 비교해 보인다. (r1 데이터는 아직 없다, §14.)

## 13. 데이터 공백

- **과거 리그를 모른다.** LeagueSpell은 관측 시점을 모르는 796개 클럽뿐이고, spell의 52%는 현재 리그 정보도 없는 클럽에 있다. "t 시점에 어느 리그에서 뛰었나"에 답하려면 시즌별 리그 소속 데이터가 따로 필요하다(§14).
- **선수의 55%(22,334명)는 spell이 없다.** 이력 미수집 21,476명, 빈 이력 40명, completed 이적이 없는 이력 818명이다. spell 보유율은 리그보다 활동 시기에 좌우된다. 1980년대생은 9.9%, 1990년대생은 42.3%, 2000년대생은 62.4%가 spell을 가진다. 5대 리그 소속(37.7%)이 나머지 리그(47.2%)보다 오히려 낮다.
- **관측 시점이 선수마다 다르다(§5.2).** API 2023 파일 선수 2,384명은 2024-07-26 이후를 모른다. 날짜가 지난 planned 2,132건 중 1,997건은 2026년 5~9월(API 2024·2025 파일 수집 이후의 2026 여름 이적)이고, 135건은 2024년 7월 스냅샷 당시의 예정이다. 3~5월에 completed가 적은 것은 이적시장이 닫혀 있었기 때문이다.
- **ProfileClaim은 날짜가 없고 일부는 낡았다.** 마지막 이벤트가 은퇴·무소속·커리어 중단인데 프로필은 클럽을 가리키는 선수가 625명이다.
- **long-tail 기준이 없다.** 인기도 지표를 외부에서 붙여야 한다(§14).
- **upstream 업데이트가 멈췄다.** transfermarkt-datasets는 2026-07-06까지의 데이터로 업데이트가 중단됐다. 따라서 r1은 다른 경로로 채워야 한다(§14).

## 14. 결정 대기 (권장안)

1. **리그 범위**
   - 먼저 원 프로젝트의 로컬 duckdb에 `games`(또는 `club_games`)가 있고 32개 리그를 덮는지 확인한다.
   - 덮으면 "시즌 s에 리그 L 경기를 치른 클럽"으로 1부 리그의 시즌 단위 LeagueSpell을 만드는 쪽을 권한다. 승강도 자연스럽게 반영된다. "t 시점에 어느 리그에서 뛰었나"는 선수–클럽과 클럽–리그라는 두 시간 관계를 겹쳐야 풀리는 2-hop 질문이라, 문서 RAG 대비 KG의 장점을 보이기 좋다. 2026-07-06 이후 시즌은 `SNAPSHOT`으로 둔다.
   - 덮지 않으면 Goal을 "선수–클럽 이력 + 현재 리그(`SNAPSHOT`)"로 좁히고, 과거 리그 질문은 평가에서 뺀다.
2. **long-tail 지표**
   - 위키백과 조회수를 주 지표로 쓴다(PopQA, Mallen et al., 2023과 같은 방식). 선수는 Wikidata의 Transfermarkt 선수 ID 속성 P2446으로 `player_id`와 연결한다.
   - 문서가 없는 선수는 별도 구간으로 둔다.
   - 시장가치는 강건성 확인용이다. `market_value_in_eur`를 쓰고, 최고값은 `player_valuations`로 계산한다(upstream에 최고값 컬럼이 있는지는 확인되지 않았다).
3. **r1 출처**
   - 로컬 캐시를 2026-10-07~08에 받은 경로가 원 프로젝트에 있으면, 그걸로 API 파일 선수 16,485명(특히 2024년 7월에 멈춘 2,384명)의 이력을 다시 받는 쪽을 권한다.
   - `transfer_id`와 클럽 ID가 그대로라 이름 연결이 필요 없고, 관측 시점 문제도 줄어든다.
   - r0→r1 비교 자체가 "2026 여름 이적"이라는 갱신 실험이 된다. planned 1,997건의 확정·취소로 갱신 연산(§9)도 그대로 시험된다.
   - 뉴스·위키 기반 갱신은 다음 확장으로 두고, 그때 날짜 허용 오차를 둔 중복 판정과 클럽 이름→ID 연결 규칙을 더한다.

## 15. 변경 이력

- **v1 (2026-10-11):** 초안.
- **v2 (2026-10-11):** 검토를 반영했다.
  - Unknown(75)을 경력 상태에서 빼고 행선지 미상(답 `NONE`)으로 바꿨다.
  - LeagueSpell의 관측일을 "미상"으로 고쳤다.
  - as-of 선택 순서를 정했다.
  - 예정일이 지난 planned 변경이 있으면 답을 `OPEN`으로 낮추는 규칙을 더했다.
  - 추론 규칙 R2를 더했다.
  - 버전과 ID 제약을 맞췄고, 파생 사실의 출처 규칙과 이벤트 정합성 제약을 추가했다.
- **v3 (2026-10-11):** 리뷰 피드백을 반영했다.
  - **선수별 관측 시점:** `history_source`, `history_observed_at`, `history_complete`를 추가했다. v2의 "데이터 최신 경계 2026년 2월 무렵"은 틀린 서술이라 원본별 관측 시점 표(§5.2)로 바꿨다.
  - **R3:** 관측 창 안 지속 규칙을 추가했다(lag 14일, 은퇴 가드 A_max 34세). as-of의 시점 상한을 전역 `source_as_of`에서 선수별 H_p로 바꿨다.
  - **분포 재계산:** §6.5를 다시 계산하고 원본별 분포를 추가했다.
  - **등록 확인 근거:** `last_confirmed_at`에 매칭된 임대 복귀를 더했다.
  - **같은 날 이벤트:** 하루 안 순서 키와 길이 0 transit spell 규칙을 정했다. 같은 날 이탈 95건 중 실제로 늦게 닫히던 것은 같은 날 입단 후 바로 떠난 29건이었고, 나머지는 임대 송출이거나 그날 입단보다 먼저 일어난 이탈이었다. 이 반영으로 R1이 45,396건에서 45,422건이 됐다.
  - **리그 답:** `SNAPSHOT` 등급을 붙였다.
  - **planned 매칭:** `transfer_id` 우선의 확정·취소·재일정·만료 규칙과 `refresh_history` 연산을 추가했다.
  - **신설:** `Club.team_level`(패턴 확정 전), 평가 프로토콜(§12), 결정 대기(§14)를 새로 두었다.
  - **확인만 한 것:** v2에서 evidence와 inferred가 둘 다 45,403건이던 것은 우연이었다. 교집합은 0건이었다.
- **v3 정정 (2026-10-14):** 구조와 수치는 그대로 두고, 구현(r0)과 다르게 적힌 문장을 고쳤다.
  - 첫머리에 구현 문서(docs/model.md, docs/updates.md)와 검증 보고서 안내, 구현에서 바뀐 표현 목록을 더했다.
  - §6.1: 관측 시점 근처에서도 `last_confirmed_at`으로 `CONFIRMED`가 될 수 있음을 밝히고, planned 이탈을 보는 구간을 (spell 시작일, t]로 적었다.
  - §6.4: 2019-06-01 행의 등록 확인 근거를 2021-06-30 임대 복귀로 고쳤다.
  - §6.5·§13: "이력이 없는 22,334명"을 "spell이 없는 22,334명"으로 고치고 내역을 적었다. §13의 연대별·리그별 비율도 spell 보유율이다.
  - §7: 파생 사실의 출처에 `last_confirmed_at`을 넣고 `SUPPORTED_BY` 총수를 적었다.
  - §9: `add_snapshot`·`retract`의 동작과 취소·재일정 규칙을 구현에 맞췄고, 한 배치 안에서 노드당 새 버전은 하나라는 규칙을 더했다.
  - §12: 평가 모집단의 기준을 구현대로 "H_p의 as-of 답"으로 고쳤다.
  - §3·§10·§12: 아직 하지 않은 일(team_level 표본 점검, 날짜 정밀도·동률 규칙, 인기도 층화, 독립 검증, r0–r1 비교)을 표시했다. §3의 team_level 비율과 §11의 리그 설명을 구현값으로 고쳤다.
  - §4: `SUPPORTED_BY`가 없는 라벨(Country·Revision)을 적었다. §5.3: 엣지 버전, void tombstone, 엔터티의 제자리 갱신을 적었다. §6.6: 예시를 실제 출력으로 바꿨다.
