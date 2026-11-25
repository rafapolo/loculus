# 13 · Process: reduce review latency and spread the review load

> **Solves:** Brings the slowest 10% of PRs from ~8 days to under 3 days to first review, and spreads reviews across more people.

| Field | Value |
|---|---|
| Issue | none yet; propose in a discussion or issue |
| Component | repo process (`.github/`) |
| PR title | `chore: add CODEOWNERS and a stale-review reminder workflow` |
| Effort | S |
| Risk | Low (social more than technical) |
| Status | proposal · 🟢 free |

## Existing work (checked 2026-11-23)
There is no `.github/CODEOWNERS` and no PR about CODEOWNERS or review reminders. Propose it in an issue or discussion first, since it's a team-process change.

## Data (merged PRs in the last 12 months, excluding Dependabot and review bots)
| Component | Median to first review | p90 to first review | p90 to merge | Top reviewer's share |
|---|---|---|---|---|
| All | 5.5 h | **191 h (~8 days)** | **381 h (~16 days)** | 36% |
| backend | 18.3 h | 216 h | 472 h | 40% |
| preprocessing | 15.2 h | 263 h | 383 h | **49%** |
| ena-deposition | 16.8 h | 149 h | 354 h | 49% |

Bus factor 1 (one author with ≥50% of commits): preprocessing/nextclade, ena-submission, ingest.

## Plan
1. Add `.github/CODEOWNERS` (there is none today), with **two owners per component** so reviews route automatically and no single person is the default.
2. A scheduled workflow that comments on, or pings in Slack, PRs with no human review after 48 h.
3. A weekly "review duty" rotation, documented in `CONTRIBUTING.md`.
4. Encourage smaller PRs: a PR-size label (e.g. `size/L` over 500 changed lines).
5. Pair a second person on the bus-factor-1 components (prepro, ENA, ingest) through paired reviews.

## Acceptance criteria
- p90 time to first review under 72 h after two months.
- No component has a reviewer above 35% of PRs.
