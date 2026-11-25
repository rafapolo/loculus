# 08 · Website: shrink the ~70 MB SeqSet page

> **Solves:** Shrinks the SeqSet page from ~70 MB to under 1 MB by paginating records instead of embedding all of them twice.

| Field | Value |
|---|---|
| Issues | [#7753](https://github.com/loculus-project/loculus/issues/7753), epic [#7754](https://github.com/loculus-project/loculus/issues/7754), [#7756](https://github.com/loculus-project/loculus/issues/7756) (sorted records), [#7757](https://github.com/loculus-project/loculus/issues/7757) (dummy-organism hack) |
| Component | website, backend, seqset |
| PR titles | `feat(backend): return seqset records sorted and paginated`, then `perf(website): paginate seqset records instead of embedding them` |
| Effort | M |
| Risk | Low–medium |
| Status | todo · 🔴 in progress |

## Existing work (checked 2026-11-23)
Active SeqSet perf epic: #7759 and #7762 (merged); **#7760 open** (sorted records, #7756), **#7761 open** (count + first page only), **#7851 open** (configurable record fields, #7757); #7749 closed. Contribute through review or e2e coverage (#7815) rather than new code.

## Problem
A SeqSet with 206k records produces a **~70 MB HTML page**: every record is serialized into the page twice (SSR props plus hydration data). Memory and CPU grow with the SeqSet size, on both the server and the client.

## Plan
1. **Backend:** make `get-seqset-records` return records **sorted** (stable order by accession), with `limit`/`offset` or keyset pagination (#7756). Sorted IDs also give LAPIS stable, cacheable request bodies.
2. **Website:** render only the first page server-side, and fetch the next pages client-side. Pass IDs or a count to islands instead of full records.
3. Compute the statistics from LAPIS with the SeqSet ID filter rather than shipping all accessions to the client.
4. Measure the page weight and server heap before and after (206k-record fixture).

## Acceptance criteria
- The page HTML is under 1 MB for a 206k-record SeqSet.
- Records are navigable with pagination; download/export still returns everything.
- e2e coverage for pagination (#7815).

## Files
- `website/src/pages/seqsets/[seqSetId].astro`, `[seqSetId].[version].astro`
- `website/src/components/SeqSetCitations/`
- `backend/src/main/kotlin/org/loculus/backend/api/v0/seqsets/SeqSetCitationsController.kt`
