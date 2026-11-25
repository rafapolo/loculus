# Loculus auth: ORCID on Keycloak 25+ and OSS alternatives

> Research from 2026-11-23 (a subagent plus my own checks). Local file, ignored through `.git/info/exclude`.
> ✅ = verified (GitHub API, repo files); ~ = inference, not tested.

## 1. The ORCID blocker is already fixed upstream ✅
- **Cause:** Keycloak 25 changed the SPI constructor `BrokeredIdentityContext(String)` → `(String, IdentityProviderModel)` and removed `setIdpConfig()`. Plugin `eosc-kc/keycloak-orcid` **1.3.0** (what Loculus uses) fails in the ORCID callback with `NoSuchMethodError`. Loculus hit this in #3374 and reverted the upgrade.
- **The fix exists:**
  - **1.4.0** (2024-12-12, "for Keycloak above 25");
  - **1.5.0** (2026-01-12, built against Keycloak 26.5).
  
  Both releases ship the asset as `keycloak-orcid.jar`, without the version in the name.
- **Loculus is still pinned to the broken version:** `keycloak/keycloakify/Dockerfile:4` sets `KEYCLOAK_ORCID_VERSION=1.3.0`, `:5` sets `KEYCLOAK_MAJOR_VERSION=24`, and the `wget` URL on `:14` expects the old file name.
- **It was already tested:** PR **#3405** ("upgrade to Keycloak 26", fork plugin 1.4.1) reports a working ORCID login and sign-up. It has been open, unreviewed, since Dec 2024 (last update Jan 2025).
- **So:** #3400 is outdated, and ORCID **no longer blocks** #3736 (Keycloak 26).

## 2. How Loculus uses auth today ✅
- **Website:** OIDC authorization code flow; the backend validates the JWT (Spring resource server).
- **Pipelines and CLI:** the password grant (ROPC), e.g. `ingest/scripts/loculus_client.py:62-63`.
- **Backend admin lookups:** `KeycloakAdapter.kt` (Keycloak admin client).
- **Realm:** ORCID IdP (alias `orcid`) with plugin mappers; self-registration with terms; user attribute `orcid` (`keycloak-config-map.yaml`).

## 3. OSS options compared
Licence and activity are ✅ (GitHub API, Nov 2026). The other cells are ~ (general knowledge, not tested).

| | Keycloak 26 | Authelia + lldap (#6419) | Authentik | Zitadel | Ory Kratos + Hydra | Casdoor | Dex |
|---|---|---|---|---|---|---|---|
| Licence | Apache-2.0 | Apache-2.0 / GPL-3.0 | MIT core + enterprise parts | AGPL-3.0 | Apache-2.0 | Apache-2.0 | Apache-2.0 |
| Latest release | 26.8.2 (Nov 2026) | 4.39.28 / lldap 0.6.3 | 2026.8.6 | v4.20.5 | v26.2.0 | v4.30.0 | v2.46.1 |
| OIDC + JWT for Spring | ✅ mature | ✅ | ✅ | ✅ | ✅ (Hydra) | ✅ | proxy only |
| ORCID login | plugin 1.5.0 or generic OIDC | ✗ (#6419 drops it) | upstream OIDC | generic OIDC | generic OIDC | OAuth/OIDC provider | OIDC connector |
| Self sign-up + terms + email | ✅ (theme exists) | ✗ needs a custom service | configurable flows | ✅ | ✅ (self-service, own UI) | ✅ | ✗ |
| Pipelines (today ROPC) | ROPC + client credentials | ✗ ROPC; device code / client credentials only | ROPC via flow + client credentials | client credentials / JWT | client credentials, no ROPC | ROPC + client credentials | limited |
| Admin API for the backend | REST (current adapter) | LDAP | REST | API | Kratos admin API | REST | none |
| 2FA, password reset | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✗ |
| Theme | keycloakify (done) | limited CSS | templates/CSS | UI customization | you build the UI | customizable | minimal |
| User migration from Keycloak | none needed | hashes don't migrate → mass reset | import via API | import with hashes | import with hashes | import | n/a |
| Ops weight | JVM + Postgres (~1 GB) | 2 light pods + custom service | server + worker + Postgres | Go binary + Postgres | 2 Go services + Postgres + UI | light | very light |

Also considered:
- **Kanidm:** OIDC, but no public self-service sign-up.
- **Logto:** sign-up, social login and admin API; SaaS-oriented.
- **SuperTokens:** a session library, not a full IdP.

## 4. Recommendation
**Short term (unblock security): stay on Keycloak and upgrade to 26.**
1. In the Dockerfile, set ORCID plugin 1.5.0, fix the asset name and set `KEYCLOAK_MAJOR_VERSION` to 26.
2. Revive or redo #3405 (and the rest of #3299) on the 26.5 line.
3. Test on an HTTPS preview with real ORCID: a new login, an existing linked user, and the `orcid` attribute mapping.
4. Take a database dump first, and keep the old image plus the dump for rollback. Then close #3400.

Why this path:
- it meets all the criteria with no rewrite;
- users and ORCID links stay intact (~ alias and ID unchanged);
- it closes the Keycloak 23 advisories (29 high in OSV).

**Long term (only if Keycloak's operational weight is the real pain):**
- Authelia + lldap is a poor trade: it loses ORCID and ROPC, adds a custom registration service and forces a password reset for every user.
- Better candidates for a PoC are **Zitadel** (light, sign-up, hash import) and **Authentik** (sign-up, upstream OIDC).
- Any migration must rebuild ORCID links from the stored `orcid` attribute and redo the theme.
- **Worth doing anyway:** move pipelines from ROPC to service accounts with client credentials, which makes any future IdP switch easier.

## 5. Open question
#3400 looks unblocked: the ORCID plugin was fixed upstream in 1.4.0, 1.5.0 is built for Keycloak 26.5, and #3405 already tested it. Is Authelia still the direction, given it drops ORCID login and the password grant the pipelines use? Moving pipelines to client credentials would make either path easier.
