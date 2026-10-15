# Validation report (revision r0)

- Checks: 182 (139 during the build, 43 on the built KG)
- Failed: 0

## Grade distribution (lag 14, A_max 34)

| t | population | CONFIRMED | INFERRED_R12 | INFERRED_R3 | OPEN_PROFILE | OPEN | NONE | registration overlaps | planned due → OPEN |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 2020-01-01 | 12,865 | 6,612 | 5,694 | 479 | 5 | 3 | 72 | 166 | 0 |
| 2023-01-01 | 16,644 | 8,255 | 6,192 | 2,007 | 137 | 10 | 43 | 225 | 0 |
| 2025-01-01 | 17,993 | 5,742 | 3,889 | 5,453 | 1,214 | 1,659 | 36 | 273 | 23 |
| 2026-01-01 | 18,291 | 1,852 | 1,173 | 11,751 | 1,693 | 1,799 | 23 | 281 | 134 |
| 2026-09-01 | 18,372 | 13 | 0 | 1,396 | 11,341 | 5,607 | 15 | 295 | 2,016 |

## All checks

| ok | section | id | expected | actual | note |
|---|---|---|---|---|---|
| ✅ | S1 | input.checksums_verified | 14 | 14 | 13 data tables + data_dictionary.csv match SHA256SUMS |
| ✅ | S1 | input.header.players | ["player_id", "player_name", "date_of_birth", "birth_date_precision", "city_of_birth", "country_of_birth", "country_of_citizenship", "current_club_id", "sour... | ["player_id", "player_name", "date_of_birth", "birth_date_precision", "city_of_birth", "country_of_birth", "country_of_citizenship", "current_club_id", "sour... | CSV header equals data_dictionary.csv (names and order) |
| ✅ | S1 | input.header.positions | ["position_id", "position", "sub_position"] | ["position_id", "position", "sub_position"] | CSV header equals data_dictionary.csv (names and order) |
| ✅ | S1 | input.header.player_positions | ["player_id", "position_id", "source_url"] | ["player_id", "position_id", "source_url"] | CSV header equals data_dictionary.csv (names and order) |
| ✅ | S1 | input.header.clubs | ["club_id", "club_name", "current_domestic_competition_id", "source_url", "metadata_status", "club_name_scope", "entity_kind", "observed_names"] | ["club_id", "club_name", "current_domestic_competition_id", "source_url", "metadata_status", "club_name_scope", "entity_kind", "observed_names"] | CSV header equals data_dictionary.csv (names and order) |
| ✅ | S1 | input.header.leagues | ["league_id", "league_name", "country_id", "country_name", "competition_type", "competition_sub_type", "source_url", "metadata_status"] | ["league_id", "league_name", "country_id", "country_name", "competition_type", "competition_sub_type", "source_url", "metadata_status"] | CSV header equals data_dictionary.csv (names and order) |
| ✅ | S1 | input.header.club_league_current | ["club_id", "league_id", "source_url"] | ["club_id", "league_id", "source_url"] | CSV header equals data_dictionary.csv (names and order) |
| ✅ | S1 | input.header.transfer_events | ["transfer_event_id", "player_id", "transfer_date", "transfer_date_precision", "transfer_season", "from_club_id", "from_club_name", "to_club_id", "to_club_na... | ["transfer_event_id", "player_id", "transfer_date", "transfer_date_precision", "transfer_season", "from_club_id", "from_club_name", "to_club_id", "to_club_na... | CSV header equals data_dictionary.csv (names and order) |
| ✅ | S1 | input.header.player_club_memberships | ["membership_id", "player_id", "club_id", "membership_type", "parent_club_id", "start_date", "end_date", "season", "start_precision", "end_precision", "deriv... | ["membership_id", "player_id", "club_id", "membership_type", "parent_club_id", "start_date", "end_date", "season", "start_precision", "end_precision", "deriv... | CSV header equals data_dictionary.csv (names and order) |
| ✅ | S1 | input.header.player_current_clubs | ["current_relation_id", "player_id", "club_id", "source_url", "source_id", "relation_kind", "temporal_scope", "start_date", "end_date"] | ["current_relation_id", "player_id", "club_id", "source_url", "source_id", "relation_kind", "temporal_scope", "start_date", "end_date"] | CSV header equals data_dictionary.csv (names and order) |
| ✅ | S1 | input.header.player_transfer_history_status | ["player_id", "transfer_history_status", "raw_source_type", "raw_source_file", "raw_transfer_event_count", "curated_transfer_event_count"] | ["player_id", "transfer_history_status", "raw_source_type", "raw_source_file", "raw_transfer_event_count", "curated_transfer_event_count"] | CSV header equals data_dictionary.csv (names and order) |
| ✅ | S1 | input.header.source_documents | ["source_id", "source_url", "source_type", "raw_locator", "retrieved_at", "content_sha256", "extraction_method"] | ["source_id", "source_url", "source_type", "raw_locator", "retrieved_at", "content_sha256", "extraction_method"] | CSV header equals data_dictionary.csv (names and order) |
| ✅ | S1 | input.header.fact_sources | ["entity_type", "entity_id", "fact_name", "source_id", "provenance_status"] | ["entity_type", "entity_id", "fact_name", "source_id", "provenance_status"] | CSV header equals data_dictionary.csv (names and order) |
| ✅ | S1 | input.header.planned_transfer_events | ["transfer_event_id", "player_id", "transfer_date", "transfer_season", "event_temporal_status", "temporal_classification_rule", "raw_source_id", "raw_source_... | ["transfer_event_id", "player_id", "transfer_date", "transfer_season", "event_temporal_status", "temporal_classification_rule", "raw_source_id", "raw_source_... | CSV header equals data_dictionary.csv (names and order) |
| ✅ | S1 | input.rows.club_league_current | 796 | 796 | row count equals database_statistics.json |
| ✅ | S1 | input.rows.clubs | 13613 | 13613 | row count equals database_statistics.json |
| ✅ | S1 | input.rows.fact_sources | 1462416 | 1462416 | row count equals database_statistics.json |
| ✅ | S1 | input.rows.leagues | 32 | 32 | row count equals database_statistics.json |
| ✅ | S1 | input.rows.planned_transfer_events | 2562 | 2562 | row count equals database_statistics.json |
| ✅ | S1 | input.rows.player_club_memberships | 110621 | 110621 | row count equals database_statistics.json |
| ✅ | S1 | input.rows.player_current_clubs | 40706 | 40706 | row count equals database_statistics.json |
| ✅ | S1 | input.rows.player_positions | 40238 | 40238 | row count equals database_statistics.json |
| ✅ | S1 | input.rows.player_transfer_history_status | 40706 | 40706 | row count equals database_statistics.json |
| ✅ | S1 | input.rows.players | 40706 | 40706 | row count equals database_statistics.json |
| ✅ | S1 | input.rows.positions | 13 | 13 | row count equals database_statistics.json |
| ✅ | S1 | input.rows.source_documents | 365128 | 365128 | row count equals database_statistics.json |
| ✅ | S1 | input.rows.transfer_events | 137776 | 137776 | row count equals database_statistics.json |
| ✅ | S1 | input.planned_subset | 2562 | 2562 | every planned event is a transfer_events row (never created twice) |
| ✅ | S1 | input.planned_columns_identical | 2562 | 2562 |  |
| ✅ | S1 | input.planned_equals_status_planned | true | true | planned table = transfer_events with event_temporal_status=planned |
| ✅ | S2 | player.history_status_one_per_player | true | true |  |
| ✅ | §3 | node.Player | 40706 | 40706 |  |
| ✅ | §3 | player.birth_date_precision | {"exact": 40386, "year": 273, "unknown": 47} | {"exact": 40386, "year": 273, "unknown": 47} |  |
| ✅ | §3 | player.history_coverage | {"valid_nonempty": 19190, "valid_empty": 40, "unavailable": 21476} | {"unavailable": 21476, "valid_nonempty": 19190, "valid_empty": 40} |  |
| ✅ | §3 | clubs.entity_kind | {"club": 13599, "special_status": 5, "unresolved_identifier": 9} | {"club": 13599, "unresolved_identifier": 9, "special_status": 5} |  |
| ✅ | §3 | club.team_level | — | {"first_team": 8303, "youth": 4659, "reserve": 637} | provisional name pattern; kg-schema estimate ~5,300 non-first-team IDs |
| ✅ | §3 | career_status.ids | ["123", "2077", "2113", "515", "75"] | ["123", "2077", "2113", "515", "75"] |  |
| ✅ | §11 | node.Country | 53 | 53 | 51 citizenship labels (Turkey+Türkiye merged) + 3 league-only |
| ✅ | §4 | edge.CITIZEN_OF.mapped | 40706 | 40706 |  |
| ✅ | §4 | edge.IN_COUNTRY.mapped | 32 | 32 |  |
| ✅ | §4 | player_positions.at_most_one_per_player | true | true |  |
| ✅ | §11 | player.without_position | 468 | 468 |  |
| ✅ | §5.1 | event.pointer_index_for_matched | 137682 | 137682 |  |
| ✅ | §5.1 | event.one_raw_file_per_player | 0 | 0 |  |
| ✅ | §5.1 | event.same_day_pointer_unique | 0 | 0 |  |
| ✅ | §4 | event.endpoint_kinds | {'Club', 'CareerStatus'} | {'Club', 'CareerStatus'} | no event touches an unresolved ID |
| ✅ | §4 | event.endpoints_known | 0 | 0 |  |
| ✅ | §11 | event.temporal_status | {"completed": 135120, "planned": 2562, "unknown": 94} | {"completed": 135120, "planned": 2562, "unknown": 94} |  |
| ✅ | §4 | event.special_endpoints | {"FROM": 2444, "TO": 3326} | {"FROM": 2444, "TO": 3326} |  |
| ✅ | S3 | profile.matches_players_current_club | 40706 | 40706 |  |
| ✅ | §4 | edge.POINTS_TO.by_label | {"Club": 40679, "UnresolvedClub": 21, "CareerStatus": 6} | {"Club": 40679, "UnresolvedClub": 21, "CareerStatus": 6} |  |
| ✅ | §8 | league_spell.one_per_club | true | true |  |
| ✅ | S3 | league_spell.equals_clubs_competition | true | true | club_league_current and clubs.current_domestic_competition_id agree; LeagueSpell built from the former |
| ✅ | §5.2 | observation.known_source_files | 0 | 0 |  |
| ✅ | §5.2 | observation.players_by_source | {"API 2025": 11979, "local cache": 2745, "API 2023": 2384, "API 2024": 2122} | {"API 2025": 11979, "local cache": 2745, "API 2023": 2384, "API 2024": 2122} |  |
| ✅ | §5.2 | observation.api_file_horizons | {"API 2023": "2024-07-26", "API 2024": "2026-02-27", "API 2025": "2026-06-11"} | {"API 2023": "2024-07-26", "API 2024": "2026-02-27", "API 2025": "2026-06-11"} |  |
| ✅ | §5.2 | observation.local_canonical_files_unique | true | true |  |
| ✅ | §5.2 | observation.local_without_retrieved_at | 1 | 1 |  |
| ✅ | §5.2 | observation.local_horizon_range | ["2026-10-07", "2026-10-08"] | ["2026-10-07", "2026-10-08"] |  |
| ✅ | §5.2 | observation.history_complete | 17980 | 17980 |  |
| ✅ | §7 | observation.one_history_record_per_player | true | true |  |
| ✅ | §5.1 | spell.membership_end_basis | {"evidence": 45403, "R1": 45422, "R2": 7, "open": 19789} | {"R1": 45422, "evidence": 45403, "open": 19789, "R2": 7} |  |
| ✅ | §5.1 | spell.status_end_basis | {"R1": 2123, "R2": 1, "open": 634} | {"R1": 2123, "open": 634, "R2": 1} |  |
| ✅ | §11 | spell.status_count | 2758 | 2758 | status 2,553 + unknown destination 205 |
| ✅ | §11 | spell.status_unknown_destination | 205 | 205 |  |
| ✅ | §5.1 | spell.transit | {"membership": 29, "status": 1} | {"membership": 29, "status": 1} |  |
| ✅ | §5.1 | spell.last_confirmed_evidence | {"loan_out": 10629, "loan_return": 9294} | {"loan_out": 10629, "loan_return": 9294} |  |
| ✅ | §5.1 | spell.last_confirmed_total | 10629 | 10629 |  |
| ✅ | §11 | reconcile.status_spells | 2758 | 2758 | 2,755 special-destination start exclusions + 3 loan returns to 75 = 2,758 status spells |
| ✅ | §7 | fact_sources.entity_types | {'club_league_current', 'position', 'league', 'player_transfer_history_status', 'transfer_event', 'club', 'player_current_club', 'planned_transfer_event', 'p... | {'club_league_current', 'league', 'position', 'player_transfer_history_status', 'transfer_event', 'club', 'player_current_club', 'planned_transfer_event', 'p... |  |
| ✅ | §7 | fact_sources.resolved | 1462416 | 1462416 |  |
| ✅ | §7 | provenance.unmapped_rows | 0 | 0 |  |
| ✅ | §7 | provenance.player_position_pairs_exist | 0 | 0 |  |
| ✅ | §7 | edge.SUPPORTED_BY.dataset | 1462416 | 1462416 |  |
| ✅ | §7 | provenance.membership_membership_start_and_type_matches_column | 110621 | 110621 |  |
| ✅ | §7 | provenance.membership_membership_end_matches_column | 45403 | 45403 |  |
| ✅ | §7 | provenance.membership_parent_club_id_matches_column | 23544 | 23544 |  |
| ✅ | §7 | edge.SUPPORTED_BY.derived | {"status": 2758, "R1": 47545, "R2": 8, "last_confirmed": 10629} | {"R1": 47545, "last_confirmed": 10629, "status": 2758, "R2": 8} |  |
| ✅ | §11 | source.referenced | {"True": 358181, "False": 6947} | {"True": 358181, "False": 6947} |  |
| ✅ | §8 | reject1.unique.Player | 0 | 0 |  |
| ✅ | §8 | reject1.unique.Club | 0 | 0 |  |
| ✅ | §8 | reject1.unique.UnresolvedClub | 0 | 0 |  |
| ✅ | §8 | reject1.unique.CareerStatus | 0 | 0 |  |
| ✅ | §8 | reject1.unique.League | 0 | 0 |  |
| ✅ | §8 | reject1.unique.Country | 0 | 0 |  |
| ✅ | §8 | reject1.unique.Position | 0 | 0 |  |
| ✅ | §8 | reject1.unique.Source | 0 | 0 |  |
| ✅ | §8 | reject1.unique.Revision | 0 | 0 |  |
| ✅ | §8 | reject1.unique.ProfileClaim | 0 | 0 |  |
| ✅ | §8 | reject1.unique.TransferEvent | 0 | 0 |  |
| ✅ | §8 | reject1.unique.Spell | 0 | 0 |  |
| ✅ | §8 | reject1.unique.LeagueSpell | 0 | 0 |  |
| ✅ | §8 | reject1.one_current_version.ProfileClaim | 0 | 0 |  |
| ✅ | §8 | reject1.unique_logical_version.ProfileClaim | 0 | 0 |  |
| ✅ | §8 | reject1.one_current_version.TransferEvent | 0 | 0 |  |
| ✅ | §8 | reject1.unique_logical_version.TransferEvent | 0 | 0 |  |
| ✅ | §8 | reject1.one_current_version.Spell | 0 | 0 |  |
| ✅ | §8 | reject1.unique_logical_version.Spell | 0 | 0 |  |
| ✅ | §8 | reject1.one_current_version.LeagueSpell | 0 | 0 |  |
| ✅ | §8 | reject1.unique_logical_version.LeagueSpell | 0 | 0 |  |
| ✅ | §8 | reject2.start_events_completed_matched | 113379 | 113379 |  |
| ✅ | §8 | reject2.end_events_completed_matched | 92956 | 92956 |  |
| ✅ | §8 | reject2.start_date_equals_valid_from | 113379 | 113379 |  |
| ✅ | §8 | reject2.end_date_equals_valid_to | 92956 | 92956 |  |
| ✅ | §8 | reject3.start_same_player | 113379 | 113379 |  |
| ✅ | §8 | reject3.end_same_player | 92956 | 92956 |  |
| ✅ | §8 | reject3.at_equals_start_to | 113379 | 113379 |  |
| ✅ | §8 | reject3.departure_from_at | 92948 | 92948 |  |
| ✅ | §8 | reject3.end_after_start | 92956 | 92956 |  |
| ✅ | §8 | reject4.has_horizon | 113379 | 113379 |  |
| ✅ | §8 | reject4.from_le_to | 92956 | 92956 |  |
| ✅ | §8 | reject4.to_le_horizon | 92956 | 92956 |  |
| ✅ | §8 | reject4.from_le_horizon | 113379 | 113379 |  |
| ✅ | §8 | reject4.transit_iff_zero_length | 0 | 0 |  |
| ✅ | §8 | reject5.membership_at_club | 0 | 0 |  |
| ✅ | §8 | reject5.status_at_career_status | 0 | 0 |  |
| ✅ | §8 | reject5.loan_parent_differs | 0 | 0 |  |
| ✅ | §8 | reject5.loans_without_parent | 3 | 3 | loans whose parent is a special-status ID |
| ✅ | §8 | reject5.loan_parent_is_club | 0 | 0 |  |
| ✅ | §8 | reject6.supported.ProfileClaim | 0 | 0 |  |
| ✅ | §8 | reject6.recorded_once.ProfileClaim | 40706 | 40706 |  |
| ✅ | §8 | reject6.supported.TransferEvent | 0 | 0 |  |
| ✅ | §8 | reject6.recorded_once.TransferEvent | 137776 | 137776 |  |
| ✅ | §8 | reject6.supported.Spell | 0 | 0 |  |
| ✅ | §8 | reject6.recorded_once.Spell | 113379 | 113379 |  |
| ✅ | §8 | reject6.supported.LeagueSpell | 0 | 0 |  |
| ✅ | §8 | reject6.recorded_once.LeagueSpell | 796 | 796 |  |
| ✅ | §8 | reject7.completed_le_horizon | 0 | 0 |  |
| ✅ | §8 | reject7.planned_after_horizon | 0 | 0 |  |
| ✅ | §8 | warning.W1_registration_overlap | — | {"pairs": 1160, "players": 358} |  |
| ✅ | §8 | warning.W1_loan_overlap | — | {"pairs": 32, "players": 31} |  |
| ✅ | §8 | warning.W2_loan_outside_parent | — | {"loans": 1327, "of": 23543} |  |
| ✅ | §8 | warning.W3_multiple_league_spells | 0 | 0 |  |
| ✅ | §8 | warning.W4_profile_mismatch_v3 | — | {"players": 4443, "with_open_spell": 18357, "of_which_status_spells": 627} | v3 definition: latest-started open spell (incl. status, excl. unknown destination and transit) |
| ✅ | §8 | warning.W4_profile_mismatch_v1_legacy | 4338 | 4338 | kg2.py definition (null end_date memberships, unstable sort by start_date) |
| ✅ | S7 | exclusions.rows_excluded | 0 | 0 | r0 loads every row: 94 unknown events as TransferEvents, 6,947 unreferenced Sources with referenced=false |
| ✅ | §11 | count.nodes | {"Player": 40706, "Club": 13599, "UnresolvedClub": 9, "CareerStatus": 5, "League": 32, "Country": 53, "Position": 13, "Source": 365128, "Revision": 1, "Profi... | {"CareerStatus": 5, "Club": 13599, "Country": 53, "League": 32, "LeagueSpell": 796, "Player": 40706, "Position": 13, "ProfileClaim": 40706, "Revision": 1, "S... |  |
| ✅ | §11 | count.edges | {"HAS_CLAIM": 40706, "POINTS_TO": 40706, "FOR_PLAYER": 137776, "FROM": 137776, "TO": 137776, "HAS_SPELL": 114175, "AT": 113379, "LOANED_FROM": 23544, "STARTE... | {"HAS_CLAIM": 40706, "POINTS_TO": 40706, "FOR_PLAYER": 137776, "FROM": 137776, "TO": 137776, "HAS_SPELL": 114175, "AT": 113379, "LOANED_FROM": 23544, "STARTE... |  |
| ✅ | §4 | count.edges.AT_by_label | {"Club": 110621, "CareerStatus": 2758} | {"Club": 110621, "CareerStatus": 2758} |  |
| ✅ | §5.2 | §5.2.table | {"API 2023": [2384, 2367, "2024-07-26", "2024-07-29"], "API 2024": [2122, 2118, "2026-02-27", "2026-05-03"], "API 2025": [11979, 11965, "2026-06-11", "2026-0... | {"API 2023": [2384, 2367, "2024-07-26", "2024-07-29"], "API 2024": [2122, 2118, "2026-02-27", "2026-05-03"], "API 2025": [11979, 11965, "2026-06-11", "2026-0... | players with events = players with a completed or planned event (2 local-cache players have only events of unknown temporal status) |
| ✅ | §5.2 | §5.2.history_incomplete | {"players": 1210, "with_spells": 396} | {"players": 1210, "with_spells": 396} |  |
| ✅ | §6.4 | §6.4.farinez | {"2017-01-01": ["Caracas FC", "Caracas FC", "INFERRED_R12"], "2019-06-01": ["Millonarios FC", "Millonarios FC", "CONFIRMED"], "2025-06-01": ["Rionegro Águila... | {"2017-01-01": ["Caracas FC", "Caracas FC", "INFERRED_R12"], "2019-06-01": ["Millonarios FC", "Millonarios FC", "CONFIRMED"], "2025-06-01": ["Rionegro Águila... |  |
| ✅ | §6.5 | §6.5.population.2020-01-01 | 12865 | 12865 |  |
| ✅ | §6.5 | §6.5.overall.2020-01-01 | [6612, 5694, 479, 5, 3, 72] | [6612, 5694, 479, 5, 3, 72] |  |
| ✅ | §6.5 | §6.5.by_source.2020-01-01.API 2023 | [1705, 56.5, 39.4, 3.5, 0.0, 0.0, 0.5] | [1705, 56.5, 39.4, 3.5, 0.0, 0.0, 0.5] |  |
| ✅ | §6.5 | §6.5.by_source.2020-01-01.API 2024 | [1331, 54.7, 44.0, 0.8, 0.0, 0.0, 0.5] | [1331, 54.7, 44.0, 0.8, 0.0, 0.0, 0.5] |  |
| ✅ | §6.5 | §6.5.by_source.2020-01-01.API 2025 | [8575, 49.6, 45.6, 4.3, 0.0, 0.0, 0.5] | [8575, 49.6, 45.6, 4.3, 0.0, 0.0, 0.5] |  |
| ✅ | §6.5 | §6.5.by_source.2020-01-01.local cache | [1254, 53.2, 41.9, 3.3, 0.2, 0.2, 1.1] | [1254, 53.2, 41.9, 3.3, 0.2, 0.2, 1.1] |  |
| ✅ | §6.5 | §6.5.population.2023-01-01 | 16644 | 16644 |  |
| ✅ | §6.5 | §6.5.overall.2023-01-01 | [8255, 6192, 2007, 137, 10, 43] | [8255, 6192, 2007, 137, 10, 43] |  |
| ✅ | §6.5 | §6.5.by_source.2023-01-01.API 2023 | [2271, 56.9, 25.7, 16.6, 0.7, 0.0, 0.1] | [2271, 56.9, 25.7, 16.6, 0.7, 0.0, 0.1] |  |
| ✅ | §6.5 | §6.5.by_source.2023-01-01.API 2024 | [1868, 56.8, 39.9, 3.1, 0.1, 0.0, 0.2] | [1868, 56.8, 39.9, 3.1, 0.1, 0.0, 0.2] |  |
| ✅ | §6.5 | §6.5.by_source.2023-01-01.API 2025 | [10916, 47.1, 39.2, 12.8, 0.7, 0.0, 0.3] | [10916, 47.1, 39.2, 12.8, 0.7, 0.0, 0.3] |  |
| ✅ | §6.5 | §6.5.by_source.2023-01-01.local cache | [1589, 47.8, 37.0, 11.2, 3.1, 0.4, 0.4] | [1589, 47.8, 37.0, 11.2, 3.1, 0.4, 0.4] |  |
| ✅ | §6.5 | §6.5.population.2025-01-01 | 17993 | 17993 |  |
| ✅ | §6.5 | §6.5.overall.2025-01-01 | [5742, 3889, 5453, 1214, 1659, 36] | [5742, 3889, 5453, 1214, 1659, 36] |  |
| ✅ | §6.5 | §6.5.by_source.2025-01-01.API 2023 | [2367, 0.0, 0.0, 0.0, 32.0, 67.8, 0.2] | [2367, 0.0, 0.0, 0.0, 32.0, 67.8, 0.2] |  |
| ✅ | §6.5 | §6.5.by_source.2025-01-01.API 2024 | [2093, 60.5, 19.9, 19.3, 0.2, 0.1, 0.0] | [2093, 60.5, 19.9, 19.3, 0.2, 0.1, 0.0] |  |
| ✅ | §6.5 | §6.5.by_source.2025-01-01.API 2025 | [11720, 33.1, 25.5, 38.5, 2.5, 0.1, 0.2] | [11720, 33.1, 25.5, 38.5, 2.5, 0.1, 0.2] |  |
| ✅ | §6.5 | §6.5.by_source.2025-01-01.local cache | [1813, 32.7, 26.7, 29.4, 8.7, 2.3, 0.3] | [1813, 32.7, 26.7, 29.4, 8.7, 2.3, 0.3] |  |
| ✅ | §6.5 | §6.5.population.2026-01-01 | 18291 | 18291 |  |
| ✅ | §6.5 | §6.5.overall.2026-01-01 | [1852, 1173, 11751, 1693, 1799, 23] | [1852, 1173, 11751, 1693, 1799, 23] |  |
| ✅ | §6.5 | §6.5.by_source.2026-01-01.API 2023 | [2367, 0.0, 0.0, 0.0, 31.6, 68.2, 0.2] | [2367, 0.0, 0.0, 0.0, 31.6, 68.2, 0.2] |  |
| ✅ | §6.5 | §6.5.by_source.2026-01-01.API 2024 | [2114, 15.0, 5.5, 73.6, 0.4, 5.3, 0.1] | [2114, 15.0, 5.5, 73.6, 0.4, 5.3, 0.1] |  |
| ✅ | §6.5 | §6.5.by_source.2026-01-01.API 2025 | [11930, 10.6, 7.3, 76.7, 5.3, 0.1, 0.1] | [11930, 10.6, 7.3, 76.7, 5.3, 0.1, 0.1] |  |
| ✅ | §6.5 | §6.5.by_source.2026-01-01.local cache | [1880, 14.6, 10.1, 55.8, 16.3, 3.0, 0.2] | [1880, 14.6, 10.1, 55.8, 16.3, 3.0, 0.2] |  |
| ✅ | §6.5 | §6.5.population.2026-09-01 | 18372 | 18372 |  |
| ✅ | §6.5 | §6.5.overall.2026-09-01 | [13, 0, 1396, 11341, 5607, 15] | [13, 0, 1396, 11341, 5607, 15] |  |
| ✅ | §6.5 | §6.5.by_source.2026-09-01.API 2023 | [2367, 0.0, 0.0, 0.0, 31.6, 68.2, 0.2] | [2367, 0.0, 0.0, 0.0, 31.6, 68.2, 0.2] |  |
| ✅ | §6.5 | §6.5.by_source.2026-09-01.API 2024 | [2117, 0.0, 0.0, 0.0, 12.4, 87.6, 0.0] | [2117, 0.0, 0.0, 0.0, 12.4, 87.6, 0.0] |  |
| ✅ | §6.5 | §6.5.by_source.2026-09-01.API 2025 | [11965, 0.0, 0.0, 0.0, 82.9, 17.1, 0.1] | [11965, 0.0, 0.0, 0.0, 82.9, 17.1, 0.1] |  |
| ✅ | §6.5 | §6.5.by_source.2026-09-01.local cache | [1923, 0.7, 0.0, 72.6, 21.6, 5.0, 0.1] | [1923, 0.7, 0.0, 72.6, 21.6, 5.0, 0.1] |  |
| ✅ | §6.5 | §6.5.registration_overlap | {"2026-01-01": 281, "2026-09-01": 295} | {"2026-01-01": 281, "2026-09-01": 295} |  |
| ✅ | §6.5 | §6.5.planned_due_open | {"2026-01-01": 134, "2026-09-01": 2016} | {"2026-01-01": 134, "2026-09-01": 2016} |  |
| ✅ | §6.3 | §6.3.sensitivity | {"2025-01-01 lag=0 A_max=34": [30.3, 16.0], "2025-01-01 lag=14 A_max=34": [30.3, 16.0], "2025-01-01 lag=28 A_max=34": [30.3, 16.0], "2025-01-01 lag=14 A_max=... | {"2025-01-01 lag=0 A_max=34": [30.3, 16.0], "2025-01-01 lag=14 A_max=34": [30.3, 16.0], "2025-01-01 lag=28 A_max=34": [30.3, 16.0], "2025-01-01 lag=14 A_max=... |  |
| ✅ | §6.3 | §6.3.near_horizon | {"2026-06-01 lag=0": 67.7, "2026-06-01 lag=14": 7.4, "2026-06-01 lag=28": 7.4, "2026-09-25 lag=0": 7.6, "2026-09-25 lag=14": 0.0, "2026-09-25 lag=28": 0.0} | {"2026-06-01 lag=0": 67.7, "2026-06-01 lag=14": 7.4, "2026-06-01 lag=28": 7.4, "2026-09-25 lag=0": 7.6, "2026-09-25 lag=14": 0.0, "2026-09-25 lag=28": 0.0} |  |
| ✅ | §12 | §12.active_population | {"active": 16517, "active_complete": 16139, "by_source": {"API 2025": 11068, "API 2023": 1851, "API 2024": 1845, "local cache": 1753}} | {"active": 16517, "active_complete": 16139, "by_source": {"API 2025": 11068, "API 2023": 1851, "API 2024": 1845, "local cache": 1753}} | age < A_max at H_p (kg-schema text says '이하'; the published numbers use '<') |
| ✅ | §6.3 | §12.r3_eligible_players | 17972 | 17972 |  |
| ✅ | §12 | §12.recent_strata | {"2025-07-01": {"spells": 8976, "players": 7922, "players_by_source": {"API 2025": 5926, "API 2024": 1396, "local cache": 600}}, "2026-01-01": {"spells": 327... | {"2025-07-01": {"spells": 8976, "players": 7922, "players_by_source": {"API 2025": 5926, "API 2024": 1396, "local cache": 600}}, "2026-01-01": {"spells": 327... |  |
| ✅ | §13 | §13.players_without_spells | 22334 | 22334 |  |
| ✅ | §13 | §13.status_then_profile_club | 626 | 626 | kg-schema/dataset docs say 625 (date-only order); 626 under the (date, order_in_day) order (player 832896: free transfer then without_club on 2024-07-01) |
| ✅ | §13 | §13.passed_planned | {"passed": 2132, "may_to_sep_2026": 1997, "other": 135} | {"passed": 2132, "may_to_sep_2026": 1997, "other": 135} |  |
