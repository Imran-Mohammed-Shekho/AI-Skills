# CI/CD as a security layer

## Workflows (GitHub Actions)

- `ci.yml` — on push/PR: backend typecheck + full tests against a real Postgres service; admin
  typecheck + build. Also `workflow_call` so deploy can reuse it.
  Concurrency group `ci-${{ github.workflow }}-${{ github.ref }}` so the copy called from deploy
  never cancels the push copy.
- `app.yml` — mobile analyze + tests, separate so it never blocks a server deploy.
- `deploy.yml` — on push to `main`: needs CI → rsync **only git-tracked files** → run
  `deploy/deploy.sh` on the server over SSH (key + pinned known_hosts in secrets).
- `deploy.sh` — DB backup (keep last 14) → `npm ci` → build → migrate → build admin →
  `pm2 reload` → health check loop; on failure print logs and exit 1.
- `security.yml` — gitleaks (needs `permissions: contents: read, pull-requests: read`),
  `npm audit --omit=dev --audit-level=high`, dependency review on PRs.
- `release-mobile.yml` — signed builds from secrets (`ANDROID_KEYSTORE_BASE64`, passwords, alias).

## Dependabot, quiet

Security fixes only, grouped, so it doesn't flood the repo with branches:
```yaml
version: 2
updates:
  - package-ecosystem: npm
    directory: /backend
    schedule: { interval: weekly }
    open-pull-requests-limit: 0        # no version-bump PRs
    groups:
      security:
        applies-to: security-updates
        patterns: ["*"]
```
(repeat per directory / ecosystem; enable "Dependabot security updates" in repo settings).

## Flaky tests are security bugs too

A failing test blocks deploys (good), so fix flakes fast: tests that rely on leftover state, random
values that sometimes collide (e.g. replacing the last char of a base64 string with a char it might
already be), background work not awaited (poll up to ~2 s instead of asserting immediately).

## Workflow habits

Feature branch → PR → CI green → merge → auto deploy. Explain this to the user in plain words if
they're new to it. Never enable auto-merge or push to main without being asked.
