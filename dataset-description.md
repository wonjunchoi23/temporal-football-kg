# Dataset Description: top50_dataset

## 1. Overall Structure

- **Target:** `top50_dataset/` (unpacked in this repository as `dataset/`)
- **Internal build version:** `final_v3` / **Original build path:** `data/top50/final_v3/`
- **Release date:** 2026-10-08 / **Country selection snapshot:** 2026-10-07
- **Purpose:** A reference dataset for linking player profiles and club affiliation histories into a knowledge graph.
- **Scope:** Players whose primary nationality is one of the top 50 countries in a user-provided snapshot of the FIFA Men's World Ranking. No league restriction is applied, and selection is not expanded to secondary nationalities.
- **Effective population:** Every player comes from the upstream curated Transfermarkt player table (`source_type=curated_player`), so the population is bounded by that table's coverage. 40,195 players (98.7%) have a profile club linked to one of the 32 leagues in `leagues.csv`. Treat the data as top-50-nationality players present in that source, not as a census of all such players.
- **Country labels:** The 50 FIFA countries map to 51 `country_of_citizenship` labels, because Türkiye appears both as `Turkey` (1,392 players) and `Türkiye` (907). Five other labels differ from the FIFA names: USA → `United States`, IR Iran → `Iran`, Côte d'Ivoire → `Cote d'Ivoire`, Korea Republic → `Korea, South`, Congo DR → `DR Congo`. The full mapping is in `database_statistics.json` → `country_counts`.

**Key Figures**

- Players: 40,706 / Positions: 13 / Player–position links: 40,238
- Club identifiers: 13,613 / Leagues: 32 / Current club–league links: 796
- Transfer events: 137,776 — completed 135,120 / planned 2,562 / unknown 94
- Historical memberships: 110,621 — permanent 26,996 / loan 23,547 / unknown 60,078
- Current club references: 40,706 / Transfer history coverage statuses: 40,706
- Source records: 365,128 / Fact–source links: 1,462,416
- Data dictionary: 131 columns / CSVs covered by checksums: 14

**File Composition**

- 13 data CSVs + 6 documentation/distribution files.
- The distribution file is `top50_dataset.zip`. A ZIP hash file may optionally be delivered alongside it.
- Read `README.md`, `schema.md`, and `data_dictionary.csv` first, then load the data.

**Link Structure**

```
Player → Position
Player → Current club → Current league
Player → Historical memberships / Transfer events
Each player / relationship / event → Source
```

Planned transfers are kept separate from completed history.

**Reading Notes**

- The figures above were aggregated directly from the CSVs in the distribution folder and match `database_statistics.json`.
- PK is the row-identifying key; FK is a key referencing another table. All identifiers are kept as TEXT. Empty CSV cells are null. League IDs are alphanumeric (`GB1`, `L1`), and numeric-looking player and club IDs must not be cast to integers.
- Read every column as text and convert deliberately. Value formats that need care:
  - Dates are `YYYY-MM-DD`, except `players.date_of_birth`, which is a bare `YYYY` when `birth_date_precision=year`.
  - Booleans are mixed: `raw_future_transfer` uses `0`/`1`, while `raw_upcoming`, `raw_from_is_special` and `raw_to_is_special` use `true`/`false`.
  - `clubs.observed_names` (an array) and `transfer_events.original_fee_json` hold JSON.
  - `transfer_season` is a two-digit `YY/YY` label, so string order is not chronological (`00/01` sorts before `99/00`).
- The coverage of collected transfer histories differs from the number of players. Do not assume full career coverage. Only 18,376 of the 40,706 players have any transfer events, and each player's history was captured at a different time, from July 2024 to October 2026 depending on its raw source (see 5.2).
- *Membership* refers to a period during which a player was attached to a club (i.e., a spell, either permanent or on loan). The current club is handled separately in `player_current_clubs.csv`.

## 2. Players / Positions

### 2.1 `players.csv`

- **Role & size:** Basic player profiles, 40,706 rows. PK: `player_id`.
- **Keys & fields:** `player_name`, `date_of_birth`, `birth_date_precision`, `city_of_birth`, `country_of_birth`, `country_of_citizenship`, `current_club_id`, `source_url`.
- **Provenance / KG:** Create `Player` nodes. `current_club_id` → `clubs.club_id`.
- **Caveats:** `country_of_citizenship` is the primary nationality used for selection. Do not merge players by name alone. Preserve missing birth dates and date precision. `players.csv` itself has no position column.
- **Missing values & precision:** `date_of_birth` is `YYYY-MM-DD` for 40,386 players (`exact`), a bare `YYYY` for 273 (`year`) and null for 47 (`unknown`). `city_of_birth` is null for 4,049 players and `country_of_birth` for 4,092.
- **Name collisions:** 833 names are shared by 2,017 players (up to 14 players named "Paulinho"), which is why name-only merging is unsafe.
- **Current club:** `current_club_id` points to an actual club for 40,679 players, an unresolved identifier for 21 and a special status for 6 (see 5.1).

### 2.2 `positions.csv`

- **Role & size:** Observed position taxonomy, 13 rows. PK: `position_id`.
- **Keys & fields:** Combination of `position` and `sub_position`, e.g., `Defender / Right-Back`.
- **Values:** `Goalkeeper` / Goalkeeper; `Defender` / Centre-Back, Left-Back, Right-Back; `Midfield` / Defensive Midfield, Central Midfield, Attacking Midfield, Left Midfield, Right Midfield; `Attack` / Centre-Forward, Left Winger, Right Winger, Second Striker. `position_id` is a deterministic hash (`pos_…`), so join on it rather than on the labels.
- **Provenance / KG:** Create `Position` nodes shared across players.
- **Caveats:** Values observed on player profiles. They do not represent season-by-season position history or tactical roles.

### 2.3 `player_positions.csv`

- **Role & size:** Player–position links, 40,238 rows. PK: (`player_id`, `position_id`).
- **Links:** `player_id` → `players` / `position_id` → `positions`. `source_url` is preserved alongside.
- **Provenance / KG:** Create `HAS_POSITION` relationships. The `entity_id` in `fact_sources` is `player_id:position_id`.
- **Caveats:** 468 players have no position link. Do not assign positions arbitrarily. Each player has at most one position link (40,238 rows for 40,238 distinct players), so this is a zero-or-one relationship in this release. `source_url` is identical to `players.source_url`.

**Link Example**

```
Player → HAS_POSITION → Position
```

Basic player attributes are kept separate from relationships: use `players.csv` for profiles and `player_positions.csv` for position links.

## 3. Teams / Leagues

### 3.1 `clubs.csv`

- **Role & size:** Registry of referenced club identifiers, 13,613 rows. PK: `club_id`.
- **Composition:** `entity_kind` = `club` 13,599 / `special_status` 5 / `unresolved_identifier` 9.
- **Metadata status:**
  - `curated` 796: actual clubs with a current league and an absolute `source_url`.
  - `transfer_label_only` 12,808: 12,803 clubs plus the 5 special statuses. `source_url` is a relative Transfermarkt href such as `/saarbrucken/transfers/verein/1/saison_id/2026`.
  - `identifier_only` 9: the unresolved identifiers, with no name, URL or observed names.
- **Special statuses:** `75` Unknown, `123` Retired, `515` Without Club, `2077` Ban, `2113` Career break. They appear as `from_club_id` in 2,444 transfer events, as `to_club_id` in 3,326, and in 6 current-club references. They never appear in memberships.
- **Unresolved identifiers:** `2225`, `4023`, `18102`, `22260`, `22838`, `23806`, `43314`, `86863`, `90371`. They appear only as the profile club of 21 players (see 5.1), never in transfer events or memberships.
- **Keys & fields:** `club_name`, `current_domestic_competition_id`, `metadata_status`, `club_name_scope`, `entity_kind`, `observed_names`, etc.
- **Provenance / KG:** Create `Club` nodes for actual clubs. Keep special statuses and unresolved identifiers separately distinguished.
- **Caveats:** `transfer_label_only` names are labels observed in transfer records. They do not establish the current official club name or past league.
- **Names and aliases:** Curated `club_name` is often a formal name. For example, `398` is "Società Sportiva Lazio S.p.A." but is observed in transfers as "Lazio". For 530 of the 796 curated clubs, `club_name` does not appear in `observed_names`. 1,068 identifiers have more than one observed name, and 106 names are shared by more than one actual club (e.g., three clubs named "Independiente"). Use `observed_names` as aliases, and do not merge clubs by name.
- **Youth and reserve teams:** Each has its own identifier (e.g., `25003` Club Brugge Yth, `66530` Club Brugge U18). No parent-club hierarchy is provided.

### 3.2 `leagues.csv`

- **Role & size:** League list, 32 rows. PK: `league_id`.
- **Keys & fields:** `league_name`, `country_id`, `country_name`, `competition_type`, `competition_sub_type`, etc.
- **Provenance / KG:** Create `League` nodes. Use `club_league_current.csv` for club links.
- **Caveats:** `country_id` is an external country identifier; there is no `countries.csv`. `COL1` is confirmed as Liga BetPlay, but its league type/sub-type metadata is unconfirmed.
- **Values:** 32 leagues in 32 countries. Every league except `COL1` has `competition_type=domestic_league` and `competition_sub_type=first_tier`. `COL1` has a null type and sub-type and `metadata_status=documented_identity_only`. Each league has 11–43 clubs in `club_league_current.csv`.
- **Codes and names:** `league_id` is a Transfermarkt competition code, not a country code: `KR1` is Croatia's SuperSport HNL and `RSK1` is South Korea's K League 1. `league_name` is not unique (`A1` and `L1` are both "Bundesliga"; `RU1` and `UKR1` are both "Premier Liga"), so key on `league_id`. `country_name` uses Transfermarkt labels (e.g., `Korea, South`, `Türkiye`).

### 3.3 `club_league_current.csv`

- **Role & size:** Currently retained club–league links, 796 rows. PK: (`club_id`, `league_id`); `club_id` is also unique.
- **Links:** `club_id` → `clubs` / `league_id` → `leagues`. `source_url` is preserved.
- **Provenance / KG:** Link with `IN_LEAGUE`. The `entity_id` in `fact_sources` is `club_id:league_id`.
- **Caveats:** Only clubs with the latest retained metadata are included. This is not a season-by-season league history.
- **Redundancy and coverage:** The 796 clubs are exactly the `curated` clubs, and `clubs.current_domestic_competition_id` holds the same league for each of them. Build `IN_LEAGUE` from only one of the two sources. 40,195 players have a profile club with an `IN_LEAGUE` link.

**Link Example**

```
Player → CURRENTLY_PLAYS_FOR → Club → IN_LEAGUE → League
```

`IN_LEAGUE` reflects the current relationship at collection time. Analyzing promotion/relegation and clubs' league affiliation in past seasons requires separate time-specific evidence.

## 4. Transfer Events and Historical Memberships

### 4.1 `transfer_events.csv`

- **Role & size:** Individual transfer and career-status events, 137,776 rows. PK: `transfer_event_id`.
- **Links & fields:** `player_id`, `from_club_id`, `to_club_id`, `transfer_date`, `transfer_season`, and evidence fields.
- **Types:** Use `event_type` and `normalized_membership_type`. The original `membership_type` is `unknown` for all 137,776 rows and must be distinguished from the normalized type column. `source_supported_membership_type` and `derived_membership_type` are identical to `normalized_membership_type` in this release.
- **Temporal status:** `event_temporal_status` is completed 135,120 / planned 2,562 / unknown 94. Not every event is completed history. The 94 `unknown` events are exactly the 86 `unmatched` and 8 `multiply_matched` events (`raw_match_status`). They have `event_type=unknown`. Every raw evidence field except `raw_match_status` is null: `raw_*`, `original_fee`, `original_fee_json`, `transfer_id` and `transfer_url`.
- **Provenance / KG:** Create `TransferEvent` nodes. Preserve date, type, temporal status, original transfer fee, and raw evidence. Do not derive memberships from these events a second time. Each membership already references its start event (`source_transfer_event_id`) and end event (`end_source_transfer_event_id`), so link to those instead.
- **Dates and seasons:** `transfer_date_precision` is `exact` for every row. Completed events run from 1993-07-01 to 2026-10-06. `transfer_season` is the original curated label (`93/94` to `27/28`). For 1,214 completed events it differs from the July–June season of `transfer_date`, so order events by `transfer_date`.
- **Fee columns:**
  - `original_transfer_fee` is the curated numeric fee with three decimals (e.g., `1000000.000`). For 14,251 of the 14,262 paid transfers it equals the `€` amount in `original_fee`. It is `0.000` on 74,556 rows and null on 48,960. Zero is not evidence of a free transfer or a loan.
  - `original_fee` is the raw label: `-` (37,137), `free transfer` (27,237), `End of loan` (23,505), `loan transfer` (21,797), `?` (11,755), or an amount such as `€1.00m`. Loan fees keep raw HTML, e.g., `Loan fee:<br /><i class="normaler-text">€500k</i>`.
  - `original_fee_json` holds the same value JSON-encoded.

**Event Types by Temporal Status**

| `event_type` | completed | planned | unknown | total | `normalized_membership_type` |
|---|---:|---:|---:|---:|---|
| `unknown` | 46,047 | 48 | 94 | 46,189 | `unknown` |
| `free_transfer` | 27,100 | 124 | 0 | 27,224 | `permanent` |
| `loan` | 23,551 | 8 | 0 | 23,559 | `loan` |
| `loan_return` | 21,744 | 1,761 | 0 | 23,505 | `loan_return` |
| `paid_transfer` | 14,148 | 114 | 0 | 14,262 | `unknown` |
| `without_club` | 2,381 | 464 | 0 | 2,845 | null |
| `retired` | 112 | 43 | 0 | 155 | null |
| `career_break` | 37 | 0 | 0 | 37 | null |

`event_type=unknown`, the largest group, covers three cases:

- 46,082 events whose raw label supports no type (`classification_rule=no_supported_explicit_label`), mostly fee `-` or `?`.
- 13 events with conflicting labels.
- The 94 unmatched events.

**Representative Types and Handling**

- `loan` marks the start of a loan; `loan_return` is evidence that an existing loan has ended. A return event alone does not create a new membership.
- `free_transfer` is evidence for the start of a permanent membership. `paid_transfer` and unsupported transfer-fee labels keep the membership type as `unknown`.
- `retired`, `without_club`, and `career_break` are status events. They do not create regular club membership relationships. More generally, no event whose destination is a special-status ID starts a membership, whatever its `event_type`. For example, `75` Unknown receives 105 `free_transfer` and 94 `event_type=unknown` events, `515` Without Club receives 71 `event_type=unknown` events, and `2077` Ban receives 10. Check the `entity_kind` of `to_club_id`, not just `event_type`.
- `completed` means a non-planned observation in the original API snapshot. It does not mean the transfer has been newly verified as completed in the real world.
- Only events that are `completed` and have `raw_match_status=matched` are used as the start and end of historical memberships. Every such event whose destination is an actual club and whose type is not `loan_return` starts exactly one membership: 110,621 = 26,996 `free_transfer` + 23,547 `loan` + 14,148 `paid_transfer` + 45,930 `unknown`. The other 24,499 completed events start none: 21,744 are `loan_return` closures and 2,755 go to special-status destinations (`build_manifest.json` → `corrections.membership_start_exclusions`).

### 4.2 `player_club_memberships.csv`

- **Role & size:** Historical membership periods, 110,621 rows. PK: `membership_id`.
- **Links & fields:** `player_id`, `club_id`, `membership_type`, `parent_club_id`, `start_date`, `end_date`, `season`, `start_precision`, `end_precision`, etc.
- **Evidence:** `source_transfer_event_id`/`source_id` and `end_source_transfer_event_id`/`end_source_id` distinguish start evidence from end evidence.
- **Provenance / KG:** Create `PLAYED_FOR` / `LOANED_TO` relationships or `Membership` nodes. Retain multiple periods for the same player–club pair: 6,871 (player, club) pairs have more than one membership.
- **Types:** `membership_type` maps one-to-one to `derivation_method`:
  - `permanent` 26,996: `explicit_free_transfer_destination`.
  - `loan` 23,547: `explicit_loan_destination`.
  - `unknown` 60,078: `explicit_destination_membership_type_unknown` (14,148 paid transfers and 45,930 unlabelled events).

  `confidence_status` is `source_supported` 50,541 / `membership_type_unknown` 60,078 / `ambiguous_interval_end` 2. It is a status, not a probability.
- **Start:** `start_date`, `club_id`, `season` and `source_id` equal the start event's `transfer_date`, `to_club_id`, `transfer_season` and `raw_source_id`. Every start is exact (1993-07-01 to 2026-08-03). For loans, `parent_club_id` is the event's `from_club_id`. Three loans have a null parent because the loan came from a special-status ID.
- **End rules:** Only two rules set `end_date` (45,403 rows, 1997-07-01 to 2026-10-06):
  - `loan_return_exact_player_borrower_parent` closes a loan with the matching loan return (21,465 loans).
  - `explicit_nonloan_departure_exact_player_club` closes a permanent or unknown membership with a later completed event that leaves the same club and has one of these types: `free_transfer` (21,562), `without_club` (2,259), `retired` (92) or `career_break` (25). That is 23,938 memberships in total.

  Departures by `paid_transfer`, `unknown` or `loan` events never close a membership. `build_manifest.json` → `corrections.membership_derivation_issues` lists 704 closing events that could not be matched unambiguously: 425 departures that matched several open memberships, 276 loan returns without a unique prior loan, and 3 others.
- **Caveats:** 60,078 rows are `unknown`. End date available: 45,403 / undetermined: 65,218. Do not interpret a null end date as the player still being at the club.
  - 48,859 of the 65,218 open memberships are followed by a later completed event leaving the same club: 27,957 `unknown`, 10,618 `loan` (the player is loaned out), 9,636 `paid_transfer` and 648 others.
  - 16,183 players have two or more memberships with no end date.
  - If an application needs approximate periods, derive them in a separate, clearly labelled layer instead of filling `end_date`.
- **Coverage:** Memberships exist for 18,372 players only.

### 4.3 `planned_transfer_events.csv`

- **Role & size:** Planned transfer references, 2,562 rows. `transfer_event_id` is both PK and FK, referencing `transfer_events`.
- **Fields:** Preserves `player_id`, `transfer_date`, `event_temporal_status`, the original future/planned flags, and raw evidence. Destination club information is obtained by joining on `transfer_event_id` to `to_club_id` in `transfer_events`.
- **Provenance / KG:** Represent as `PlannedTransferEvent` or `PLANNED_TRANSFER`. These rows are also included as-is in `transfer_events`, so do not create duplicate events.
- **Caveats:** If the original has `futureTransfer=1` or `upcoming=true`, the event remains `planned` even after the scheduled date has passed. `planned` events and events with `unknown` temporal status are not used as start/end evidence for historical memberships.
- **Values:** All 2,562 were classified by explicit flags (`temporal_classification_rule=explicit_raw_future_or_upcoming`, `raw_future_transfer=1`, `raw_upcoming=true`).
  - Scheduled dates run from 2024-07-29 to 2030-06-30. 2,132 are on or before the release date (2026-10-08), and 430 are after it.
  - "Planned" is relative to the snapshot of the player's raw source, not to the release date (see 5.2). By source: API 2023 file 135 (from 2024-07-29), API 2024 file 413 (from 2026-05-03), API 2025 file 1,954 (from 2026-06-15), local cache 60 (from 2026-11-30).
  - Of the 2,132 past-dated events, 1,997 fall in May–September 2026, mostly the 2026 summer window after the API 2024 and 2025 snapshots. The other 135 were still in the future at the July 2024 snapshot.
  - By type: 1,761 `loan_return`, 464 `without_club`, 124 `free_transfer`, 114 `paid_transfer`, 48 `unknown`, 43 `retired`, 8 `loan`.
  - All 11 non-key columns are identical to the matching `transfer_events` rows.

## 5. Current Clubs and Sources

### 5.1 `player_current_clubs.csv`

- **Role & size:** Current profile club references, 40,706 rows. PK: `current_relation_id`; `player_id` is also unique.
- **Links:** `player_id` → `players` / `club_id` → `clubs` / `source_id` → `source_documents`. Matches `players.current_club_id`.
- **Status:** `profile_current_club_reference` 40,700 / `special_status_reference` 6. `temporal_scope` is `latest_retained_player_profile`. The 40,700 profile references are 40,679 to actual clubs and 21 to the 9 unresolved identifiers. The 6 special-status references are 5 to `515` Without Club and 1 to `123` Retired.
- **Provenance / KG:** Regular profile references can be represented as `CURRENTLY_PLAYS_FOR`. The 6 `special_status_reference` rows are not turned into regular club edges. Decide explicitly how to treat the 21 references to unresolved identifiers, for example with an `UnresolvedClubIdentifier` node, or by excluding them and recording the count. Otherwise a loader that creates `Club` nodes only for `entity_kind=club` will leave them dangling.
- **Caveats:** The current club is not guaranteed to be up to date. It is the club on the retained profile, which for retired or inactive players can be their last club. For example, player `10` (Miroslav Klose, born 1978) points to `398` Lazio. 625 players whose latest completed transfer event moves them to Retired, Without Club or Career break still have an actual club here. `start_date` and `end_date` are both null; join dates and loan types are not generated.

### 5.2 `player_transfer_history_status.csv`

- **Role & size:** Per-player coverage status of the original transfer history, 40,706 rows. `player_id` is both PK and FK.
- **Status:** `valid_nonempty` 19,190 / `valid_empty` 40 / `unavailable` 21,476 players.
- **Fields:** Distinguishes `raw_transfer_event_count` from `curated_transfer_event_count`. For `unavailable`, the raw count is null; for `valid_empty`, it is 0.
  - `raw_transfer_event_count` counts the events in the raw API history response.
  - `curated_transfer_event_count` equals the number of rows the player has in `transfer_events` (verified for every player).
  - `raw_source_type` is `dvc_raw_api` for 16,485 players, `local_api_cache` for 2,745, and null for `unavailable`.
- **Provenance / KG:** Keep as a collection-status attribute on `Player`.
- **Caveats:** `unavailable` does not mean "no history." Even when there are 0 curated events, do not conclude that the player's actual career is empty.
- **Coverage:** The status describes the raw history response, not `transfer_events`. Transfer events exist only for `valid_nonempty` players, and not for all of them. 18,376 players have at least one event, while 814 `valid_nonempty` players have none (e.g., player `10` has 11 raw events and 0 curated). Across `valid_nonempty` players, raw counts total 145,080, against 137,776 curated events.
- **Observation time:** Each player's history comes from one canonical raw response, and the raw sources were captured at different times. The bounds below come from event dates: every completed event is on or before the snapshot, and every planned event is after it.

| Canonical source (`raw_source_file`) | Players | Players with events | Last completed date | First planned date | Snapshot |
|---|---:|---:|---|---|---|
| `transfermarkt-api/2023/transfers.json` | 2,384 | 2,367 | 2024-07-26 | 2024-07-29 | Late July 2024 |
| `transfermarkt-api/2024/transfers.json` | 2,122 | 2,118 | 2026-02-27 | 2026-05-03 | Between 2026-02-27 and 2026-05-03 |
| `transfermarkt-api/2025/transfers.json` | 11,979 | 11,965 | 2026-06-11 | 2026-06-15 | Mid-June 2026 |
| `data/raw/transfer_history*/<player_id>.json` (local cache) | 2,745 | 1,924 | 2026-10-06 | 2026-11-30 | `retrieved_at` 2026-10-07 to 2026-10-08 |

  For the 2,438 players with both bounds, no completed event falls after the player's first planned date. Within each file, the last completed date also precedes the first planned date, so each file behaves like a single snapshot. A history says nothing about moves after its snapshot. The few completed events dated March–May 2026 reflect the closed transfer window, not the end of the data.
- **Completeness:** `curated_transfer_event_count` equals `raw_transfer_event_count` for 17,980 of the 19,190 `valid_nonempty` players. It is lower for 1,207 (including the 814 with no curated events) and higher for 3. For those 1,210 players, gaps between recorded moves may hide unrecorded ones.
- **Who lacks history:** Coverage tracks how recently a player was active more than league prominence. By birth decade, 9.9% of players born in the 1980s have memberships, against 42.3% for the 1990s and 62.4% for the 2000s. By profile club, the figure is 37.7% for the top-5 leagues and 47.2% for the other 27 leagues.

### 5.3 `source_documents.csv`

- **Role & size:** Source records, 365,128 rows. PK: `source_id`.
- **Keys & fields:** `source_url`, `source_type`, `raw_locator`, `retrieved_at`, `content_sha256`, `extraction_method`.
- **Provenance / KG:** Create `SourceDocument` nodes. A record may represent an individual source row or event rather than an entire web page.
- **Caveats:** Unknown retrieval times and hashes remain null. `raw_locator` points to files in the original project (`data/raw/...`, `transfermarkt-datasets/...`, `data/top50/intermediate/...`). None of these raw files are included in the ZIP, so treat locators as opaque provenance strings.
- **Source types:**

| `source_type` | Records | `raw_locator` points to | `source_url` | `content_sha256` |
|---|---:|---|---|---|
| `dvc_raw_api` | 143,098 | Raw API transfer JSON (`transfermarkt-datasets/data/raw/...`) | Relative href on 126,613 | File hash (3 distinct) |
| `existing_transfer_record` | 137,776 | Curated transfer row (`data/raw/transfermarkt.duckdb`, `main.transfers`) | null | null |
| `curated_player` | 40,706 | Curated player row (`main.players`) | Absolute profile URL | null |
| `local_coverage_audit` | 21,476 | Coverage audit row (`data/top50/intermediate/...`) | null | File hash (1 distinct) |
| `local_api_cache` | 21,212 | Cached per-player history JSON (`data/raw/transfer_history*/`) | Relative href on 18,467 | File hash (2,745 distinct) |
| `latest_curated_club` | 796 | Curated club row (`main.clubs`) | Absolute | null |
| `existing_competition` | 62 | Competition row (`main.competitions`) | Absolute | null |
| `curated_country` | 1 | Country row (`main.countries`) | Absolute | Set |
| `local_repository_configuration` | 1 | `transfermarkt-datasets/config.yml` | null | Set |

- **Timestamps and hashes:** `retrieved_at` is populated only for 21,192 `local_api_cache` records. 2,744 of the 2,745 canonical per-player responses among them were fetched between 2026-10-07 14:28 and 2026-10-08 07:55 UTC. `dvc_raw_api` records carry no retrieval time, but each file's snapshot date can be bounded from event dates (see 5.2). `content_sha256` is the hash of the whole preserved source file, shared by every record from that file, not a per-record hash.
- **Unreferenced records:** 6,947 records (6,915 `local_api_cache`, 32 `dvc_raw_api`) are not referenced by any fact or foreign key. They point to individual raw history events that no retained row uses (e.g., `data/raw/transfer_history/261393.json#pointer=/transfers/9`). Loading every record creates that many isolated `SourceDocument` nodes.

