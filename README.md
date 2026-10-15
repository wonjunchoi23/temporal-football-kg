# Football temporal knowledge graph

A temporal knowledge graph of the club careers of 40,706 football players whose primary citizenship is one of the FIFA top-50 countries. It is built from a Transfermarkt-derived relational release (`final_v3`, 2026-10-08) and implements the design in [kg-schema.md](kg-schema.md) (v3).

The graph answers **"where did player *p* play on date *t*, how sure is that, and why?"** Every answer comes with a validity interval, the player's own observation horizon, a certainty grade and the source records behind it.

This repository holds the built KG (revision **r0**, built 2026-10-12), its input dataset and the documentation, all readable with pandas or any other Parquet reader. The Python package that built the KG (`football_kg`) is not included.

| | Count |
|---|---:|
| Nodes | 712,203 (Player 40,706 · TransferEvent 137,776 · Spell 113,379 · Source 365,128 · Club 13,599 · …) |
| Relationships | 2,849,958 (SUPPORTED_BY 1,523,356 provenance links · FOR_PLAYER/FROM/TO · HAS_SPELL/AT/STARTED_BY/ENDED_BY · …) |
| Validation | 182 checks recompute the kg-schema v3 numbers from the built graph, and all pass ([validation report](build/r0/reports/validation_report.md)). |

## What is in this repository

| Folder | Contents |
|---|---|
| `build/r0/kg/` | Parquet files per node label (`nodes/`) and per edge type and endpoint-label pair (`edges/`), side tables (`tables/`), and `manifest.json` with each file's rows, columns and sha256 |
| `build/r0/eval/` | `answer_timeline.parquet` (precomputed answers, see below), the gold pool, 1,000 QA items and a verification sample |
| `build/r0/reports/` | The build checks and the validation report |
| `dataset/` | The input release's README, schema, data dictionary, statistics, build manifest and checksums |
| `dataset-gz/` | The 13 input CSVs, gzip-compressed (two exceed GitHub's 100 MB limit uncompressed) |

## Quick start (pandas only)

```bash
pip install pandas pyarrow
```

```python
import pandas as pd

players = pd.read_parquet("build/r0/kg/nodes/Player.parquet")
clubs = pd.read_parquet("build/r0/kg/nodes/Club.parquet")      # club_id, name, aliases, ...
spells = pd.read_parquet("build/r0/kg/nodes/Spell.parquet")    # who was at which club, from when to when

# "Where did player p play on date t?"
t = pd.read_parquet("build/r0/eval/answer_timeline.parquet")
day = pd.Timestamp("2026-03-01")
start, end = pd.to_datetime(t.seg_from), pd.to_datetime(t.seg_to)
a = t[(t.player_id == "373714") & (start <= day) & (end.isna() | (day < end))]
a[["club_id", "registered_club_id", "category"]]
# 17425 (Internacional de Bogotá, on loan), 531 (Caracas FC), INFERRED_R3
```

- `answer_timeline.parquet` gives the answer for every date as segments `[seg_from, seg_to)`; an empty `seg_to` means "until further notice". It covers the 18,372 players with at least one spell, with the default parameters (lag 14 days, A_max 34 years). Any other player gets `NONE` on every date.
- `answer_kind` is `club`, `status` (without club, retired, career break or ban; see `status_code`) or `none`. On a loan, `club_id` is the club the player played for and `registered_club_id` the parent club.
- Club names are in `Club.parquet`, status IDs (`at_id` of a status answer) in `CareerStatus.parquet`.
- The answers were computed from the stored spells and events by the `football_kg` package, using the rules in [docs/model.md](docs/model.md) ("As-of rules").

To restore the input CSVs:

```bash
for f in dataset-gz/*.csv.gz; do gunzip -c "$f" > "dataset/$(basename "$f" .gz)"; done
(cd dataset && sha256sum --check SHA256SUMS)    # the 13 CSVs and data_dictionary.csv: OK
```

`dataset/data_dictionary.csv` defines every column. `fact_sources.csv` (1.46M rows) and `source_documents.csv` (365k rows) are large; load them with pyarrow or in chunks.

## Reading the answers

| Grade | Meaning |
|---|---|
| `CONFIRMED` | t lies inside a recorded interval, or the registration was confirmed on or after t (a later loan-out or a matched loan return) |
| `INFERRED (R1/R2)` | the end of the spell is inferred: the player left the club (R1) or the next spell started (R2) |
| `INFERRED (R3)` | an open spell with a complete history, t more than 14 days before the player's observation horizon, and a known birth date with the player under 34 at t |
| `OPEN_PROFILE` | an open spell that the player's (undated) profile also points to |
| `OPEN` | an open spell with nothing to back it up, or a planned departure that was due and never confirmed |
| `NONE` | no spell covers t: no history, a gap, or an unknown destination |

Keep in mind:

- **Absence is not a negative fact.** 22,334 players have no spell and always get `NONE`: 21,476 have no available transfer history, 40 have a verified empty one, and 818 have no completed transfer in the release. None of this means they had no club.
- **Each player with an available history (19,230) has an observation horizon** (`history_observed_at`, 2024-07-26 to 2026-10-08). The KG knows nothing about the player after it.
- **League answers are snapshots** (`SNAPSHOT`): the only league data is the latest club→league link.
- **Provisional fields:** `Club.team_level` (youth / reserve / first team) is a name pattern, and the popularity stratum in the evaluation files is a placeholder (kg-schema §14).

## Documentation

| Document | Contents |
|---|---|
| [docs/model.md](docs/model.md) | The model as built, including every clarification of kg-schema v3 |
| [kg-schema.md](kg-schema.md) | The agreed design, v3 |
| [dataset-description.md](dataset-description.md) | The input dataset, table by table |
| [dataset/README.md](dataset/README.md) | The release's own notes and per-table limitations |
| [docs/queries.md](docs/queries.md), [docs/updates.md](docs/updates.md) | The query API and update batches of the `football_kg` package (not in this repository) |

## Data and attribution

The data come from Transfermarkt (via the transfermarkt-datasets project and the Transfermarkt API) and contain personal data of real people (names, birth dates and birthplaces). This repository is for academic use within the course. Do not redistribute it publicly.
