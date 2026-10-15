# Football temporal knowledge graph

A temporal knowledge graph of the club careers of 40,706 football players whose primary citizenship is one of the FIFA top-50 countries. It is built from a Transfermarkt-derived relational release (`final_v3`, 2026-10-08) and implements the design in [kg-schema.md](kg-schema.md) (v3, in Korean).

The graph answers **"where did player *p* play on date *t*, how sure is that, and why?"** Every answer comes with:

- a validity interval
- the player's own observation horizon
- a certainty grade
- the source records behind it

New information becomes a new revision instead of overwriting the old one: changed events, spells and profile claims get new versions, and each revision keeps its own directory. Earlier states of the KG can therefore be queried again.

Release **r0** (built 2026-10-12) contains:

| | Count |
|---|---:|
| Nodes | 712,203 (Player 40,706 · TransferEvent 137,776 · Spell 113,379 · Source 365,128 · Club 13,599 · …) |
| Relationships | 2,849,958 (SUPPORTED_BY 1,523,356 provenance links · FOR_PLAYER/FROM/TO · HAS_SPELL/AT/STARTED_BY/ENDED_BY · …) |
| Validation | 182 build and validation checks recompute the kg-schema.md v3 numbers from the built graph, and all pass. The derivation also matches the reference scripts spell by spell. |

## Quick start

### 1. Get the files

Clone the repository. Everything is in it (about 185 MB):

| Folder | Contents |
|---|---|
| `build/r0/kg/` | the KG as Parquet tables (nodes, edges, side tables) and `manifest.json` with each file's sha256 |
| `build/r0/reports/` | the build checks and the validation report |
| `build/r0/eval/` | the evaluation material: answer timelines, gold pool, verification sample, QA items |
| `dataset-gz/` | the 13 input CSVs, gzip-compressed (two exceed GitHub's 100 MB limit uncompressed), for rebuilding |

### 2. Query in Python (no database needed)

```bash
pip install -e .                                    # Python >= 3.10 (pandas, pyarrow, numpy)
python -m football_kg asof 373714 2026-03-01        # JSON answer
python -m football_kg context 373714 2026-03-01     # the text block given to an LLM (kg-schema §6.6)
```

```python
from football_kg.kg import KG
from football_kg.query.asof import AsOfEngine

engine = AsOfEngine(KG.load("build/r0"))            # lag=14 days, a_max=34 by default
answer = engine.as_of("373714", "2026-03-01")
answer["club_id"], answer["registered_club_id"], answer["category"]   # ('17425', '531', 'INFERRED_R3')
```

More examples, including batch queries and answer timelines, are in [docs/queries.md](docs/queries.md).

### 3. Rebuild from the dataset

```bash
for f in dataset-gz/*.csv.gz; do gunzip -c "$f" > "dataset/$(basename "$f" .gz)"; done   # restore the 13 CSVs
(cd dataset && sha256sum --check SHA256SUMS)              # they match the release checksums
python -m football_kg build --out build/rebuild           # about 40 s; build/r0 stays untouched
python -m football_kg validate build/rebuild              # build/rebuild/reports/validation_report.md
python -m football_kg eval --build-dir build/rebuild      # build/rebuild/eval
pytest && pytest -m slow                                  # unit tests, then full-data tests (about 10 min)
```

Build into a separate folder as above. A Parquet file records the pyarrow version that wrote it, so a rebuild with other library versions differs byte for byte from the committed `build/r0`. The table contents are the same, except the code version (git commit) recorded on the Revision node.

## Reading the answers

| Grade | Meaning |
|---|---|
| `CONFIRMED` | t lies inside a recorded interval, or the registration was confirmed on or after t (a later loan-out or a matched loan return) |
| `INFERRED (R1/R2)` | the end of the spell is inferred: the player left the club (R1) or the next spell started (R2) |
| `INFERRED (R3)` | the spell is open, but the history is complete, t is more than 14 days before the player's observation horizon, and the player's birth date is known and they were under 34 at t |
| `OPEN_PROFILE` | an open spell that the player's (undated) profile also points to |
| `OPEN` | an open spell with nothing to back it up, or a planned departure that was due and never confirmed |
| `NONE` | no spell covers t: no history, a gap, or an unknown destination |

Keep in mind:

- **Absence is not a negative fact.** 22,334 players have no spell, so they always get `NONE`: 21,476 have no available transfer history, 40 have a verified empty one, and 818 have a history in the source but no completed transfer in the release. None of this means they had no club.
- **Each player with an available history has an observation horizon** (`history_observed_at`, between 2024-07-26 and 2026-10-08; 19,230 players). The KG knows nothing about that player after this date.
- **League answers are snapshots.** The only league data is the latest club→league link, so player-league answers are graded `SNAPSHOT`.
- **Provisional and placeholder fields:**
  - `Club.team_level` (youth / reserve / first team) is a provisional name pattern.
  - The popularity stratum in the evaluation files is a placeholder; the metric is still pending (kg-schema §14).

## Documentation

| Document | Contents |
|---|---|
| [docs/model.md](docs/model.md) | The model as built: labels, properties, time model, grades, versioning, provenance, constraints, and every clarification of kg-schema v3 |
| [docs/queries.md](docs/queries.md) | Python query examples |
| [docs/updates.md](docs/updates.md) | Update batches (kg-schema §9): format, rules, revisions and time travel |
| [kg-schema.md](kg-schema.md) | The agreed design, v3 (Korean) |
| [dataset-description.md](dataset-description.md) | The input dataset, table by table |
| [reference/schema_v3](reference/schema_v3) | The scripts that produced the kg-schema v3 numbers (used as a test oracle) |

## Repository layout

```
src/football_kg/   build/ (S1-S7 pipeline) · query/ (as-of, timeline, context) · update/ (§9) · evalset.py · validate.py · cli.py
tests/             rule tests on synthetic careers + slow full-data tests (reference parity, determinism, §9 scenarios)
reference/         the scripts behind the kg-schema v3 numbers (test oracle; not edited)
build/r0/          the built KG (kg/), its reports and the evaluation material (eval/)
dataset/           final_v3 documentation and checksums (restore the CSVs from dataset-gz/, see Quick start)
dataset-gz/        the 13 final_v3 CSVs, gzip-compressed
```

## Data and attribution

The data come from Transfermarkt, via the transfermarkt-datasets project and the Transfermarkt API, and contain personal data of real people (names, birth dates and birthplaces). This repository and its releases are for academic use within the course. Do not redistribute them publicly.