### 5.4 `fact_sources.csv`

- **Role & size:** Fact–source links, 1,462,416 rows. PK: (`entity_type`, `entity_id`, `fact_name`, `source_id`).
- **Links:** `source_id` → `source_documents`. `entity_type`/`entity_id` are polymorphic references to the target tables and require separate validation.
- **Provenance / KG:** Link each `Fact`, identified by `entity_type`, `entity_id`, and `fact_name`, to its source via `SUPPORTED_BY`. Facts can also be mapped to relationship, membership, or event nodes.
- **Status:** All rows have `provenance_status=resolved`; there are 0 null `source_id` values and 0 broken entity/source ID references.
- **Entity types and facts:**

| `entity_type` | `entity_id` | Links | `fact_name` values |
|---|---|---:|---|
| `transfer_event` | `transfer_event_id` | 826,280 | `transfer_event`, `event_temporal_status` (all 137,776 events); `event_type`, `normalized_membership_type`, `classification_rule`, `raw_transfer_evidence` (the 137,682 matched events) |
| `player` | `player_id` | 276,754 | `player_name`, `country_of_citizenship`, `birth_date_precision`, `current_club_id` (40,706 each); `date_of_birth` 40,659; `city_of_birth` 36,657; `country_of_birth` 36,614 |
| `player_club_membership` | `membership_id` | 179,568 | `membership_start_and_type` 110,621; `membership_end` 45,403; `parent_club_id` 23,544 |
| `player_current_club` | `current_relation_id` | 40,706 | `profile_current_club_reference` |
| `player_transfer_history_status` | `player_id` | 40,706 | `transfer_history_status` |
| `player_position` | `player_id:position_id` | 40,238 | `position` |
| `position` | `position_id` | 40,238 | `position_definition` (13 facts) |
| `club` | `club_id` | 14,409 | `club_name` 796; `club_name_observation` 12,808; `current_domestic_competition_id` 796; `identifier_reference` 9 |
| `planned_transfer_event` | `transfer_event_id` | 2,562 | `planned_transfer_observation` |
| `club_league_current` | `club_id:league_id` | 796 | `latest_league` |
| `league` | `league_id` | 159 | `league_name`, `country_id`, `country_name` (32 each); `competition_type`, `competition_sub_type` (31 each); `league_id` (1) |

