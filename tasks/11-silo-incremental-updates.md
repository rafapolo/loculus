# 11 · Search: avoid full SILO rebuilds

> **Solves:** Shortens the delay from approval to search visibility by avoiding full SILO index rebuilds on every data change.

| Field | Value |
|---|---|
| Issues | [#7733](https://github.com/loculus-project/loculus/issues/7733) (incremental updates), [#5707](https://github.com/loculus-project/loculus/issues/5707) (sorting during load) |
| Component | loculus-silo (importer), SILO/LAPIS |
| PR titles | `perf(silo): pre-sort ndjson before SILO preprocessing`, then a design doc for incremental updates |
| Effort | L (depends on SILO's upstream capabilities) |
| Risk | Medium–high: search availability and correctness |
| Status | design · 🔴 in progress / 🟢 #5707 free |

## Existing work (checked 2026-11-23)
#7733 is **assigned to clonder**, with **#7746 open** (`get-released-data-changes` endpoint) and **#7747 open** (incremental SILO import). #5707 (sorting during load) has no PR.

## Problem
Every change to released data triggers a **full download, transform and SILO preprocessing** of the organism. For large organisms this delays search freshness (from approval to "visible in search") and uses a lot of CPU and memory.

## Where
- `loculus-silo/src/silo_import/` (`runner.py`, `download_manager.py`, `transformer.py`, `instruct_silo.py`, `lineage.py`)
- `kubernetes/loculus/templates/silo-deployment.yaml`

## Plan
1. **Quick win (#5707):** sort the NDJSON in the importer, streaming and with an external sort if needed, so SILO's preprocessing does less work. Benchmark the preprocessing time before and after.
2. **Measure** the time per stage (download, transform, preprocess, swap) and expose it as metrics or logs.
3. **Design (#7733):** check what LAPIS-SILO supports upstream (append and delete-by-key, or partitioned stores). Options:
   - append-only daily partitions plus a periodic full rebuild;
   - a delta import keyed on the backend's `lastTimeUpdated`, which would need a "changed since" mode in `get-released-data` (connects to task 05).
4. Write the design as an issue comment or ADR before coding.

## Acceptance criteria
- Step 1 is a measurable reduction in preprocessing time, with no change in query results (diff the LAPIS aggregates before and after).
- Step 3 is a design agreed with the maintainers.
