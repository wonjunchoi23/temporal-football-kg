# FIFA Top 50 canonical relational database — final v3

Release/audit reference date: **2026-10-08**. Country scope: the user-provided FIFA ranking dated **2026-10-07**, 50 countries, **40,706 primary-citizenship players**. No league filter, no secondary-citizenship expansion, no external requests. `final_v2` and its raw/audit inputs remain unchanged.

This release fixes only the scheduled-membership and missing-provenance failures. All original curated transfer fields and fee/type classifications remain unchanged. The original transfer_events.membership_type column remains unknown; use normalized_membership_type. Paid transfers remain unknown membership. Unavailable raw history is never interpreted as empty.

## Temporal evidence and historical membership

`event_temporal_status` is independent of event_type and membership_type. Explicit futureTransfer=1 or upcoming=true yields planned even if the calendar date has passed. A future date is also a planned guard and never completed. Completion requires both explicit source negatives (futureTransfer=0 and upcoming=false), an exact source date, and a date no later than the release reference. Missing/ambiguous raw evidence remains unknown. Completed denotes the API snapshot’s explicit non-future/non-upcoming observation; no independent real-world confirmation or new retrieval is claimed.

Only completed events participate in historical membership starts **or** closures. Planned/unknown events cannot disambiguate or close a loan. Loan return terminates a unique loan with matching player, borrower, and parent; it does not create a permanent start. Free transfer supports permanent membership; paid/unsupported fees do not. Same-day order, missing endpoints, and conflicting closed intervals remain unresolved. Current profile-club references stay separate and undated.

## Provenance correction

All 796 absent raw-club source records are removed from the canonical ledger and retained unchanged in reports/top50/final_v3_provenance_quarantine.csv. All 1,592 affected club facts have equivalent existing DuckDB evidence whose club ID and value were checked. Those equivalent links already existed, so broken duplicate evidence was removed without inventing paths or URLs. fact_sources.provenance_status explicitly distinguishes resolved from unresolved; null source_id is permitted only for unresolved facts. This release has zero unresolved facts. Every retained local locator must exist and resolve to its row/event. Original URLs are retained and syntax-checked offline; no network verification is claimed.

## Tables

| Table | Rows | Purpose |
|---|---:|---|
| [players.csv](players.csv) | 40,706 | Top 50 primary-citizenship player profiles, regardless of league. |
| [positions.csv](positions.csv) | 13 | Deterministic vocabulary of observed broad/specific position pairs. |
| [player_positions.csv](player_positions.csv) | 40,238 | Associate players with retained position observations. |
| [clubs.csv](clubs.csv) | 13,613 | Registry of referenced club identifiers with supported curated or observed metadata. |
| [leagues.csv](leagues.csv) | 32 | Referenced competition identities, retaining canonical Transfermarkt competition codes. |
| [club_league_current.csv](club_league_current.csv) | 796 | Currently retained/latest curated club-to-competition association. |
| [transfer_events.csv](transfer_events.csv) | 137,776 | All original curated transfers plus validated, exactly matched raw classifications and evidence. |
| [player_club_memberships.csv](player_club_memberships.csv) | 110,621 | Historical destination-club membership observations with conservative evidence-backed starts and optional ends. |
| [player_current_clubs.csv](player_current_clubs.csv) | 40,706 | One separate latest player-profile club reference for every player, independent of historical coverage. |
| [player_transfer_history_status.csv](player_transfer_history_status.csv) | 40,706 | Explicit canonical raw-history availability for every player. |
| [source_documents.csv](source_documents.csv) | 365,128 | Source-document/record ledger for curated, raw, configuration, and local audit evidence. |
| [fact_sources.csv](fact_sources.csv) | 1,462,416 | Lightweight fact-to-source links for entities and associations. |
| [planned_transfer_events.csv](planned_transfer_events.csv) | 2,562 | Explicit planned-event references; original events remain in transfer_events. |

### players

Purpose: Top 50 primary-citizenship player profiles, regardless of league.

Primary key: player_id.

Foreign keys: current_club_id → clubs.club_id.

Important fields: country_of_citizenship; date_of_birth and birth_date_precision; birth location; current_club_id.

Provenance: source_url; fact_sources(player, player_id) points to retained curated player records.

Known limitations: Secondary citizenship is not used. Birth precision is explicit. A profile club reference can be a special status and does not establish historical dates.