- **Granularity:**
  - Facts exist only for non-null values (e.g., 40,659 `date_of_birth` facts for 40,659 known birth dates).
  - Some fact names are bundles covering several columns (`transfer_event`, `raw_transfer_evidence`, `membership_start_and_type`).
  - Every row of every table has at least one fact.
  - Each of the 1,422,191 distinct facts has exactly one source, except the 13 `position_definition` facts. Each of those links to every player profile showing that position (up to 7,183 sources).
- **Caveats:** A valid source link does not guarantee factual accuracy or that the web URL is currently accessible.

## 6. Documentation and Distribution Files

### 6.1 `README.md`

- Overall usage guide and the starting point for checking each file's role and constraints. Read it before loading into the KG.
- The paths it mentions belong to the original project and are not part of the distribution. These include the `src/*.py` build and validation scripts and the `reports/top50/...` reports.

### 6.2 `schema.md`

- For checking table relationships and the actual PKs/FKs. Use it to design KG uniqueness constraints, join keys, and loading order.

### 6.3 `data_dictionary.csv`

- Describes the actual 131 columns. Identified by (`table`, `column`) and matches the headers of all data CSVs, including column order. Map attributes using `type`, `nullable`, `description`, and `provenance_note`.
- Logical types: `TEXT` 116, `BOOLEAN` 4, `DATE` 4, `INTEGER` 4, `JSON` 2, `TIMESTAMP WITH TIME ZONE` 1. Most date columns (`transfer_date`, `start_date`, `date_of_birth`) are typed `TEXT`. Only `temporal_as_of_date`, `player_club_memberships.end_date` and the always-null `player_current_clubs` dates are typed `DATE`.

