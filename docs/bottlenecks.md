# Loculus: current bottlenecks and optimization opportunities

> Snapshot from 2026-11-23, based on the 19 open issues labelled `performance` in `loculus-project/loculus`, cross-checked against our own measurements (CI, review, backlog; see [analyses](analyses.md)).
> Items marked **verified** were confirmed by reading the code.
> Local file, ignored through `.git/info/exclude`. See `tasks/README.md` for which items already have open PRs.

## Summary

| Priority | Area | Bottleneck | Effort | Issue |
|---|---|---|---|---|
| 1 | Ingest | Quadratic lookup in `prepare_files.py` (**verified**) | XS | #7443 |
| 2 | Deployment | Expensive website health probe causes restarts | S | #7751 |
| 3 | Website | SeqSet page waits for the slowest of ~48 LAPIS calls | S | #7845 |
| 4 | Database | `table_update_tracker` trigger serializes writes (**verified**) | M | #7167 |
| 5 | Backend | `get-released-data` times out on large organisms | M–L | #3511 |
| 6 | Backend | Bulk upload, reprocessing and inefficient queries | M–L | #6944, #6541, #4858, #6544 |
| 7 | Backend | Slow SeqSet creation and updates | M | #7739, #7783 |
| 8 | Website | ~70 MB SeqSet page | M | #7753, #7754, #7756 |
| 9 | Website | ~32 MB review page for mpox | M | #7185, #7825 |
| 10 | Website | Async zodios validation, slow combobox | S–M | #7765, #7444 |
| 11 | Search | SILO rebuilds the full index on every change | L | #7733, #5707 |
| 12 | CI | Integration tests: 22 min median, 14% failure rate | M | — |
| 13 | Process | Review p90 of 8 days, concentrated on one person | S | — |
| 14 | Process | ~70 Dependabot PRs merged by hand every month | S | — |

---

## Quick fixes: high impact, low effort

### 1. Quadratic ingest (#7443): **verified**
- **Symptom:** 15+ minutes for 200k SARS-CoV-2 submissions.
- **Cause:** in `ingest/scripts/prepare_files.py`, `to_submit`, `to_revise` and `to_revoke` are loaded from JSON as **lists** (line 106). The loop over every metadata record then checks `if fasta_id in to_submit:` (line 135), which scans the list once per record. With 200k records that is O(n²), roughly 10¹⁰ comparisons.
- **Fix:** wrap the three in `set(...)`. It's a 3-line change, and this step should go from minutes to seconds.
- **Proof:** a micro-benchmark (list vs. set with 200k IDs), plus a unit test that keeps the output identical.

### 2. Avoidable website restarts (#7751)
- **Cause:** `livenessProbe` and `readinessProbe` in `kubernetes/loculus/templates/loculus-website.yaml` hit `/`, which renders the full home page, with the default 1 s timeout.
- **Fix:** a cheap `/health` endpoint and a higher `timeoutSeconds`/`failureThreshold`.

### 3. SeqSet page blocked by the slowest LAPIS call (#7845)
- **Cause:** about 48 aggregate requests run in parallel, and the page waits for all of them. This can take up to 60 s.
- **Fix:** `AbortSignal.timeout` per request, render partial results, and mark the ones that timed out.

---

## Database and backend: biggest impact, medium effort

### 4. `table_update_tracker` trigger serializes writes (#7167): **verified**
- **Cause:** migration `V1.2__add_table_update_tracker.sql` adds a `FOR EACH STATEMENT` trigger to the tables. Every INSERT, UPDATE, DELETE or TRUNCATE upserts **the same row per table** in `table_update_tracker`, and that row stays locked until the transaction commits. Long transactions, such as a large submission or the mpox reprocessing, therefore block every other writer to the same table.
- **Options:**
  - derive "last updated" from `pg_stat_user_tables` or a read-side check;
  - move the bookkeeping out of the main transaction (`NOTIFY` plus a listener, or a deferred job);
  - use an append-only change log that readers aggregate.