### positions

Purpose: Deterministic vocabulary of observed broad/specific position pairs.

Primary key: position_id.

Foreign keys: None.

Important fields: position; sub_position.

Provenance: fact_sources(position, position_id) links the normalized vocabulary to profile evidence.

Known limitations: Only retained profile positions are represented; this is not a season-by-season position history.

### player_positions

Purpose: Associate players with retained position observations.

Primary key: (player_id, position_id).

Foreign keys: player_id → players.player_id; position_id → positions.position_id.

Important fields: player_id; position_id.

Provenance: source_url; fact_sources(player_position, player_id:position_id).

Known limitations: No position dates are invented; players with missing position data have no association row.

### clubs

Purpose: Registry of referenced club identifiers with supported curated or observed metadata.

Primary key: club_id.

Foreign keys: current_domestic_competition_id → leagues.league_id (nullable).

Important fields: club_name; metadata_status; club_name_scope; entity_kind; observed_names (JSON array).

Provenance: source_url where available; fact_sources(club, club_id) retains name observations or identifier references.

Known limitations: Includes 5 special-status identifiers and 9 unresolved identifiers. Transfer-label-only names do not assert current legal names or historical leagues. Missing metadata stays null.

### leagues

Purpose: Referenced competition identities, retaining canonical Transfermarkt competition codes.

Primary key: league_id.

Foreign keys: country_id refers to an upstream country identifier; no countries table is released..

Important fields: league_name; country fields; competition_type/sub_type; metadata_status.

Provenance: source_url; fact_sources(league, league_id), including existing country/configuration evidence.

Known limitations: COL1 has documented identity (Liga BetPlay, Colombia) but no supported competition type/subtype. Country references are external, not enforced release foreign keys.

### club_league_current

Purpose: Currently retained/latest curated club-to-competition association.

Primary key: (club_id, league_id); club_id also unique in this snapshot.

Foreign keys: club_id → clubs.club_id; league_id → leagues.league_id.

Important fields: club_id; league_id.

Provenance: source_url; fact_sources(club_league_current, club_id:league_id).

Known limitations: This is never a historical club-to-league mapping and covers only clubs with retained curated competition metadata.

### transfer_events

Purpose: All original curated transfers plus validated, exactly matched raw classifications and evidence.

Primary key: transfer_event_id.

Foreign keys: player_id → players.player_id; from_club_id/to_club_id → clubs.club_id (nullable); raw_source_id → source_documents.source_id (nullable).

Important fields: Original curated fields; event_type; normalized_membership_type; raw_match_status; original_fee versus original_transfer_fee; event_temporal_status; raw_future_transfer; raw_upcoming; temporal_classification_rule; temporal_as_of_date.

Provenance: source_locator; transfer_url; original_fee/original_fee_json; classification_rule; raw_source_type/file/id/event_locator/sha256; fact_sources(transfer_event, transfer_event_id).

Known limitations: membership_type is the unchanged original curated column: use normalized_membership_type. paid_transfer remains unknown membership. 86 unmatched and 8 multiply matched records remain unsupported; no fuzzy matching or zero-fee inference.

### player_club_memberships

Purpose: Historical destination-club membership observations with conservative evidence-backed starts and optional ends. Only completed raw events are eligible.

Primary key: membership_id.

Foreign keys: player_id → players.player_id; club_id/parent_club_id → clubs.club_id; source_transfer_event_id/end_source_transfer_event_id → transfer_events.transfer_event_id; source_id/end_source_id → source_documents.source_id (parent/end fields nullable).

Important fields: membership_type; parent_club_id; start/end dates and precision; source_supported_membership_type; derived_membership_type; confidence_status.

Provenance: source_transfer_event_id/source_id; end_source_transfer_event_id/end_source_id; derivation_method/end_derivation_method; fact_sources(player_club_membership, membership_id). Raw fee/URL/file provenance is available by joining the source event.

Known limitations: Loan returns close unique prior loans rather than create permanent starts. Permanent starts here require explicit free-transfer labels. Paid/unsupported transfers remain unknown. Null end dates do not assert continuous or ongoing careers; ambiguous closures are withheld.

### player_current_clubs

Purpose: One separate latest player-profile club reference for every player, independent of historical coverage.

Primary key: current_relation_id; player_id also unique.