### 6.4 `database_statistics.json`

- Stores key statistics such as final counts and history coverage. Use it to compare counts before and after loading from the source data into the KG.
- It also holds `country_counts` (FIFA rank → citizenship labels → players) and several policy flags:
  - `planned_events_allowed_in_historical_memberships`, `paid_transfer_semantic_rule_used` and `secondary_citizenship_included` are all `false`.
  - `external_requests_made` is 0.

### 6.5 `build_manifest.json`

- Records release metadata (`release_version`, `release_as_of_date`, `input_database`) and `external_requests_made` (0).
- Holds SHA-256 hashes of the 5,527 upstream input files (`input_sha256`) and of the 13 data tables (`output_sha256`, identical to `SHA256SUMS`).
- Contains a `corrections` summary of the changes from `final_v2`. It includes `membership_start_exclusions` (why completed events did not start memberships) and `membership_derivation_issues` (704 closing events that could not be matched unambiguously).
- It records what was built and changed, not the build procedure itself. Keep a record of the version used. Optionally manage it as Dataset/Release metadata.

### 6.6 `SHA256SUMS`

- SHA-256 list for 14 CSVs: the 13 data tables plus `data_dictionary.csv`. Use it to detect file changes or corruption; do not use it to create KG relationships or to verify semantic correctness.

