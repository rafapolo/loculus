The `backend`, `website`, `integration-tests` and `raw-reads-processing` directories each contain their own `AGENTS.md` files with additional specific instructions.

Use conventional commits as titles for PRs, e.g. feat(deployment):xx, fix!(website):xx, chore(backend):xx.
Components include: website, backend, deployment, preprocessing, ingest, deposition.

Write detailed PR summaries, not just short bullet points. When creating PRs, you should generally create them as a draft PR. If you have access to https://github.com/loculus-project/agent_store/ you can upload screenshots (which you might take using `playwright-cli` there to be able to use in PR descriptions, and maybe issue descriptions).

## Preventing flaky Playwright tests (website)

When adding or modifying interactive components in the website, ensure they are disabled until React hydration completes to prevent race conditions in Playwright tests:

- **For buttons**: Use `Button` from `src/components/common/Button.tsx` instead of native `<button>`
- **For Headless UI Combobox**: Import from `src/components/common/headlessui/Combobox.tsx` instead of `@headlessui/react`
- **For other interactive elements**: Wrap with `DisabledUntilHydrated` or use the `useClientFlag` hook

These wrappers automatically disable components until client-side hydration is complete, preventing Playwright from interacting with them before they're ready.

## Updating Conda Environment Dependencies

Dependabot updates conda dependencies in the `environment.yml` files listed in `.github/dependabot.yml`. Some packages are pinned on purpose with `ignore:` blocks (for example `nextclade`, see #7784).
The `maintenance-scripts/` folder contains a script to check conda packages by hand, for example pinned ones or environments not listed in `dependabot.yml`.

