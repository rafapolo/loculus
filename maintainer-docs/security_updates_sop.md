# Security Updates Standard Operating Procedure

## Background

Most dependency updates reach us through Dependabot (see the [Dependabot duty SOP](./dependabot_duty_sop.md)). Dependabot only sees what is declared in [`.github/dependabot.yml`](https://github.com/loculus-project/loculus/blob/main/.github/dependabot.yml) and what its parsers understand, so some parts of a Loculus deployment never get an automatic update or security alert. This SOP lists those blind spots and describes how to check them.

Vulnerability reports from outside the team follow [`SECURITY.md`](../SECURITY.md) and are not covered here.

## What Dependabot covers

| Ecosystem      | Directories                                                                                                                           |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| GitHub Actions | `/`                                                                                                                                   |
| npm            | `website/`, `keycloak/keycloakify`, `integration-tests/` (other `package.json` files, such as `docs/`, only receive security updates) |
| Gradle         | `backend/`                                                                                                                            |
| conda          | `preprocessing/nextclade`, `ingest`, `ena-submission`, `raw-reads-processing`, `taxonomy/taxonomy_service`                            |
| pip            | `preprocessing/nextclade`, `cli`, `ena-submission`, `loculus-silo`                                                                    |
| Docker         | Dockerfiles and Kubernetes manifests in the directories listed under `docker`                                                         |

Some dependencies are pinned on purpose through `ignore:` blocks (for example `nextclade`, see #7784). Check the linked issue before overriding a pin.

## Blind spots

### Third-party images mirrored to GHCR

Third-party images are mirrored to `ghcr.io/loculus-project/mirror/*` from [`.github/mirror-images.txt`](https://github.com/loculus-project/loculus/blob/main/.github/mirror-images.txt) and referenced in `kubernetes/loculus/values.yaml` as separate `repository` and `tag` fields. Dependabot does not open pull requests for these, so new upstream releases and their security fixes have to be picked up by hand. This includes:

- Keycloak (`quay.io/keycloak/keycloak`), which handles authentication for every instance
- PostgreSQL (development and stand-in databases, Flyway)
- MinIO (development S3)
- busybox, alpine and kubectl (init and helper containers)

### Other sources

- **Transitive dependencies** that are held back by a parent package (for example an old major version that only a build tool depends on). Dependabot cannot update them until the parent allows it.
- **Tools and data downloaded at build or run time**, such as the `readtools` JAR in `raw-reads-processing`, Nextclade datasets and the SILO version used by the importer.
- **Base images** of our own Dockerfiles, between Dependabot runs.

## Monthly check

Whoever is on Dependabot duty at the start of the month should also do the following:

1. **Mirrored images:** compare each entry in `.github/mirror-images.txt` with the latest upstream release. For Keycloak and PostgreSQL, read the upstream security advisories for the version we run. If an update is needed, add the new tag to `mirror-images.txt`, publish it, and only then update `values.yaml` (see `kubernetes/README.md`).
2. **Lockfiles:** run a vulnerability scan over the lockfiles, for example:
   - `npm audit --package-lock-only` in each npm project;
   - [`osv-scanner`](https://github.com/google/osv-scanner) with `--lockfile` for `gradle.lockfile`, `uv.lock`, `yarn.lock` and `package-lock.json`.
3. **Images:** scan the published `ghcr.io/loculus-project/*` images and the mirrored images with a container scanner such as [Trivy](https://github.com/aquasecurity/trivy) (`trivy image <image>`).
4. **GitHub security tab:** if you have access, review open code scanning and Dependabot alerts.

## Triage

For every finding, decide:

- **Is it reachable in production?** A vulnerability in a runtime dependency of the backend, website, Keycloak or a pipeline service matters more than one in a build-only tool (for example `keycloak/keycloakify` dependencies that are only used to build the theme) or a test dependency.
- **Is there a fix?** If there is, open a PR (or merge the Dependabot PR). If there isn't, note the mitigation and link the upstream issue.
- **Does it need a private advisory?** Track vulnerabilities in our own code privately, following `SECURITY.md`, not in a public issue.

Upgrades that need migration work, such as a new Keycloak major version, should get their own issue with a test plan on a preview deployment.
