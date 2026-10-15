# Querying the KG

## Python

```python
from football_kg.kg import KG
from football_kg.query.asof import AsOfEngine

kg = KG.load("build/r0")                  # current versions; KG.load(path, revision="r0") for an older revision
engine = AsOfEngine(kg, lag_days=14, a_max=34)
```

### One player, one date

```python
a = engine.as_of("373714", "2026-03-01")   # Wuilker Fariñez (kg-schema §6.4)
a["club_id"], a["registered_club_id"]      # '17425' (Internacional de Bogotá, on loan), '531' (Caracas FC)
a["category"], a["rule"]                   # 'INFERRED_R3', 'R3'
a["valid_from"], a["valid_to"]             # '2026-01-08', None
a["history_observed_at"]                   # '2026-06-11': nothing is known about this player after this date
a["planned_pending"]                       # [{'event_id': 'transfer_997a…', 'date': '2026-12-31', 'to_club_id': '531', 'due': False}]
a["sources"]                               # source records: start event, end/inference basis, history response
```

The answer also contains:

- `answer_kind`: club, status or none
- `status_code`: for status answers
- `none_reason`: no_spells, before_first_spell, no_covering_spell or unknown_destination
- `registration_conflict`: true when two registration spells overlap at t
- the spell and event IDs

### Many queries at once (vectorized)

```python
import pandas as pd
q = pd.DataFrame({"player_id": ["373714", "10"], "t": ["2019-06-01", "2010-01-01"]})
engine.as_of_many(q)[["player_id", "t", "club_id", "category", "rule"]]

engine.grade_table("2026-01-01").category.value_counts()   # the §6.5 distribution for one date
```

### Whole careers as answer segments

```python
from football_kg.query.timeline import timeline, lookup
seg = timeline(engine, players=["373714"])   # piecewise-constant answers [seg_from, seg_to)
lookup(seg, "373714", "2025-06-01")          # same result as engine.as_of(...)
```

### Text for an LLM, and the league

```python
from football_kg.query.context import render_context, league_as_of
print(render_context(engine, "373714", "2026-03-01"))          # lang="ko" for Korean labels
league_as_of(engine, "373714", "2026-03-01")                   # {'league_id': 'COL1', ..., 'grade': 'SNAPSHOT'}
```

```
[KG as-of 2026-03-01] Wuilker Fariñez (player 373714)
Playing for      Internacional de Bogotá (on loan)
Registered with  Caracas FC
Interval         2026-01-08 → no recorded end
Grade            INFERRED (R3)
Observed         history observed 2026-06-11 (API 2025); nothing is known after this date
Planned          2026-12-31 → Caracas FC (planned, not applied)
Sources          start: src_c10955af2882cc9b7fd181f0, history_response: src_b4307d5d5351db95fa4b76eb
```

### Evaluation material (kg-schema §12)

```python
from football_kg.evalset import gold_pool, qa_items, score, active_population, recent_strata
pool = gold_pool(engine)                   # CONFIRMED / INFERRED segments with t < H_p - lag, complete histories
items = qa_items(engine, pool, n=500, seed=1)
# an answer matches by club ID or any alias, compared after HTML-unescaping and case-folding
score(items, {"q00000": "Troyes"})         # {'n': 500, 'accuracy': 0.002, ...}: unanswered items count as wrong
```

`python -m football_kg eval` writes the following to `build/r0/eval/`:

- `answer_timeline.parquet` and `gold_pool.parquet`
- `verification_sample.csv`: 50 rows per grade, plus three oversampled risk groups, with empty columns for manual checks
- `qa_default.jsonl`: 1,000 questions

### Plain tables

All tables are Parquet files. pandas (or pyarrow, polars) can read them directly:

```python
import pandas as pd
spell = pd.read_parquet("build/r0/kg/nodes/Spell.parquet")
spell.type.value_counts()
sb = pd.read_parquet("build/r0/kg/edges/SUPPORTED_BY__Spell__Source.parquet")
sb[sb.src == "membership_2f7df71a628c78652953566b@v1"]
```

`football_kg.io.read_parquet(path)` returns the same tables the way the package uses them: ISO date strings and nullable integer and boolean columns.
