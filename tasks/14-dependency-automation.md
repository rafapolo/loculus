# 14 · Process: automate dependency updates and close the blind spots

> **Solves:** Removes ~70 manual Dependabot merges a month and closes the CVE blind spots: Keycloak 23 and other mirrored images, and transitive pins in keycloakify.

| Field | Value |
|---|---|
| Issues | [#3736](https://github.com/loculus-project/loculus/issues/3736) (Keycloak 26), [#7784](https://github.com/loculus-project/loculus/issues/7784) and [#7806](https://github.com/loculus-project/loculus/issues/7806) (Nextclade) |
| Component | CI, deployment |
| PR titles | `ci: auto-merge patch/minor dependabot updates on green CI`, `ci: track mirrored image tags (Renovate or a scheduled check)`, `ci: scan published images with Trivy` |
| Effort | S–M |
| Risk | Low for auto-merge (with green CI as the gate); **high for the Keycloak upgrade itself** |
| Status | todo · 🟢 free / 🟡 Keycloak |

## Existing work (checked 2026-11-23)
There is already a **Dependabot duty SOP** (`maintainer-docs/dependabot_duty_sop.md`): a rotating person reviews Dependabot PRs daily, with a preview and visual check for UI deps. Build on it (auto-merge only the classes of update the SOP already treats as routine) rather than replacing it. There are no PRs for Dependabot auto-merge, Renovate or Trivy. Keycloak: #3735 (merged, old flags); **#6419 and #6707 are open** (swap Keycloak for Authelia + lldap); the upgrade to 26 (#3736) is now task 16 (stale PR #3405 already tested ORCID on 26). Nextclade: #7780 (merged) **reverted** to 3.24.0, so check the reason before bumping (#7784, #7806). The `uv` bump in `ena-submission` is free.

## Data (last 12 months)
- Dependabot opened **734 PRs, 41% of all PRs**: 546 merged and 185 closed as superseded. The median merge time is 20 h, all merged by hand (≈70/month).
- **Blind spots** that Dependabot doesn't see:
  - **Keycloak `23.0`**, pinned as an image tag in `kubernetes/loculus/values.yaml`. OSV lists **91 advisories (28 high)** for 23.0.x, e.g. CVE-2024-3656 and CVE-2024-1132.
  - **conda `environment.yml` pins**: Nextclade is stuck at 3.24.0, and `uv=0.12.21` in `ena-submission` has a CRITICAL advisory (fixed in 0.13.4).
  - **`keycloak/keycloakify/yarn.lock`**: `tar 6.2.1` with 1 critical and 8 high advisories (build-time only).

## Plan
1. **Auto-merge:** a workflow that enables `gh pr merge --auto --squash` on Dependabot PRs whose `update-type` is patch or minor (`dependabot/fetch-metadata`), gated on the required checks.
2. **Grouping:** review the `groups:` in `.github/dependabot.yml` to group per ecosystem per week. Add the keycloakify `yarn` directory if it's missing.
3. **Renovate** (or a scheduled job built on `maintenance-scripts/`) for:
   - conda `environment.yml` pins;
   - Helm `values.yaml` image tags (`images.*.tag`).
4. **Trivy** in CI on the published images (`ghcr.io/loculus-project/*`) and on the mirrored third-party images. Fail on new CRITICAL findings and report HIGH.
5. **Quick fix now:** bump `uv` in `ena-submission/environment.yml` to ≥ 0.13.4, aligned with the other environments (0.13.8 and 0.13.9).
6. **Keycloak 26** (#3736): now its own task, **[16](16-keycloak-26-orcid.md)**. The ORCID blocker (#3400) is already fixed upstream in keycloak-orcid 1.4.0/1.5.0.

## Acceptance criteria
- Under 10 Dependabot PRs per month need manual action.
- The CVE scan runs on every image build, and Keycloak and conda show up in update PRs.
