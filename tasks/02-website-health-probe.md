# 02 · Deployment: cheap health endpoint for the website probes

> **Solves:** Stops avoidable website pod restarts by probing a cheap health endpoint instead of rendering the home page within a 1 s timeout.

| Field | Value |
|---|---|
| Issue | [#7751](https://github.com/loculus-project/loculus/issues/7751) |
| Component | website, deployment |
| PR title | `fix(deployment): point website probes at a lightweight health endpoint` |
| Effort | S (½ day) |
| Risk | Low: watch for a probe that reports healthy while SSR is broken |
| Status | todo · 🔴 taken |

## Existing work (checked 2026-11-23)
**PR #7771 is open** (`fix(website,deployment): point website k8s probes at dedicated…`, by corneliusroemer, 2026-11-13, review required). Don't duplicate it: review or test #7771, or pick another task.

## Problem
The website pod restarts even when nothing is wrong.

## Root cause
- `kubernetes/loculus/templates/loculus-website.yaml:38-49`: both `livenessProbe` and `readinessProbe` call `GET /` on port 3000.
- `/` renders the full home page, which may call the backend and LAPIS. The probe uses the default `timeoutSeconds: 1`, so a slow render fails the probe and the pod gets restarted.

## Plan
1. Add an Astro API route, e.g. `website/src/pages/api/health.ts`, that returns `200 {"status":"ok"}` without calling upstream services.
   - **Liveness:** the process answers.
   - **Readiness:** optionally `/api/ready`, which checks that the runtime config is loaded. It should not call the backend, so a backend outage doesn't remove every website pod from the Service.
2. Point both probes at the new routes and set explicit `timeoutSeconds` (3–5 s) and `failureThreshold` (3–6).
3. Exclude the routes from the auth middleware (`website/src/middleware/authMiddleware.ts`) and from request logging.
4. Consider a `startupProbe` so the slow cold start doesn't need a long `initialDelaySeconds`.

## Acceptance criteria
- `curl /api/health` answers in a few ms and doesn't touch the backend (check the logs).
- Helm renders the new probe paths (`helm template`), and the schema lint passes.
- A unit test for the route (vitest); the existing e2e tests pass.

## Files
- `website/src/pages/api/health.ts` (new)
- `website/src/middleware/authMiddleware.ts`
- `kubernetes/loculus/templates/loculus-website.yaml`
