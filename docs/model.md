# The model as built (revision r0)

This document describes the graph produced by `python -m football_kg build`. The design is [kg-schema.md](../kg-schema.md) (v3, in Korean). The section "Clarifications of kg-schema v3" lists every point where the build had to make that design more precise.

```
Player -HAS_CLAIM-> ProfileClaim -POINTS_TO-> Club | UnresolvedClub | CareerStatus        (undated profile snapshot)
TransferEvent -FOR_PLAYER-> Player ; TransferEvent -FROM / TO-> Club | CareerStatus        (one change at one date)
Player -HAS_SPELL-> Spell -AT-> Club | CareerStatus ; Spell -LOANED_FROM-> Club             (an interval)
Spell -STARTED_BY-> TransferEvent ; Spell -ENDED_BY {basis, derivation}-> TransferEvent
Club -HAS_SPELL-> LeagueSpell -IN-> League -IN_COUNTRY-> Country ; Player -CITIZEN_OF-> Country ; Player -HAS_POSITION-> Position
(Spell | TransferEvent | ProfileClaim | LeagueSpell | entities) -SUPPORTED_BY {fact_name, derivation}-> Source
(versioned nodes) -RECORDED_IN-> Revision ; new version -SUPERSEDES-> old version
```

There is no `CURRENTLY_PLAYS_FOR` edge and no Fact node. "Where did p play at t" is computed from spells by the as-of rules described below.

## Nodes

The key column is unique within its label. `*` marks versioned labels: every row also carries `version`, `uid = <id>@v<version>`, `tx_from`, `tx_to` (null means current) and `revision_id`.

| Label | Key | r0 | Properties |
|---|---|---:|---|
| Player | `player_id` | 40,706 | `name`, `birth_date` (as given: `YYYY-MM-DD` or `YYYY`), `birth_date_precision` (exact/year/unknown), `birth_date_floor` (`YYYY-01-01` for year precision, used for the age guard), `birth_city`, `birth_country`, `citizenship_label`, `history_coverage` (valid_nonempty/valid_empty/unavailable), `history_source`, `history_source_type`, `history_source_label` (API 2023/2024/2025, local cache), `history_observed_at` (H_p), `history_observed_at_basis`, `history_complete`, `history_record_source_id`, raw/curated event counts, `source_url` |
| Club | `club_id` | 13,599 | `name`, `name_scope`, `metadata_status`, `aliases[]` (observed names, verbatim; may contain HTML entities), `source_url`, `team_level` (first_team/reserve/youth, provisional), `team_level_token`, `team_level_basis` |
| UnresolvedClub | `club_id` | 9 | `metadata_status`. These IDs have no name; only 21 profile claims point to them. |
| CareerStatus | `club_id` | 5 | `code`: 123 retired · 515 without_club · 2077 ban · 2113 career_break · 75 unknown_destination. `is_career_status` is false for 75. |
| League | `league_id` | 32 | `name`, `country_id`, `country_name`, `competition_type`, `competition_sub_type`, `metadata_status`, `source_url` |
| Country | `name` | 53 | `aliases[]`, `fifa_name`, `fifa_rank`, `is_top50`, `players_in_release`. The key is the Transfermarkt label. Turkey and Türkiye are merged into Türkiye. Czech Republic, Romania and Saudi Arabia appear only as league countries. |
| Position | `position_id` | 13 | `position`, `sub_position` |
| Source | `source_id` | 365,128 | `source_type`, `url`, `raw_locator` (an opaque upstream locator), `retrieved_at`, `content_sha256`, `extraction_method`, `referenced` (false for the 6,947 records nothing uses) |
| Revision | `revision_id` | 1 | `ordinal`, `recorded_at`, `source_as_of`, `origin`, `operation`, `parent_revision_id`, `input_sha256`, `input_row_counts`, `code_version`, `parameters` |
| ProfileClaim\* | `current_relation_id` | 40,706 | `player_id`, `target_id`, `target_kind`, `relation_kind`, `temporal_scope`, `observed_at` (null: undated), `observed_before` (2026-10-08), `stale` (null in r0), `source_url`, `source_id` |
| TransferEvent\* | `transfer_event_id` | 137,776 | All 39 release columns, plus `date`, `season`, `temporal_status` (completed/planned/unknown, plus cancelled after updates), `event_type`, `normalized_membership_type`, `fee_raw`, `fee_curated`, `fee_eur`, `transfer_id`, raw evidence (`raw_*`), `from_kind`/`to_kind`, `in_planned_table`, and the same-day order fields `pointer_index`, `order_in_day`, `seq` |
| Spell\* | `spell_id` | 113,379 | `player_id`, `at_id`, `at_kind`, `type` (permanent/loan/unknown/status), `status_code`, `loaned_from_id`, `valid_from`, `valid_to` (the effective end), `source_end_date` (the dataset value, never overwritten), `end_basis` (evidence/inferred/open), `end_derivation` (R1/R2), `evidence_end_method`, `start_event_id`, `end_event_id`, `start_seq`/`end_seq`, `last_confirmed_at` (+ event and basis), `transit`, `origin` (membership/status), dataset `membership_derivation_method` and `confidence_status`, `record_status` |
| LeagueSpell\* | `league_spell_id` (`club_id:league_id`) | 796 | `club_id`, `league_id`, `valid_from`/`valid_to`/`observed_at` (all unknown), `basis` = snapshot |

