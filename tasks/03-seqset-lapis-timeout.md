# 03 · Website: time-box the SeqSet page's LAPIS aggregate calls

> **Solves:** Makes the SeqSet page render in seconds even when some of its ~48 LAPIS queries are slow, by time-boxing them and showing partial stats.

| Field | Value |
|---|---|
| Issue | [#7845](https://github.com/loculus-project/loculus/issues/7845) |
| Component | website, seqset |
| PR title | `fix(website): add per-request timeout to SeqSet statistics and render partial results` |
| Effort | S (½–1 day) |
| Risk | Low |
| Status | todo · 🟢 free |

## Existing work (checked 2026-11-23)
Issue #7845 is open and unassigned, with **no linked PR**. Older related work: #2020 (merged, LAPIS timeouts on the first page).

## Problem
The SeqSet details page waits for **the slowest of ~48 LAPIS aggregate requests**, which can take up to 60 s before anything renders.

## Where
- `website/src/components/SeqSetCitations/getSeqSetStatistics.ts`, which builds the statistics
- `website/src/pages/seqsets/[seqSetId].astro` and `[seqSetId].[version].astro`

## Plan
1. Wrap each LAPIS call with `AbortSignal.timeout(N)` (N configurable, default 5–10 s).
2. Use `Promise.allSettled` and render every statistic that resolved; mark the ones that timed out as "unavailable".
3. Log the timeouts with the organism and query, so the slow LAPIS queries can be found later (connects to task 08).
4. Consider moving the statistics to a client-side island, so the page shell renders right away.

## Acceptance criteria
- The page renders in ≤ N s even when a LAPIS mock hangs.
- Unit tests in `getSeqSetStatistics.spec.ts` for the success, partial and all-timeout cases.
- An integration test covering the page loading with partial stats (relates to #7815).

## Files
- `website/src/components/SeqSetCitations/getSeqSetStatistics.ts` and its spec
- `website/src/pages/seqsets/*.astro`
