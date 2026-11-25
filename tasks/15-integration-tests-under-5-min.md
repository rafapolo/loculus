# 15 · CI: bring the integration test run from >20 min to <5 min

> **Solves:** Gets PR integration feedback from ~22 min to under 5 min of test execution (and about 8–10 min end to end) by parallelizing across runners and workers and removing the serial waits for infrastructure.

| Field | Value |
|---|---|
| Issue | none yet; open one with the timing breakdown below (extends task 12) |
| Component | CI, integration-tests, deployment |
| PR titles | `ci: shard playwright integration tests across runners`, `ci: raise playwright workers and run firefox only on main`, `ci: pre-pull images and speed up pod readiness`, `test(integration): seed shared data once instead of per test` |
| Effort | M–L (4–6 incremental PRs) |
| Risk | Low–medium: CI cost goes up with more runners, and higher parallelism can surface hidden coupling between tests |
| Status | todo · 🟢 free |

## Existing work (checked 2026-11-23)
No sharding PR exists. Related: **#7393 open** (3 workers) and #6370 (merged, 4→2 workers: runtime only went up ~1 min, so execution is dominated by serial waits, not CPU parallelism). Coordinate with #7393.

## Where the time goes today (latest green run on `main`)

| Step | chromium | firefox | cli |
|---|---|---|---|
| Create k3d cluster | 48 s | 38 s | 41 s |
| Install deps + Playwright browsers | 29 s | 58 s | 39 s |
| **Wait for pods to be ready** | **224 s** | **207 s** | **254 s** |
| Fixed `sleep 10` | 10 s | 10 s | 10 s |
| **Run integration tests** | **534 s (8.9 min)** | **812 s (13.5 min)** | 101 s |
| ≈ Job total | ~15 min | ~20 min | ~8 min |

The facts behind these numbers:
- `npx playwright test --workers=2` runs on `ubuntu-latest` (4 vCPU), and the same machine hosts the whole k3d cluster. That's only 2 workers for about 100 tests in 55 spec files.
- `fullyParallel: true` and `timeout: 60_000` are set, and `retries: 0`.
- The projects are `readonly-setup`, then `*-with-dep` (the `.dependent.spec.ts` files) and `*-without-dep`, plus `cli-tests`.
- **Infrastructure costs ~5 min** before the first test, mostly waiting for pods (Keycloak, the backend, preprocessing and SILO's first import).
- Firefox repeats the whole browser suite on every PR and is the slowest job.

## Target budget (per PR)

| Phase | Today | Target |
|---|---|---|
| Cluster + images + pods ready | ~5 min | **≤ 2.5 min** |
| Test execution (slowest shard) | 9–13.5 min | **< 5 min** |
| Wall time until all checks are green | ~22 min | **~8–10 min** |

## Plan

### A. Parallelize test execution (biggest lever)
1. **Shard across runners:** add `shard: [1, 2, 3, 4]` to the matrix and run `npx playwright test --shard=${{ matrix.shard }}/4`. Upload the blob reports and merge them in a final job (`npx playwright merge-reports`).
2. **Not more workers on the same runner:** #6370 showed that going from 4 to 2 workers cost only ~1 min and removed `ERR_EMPTY_RESPONSE` flakes, because the 4-vCPU runner is saturated by the cluster. Parallelism has to come from **more runners (shards)**, or from a larger runner (8 vCPU) if the cost is acceptable. Coordinate with the open #7393 (3 workers).
3. **Firefox off the PR critical path:** run chromium plus cli on PRs, and the full chromium + firefox matrix on `main`, nightly, and on a `test:firefox` label.
4. **Balance shards by duration:** keep a `durations.json` from the `main` runs. Playwright shards by file, so split the long spec files (`file-sharing.spec.ts`, 778 lines, and the search specs) into smaller files.

### B. Shrink the infrastructure phase
1. **Images:** pre-pull or side-load the images while the cluster starts (`k3d image import` from the Actions cache, or a registry mirror). Build or download them in parallel with `npm ci` and the Playwright install.
2. **Readiness:** replace the `wait_for_pods_to_be_ready.py` polling and `sleep 10` with targeted `kubectl wait --for=condition=Ready` on the services the tests need first. Start the tests that don't need SILO data (auth, navigation, account) as soon as the backend, website and Keycloak are ready.
3. **Lighter deployment for CI:**
   - Keycloak in dev mode with an imported realm and no theme build;
   - fewer organisms in `values_e2e_and_dev.yaml` (only the ones the tests touch);
   - dummy preprocessing where Nextclade output isn't asserted;
   - smaller resource requests.
4. **Overlap:** start the deploy first, then install the deps and browsers while it comes up (today these steps run strictly in sequence).

### C. Make tests cheaper
1. **Seed once:** extend `readonly.setup.ts` so submissions and releases are created once per shard and reused, instead of each spec submitting and then polling until preprocessing and SILO catch up.
2. **No blind waits:** replace fixed timeouts with `expect.poll` or `toPass` on a precise condition, and lower the 60 s timeout where possible to fail fast.
3. **API shortcuts for setup:** log in through a stored `storageState` per worker (auth fixture) and create data through the backend API instead of clicking through the UI, unless the UI flow itself is under test.
4. **Isolation:** one group per worker (`testInfo.parallelIndex`) so higher worker counts don't collide.

### D. Keep it fast and green
- `retries: 1` on CI, plus a report of tests that passed only on retry (flaky list), with a quarantine (`test.fixme` plus an issue).
- A CI-duration dashboard: per-job and per-shard p50/p90 from `gh run list`, posted weekly.

## Rollout (incremental PRs)
1. Measurement: upload the Playwright JSON reporter output, and record the per-test durations on `main`.
2. Firefox only on `main`, nightly and on label: quick win (−20 min job off PRs).
3. Shards (4) with merged reports, plus `--workers` tuning.
4. Infrastructure overlap, image pre-pull and targeted readiness.
5. Seeding refactor and API-based setup (largest test change; split by feature folder).

## Acceptance criteria
- On PRs, the slowest shard's **test step takes < 5 min** (p90 over 2 weeks).
- **Wall time to green checks ≤ 10 min** p50.
- The flakiness rate doesn't increase (failures on `main` ≤ 1%).
- The CI minute cost is reported before and after. Shards add runners, but removing Firefox from PRs offsets part of it.

## Files
- `.github/workflows/integration-tests.yml`
- `.github/scripts/wait_for_pods_to_be_ready.py`
- `integration-tests/playwright.config.ts`, `integration-tests/tests/**` (`readonly.setup.ts`, fixtures)
- `kubernetes/loculus/values_e2e_and_dev.yaml`