- **Note:** this changes how consumers detect updates (ETags for SILO and the website), so bring it as a design proposal rather than an unsolicited PR.

### 5. `get-released-data` times out (#3511, open since 2024)
- This is the endpoint that feeds SILO and ENA deposition.
- **Fix:** server-side cursor streaming, keyset pagination, and a materialized snapshot per released version.

### 6. Bulk upload, reprocessing and query efficiency
- Uploading 1M dummy sequences fails (#6944).
- The backend is slow on mpox reprocessing (#6541).
- There are inefficient database queries (#4858).
- `/get-original-data` is not streamed (#6544).
- **Fix:** batch inserts with `COPY`, `EXPLAIN ANALYZE` on the hot queries, review the indexes, and stream large responses end to end.

### 7. Slow SeqSet creation and updates (#7739)
- **Cause:** row-by-row inserts, plus a redundant `seqset_to_records` table (#7783).
- **Fix:** batch inserts and simplify the schema.

---

## Website: memory and payload

### 8. ~70 MB SeqSet page (#7753, epic #7754)
- **Cause:** all records are serialized into the HTML twice (206k records).
- **Fix:** sort `get-seqset-records` on the server so it can be paginated and LAPIS can cache the requests (#7756), and send only IDs to the client.

### 9. ~32 MB review page for mpox (#7185)
- **Cause:** every review card calls `get-data-to-edit` and downloads full sequence data.
- **Fix:** a summary endpoint with sequences fetched on demand (related to #7825, `/review-data`).

### 10. Client-side responsiveness
- zodios' async response validation is expensive (#7765). Replace it with a synchronous `zod.parse`; this also gets rid of the abandoned `@zodios/*` packages, which carry an unfixed HIGH advisory through `zod <4.6.6`.
- The search combobox blocks for over 1 s with 1,000+ options (#7444). Fix it with a virtualized list and debounced filtering.

---

## Search pipeline

### 11. SILO full rebuilds (#7733)
- **Cause:** the SILO importer rebuilds the whole index whenever released data changes.
- **Fix:** incremental updates, or at least cheaper sorting during the load (#5707). This reduces the delay between approval and the sequence appearing in search.

---

## Process bottlenecks (from our own measurements)

### 12. Integration tests are the CI bottleneck
- 22 min median, 14% failure rate on branches, 3.5% re-runs, 1.8% failures on `main`, against 0.4–4 min for unit test workflows.
- **Fix:** shard by feature or organism, run only the affected tests on PRs and the full suite on `main`, cache the kind cluster and images, and keep a flaky-test report with quarantine.

### 13. Review latency
- The median time to the first human review is 5.5 h, but the p90 is 8 days and the p90 to merge is 16 days. One reviewer covers 36–49% of the PRs in several components.
- **Fix:** CODEOWNERS with two owners per component, a weekly review-duty rotation, and alerts for PRs without a review after 48 h.

### 14. Dependency churn
- Dependabot opens 41% of all PRs, about 70 per month are merged by hand, and 25% are closed as superseded.
- **Fix:** auto-merge patch and minor updates on green CI, weekly grouping per ecosystem, and a way to track the third-party images mirrored to GHCR and pinned in Helm values (Keycloak 23.0 gets no Dependabot PRs). Conda is already covered by Dependabot.

---

## Where to start

1. **#7443 (`set()` in ingest):** easy to prove with a benchmark and before/after numbers.
2. **#7845 (SeqSet LAPIS timeout):** small, free (no linked PR), and user-visible. Note that #7751 (health probe) already has an open PR, **#7771**, so review that one instead of duplicating it.
3. **#7167 (trigger lock):** the highest technical impact (Postgres locking and concurrency). Worth discussing as a proposal first.

## Dive deeper

- [Analyses: the generated charts, with takeaways](analyses.md)
- [Tasks: 18 concrete contributions](../tasks/README.md)
- [Data map: where data lives](data-map.md)
- [Python pipelines](python-pipelines.md)
- [CVEs and advisories](cves.md)
- [Security review](security.md)
