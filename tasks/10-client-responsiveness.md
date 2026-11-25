# 10 · Website: replace zodios validation and speed up large comboboxes

> **Solves:** Removes the unmaintained, vulnerable zodios client and makes search comboboxes with thousands of options responsive.

| Field | Value |
|---|---|
| Issues | [#7765](https://github.com/loculus-project/loculus/issues/7765) (sync zod parse), [#7818](https://github.com/loculus-project/loculus/issues/7818) (unify how the website calls endpoints), [#7444](https://github.com/loculus-project/loculus/issues/7444) (combobox with 1000+ options) |
| Component | website |
| PR titles | `refactor(website): replace zodios with a typed fetch client and sync zod parsing`, `perf(website): virtualize search combobox options` |
| Effort | S–M each |
| Risk | Medium for the client refactor (14 files use zodios); low for the combobox |
| Status | todo · 🔴 in progress / 🟢 combobox free |

## Existing work (checked 2026-11-23)
zodios removal is underway as a **stacked PR series**: #7813, #7832, #7833, #7834, #7836, #7837, #7840 and #7843 are open; #7766 is open; #7758 is merged; #7821 was closed. **#7444 (combobox) has no PR, so that slice is free.**

## Problems
1. zodios runs response validation asynchronously, which is expensive and was turned off. zodios is also **unmaintained**, and `npm audit` reports HIGH advisories for `@zodios/*` (no fix available) and for `zod <4.6.6` (GHSA-mrhx-c7xc-m3w5, memory exhaustion).
2. The search comboboxes block the main thread for over 1 s on open or typing when a field has 1,000+ options.

## Plan
### Client
1. Write a small typed fetch wrapper: `request(endpoint, schema)` with a synchronous `schema.parse` (zod ≥ 4.6.6).
2. Migrate the endpoints from `website/src/services/backendApi.ts` and `website/src/services/*Client.ts` one group at a time.
3. Remove `@zodios/core` and `@zodios/react`, then re-run `npm audit`.

### Combobox
1. Use the wrapper at `src/components/common/headlessui/Combobox.tsx`, as `AGENTS.md` requires.
2. Virtualize the options (e.g. `@tanstack/react-virtual`), debounce the filtering (150 ms), and precompute lowercased labels.
3. Affected fields: `SingleChoiceAutoCompleteField.tsx`, `MultiChoiceAutoCompleteField.tsx`, `MutationField.tsx`.

## Acceptance criteria
- `npm audit` no longer lists `@zodios/*` or `zod`.
- The combobox opens in under 100 ms with 5,000 options (vitest plus a profiler check).
- The website unit tests and the search e2e tests pass.