**Distribution Files**

- `top50_dataset.zip` is the package delivered to team members.
- `top50_dataset.zip.sha256` is an optional file that verifies the ZIP itself.
- `SHA256SUMS` inside the package verifies the CSVs.

Documentation, statistics, and checksums are kept separate from regular domain data and do not need to be loaded as Player/Club relationships. A matching checksum does not guarantee the semantic correctness of the content.

## 7. Recommended KG Structure and Data Flow

> For this repository's KG, the design in [kg-schema.md](kg-schema.md) supersedes this section. It reifies relationships as nodes instead of adding `Fact` nodes, models the profile club as `ProfileClaim` instead of `CURRENTLY_PLAYS_FOR`, and adds a per-player observation time. The rest of this section is kept as general guidance for using the dataset.

### 7.1 Base Structure

```
Player → HAS_POSITION → Position
Player → CURRENTLY_PLAYS_FOR → Club
Player → PLAYED_FOR → Club
Club → IN_LEAGUE → League
Fact → SUPPORTED_BY → SourceDocument
```

- Preserve membership type, period, season, and evidence identifiers on `PLAYED_FOR`.
- Represent loans as `membership_type=loan` or as `LOANED_TO`. Avoid double counting.
- If repeated memberships or multiple sources matter, use the structure `Player → HAS_MEMBERSHIP → Membership → AT_CLUB → Club`.
- Add `TransferEvent` when analyzing transfer events separately. Distinguish planned transfers with a separate node or a `planned` status.

