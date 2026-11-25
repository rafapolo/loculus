# 06 · Backend: bulk upload, reprocessing and query efficiency

> **Solves:** Makes million-sequence uploads, mpox reprocessing and hot backend queries fast and bounded in memory, measured before and after.

| Field | Value |
|---|---|
| Issues | [#6944](https://github.com/loculus-project/loculus/issues/6944) (1M upload fails), [#6541](https://github.com/loculus-project/loculus/issues/6541) (slow mpox reprocessing), [#4858](https://github.com/loculus-project/loculus/issues/4858) (inefficient queries), [#6544](https://github.com/loculus-project/loculus/issues/6544) (stream `/get-original-data`) |
| Component | backend |
| PR titles | split into 2–4 PRs, e.g. `perf(backend): stream get-original-data`, `perf(backend): add missing indexes for …` |
| Effort | M–L (investigation first) |
| Risk | Medium |
| Status | investigate · 🟡 partial |

## Existing work (checked 2026-11-23)
#6944: #7078 and #7096 (merged) batch some inserts and accession updates. #4858 is **assigned to clonder and anna-parker**; #7074 was closed and #7431 is open (experiments). #6544: #5856 (merged) is related but doesn't stream. #6541 has no PR. Free slice: profile mpox reprocessing (#6541).

## Problem
- Uploading 1M dummy sequences fails.
- Reprocessing mpox is slow.
- Some queries are known to be inefficient.
- `/get-original-data`, which the preprocessing pipelines pull, isn't streamed end to end.

## Plan
1. **Measure first.**
   - Enable `pg_stat_statements` in a dev deployment and replay an mpox reprocessing.
   - Rank the queries by total time, then run `EXPLAIN (ANALYZE, BUFFERS)` on the top 10.
2. **Indexes:** add or adjust indexes for the hot filters (`organism`, `status`, `pipeline_version`, `(accession, version)` joins). Each one gets its own Flyway migration, created with `CONCURRENTLY` where possible.
3. **Upload path** (`UploadDatabaseService.mapAndCopy`):
   - check memory with 1M rows;
   - use `COPY`-style batch inserts and chunked processing;
   - fail with a clear error instead of an OOM.
4. **`/get-original-data`:** stream it like task 05 (cursor + NDJSON flushing).
5. **Reprocessing:** batch the state transitions instead of updating row by row (inspect `SubmissionDatabaseService.kt`).

## Acceptance criteria
- Before and after numbers in each PR: query time, upload time for 100k and 1M rows, and peak heap.
- A 1M upload either succeeds or fails fast with a documented limit.

## Files
- `backend/src/main/kotlin/org/loculus/backend/service/submission/` (`UploadDatabaseService`, `SubmissionDatabaseService`)
- `backend/src/main/resources/db/migration/`
