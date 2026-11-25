# 16 · Auth: upgrade Keycloak 23 → 26 with the fixed ORCID plugin

> **Solves:** Unblocks the Keycloak 26 upgrade by switching to the fixed `keycloak-orcid` plugin (1.5.0), closing Keycloak 23's known advisories (29 high, 52 moderate, 18 low in OSV) without losing ORCID login or changing how pipelines authenticate.

| Field | Value |
|---|---|
| Issues | [#3400](https://github.com/loculus-project/loculus/issues/3400) (ORCID plugin breaks on Keycloak ≥ 25, outdated), [#3736](https://github.com/loculus-project/loculus/issues/3736) (upgrade to Keycloak 26) |
| Related PRs | [#3405](https://github.com/loculus-project/loculus/pull/3405) (upgrade to Keycloak 26, ORCID tested with plugin 1.4.1, open since 2024-12); #3299 (earlier attempt, closed); #3374 (reverted upgrade); #6419 and #6707 (Authelia alternative, drafts) |
| Component | deployment, keycloak, website (theme) |
| PR titles | 1) `fix(deployment): bump keycloak-orcid plugin to 1.5.0`; 2) `feat(deployment)!: upgrade Keycloak to 26` |
| Effort | S for the plugin bump; M–L for the full upgrade (migration + testing) |
| Risk | **High**: authentication for every instance. Needs a DB backup, a preview with real ORCID, and a rollback plan |
| Status | todo · 🟢 free (no active PR; #3405 is stale) |

## Existing work (checked 2026-11-23)
- **The upstream fix exists** (verified through the GitHub API):
  - `eosc-kc/keycloak-orcid` **1.4.0** (2024-12-12) supports Keycloak ≥ 25;
  - **1.5.0** (2026-01-12) is built against Keycloak 26.5.
  
  The cause is the SPI change `BrokeredIdentityContext(String)` → `(String, IdentityProviderModel)` (eosc-kc/keycloak-orcid#2). With the old plugin, the ORCID callback fails with `NoSuchMethodError`.
- **Loculus still pins the broken plugin:** `keycloak/keycloakify/Dockerfile:4` sets `KEYCLOAK_ORCID_VERSION=1.3.0`, `:5` sets `KEYCLOAK_MAJOR_VERSION=24`, and `:14` downloads `keycloak-orcid-${VERSION}.jar`, while new releases ship `keycloak-orcid.jar`.
- **#3405** already upgraded to 26 using the 1.4.1 build from the fork `loculus-project/keycloak-orcid` and reported that ORCID login and sign-up worked on preview `orcid.loculus.org`. Nobody reviewed it.
- The Keycloak image is the GHCR mirror `ghcr.io/loculus-project/mirror/keycloak:23.0` (`kubernetes/loculus/values.yaml:2897-2899`, `.github/mirror-images.txt`). Dependabot doesn't open PRs for it (see `maintainer-docs/security_updates_sop.md` on the docs branch).
- Full comparison with other OSS IdPs: `docs/auth-options.md`. Authelia drops ORCID and ROPC; Zitadel and Authentik are the long-term candidates if Keycloak's weight becomes the problem.

## Plan

### Step 0: verify before touching anything (½ day)
1. Start Keycloak **26.8.x** (latest; **26.6.4 is the minimum that fixes all HIGH advisories below**) locally (container) with `keycloak-orcid` **1.5.0** in `/opt/keycloak/providers`, and import the realm from `keycloak-config-map.yaml`.
2. Confirm the `orcid` identity provider and the `orcid-user-attribute-mapper` load without errors (`kc.sh build`, start-up logs, admin console → Identity providers).
3. Record the result and comment on **#3400** (draft text below). Ask the user before posting.

### Step 1: PR "bump keycloak-orcid plugin" (S)
- `keycloak/keycloakify/Dockerfile`:
  - `KEYCLOAK_ORCID_VERSION=1.5.0`;
  - the URL `…/releases/download/${KEYCLOAK_ORCID_VERSION}/keycloak-orcid.jar`;
  - the copy glob on line ~34 so the new file name is picked up.
- On its own, this doesn't work on Keycloak 23: 1.4.0+ requires Keycloak ≥ 25. So either merge it **together with** step 2, or keep the version selectable per `KEYCLOAK_MAJOR_VERSION`.

### Step 2: PR "upgrade Keycloak to 26" (M–L)
1. Add `quay.io/keycloak/keycloak:26.8.x` (≥ 26.6.4, see the CVE table) to `.github/mirror-images.txt` and publish it to GHCR **before** changing consumers (see `kubernetes/README.md`).
2. In `values.yaml`, set `images.keycloak.tag` to 26.8.x; set `KEYCLOAK_MAJOR_VERSION` to 26 in the keycloakify Dockerfile (the theme JAR is built per major version).
3. In `keycloak-deployment.yaml`:
   - resolve the `TODO #1221` / #3736 items;
   - move to the `KC_BOOTSTRAP_ADMIN_USERNAME`/`PASSWORD` variables (Keycloak 26 replaced `KEYCLOAK_ADMIN*`);
   - apply hostname v2 (`KC_HOSTNAME` as a full URL, remove `KC_HOSTNAME_STRICT*` and the old flags; see #3735);
   - check proxy headers (`KC_PROXY_HEADERS`).
4. Realm import (`keycloak-config-map.yaml`): check deprecated fields against the 24/25/26 upgrade notes (user profile is always on in 24+; the `orcid` attribute is already declared there).
5. Website (`website/src/middleware/authMiddleware.ts`, `KeycloakClientManager.ts`) and backend (`KeycloakAdapter.kt`, `keycloak-admin-client` is already 26.0.13 in `build.gradle`): confirm that the issuer URL, token claims and admin API still match.
6. Use **#3405** as the reference diff; rebase or redo it on current `main`.

### Step 3: test on an HTTPS preview (required)
- Integration tests: auth, registration, edit-account, OIDC flow, CLI.
- Manual tests with **real ORCID** (the sandbox is not enough; it needs HTTPS and real credentials):
  - (a) new user signs up through ORCID;
  - (b) **an existing user linked to ORCID on 23 still logs in** (the alias `orcid` and the federated ID should be unchanged; ~ inference);
  - (c) the `orcid` attribute is filled in.
- Pipelines: ingest and preprocessing still get tokens with the password grant (ROPC is still supported in 26).

### Step 4: migration and rollback for real instances
- Keycloak migrates its database on start-up. Jumping from 23 to 26 crosses 24 and 25, so read all three upgrade guides.
- **Before the upgrade:** dump the Keycloak database. **Rollback:** restore the dump and the 23.0 image. A database migrated by 26 can't be used by 23.
- Write an upgrade note in `docs/.../reference/upgrade-guides/` for other instances.

## Acceptance criteria
- Keycloak 26.x is running on a preview, the integration tests are green, and the ORCID scenarios (a), (b) and (c) pass by hand.
- **All 29 HIGH advisories below are fixed** (the image is ≥ 26.6.4), checked again with OSV or Trivy after the upgrade.
- The upgrade guide and rollback steps are documented. #3400 and #3736 are closed.

## Open advisories in Keycloak 23.0.x that the upgrade fixes (OSV, checked 2026-11-23)
OSV lists **99 advisories** for `org.keycloak` 23.0.x: **29 high, 52 moderate, 18 low** (no critical). Below are the 29 HIGH ones. **"Fixed in" shows that the 26 line needs at least 26.6.4**, so target the latest 26.8.x, not 26.5.
⚠️ The ORCID plugin 1.5.0 is built against 26.5, so step 0 must confirm it also loads on 26.8 (minor-version SPI changes are possible).

| CVE | Summary | Fixed in | Relevance to Loculus | Advisory |
|---|---|---|---|---|
| CVE-2023-6841 | Keycloak Denial of Service vulnerability | 24.0.0 | Applies | [GHSA-w97f-w3hq-36g2](https://osv.dev/vulnerability/GHSA-w97f-w3hq-36g2) |
| CVE-2024-10039 | Keycloak mTLS Authentication Bypass via Reverse Proxy TLS Termination | 26.0.6 | mTLS / build-time | [GHSA-93ww-43rr-79v3](https://osv.dev/vulnerability/GHSA-93ww-43rr-79v3) |
| CVE-2024-10270 | org.keycloak:keycloak-services has Inefficient Regular Expression Complexity | 26.0.6 | Applies | [GHSA-wq8x-cg39-8mrr](https://osv.dev/vulnerability/GHSA-wq8x-cg39-8mrr) |
| CVE-2024-10451 | Keycloak Build Process Exposes Sensitive Data | 26.0.6 | mTLS / build-time | [GHSA-v7gv-xpgf-6395](https://osv.dev/vulnerability/GHSA-v7gv-xpgf-6395) |
| CVE-2024-1132 | Keycloak path traversal vulnerability in redirection validation | 22.0.10, 24.0.3 | Applies | [GHSA-72vp-xfrc-42xm](https://osv.dev/vulnerability/GHSA-72vp-xfrc-42xm) |
| CVE-2024-1249 | Keycloak's unvalidated cross-origin messages in checkLoginIframe leads to DDoS | 22.0.10, 24.0.3 | Applies | [GHSA-m6q9-p373-g5q8](https://osv.dev/vulnerability/GHSA-m6q9-p373-g5q8) |
| CVE-2024-2419 | Keycloak path traversal vulnerability in the redirect validation | 22.0.10, 24.0.3 | Applies | [GHSA-mrv8-pqfj-7gp5](https://osv.dev/vulnerability/GHSA-mrv8-pqfj-7gp5) |
| CVE-2024-3656 | Keycloak's admin API allows low privilege users to use administrative functions | 24.0.5 | Applies | [GHSA-2cww-fgmg-4jqc](https://osv.dev/vulnerability/GHSA-2cww-fgmg-4jqc) |
| CVE-2024-4540 | Keycloak exposes sensitive information in Pushed Authorization Requests (PAR) | 24.0.5 | Applies | [GHSA-69fp-7c8p-crjr](https://osv.dev/vulnerability/GHSA-69fp-7c8p-crjr) |
| CVE-2024-7341 | Keycloak has session fixation in Elytron SAML adapters | 22.0.12, 24.0.7 | SAML (not used by Loculus) | [GHSA-5rxp-2rhr-qwqv](https://osv.dev/vulnerability/GHSA-5rxp-2rhr-qwqv) |
| CVE-2024-8698 | Improper Verification of SAML Responses Leading to Privilege Escalation in Keycloak | 22.0.13, 24.0.8 | SAML (not used by Loculus) | [GHSA-xgfv-xpx8-qhcr](https://osv.dev/vulnerability/GHSA-xgfv-xpx8-qhcr) |
| CVE-2025-3501 | Keycloak hostname verification | 26.2.2 | Applies | [GHSA-hw58-3793-42gg](https://osv.dev/vulnerability/GHSA-hw58-3793-42gg) |
| CVE-2026-11800 | Keycloak has authentication bypass via jwt algorithm confusion | 26.6.4 | Applies | [GHSA-j97h-3f8r-mrjr](https://osv.dev/vulnerability/GHSA-j97h-3f8r-mrjr) |
| CVE-2026-1486 | Keycloak fails to verify if an Identity Provider (IdP) is enabled before issuing tokens | 26.4.9, 26.5.3 | Applies | [GHSA-37gf-gmxv-74wv](https://osv.dev/vulnerability/GHSA-37gf-gmxv-74wv) |
| CVE-2026-1529 | Keycloak affected by improper invitation token validation | 26.2.13, 26.4.9 | Applies | [GHSA-hcvw-475w-8g7p](https://osv.dev/vulnerability/GHSA-hcvw-475w-8g7p) |
| CVE-2026-2092 | Keycloak: Unauthorized access via improper validation of encrypted SAML assertions | 26.5.5 | SAML (not used by Loculus) | [GHSA-794g-x443-36f7](https://osv.dev/vulnerability/GHSA-794g-x443-36f7) |
| CVE-2026-2603 | Keycloak: Unauthorized authentication via disabled SAML Identity Provider | 26.5.5 | SAML (not used by Loculus) | [GHSA-x4p7-7chp-64hq](https://osv.dev/vulnerability/GHSA-x4p7-7chp-64hq) |
| CVE-2026-3009 | Keycloak allows authentication using an Identity Provider (IdP) even after it has been disabled | 26.5.5 | Applies | [GHSA-m297-3jv9-m927](https://osv.dev/vulnerability/GHSA-m297-3jv9-m927) |
| CVE-2026-3872 | Keycloak: Redirect URI validation bypass via ..;/ path traversal in OIDC auth endpoint | 26.5.7 | Applies | [GHSA-cjm2-j6cm-6p6m](https://osv.dev/vulnerability/GHSA-cjm2-j6cm-6p6m) |
| CVE-2026-4282 | Keycloak: Privilege escalation via forged authorization codes due to SingleUseObjectProvider is | 26.5.7 | Applies | [GHSA-hj93-h7pg-fh6v](https://osv.dev/vulnerability/GHSA-hj93-h7pg-fh6v) |
| CVE-2026-4634 | Keycloak: Application-Level DoS via Scope Processing | 26.5.7 | Applies | [GHSA-h4wv-g838-66g3](https://osv.dev/vulnerability/GHSA-h4wv-g838-66g3) |
| CVE-2026-4636 | Keycloak: UMA Policy Resource Injection Allows Unauthorized Cross-User Permission Grants | 26.5.7 | Applies | [GHSA-f2hx-5fx3-hmcv](https://osv.dev/vulnerability/GHSA-f2hx-5fx3-hmcv) |
| CVE-2026-7307 | Keycloak: Denial of Service via specially crafted SAML input | 26.6.2 | SAML (not used by Loculus) | [GHSA-p5mv-gj8j-xqgf](https://osv.dev/vulnerability/GHSA-p5mv-gj8j-xqgf) |
| CVE-2026-7504 | Keycloak: Open redirect when using wildcard valid redirect URIs in Keycloak | 26.6.2 | Applies | [GHSA-rp95-xpg9-c2cq](https://osv.dev/vulnerability/GHSA-rp95-xpg9-c2cq) |
| CVE-2026-7507 | Keycloak: Session fixation in OIDC login flow that can lead to account takeover | 26.6.2 | Applies | [GHSA-hf67-5vvq-fm3r](https://osv.dev/vulnerability/GHSA-hf67-5vvq-fm3r) |
| CVE-2026-7571 | Keycloak: Access token disclosure and implicit flow bypass via forged client data | 26.6.2 | Applies | [GHSA-hq3p-w4xv-x7vp](https://osv.dev/vulnerability/GHSA-hq3p-w4xv-x7vp) |
| CVE-2026-9086 | Keycloak vulnerable to cross-site scripting (xss) via case-insensitive uri validation bypass | 26.6.4 | Applies | [GHSA-v3f7-2p4r-mwfw](https://osv.dev/vulnerability/GHSA-v3f7-2p4r-mwfw) |
| CVE-2026-9795 | Keycloak has privilege escalation via improper scope mapping enforcement | 26.6.4 | Applies | [GHSA-32h4-44jj-c5vx](https://osv.dev/vulnerability/GHSA-32h4-44jj-c5vx) |
| CVE-2026-9800 | Keycloak vulnerable to authorization bypass via incorrect uri comparison | 26.0.10, 26.6.4 | Applies | [GHSA-f5p5-6xmx-p252](https://osv.dev/vulnerability/GHSA-f5p5-6xmx-p252) |

Note: OSV ranges come from the Maven artifacts. The "Relevance" column is a first triage: Loculus uses OIDC only, not SAML or mTLS.

## Draft comment for #3400 (post only after step 0, and with the user's OK)
> keycloak-orcid has fixed this upstream: **1.4.0** (2024-12-12) adds support for Keycloak ≥ 25, and **1.5.0** (2026-01-12) is built against Keycloak 26.5 ([releases](https://github.com/eosc-kc/keycloak-orcid/releases)). The fix is the `BrokeredIdentityContext(String, IdentityProviderModel)` constructor change from eosc-kc/keycloak-orcid#2, which is on `main` even though that PR is still open.
>
> So a fork may no longer be needed. In `keycloak/keycloakify/Dockerfile`, the asset is now `keycloak-orcid.jar` (no version suffix), and `KEYCLOAK_ORCID_VERSION` / `KEYCLOAK_MAJOR_VERSION` (lines 4–5) need bumping. #3405 already reported a working ORCID login on Keycloak 26 with the 1.4.x plugin. [I verified that 1.5.0 loads on Keycloak 26.5 / I haven't tested 1.5.0 yet.] Happy to open a PR if that's useful.
>
> For context, Keycloak 23.0.x currently has 29 HIGH advisories in OSV (e.g. CVE-2024-3656 admin API privilege issue, CVE-2024-1132 / CVE-2024-2419 / CVE-2026-3872 redirect validation bypasses, CVE-2026-7507 session fixation, CVE-2026-11800 JWT algorithm confusion). Most are fixed in 26.6.4+, so targeting the latest 26.8.x would close them.
>
> <details><summary>Full list of HIGH advisories (OSV)</summary>
>
> | CVE | Summary | Fixed in | Relevance to Loculus | Advisory |
> |---|---|---|---|---|
> | CVE-2023-6841 | Keycloak Denial of Service vulnerability | 24.0.0 | Applies | [GHSA-w97f-w3hq-36g2](https://osv.dev/vulnerability/GHSA-w97f-w3hq-36g2) |
> | CVE-2024-10039 | Keycloak mTLS Authentication Bypass via Reverse Proxy TLS Termination | 26.0.6 | mTLS / build-time | [GHSA-93ww-43rr-79v3](https://osv.dev/vulnerability/GHSA-93ww-43rr-79v3) |
> | CVE-2024-10270 | org.keycloak:keycloak-services has Inefficient Regular Expression Complexity | 26.0.6 | Applies | [GHSA-wq8x-cg39-8mrr](https://osv.dev/vulnerability/GHSA-wq8x-cg39-8mrr) |
> | CVE-2024-10451 | Keycloak Build Process Exposes Sensitive Data | 26.0.6 | mTLS / build-time | [GHSA-v7gv-xpgf-6395](https://osv.dev/vulnerability/GHSA-v7gv-xpgf-6395) |
> | CVE-2024-1132 | Keycloak path traversal vulnerability in redirection validation | 22.0.10, 24.0.3 | Applies | [GHSA-72vp-xfrc-42xm](https://osv.dev/vulnerability/GHSA-72vp-xfrc-42xm) |
> | CVE-2024-1249 | Keycloak's unvalidated cross-origin messages in checkLoginIframe leads to DDoS | 22.0.10, 24.0.3 | Applies | [GHSA-m6q9-p373-g5q8](https://osv.dev/vulnerability/GHSA-m6q9-p373-g5q8) |
> | CVE-2024-2419 | Keycloak path traversal vulnerability in the redirect validation | 22.0.10, 24.0.3 | Applies | [GHSA-mrv8-pqfj-7gp5](https://osv.dev/vulnerability/GHSA-mrv8-pqfj-7gp5) |
> | CVE-2024-3656 | Keycloak's admin API allows low privilege users to use administrative functions | 24.0.5 | Applies | [GHSA-2cww-fgmg-4jqc](https://osv.dev/vulnerability/GHSA-2cww-fgmg-4jqc) |
> | CVE-2024-4540 | Keycloak exposes sensitive information in Pushed Authorization Requests (PAR) | 24.0.5 | Applies | [GHSA-69fp-7c8p-crjr](https://osv.dev/vulnerability/GHSA-69fp-7c8p-crjr) |
> | CVE-2024-7341 | Keycloak has session fixation in Elytron SAML adapters | 22.0.12, 24.0.7 | SAML (not used by Loculus) | [GHSA-5rxp-2rhr-qwqv](https://osv.dev/vulnerability/GHSA-5rxp-2rhr-qwqv) |
> | CVE-2024-8698 | Improper Verification of SAML Responses Leading to Privilege Escalation in Keycloak | 22.0.13, 24.0.8 | SAML (not used by Loculus) | [GHSA-xgfv-xpx8-qhcr](https://osv.dev/vulnerability/GHSA-xgfv-xpx8-qhcr) |
> | CVE-2025-3501 | Keycloak hostname verification | 26.2.2 | Applies | [GHSA-hw58-3793-42gg](https://osv.dev/vulnerability/GHSA-hw58-3793-42gg) |
> | CVE-2026-11800 | Keycloak has authentication bypass via jwt algorithm confusion | 26.6.4 | Applies | [GHSA-j97h-3f8r-mrjr](https://osv.dev/vulnerability/GHSA-j97h-3f8r-mrjr) |
> | CVE-2026-1486 | Keycloak fails to verify if an Identity Provider (IdP) is enabled before issuing tokens | 26.4.9, 26.5.3 | Applies | [GHSA-37gf-gmxv-74wv](https://osv.dev/vulnerability/GHSA-37gf-gmxv-74wv) |
> | CVE-2026-1529 | Keycloak affected by improper invitation token validation | 26.2.13, 26.4.9 | Applies | [GHSA-hcvw-475w-8g7p](https://osv.dev/vulnerability/GHSA-hcvw-475w-8g7p) |
> | CVE-2026-2092 | Keycloak: Unauthorized access via improper validation of encrypted SAML assertions | 26.5.5 | SAML (not used by Loculus) | [GHSA-794g-x443-36f7](https://osv.dev/vulnerability/GHSA-794g-x443-36f7) |
> | CVE-2026-2603 | Keycloak: Unauthorized authentication via disabled SAML Identity Provider | 26.5.5 | SAML (not used by Loculus) | [GHSA-x4p7-7chp-64hq](https://osv.dev/vulnerability/GHSA-x4p7-7chp-64hq) |
> | CVE-2026-3009 | Keycloak allows authentication using an Identity Provider (IdP) even after it has been disabled | 26.5.5 | Applies | [GHSA-m297-3jv9-m927](https://osv.dev/vulnerability/GHSA-m297-3jv9-m927) |
> | CVE-2026-3872 | Keycloak: Redirect URI validation bypass via ..;/ path traversal in OIDC auth endpoint | 26.5.7 | Applies | [GHSA-cjm2-j6cm-6p6m](https://osv.dev/vulnerability/GHSA-cjm2-j6cm-6p6m) |
> | CVE-2026-4282 | Keycloak: Privilege escalation via forged authorization codes due to SingleUseObjectProvider is | 26.5.7 | Applies | [GHSA-hj93-h7pg-fh6v](https://osv.dev/vulnerability/GHSA-hj93-h7pg-fh6v) |
> | CVE-2026-4634 | Keycloak: Application-Level DoS via Scope Processing | 26.5.7 | Applies | [GHSA-h4wv-g838-66g3](https://osv.dev/vulnerability/GHSA-h4wv-g838-66g3) |
> | CVE-2026-4636 | Keycloak: UMA Policy Resource Injection Allows Unauthorized Cross-User Permission Grants | 26.5.7 | Applies | [GHSA-f2hx-5fx3-hmcv](https://osv.dev/vulnerability/GHSA-f2hx-5fx3-hmcv) |
> | CVE-2026-7307 | Keycloak: Denial of Service via specially crafted SAML input | 26.6.2 | SAML (not used by Loculus) | [GHSA-p5mv-gj8j-xqgf](https://osv.dev/vulnerability/GHSA-p5mv-gj8j-xqgf) |
> | CVE-2026-7504 | Keycloak: Open redirect when using wildcard valid redirect URIs in Keycloak | 26.6.2 | Applies | [GHSA-rp95-xpg9-c2cq](https://osv.dev/vulnerability/GHSA-rp95-xpg9-c2cq) |
> | CVE-2026-7507 | Keycloak: Session fixation in OIDC login flow that can lead to account takeover | 26.6.2 | Applies | [GHSA-hf67-5vvq-fm3r](https://osv.dev/vulnerability/GHSA-hf67-5vvq-fm3r) |
> | CVE-2026-7571 | Keycloak: Access token disclosure and implicit flow bypass via forged client data | 26.6.2 | Applies | [GHSA-hq3p-w4xv-x7vp](https://osv.dev/vulnerability/GHSA-hq3p-w4xv-x7vp) |
> | CVE-2026-9086 | Keycloak vulnerable to cross-site scripting (xss) via case-insensitive uri validation bypass | 26.6.4 | Applies | [GHSA-v3f7-2p4r-mwfw](https://osv.dev/vulnerability/GHSA-v3f7-2p4r-mwfw) |
> | CVE-2026-9795 | Keycloak has privilege escalation via improper scope mapping enforcement | 26.6.4 | Applies | [GHSA-32h4-44jj-c5vx](https://osv.dev/vulnerability/GHSA-32h4-44jj-c5vx) |
> | CVE-2026-9800 | Keycloak vulnerable to authorization bypass via incorrect uri comparison | 26.0.10, 26.6.4 | Applies | [GHSA-f5p5-6xmx-p252](https://osv.dev/vulnerability/GHSA-f5p5-6xmx-p252) |
>
> </details>

## Follow-up (separate task)
- Move pipelines (ingest, preprocessing, CLI) from the password grant to **service accounts with client credentials**. This is better security and makes any future IdP change (Zitadel, Authentik) possible.