Foreign keys: player_id → players.player_id; club_id → clubs.club_id; source_id → source_documents.source_id.

Important fields: relation_kind; temporal_scope; player_id; club_id; start_date/end_date (both null).

Provenance: Player-profile source_url/source_id; fact_sources(player_current_club, current_relation_id).

Known limitations: A current reference is not a dated historical membership or a real-time assertion. special_status_reference must not be converted into a normal club edge.

### player_transfer_history_status

Purpose: Explicit canonical raw-history availability for every player.

Primary key: player_id.

Foreign keys: player_id → players.player_id.

Important fields: transfer_history_status; raw_transfer_event_count; curated_transfer_event_count.

Provenance: raw_source_type/file when available; fact_sources(player_transfer_history_status, player_id) links valid responses or local inventory audit evidence.

Known limitations: unavailable is not verified empty. raw_transfer_event_count is null for unavailable histories; only valid_empty has a verified zero. Zero curated events does not prove an empty real career.

### source_documents

Purpose: Source-document/record ledger for curated, raw, configuration, and local audit evidence.

Primary key: source_id.

Foreign keys: None.

Important fields: source_type; source_url; raw_locator; retrieved_at; content_sha256; extraction_method.

Provenance: This table is the evidence ledger itself; each source has at least a URL or a raw locator.

Known limitations: A record may represent a source row/event rather than a whole webpage. Relative hrefs stay relative. Unknown historical retrieval times and retained original hashes remain null; release time is not retrieval time.

### fact_sources

Purpose: Lightweight fact-to-source links for entities and associations.

Primary key: (entity_type, entity_id, fact_name, source_id).

Foreign keys: source_id → source_documents.source_id; entity_type/entity_id form a validated polymorphic reference (see schema.md).

Important fields: entity_type; entity_id; fact_name; source_id; provenance_status.

Provenance: Each row explicitly identifies the source supporting a named fact or fact bundle.

Known limitations: Polymorphic entity references require application-level validation. Complete evidence linkage does not mean complete metadata or exact retrieval times. Missing raw-club sources were quarantined, not retained as valid evidence.

### planned_transfer_events

Purpose: retain scheduled observations separately while preserving their full original transfer_events rows. Primary key and foreign key: transfer_event_id → transfer_events. player_id → players; raw_source_id → source_documents. Important fields: planned status, original temporal flags, scheduled date, raw locator. Provenance: raw evidence and fact_sources(planned_transfer_event, transfer_event_id). Limitation: a planned date passing does not prove execution; never turn this table into completed historical edges.

## Limitations

Raw coverage remains 19,190 valid nonempty, 40 verified empty, and 21,476 unavailable histories. Unavailable players have profile-based current-club references; absence does not prove an empty career. Primary citizenship may miss secondary nationalities. Snapshot years are acquisition years, not event-year filters or retrieval timestamps. Club labels may be observed transfer labels, the identifier registry includes special statuses, and league associations are latest retained only. COL1 identity is supported but competition-type metadata remains incomplete. Null membership ends do not prove continued membership.

## Reproduction and validation

Use the existing local environment and exactly the protected final_v2/raw inputs. The builder refuses to overwrite an existing v3. To reproduce the database, use a separate project copy without final_v3; no downloading or collector is needed.

```bash
.venv/bin/python -B src/build_top50_final_v3.py --as-of 2026-10-08
.venv/bin/python -B src/package_top50_final_v3.py
.venv/bin/python -B src/validate_top50_final_v3.py
```

For an existing release, run only the independent validator and checksum check:

```bash
.venv/bin/python -B src/validate_top50_final_v3.py
(cd data/top50/final_v3 && sha256sum --check SHA256SUMS)
```

Full per-record changes: reports/top50/final_v3_changes.csv. The wildcard schema-change entry documents provenance_status on every retained fact link; all removed/repaired/new evidence records and every temporal event are listed individually. Final independent validation: reports/top50/final_v3_validation.json. Correction report: reports/top50/final_v3_correction_report.md. Checksums cover the 13 database tables and data_dictionary.csv. build_manifest.json protects v2, raw files, caches, existing audit outputs, and the new data tables. Unknown historical retrieval times remain null.

See [schema.md](schema.md), [data_dictionary.csv](data_dictionary.csv), and [database_statistics.json](database_statistics.json).
