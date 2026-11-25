# 17 · Docs: complete the system overview and add a security-updates SOP

> **Solves:** Fixes the wrong "all data is in two Postgres databases" backup claim, documents the missing optional services, and writes down how to keep the components Dependabot can't see (mirrored images such as Keycloak) up to date.

| Field | Value |
|---|---|
| Branch | `docs/system-overview-data-storage`, commit `f602fb33`, pushed to the fork `rafapolo/loculus` |
| Component | docs |
| PR title | `docs: list optional services and where data is stored; add security updates SOP` (or split into 2 PRs) |
| Effort | S (done; needs review, commit and PR) |
| Risk | Very low (docs only) |
| Status | pushed to the fork · 🟡 upstream PR not opened yet |

## Existing work (checked 2026-11-23)
No open docs PR touches `system-overview.md` or adds a security SOP. The only maintainer SOP so far is `maintainer-docs/dependabot_duty_sop.md`.

## Changes on the branch
| File | Change |
|---|---|
| `docs/src/content/docs/introduction/system-overview.md` | Adds the 4 optional services (Ingest, ENA deposition, Raw reads processing, Taxonomy service) with their Helm toggles. Fixes the backup claim: S3 also holds data when file sharing is enabled. New section **"Where data is stored"** (which store, what it holds, whether it needs a backup) and a warning that the dev databases have no persistent volume |
| `maintainer-docs/security_updates_sop.md` (new) | What Dependabot covers, the blind spots (third-party images mirrored to GHCR such as Keycloak, PostgreSQL and MinIO; transitive pins; downloaded tools), a monthly check (`npm audit`, `osv-scanner`, Trivy) and triage |
| `maintainer-docs/README.md` | Link to the new SOP |
| `AGENTS.md`, `maintenance-scripts/README.md` | Fixes the outdated "Dependabot doesn't update conda" note. It does: see #7793, #7795, #7852 and the conda entries in `.github/dependabot.yml`; Nextclade is pinned on purpose |

All files are formatted with the repo's Prettier settings (Markdown `tabWidth: 2`).

## Next steps
1. Decide between **1 PR** and **2 PRs** (docs/overview vs. SOP + conda notes).
2. Commit (conventional commit, no AI attribution in commits), push, and open a **draft PR** with a detailed description.
3. Optional: run `npm run check-format` and the docs build in `docs/` after `npm ci`.
