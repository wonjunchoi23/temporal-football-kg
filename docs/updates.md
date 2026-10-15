# Updating the KG (kg-schema §9)

A batch of new information becomes a new **Revision** (r1, r2, …), written as a new directory. The base directory is never modified.

- Changed events, spells and profile claims get new versions. Old versions keep their data, and their `tx_to` is set to the batch time.
- A node that several operations of one batch change gets a single new version.
- Entity labels are updated in place in the new revision: the Player's observation fields, new clubs, unresolved club IDs that become known clubs, and Source records.
- Earlier revisions can still be queried (see [Guarantees](#guarantees)).

```bash
python -m football_kg update --base build/r0 --batch batch.json --out build/r1
python -m football_kg asof 373714 2026-09-01 --build-dir build/r1                  # current
python -m football_kg asof 373714 2026-09-01 --build-dir build/r1 --revision r0    # as the KG knew it at r0
```

No real r1 data exist yet (kg-schema §14). The operations are tested with synthetic batches built from the r0 graph (`tests/test_updates.py`).

## Batch format

```json
{
  "revision_id": "r1",
  "recorded_at": "2027-01-21T00:00:00+00:00",
  "origin": "local_api_cache refresh",
  "responses": [{
    "player_id": "373714",
    "status": "valid_nonempty",
    "source": {"source_type": "local_api_cache", "raw_locator": "data/raw/transfer_history_r1/373714.json",
               "retrieved_at": "2027-01-20T09:00:00+00:00", "content_sha256": null},
    "transfers": [
      {"pointer": 0, "transfer_id": "6106493", "date": "2026-12-31", "season": "26/27",
       "from": {"club_id": "17425", "name": "Internacional de Bogotá", "is_special": false},
       "to": {"club_id": "531", "name": "Caracas FC", "is_special": false},
       "fee": "End of loan", "future_transfer": 0, "upcoming": false}
    ]
  }],
  "snapshots": [{"player_id": "373714", "club_id": "531", "observed_at": "2026-09-01"}],
  "retractions": []
}
```

- `recorded_at` is the batch time. It needs a UTC offset and must be later than the base revision's `recorded_at`. This is not checked, and an earlier time corrupts the views of earlier revisions.
- A response lists the player's **whole** history, newest first (`pointer` 0 is the latest), in the same shape as the raw Transfermarkt API.
  - The example shows only the newest transfer to stay short.
  - Applied as it is, the omitted completed events are kept and logged as `missing_in_refresh`, but `player_observation` records a history of one transfer.
  - `refreshed()` in `tests/update_fixtures.py` builds complete responses from the r0 graph.
- `status` is `valid_nonempty`, `valid_empty` or `unavailable`. An `unavailable` response changes nothing and is only logged, because absence is not a negative fact.
- `source.retrieved_at` needs a UTC offset. Its date becomes the player's new observation time.
- A snapshot is a dated profile observation: `player_id`, `club_id`, `observed_at` and an optional `source`.
- A retraction is `{"source_id": "<Source id>"}`.

## What happens to each response (`refresh_history`)

1. **Classify.**
   - Event type: fee labels and special destinations, using the dataset's rules.
   - Temporal status: an event is completed only with explicit `future_transfer = 0`, `upcoming = false` and a date on or before the retrieval day.
   - Both rules reproduce all 137,776 r0 events.
2. **Match each transfer to a known event**, first by `transfer_id`, then by (player, from, to) with dates at most N = 31 days apart. Matched pairs are handled as follows; no change means no new version, and the new response is recorded as an extra source.

   | Situation | Operation |
   |---|---|
   | planned → completed | `confirm_planned` (the date shift is logged) |
   | planned → planned on another date | `reschedule_planned` |
   | unknown → completed or planned | `resolve_unknown` |
   | cancelled → listed again | `reinstate` (a conflict is logged) |
   | changed completed event | correction |
   | completed → listed as planned | the completed version is kept and a conflict is logged |
   | unmatched transfer | `add_event` |

3. **Planned events the response no longer confirms** are voided (`temporal_status = cancelled`, with a `cancel_reason`) in these cases:
   - `contradicted`: a newly completed move leaves the same club for another club, after the previous H_p and no later than the planned date + N
   - `rescheduled`: a new planned event has the same route and a different date
   - `expired`: the new H_p is more than N days past the planned date

   A planned event still listed as upcoming after its date stays planned and is logged as `planned_overdue`. Completed events missing from a refresh are kept.
4. **Recompute the player's spells** with the r0 code:
   - The dataset's membership rules run only for the events this batch adds or changes: new, newly completed or corrected. They reproduce all 110,621 r0 memberships, 45,403 ends and 704 withheld closures. A withheld or ambiguous closure is logged as a conflict, and R1/R2 still apply.
   - Status spells, R1/R2, transit spells and `last_confirmed_at` are recomputed.
   - Changed spells get new versions. Spells whose start event is no longer completed get a `void` tombstone. A void spell that is derived again gets a new active version.
5. **Stale profile claim.** A completed departure from the profile's club dated after the profile was observed (2026-10-08) creates a ProfileClaim version with `stale = true`.
6. **Observation time.** The player's observation time moves to the retrieval date. This is a new `player_observation` version, and the Player node is updated in place and logged.

New nodes get these IDs: an added event `te_kg_<hash of player and transfer_id>`, a new spell `spell_<start event id>`, a new source `ksrc_<hash>`.

The other operations:

- `add_snapshot` adds a dated ProfileClaim version and logs a conflict if it disagrees with the as-of answer. When a refresh in the same batch has already marked the claim stale, the snapshot replaces that version's content, so the claim still gets one new version.
- `retract` withdraws a source:
  - Its Source record gets `retracted_in`.
  - Events whose own evidence (temporal status and raw transfer evidence) relied only on it become `unknown` (`raw_match_status = retracted`), and the player's spells are recomputed.
  - Nodes that do not change keep their support edges to the source.
  - A retraction that would leave an event with no source at all is rejected by §8 reject rule 6, and the batch is not applied.

## Guarantees

- **Idempotent.** A batch whose content hash was already applied is a no-op. Re-sending unchanged histories creates no new versions.
- **Time travel.**
  - Edges carry `[tx_from, tx_to)`.
  - Edges into a superseded version are closed and re-created against the new version, so a revision view's edges connect nodes of that view. There are two exceptions:
    - `SUPERSEDES` links a version to the one it replaced, which is no longer current.
    - After an unresolved club ID becomes a known club, earlier views keep the profile claim's `POINTS_TO` edge to it but no longer have its `UnresolvedClub` node.
  - The r0 view of an updated store (`--revision r0`) has the r0 events, spells, profile claims, edges and observation times, so it gives the r0 as-of answers. Entity labels show their current values, for example a Player's `history_observed_at`. For the exact r0 state, load the r0 directory.
- **Checked.**
  - The §8 reject rules run on the new current view.
  - Rule 7's "H_p < planned date" is a warning for update batches, because of overdue planned events. It is reported in the revision's `reports/build_checks.json`.
  - With `apply_to_dir` (which the CLI uses), the §8 warnings are recomputed on the new current view. Warnings not yet in the conflict log are added with the new revision id. Warnings that persist keep their first revision id, and resolved ones stay in the log.
- **Logged.** Operations on events, spells, profile claims, clubs and players are recorded in `tables/revision_log.parquet`: operation, label, key and changed fields. New Source records are added without a log entry.
