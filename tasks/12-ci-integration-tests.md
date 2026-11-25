# 12 · CI: make integration tests faster and less flaky

> **Solves:** Gives faster, more reliable PR feedback by sharding integration tests, running only affected specs and quarantining flaky ones.

| Field | Value |
|---|---|
| Issue | none yet; open one with the numbers below |
| Component | CI, integration-tests |
| PR titles | `ci: shard integration tests`, `ci: report flaky integration tests` |
| Effort | M |
| Risk | Low: CI only |
| Status | todo · 🟡 related |

## Existing work (checked 2026-11-23)
No sharding PR exists. **#7393 is open** (3 Playwright workers instead of 2). #6370 (merged) cut workers from 4 to 2 because the runner was overloaded (`ERR_EMPTY_RESPONSE`), at the cost of only ~1 min. More workers on one runner isn't the lever.

## Data (≈90 days, `gh run list`)
| Workflow | Runs | Failure rate | Re-runs | Median | p90 |
|---|---|---|---|---|---|
| Integration tests | 1,000 (since 2026-10-05) | **14.0%** | **3.5%** | **21.7 min** | 26.3 min |
| website-tests | 1,000 | 7.7% | 0.7% | 2.2 min | 2.9 min |
| backend-tests | 1,000 | 5.2% | 0.6% | 4.3 min | 6.0 min |

Integration tests also fail **1.8% of the time on `main`**, which points to flakiness rather than broken PRs.

## Where
- `.github/workflows/integration-tests.yml`: the matrix today is only `["chromium","firefox","cli"]`, with `timeout-minutes: 45`.
- `integration-tests/tests/specs/` (`auth`, `backend`, `cli`, `features`, `group`, …)

## Plan
1. **Shard:** add `--shard=i/N` to the Playwright run (e.g. N=3 per browser), and merge the reports with `playwright merge-reports`.
2. **Selective runs on PRs:** a `paths` filter, plus tagging the specs (`@website`, `@backend`, `@ingest`), so PRs run the relevant subset and `main` runs everything.
3. **Cluster start-up:** measure how much of the 22 min is spent creating the kind cluster and pulling images. Cache the images, or reuse a pre-built cluster image.
4. **Flakiness:**
   - turn on `retries: 1` together with reporting of the tests that passed on retry;
   - publish a weekly flaky-test summary (a small script over the JUnit reports);
   - quarantine tests with `test.fixme` plus an issue.
5. Keep the existing `DisabledUntilHydrated`/`Button` rules (root `AGENTS.md`) to stop new flakes.

## Acceptance criteria
- PR feedback (time until all checks are green) median under 12 min.
- Branch failure rate under 5% over the following month; `main` failures under 0.5%.