### 7.2 Source Linking

- In a standard property graph, a new edge cannot be drawn directly from an edge to a source. Create `Fact` as a node and retain `entity_type`, `entity_id`, and `fact_name` to link it back to the original fact.
- Even when the fact is itself a node (e.g., `Membership`, `TransferEvent`), the correspondence between fact attributes and their sources must not be lost.
- If a single fact has multiple sources, preserve all of them. Do not substitute a single player URL for relationship-level provenance.

### 7.3 End-to-End Data Flow

1. Unzip → verify the 14 CSVs against `SHA256SUMS` → record the internal build version and reference dates from `build_manifest`.
2. Based on `schema` and `data_dictionary`, define TEXT identifiers, nulls, PKs/FKs, and the KG mapping.
3. Create Player, actual Club, League, Position, and Source nodes; keep special statuses and unresolved IDs distinguished.
4. Link position, profile club, and currently retained league relationships. Exclude the 6 special-status rows from regular current-club relationships, and decide explicitly how to treat the 21 profile references to unresolved identifiers.
5. Load memberships as given rather than re-deriving them from events. For events, preserve temporal status and normalized type, and do not create duplicate planned events.
6. Link sources at the fact level using `fact_sources`, and validate counts, duplicates, and references. Record the reason and count for any excluded data.

