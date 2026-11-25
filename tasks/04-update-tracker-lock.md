# 04 · Database: stop `table_update_tracker` from serializing writers

> **Solves:** Lets concurrent submissions of the same organism run in parallel by removing the long-lived row lock taken by the update-tracker trigger.

| Field | Value |
|---|---|
| Issue | [#7167](https://github.com/loculus-project/loculus/issues/7167) (context: PR #7165) |
| Component | backend (Postgres) |
| PR title | `perf(backend): avoid long-lived row locks from the table update tracker` |
| Effort | M (design + 2–4 days) |
| Risk | **Medium–high**: SILO, the website and ETag-based caching depend on "last modified" |
| Status | proposal, discuss with maintainers first · 🟡 partial |

## Existing work (checked 2026-11-23)
Issue #7167 is open and unassigned, with no fix PR. #7551 (merged, lease-based preprocessing work queue) and migrations V1.28/V1.32 narrowed the lock scope to `(table, organism, pipeline_version)` but didn't remove it. Bring a design proposal.

## Problem
Concurrent submissions of the **same organism** are serialized. A 100k-sequence upload holds the tracker row for minutes, and a second submission waits behind it.

## Root cause (verified)
- `V1.2__add_table_update_tracker.sql` added a `FOR EACH STATEMENT` trigger that upserts a row in `table_update_tracker`.
- `V1.28__scope_update_tracker_by_organism_and_pipeline_version.sql` and `V1.32` scoped the key to `(table_name, organism, pipeline_version)`. That reduced the contention but didn't remove it: the first write in a transaction locks that row until commit.
- `UploadDatabaseService.mapAndCopy` is a single transaction over a whole upload.
- Rows without an organism (`metadata_upload_aux_table`, `sequence_upload_aux_table`, `groups_table`, `user_groups_table`) are shared by **every** organism. A future long transaction touching them would serialize submissions across all organisms.
- The reader is `ReleasedDataModel.kt:122-136`, which computes `lastTimeUpdated` for ETags.

## Options
| Option | Idea | Trade-off |
|---|---|---|
| A. Append-only change log | The trigger inserts a new row (no upsert), and readers take `max(ts)` per key; a periodic job compacts it | No lock conflicts. The table grows until compacted |
| B. Deferred update | Record "dirty" keys in a transaction-local temp table and upsert once in a `DEFERRABLE INITIALLY DEFERRED` constraint trigger at commit | The lock is held only during commit. More complex |
| C. Advisory lock + `SKIP LOCKED` upsert | If the row is locked, skip, since another transaction will bump the timestamp anyway | Cheap. The skipped transaction may commit later than the other, so its timestamp can be stale; needs care |
| D. Derive from data | Readers compute `max(updated_at)` from the source tables, with indexes | Removes the triggers. Reads cost more |

The recommendation is **A**: the simplest correct option, and easy to reason about in review.

## Acceptance criteria
- A test with two concurrent large uploads of the same organism shows they run in parallel (the elapsed time of both is close to the slower one, not their sum).
- The ETag/`lastTimeUpdated` behaviour is unchanged for SILO and the website (existing tests plus a new test for "update visible after commit").
- A Flyway migration with a rollback note.

## Files
- `backend/src/main/resources/db/migration/` (new migration)
- `backend/src/main/kotlin/org/loculus/backend/service/submission/dbtables/UpdateTrackerTable.kt`
- `backend/src/main/kotlin/org/loculus/backend/model/ReleasedDataModel.kt`