Renamed release columns: `transfer_date`→`date`, `transfer_date_precision`→`date_precision`, `transfer_season`→`season`, `event_temporal_status`→`temporal_status`, `original_fee`→`fee_raw`, `original_fee_json`→`fee_json`, `original_transfer_fee`→`fee_curated` (parsed into `fee_eur`; 0 does **not** mean free), and `membership_type`→`upstream_membership_type`. The last one is always "unknown"; use `normalized_membership_type` instead.

## Relationships

| Type | From → to | r0 | Properties |
|---|---|---:|---|
| HAS_CLAIM / POINTS_TO | Player → ProfileClaim → Club, UnresolvedClub or CareerStatus | 40,706 each | |
| FOR_PLAYER / FROM / TO | TransferEvent → Player / Club or CareerStatus | 137,776 each | |
| HAS_SPELL | Player → Spell, and Club → LeagueSpell | 113,379 + 796 | The same type is used for both; filter by label. |
| AT / LOANED_FROM | Spell → Club or CareerStatus / Club | 113,379 / 23,544 | |
| STARTED_BY / ENDED_BY | Spell → TransferEvent | 113,379 / 92,956 | ENDED_BY carries `basis` (evidence/inferred) and `derivation` (the dataset rule, or R1/R2) |
| IN / IN_COUNTRY / CITIZEN_OF / HAS_POSITION | | 796 / 32 / 40,706 / 40,238 | |
| SUPPORTED_BY | any fact node → Source | 1,523,356 | `fact_name`, `derivation`, `orig_entity_type`, `orig_entity_id`, `provenance_status` |
| RECORDED_IN / SUPERSEDES | version → Revision / older version | 292,657 / 0 | |

Every relationship also carries `tx_from` and `tx_to`.

## Time model

- **Validity.** Intervals are half-open: `[valid_from, valid_to)`. On a transfer day the new spell covers the date.
- **Same-day order.** Events of one player on one date are ordered by the raw API pointer. The list is newest first, so a larger index means an earlier event. `order_in_day` is 1 for the earliest event, and `seq` is `days_since_1970 * 1000 + order_in_day`.
- **Spells.**
  - The 110,621 dataset memberships are loaded as given.
  - Every completed event whose destination is a special ID adds a status spell (2,758). This includes three loan returns and four loans to 75 "Unknown".
- **Ends.** `end_basis` takes one of three values (counts are memberships / status spells):

  | `end_basis` | Meaning | Count |
  |---|---|---:|
  | evidence | the dataset end (a loan return, or a free/without-club/retired/career-break departure) | 45,403 / 0 |
  | inferred | R1: the earliest later completed departure from the same club (registration spells ignore loan-outs). R2: an open loan or status spell ends when the player's next spell starts. When both apply, the earlier one wins; on a tie, R1. | 45,429 / 2,124 |
  | open | no end | 19,789 / 634 |

- **Transit spells.** A spell that ends on the day it starts is a transit spell (29 + 1). It links the event chain but is never an answer.
- **`last_confirmed_at`.** Set on registration spells without an evidence end: the latest loan-out from the club, or loan return to it that the dataset matched to a loan, after the spell's start and before its inferred end (10,629 spells: 3,473 open, 7,156 with an R1/R2 end).
- **Observation horizon H_p.**
  - API files use the last completed date in the file: 2024-07-26, 2026-02-27 and 2026-06-11.
  - The local cache uses each player's `retrieved_at` date in UTC (2026-10-07/08; one player defaults to 2026-10-07).
  - `history_complete` means `valid_nonempty` and every raw event was curated (17,980 players).

## As-of rules (kg-schema §6)

1. Candidates are the non-transit spells with `valid_from ≤ t` that either have no end or have `t < valid_to`.
2. A loan spell wins if there is one; its `LOANED_FROM` is the registered club. Otherwise the choice is among registration and status spells.
3. Rank by CONFIRMED (3) > INFERRED (R1, R2 and R3 alike, 2) > OPEN (1), then by the later `start_seq`.
4. Grades:
   - CONFIRMED: an evidence end, or `last_confirmed_at ≥ t`.
   - INFERRED R1/R2: an inferred end exists.
   - INFERRED R3: the spell is open, `history_complete`, the birth date is known, `t < H_p − lag` and `t < birth_date_floor + A_max`.
   - OPEN: anything else.
5. An OPEN answer stays OPEN if a planned departure from that club has a date in `(valid_from, t]`. Otherwise it becomes OPEN_PROFILE if the profile claim names the same club.
6. Status spells answer with the status. `unknown_destination` answers NONE.

The defaults are `lag = 14` days and `A_max = 34` years. The §6.3 sensitivity figures are reproduced for lag 0/14/28 and A_max 34/36/38.

