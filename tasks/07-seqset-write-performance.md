# 07 · Backend: faster SeqSet creation and updates

> **Solves:** Makes creating and updating large SeqSets take seconds instead of minutes through batch inserts and a simpler schema.

| Field | Value |
|---|---|
| Issues | [#7739](https://github.com/loculus-project/loculus/issues/7739), [#7783](https://github.com/loculus-project/loculus/issues/7783) (`seqset_to_records` not needed), [#7781](https://github.com/loculus-project/loculus/issues/7781) (accession vs. accessionVersion), [#7764](https://github.com/loculus-project/loculus/issues/7764) (unused `type`) |
| Component | backend, seqset |
| PR titles | `perf(backend): batch-insert seqset records`, then `refactor(backend): drop seqset_to_records` |
| Effort | M |
| Risk | Medium: schema migration on user-facing citation data |
| Status | todo · 🔴 in progress |

## Existing work (checked 2026-11-23)
#7849 (merged) added SeqSet indexes; #7817 (merged) dropped `type` and `seqSetRecordId`; **#7805 is open** (flatten `seqset_records`, split accession and version), covering #7783 and #7781. Little left: review #7805 and benchmark #7739.

## Problem
Creating or updating a large SeqSet (100k+ records) is slow.

## Where
- `backend/src/main/kotlin/org/loculus/backend/service/seqsetcitations/SeqSetCitationsDatabaseService.kt` (708 lines)
- `backend/src/main/kotlin/org/loculus/backend/service/seqsetcitations/SeqSetsTables.kt`

## Plan
1. Profile creating a SeqSet with 200k records and count the round trips.
2. Replace per-record inserts with Exposed `batchInsert` (or `COPY`), and deduplicate records in memory first.
3. Schema cleanup, as separate PRs with Flyway migrations:
   - fold `seqset_to_records` into `seqset_records` with a `seqset_id` FK (#7783);
   - split `accession` and `accession_version` into proper columns (#7781);
   - drop the unused `type` (#7764).
4. Keep the API contract unchanged; the website depends on it.

## Acceptance criteria
- Creating 200k records takes seconds, not minutes (benchmark in the PR).
- Migrations are tested on a copy with existing SeqSets, and DOIs and versions are preserved.
- Existing SeqSet tests pass.
