# 05 · Backend: make `get-released-data` scale to large organisms

> **Solves:** Keeps the released-data feed for SILO and ENA from timing out on large organisms by streaming it with a server-side cursor.

| Field | Value |
|---|---|
| Issue | [#3511](https://github.com/loculus-project/loculus/issues/3511) (open since 2024-11) |
| Component | backend, silo/LAPIS, ena-deposition |
| PR title | `perf(backend): stream get-released-data with a server-side cursor` |
| Effort | M–L |
| Risk | Medium: it's the feed for SILO and ENA, so a regression breaks search |
| Status | todo · 🟡 explored |

## Existing work (checked 2026-11-23)
Issue #3511 is open. #6926 (`spool released exports safely`) was **closed unmerged**; **#7431 is open** ("query experiments, not to merge"). Read both before proposing anything.

## Problem
`/get-released-data` times out when an organism has many sequences. It feeds:
- the **SILO importer** (`loculus-silo/src/silo_import/download_manager.py`), every 30 s with an ETag;
- **ENA deposition** (`ena-submission/src/ena_deposition/call_loculus.py`);
- the website's download links.

## Where
- `backend/src/main/kotlin/org/loculus/backend/api/v0/submission/SubmissionController.kt` (endpoint)
- `backend/src/main/kotlin/org/loculus/backend/model/ReleasedDataModel.kt` (row mapping and ETag)

## Plan
1. Profile on a realistic dataset (seed 1M rows locally): `EXPLAIN (ANALYZE, BUFFERS)` the main query and measure time to first byte and peak heap.
2. Make sure the response is **truly streamed**: JDBC `fetchSize` with autocommit off (a server-side cursor), Exposed's lazy iteration, and an NDJSON writer that flushes periodically, without materializing the whole list.
3. Optional: build a compressed snapshot per release version (zstd NDJSON in S3) when the tracker changes, and serve or redirect to it. SILO then just downloads a file.
4. Add a metric for duration and rows per request (`SubmissionMetricNames.kt`).

## Acceptance criteria
- Flat memory, independent of the row count (check heap with 1M rows).
- Time to first byte under 2 s.
- Existing backend tests pass, plus a streaming test asserting that rows arrive before the query ends.

## Files
- `SubmissionController.kt`, `ReleasedDataModel.kt`, and Exposed query code in `service/submission/`