## Versioning and revisions

- **Versions.** A changed versioned node gets version v+1. The old version's `tx_to` is set to the batch time, and SUPERSEDES links the new version to the old one. Versions are never deleted: a spell whose start event is withdrawn gets a tombstone version with `record_status = void`. One batch gives a node at most one new version.
- **Edges.** Every edge has its own `[tx_from, tx_to)`. Edges into a superseded version are closed and re-created against the new version. The KG at revision r is every node and edge whose interval contains r.
- **Entities.** Player, Club and the other entity labels are updated in place in each new revision directory. An UnresolvedClub whose ID becomes a known club is replaced by a Club node. Changes are logged in `tables/revision_log.parquet`, except new Source records.
- **Observation fields.** The as-of grade at revision r needs that revision's H_p, so the per-player observation fields are also versioned in `tables/player_observation.parquet`.
- **Time travel.** `KG.load(path, revision="r0")` returns the versioned part of an earlier state: events, spells, profile claims, edges and observation fields as they were at r0, which is everything the as-of rules use. Entity labels show their current values. Each revision's own directory keeps its exact state. See [updates.md](updates.md).

## Provenance

- **Dataset facts.** Each of the 1,462,416 `fact_sources` rows is one SUPPORTED_BY edge, with `fact_name` and the original `entity_type`/`entity_id` kept. Relationship facts attach to the reified node:

  | `entity_type` | Node |
  |---|---|
  | player_club_membership | Spell |
  | club_league_current | LeagueSpell |
  | player_current_club | ProfileClaim |
  | player_position | Player |

- **Derived facts.** These cite the raw record of the event they rest on, with `derivation` set:

  | Fact | `fact_name` | `derivation` | Count |
  |---|---|---|---:|
  | status-spell start | membership_start_and_type | status | 2,758 |
  | inferred end | membership_end | R1 / R2 | 47,553 |
  | `last_confirmed_at` | last_confirmed_at | last_confirmed_loan_out / last_confirmed_loan_return | 10,629 |

- **R3 answers** also return the player's history response record (`history_record_source_id`).

## Constraints and conflicts (kg-schema §8)

- **Reject rules 1–7** must have zero violations, or the build stops. They cover:
  - unique ids and versions
  - boundary events that are completed and matched
  - the same player; AT equals the start event's TO; departures leave the AT club
  - `valid_from ≤ valid_to ≤ H_p`, and transit exactly when the length is zero
  - the node kinds AT and LOANED_FROM may point to
  - SUPPORTED_BY and RECORDED_IN present
  - completed dates ≤ H_p < planned dates
- **Warnings** go to `tables/conflicts.parquet` (6,962 rows in r0):

  | Warning | Rows | Pairs / players |
  |---|---:|---|
  | W1 overlapping registration spells | 1,160 | 358 players |
  | W1 overlapping loans | 32 | |
  | W2 loan not inside a registration at the parent club | 1,327 | |
  | W4 latest open spell ≠ profile claim | 4,443 | |

  Point counts of registration overlaps: 281 players at 2026-01-01 and 295 at 2026-09-01.

## Clarifications of kg-schema v3

1. **Labels.** Unresolved club IDs get their own label, `UnresolvedClub`, so `(:Club)` matches only the 13,599 real clubs.
2. **Spell properties.** The design's `derivation` property is split into `origin` (membership/status), `end_derivation` (R1/R2) and `evidence_end_method`.
3. **Observation fields** are versioned in a side table. The Player node shows only the current values.
4. **Profile claims.** `ProfileClaim.stale` stays null in r0. A claim becomes stale only when a completed departure from the claimed club dated after 2026-10-08 arrives (§9).
5. **Unreferenced sources.** All 365,128 sources are loaded; the 6,947 that nothing references have `referenced = false`.
6. **§12 active population.** It uses age **<** A_max. The kg-schema text says "이하" (≤), but the published 16,517 / 16,139 use "<".
7. **§13.** The "625 players whose last event is a status but whose profile names a club" is **626** under the same-day order. Player 832896 has a free transfer and then a without-club event on 2024-07-01.
8. **§8 warning 4.** The legacy definition reproduces 4,338. It uses null-`end_date` memberships with pandas' unstable sort; a stable sort gives 4,336. The v3 definition, recorded in the conflict log, gives 4,443.
9. **§5.2.** "Players with events" counts players with a completed or planned event: 1,924 in the local cache. Two players there have only events of unknown temporal status.
10. **Update batches.** Planned events still listed as upcoming after their date stay planned and are reported as `planned_overdue`. Rule 7's "H_p < planned date" is a reject rule for r0 only.

## Known limitations

- **League history.** The only league data is the latest club→league snapshot, so league answers are `SNAPSHOT` and there is no league history (kg-schema §13/§14).
- **No r1 data.** The update operations are implemented and tested with synthetic batches only.
- **Provisional and placeholder parts.** `team_level` is a provisional pattern (5,296 club IDs, 26.3% of memberships), and the popularity stratum is a placeholder.
