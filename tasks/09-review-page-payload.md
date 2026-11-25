# 09 · Website: stop downloading full sequences for every review card

> **Solves:** Cuts the mpox review page from ~32 MB to under 1 MB by loading card summaries and fetching sequences only on demand.

| Field | Value |
|---|---|
| Issues | [#7185](https://github.com/loculus-project/loculus/issues/7185), [#7825](https://github.com/loculus-project/loculus/issues/7825) (switch to `/review-data`, drop `includeReviewData`) |
| Component | website, backend |
| PR title | `perf(website): load review cards from /review-data and fetch sequences on demand` |
| Effort | M |
| Risk | Low–medium: the review flow is core for submitters |
| Status | todo · 🟡 mostly done |

## Existing work (checked 2026-11-23)
**Assigned to clonder.** #7427 (merged) loads review card data in one request; #7699 (merged) adds `/review-data` and deprecates `includeReviewData`. What's left is #7825 (switch the page to `/review-data`, drop the flag): ask first.

## Problem
The review page calls `get-data-to-edit` for **every card**, downloading full sequence data. For mpox that is **~32 MB per page load**.

## Where
- `website/src/components/ReviewPage/` (`ReviewPage.tsx` and the card components)
- `website/src/services/backendApi.ts`, `website/src/services/backendClient.ts`
- backend: the `/review-data` endpoint and the `includeReviewData` flag

## Plan
1. Use `/review-data` for the list, with metadata, status, errors/warnings and sequence lengths only (#7825).
2. Fetch the full sequence only when the user expands a card or opens the editor (lazy fetch, cached per accession and version).
3. Remove `includeReviewData` once nothing uses it.
4. Use `Button` and `DisabledUntilHydrated` for the new interactive pieces (see the root `AGENTS.md`).

## Acceptance criteria
- The initial payload for an mpox review page of 100 cards is under 1 MB.
- Expanding a card shows the sequence within one request.
- `ReviewPage.spec.tsx` is updated; the review e2e tests pass.