### 7.4 Reference Counts for Validation

| Item | Expected count |
|---|---:|
| `Player` | 40,706 |
| `Position` / `HAS_POSITION` | 13 / 40,238 |
| `Club` (actual clubs) | 13,599 (plus 5 special statuses and 9 unresolved identifiers, if loaded) |
| `League` / `IN_LEAGUE` | 32 / 796 |
| `CURRENTLY_PLAYS_FOR` | 40,679 to actual clubs (plus 21 to unresolved identifiers; the 6 special-status references are excluded) |
| Memberships | 110,621 (permanent 26,996 / loan 23,547 / unknown 60,078) |
| `TransferEvent` | 137,776 (2,562 of them planned) |
| `SourceDocument` | 365,128 (358,181 referenced by a fact or foreign key) |
| Distinct facts / fact–source links | 1,422,191 / 1,462,416 |

## 8. Additional Information

### 8.1 File Encoding

- All CSV files in this dataset are saved in UTF-8 encoding without a byte-order mark (BOM), so Excel has to guess the encoding.
- If you open a CSV file by double-clicking it in Excel, special and European characters such as Ú, é, ö, and ü may appear garbled.
- This is not an error in the data itself; it occurs because Excel misidentifies the file encoding as CP949 or EUC-KR.
- CSV files must be opened with UTF-8 (65001) encoding:

1. Launch Excel.
2. Select the **Data** tab in the top menu.
3. Click **From Text/CSV**.
4. Select the CSV file you want to open.
5. In the preview window, find the **File Origin** field.
6. Set File Origin to **65001: Unicode (UTF-8)**.
7. Check that the characters display correctly.
8. Click **Load**.

### 8.2 Line Endings

- Line endings are mixed. `club_league_current.csv`, `player_positions.csv`, `players.csv` and `positions.csv` use LF, and the other 10 CSVs use CRLF.
- CSV parsers handle both. Line-oriented shell tools such as `cut` or `awk`, however, keep a trailing `\r` in the last field of CRLF files.