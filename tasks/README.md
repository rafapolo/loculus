# Tasks: Loculus

Contribution backlog built from the repository analysis (2026-11-23). Each task file has a one-line **Solves**, a header table (issues, component, PR title, effort, risk, status), an **Existing work** section with the related issues and PRs, a plan, acceptance criteria and the files involved.

Ordered by priority: security and backend performance first (Kotlin/Spring + PostgreSQL), then testing and CI/CD, then website and pipeline improvements.

| # | Task | Problem | Expected result |
|---|---|---|---|
| 1 | [Keycloak 26 + fixed ORCID plugin](16-keycloak-26-orcid.md) | Keycloak 23 has 29 HIGH advisories; the upgrade was blocked by an ORCID plugin that's already fixed upstream | All 29 HIGH advisories closed, ORCID login kept |
| 2 | [Backend tests faster + parallel](18-backend-tests-speed.md) | Serial tests in one JVM, a `docker exec psql` reset before every test, many Spring contexts, no Gradle build cache | Backend CI test step: 210 s → ≤ 90 s |
| 3 | [Update-tracker row lock](04-update-tracker-lock.md) | A tracker trigger holds a row lock until commit, so same-organism submissions run one after another | Concurrent submissions run in parallel |
| 4 | [Stream get-released-data](05-released-data-timeouts.md) | The released-data feed for SILO and ENA times out on large organisms | Flat memory and < 2 s to first byte, no timeouts |
| 5 | [Bulk upload & queries](06-bulk-upload-and-queries.md) | 1M-sequence uploads fail; mpox reprocessing and hot queries are slow | 1M uploads succeed or fail fast; measured query speed-ups |
| 6 | [Integration tests < 5 min](15-integration-tests-under-5-min.md) | ~5 min of infrastructure start-up plus 9–13.5 min of serial tests on one runner | Test run < 5 min; checks green in ~8–10 min |
| 7 | [CI integration tests](12-ci-integration-tests.md) | Integration tests take 22 min with a 14% failure rate on branches | Faster and less flaky PR feedback (< 5% failures) |
| 8 | [Dependency automation & CVEs](14-dependency-automation.md) | ~70 Dependabot PRs merged by hand every month; mirrored images (Keycloak) are never updated | < 10 manual Dependabot PRs a month; CVE scans on every image build |
| 9 | [Website health probe](02-website-health-probe.md) | Liveness and readiness probes render the full home page with a 1 s timeout | No more avoidable website pod restarts |
| 10 | [Docs: system overview + security SOP](17-docs-system-overview-security-sop.md) | The docs claim all data is in two Postgres databases (S3 is missing) and list no optional services or CVE process | Correct backup guidance and a documented security-update routine |
| 11 | [Review latency](13-review-latency.md) | p90 to first review is ~8 days, concentrated on one reviewer | p90 < 72 h; no reviewer above 35% of PRs |
| 12 | [SILO incremental updates](11-silo-incremental-updates.md) | The SILO index is fully rebuilt on every data change | Shorter delay from approval to search; cheaper rebuilds |
| 13 | [SeqSet write performance](07-seqset-write-performance.md) | Large SeqSets are inserted row by row, with a redundant schema | Create/update of 200k records: minutes → seconds |
| 14 | [Ingest quadratic lookup](01-ingest-quadratic-lookup.md) | `prepare_files.py` checks every record against Python lists, so preparation is O(n²) | 200k-sequence prep: 15+ min → seconds |
| 15 | [SeqSet LAPIS timeout](03-seqset-lapis-timeout.md) | The SeqSet page waits for the slowest of ~48 LAPIS queries | Page renders in ≤ 5–10 s with partial stats instead of up to 60 s |
| 16 | [SeqSet page payload](08-seqset-page-payload.md) | All records are serialized into the SeqSet page HTML twice | ~70 MB page → < 1 MB |
| 17 | [Review page payload](09-review-page-payload.md) | Every review card downloads full sequence data | mpox review page: ~32 MB → < 1 MB |
| 18 | [zodios → typed fetch; combobox](10-client-responsiveness.md) | An unmaintained, vulnerable zodios client and comboboxes that freeze with 1,000+ options | `@zodios`/`zod` advisories gone; combobox opens in < 100 ms |

## Dive deeper

- [Analyses: the generated charts, with takeaways](../docs/analyses.md)
- [Bottlenecks and optimization opportunities](../docs/bottlenecks.md)
- [Data map: where data lives](../docs/data-map.md)
- [Python pipelines](../docs/python-pipelines.md)
- [CVEs and advisories](../docs/cves.md)
- [Security review](../docs/security.md)
