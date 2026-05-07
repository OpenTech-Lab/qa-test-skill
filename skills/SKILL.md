---
name: repo-qa-initializer
description: Initialize or update a repository-level QA workflow. Use when the user wants an AI agent to inspect the current repo, add missing QA scripts/workflows, create test planning docs based on changed files, run local QA once, and enable PR comment `/qa` testing through GitHub Actions.
---

# Repo QA Initializer Skill

## Purpose

This skill helps initialize a repository so an AI/dev workflow can run lightweight product-level QA both locally and from a GitHub PR comment.

Target flow:

```text
local repo changes
  -> AI inspects repo
  -> AI adds or updates QA scripts and GitHub workflow if missing
  -> AI creates a test plan document for the current change scope
  -> AI runs local QA once if possible
  -> AI opens/updates PR
  -> PR comment `/qa` triggers GitHub Actions QA run
  -> workflow uploads reports and comments result back to PR
```

## When to use

Use this skill when the user asks for any of these:

- initialize QA for this repo
- add `/qa` PR command
- add QA workflow
- create affected test plan from current diff
- run local QA before PR
- prepare repo for Playwright/API/E2E smoke testing
- make a repeatable QA gate for small projects

## Core principles

1. Do not assume every repo is Node.js. Detect the repo first.
2. Prefer minimal, non-invasive changes.
3. Do not rewrite unrelated files.
4. If a repo already has tests, reuse them instead of replacing them.
5. If a repo has no tests, create a minimal smoke test scaffold only when appropriate.
6. Always create or update a QA documentation file for the current change scope.
7. Local QA should run before relying on PR `/qa`, when dependencies and environment allow it.
8. PR `/qa` is for traceability and reproducibility, not a replacement for local testing.
9. Secrets must never be committed. Refer to GitHub Actions secrets by name only.
10. For PR-comment-triggered workflows, checkout the PR head SHA, not the base branch.

## Repo inspection checklist

Before editing, inspect:

```bash
git status --short
git diff --name-only origin/main...HEAD 2>/dev/null || git diff --name-only HEAD~1...HEAD 2>/dev/null || git diff --name-only
ls -la
find . -maxdepth 3 -type f \( -name "package.json" -o -name "pnpm-lock.yaml" -o -name "package-lock.json" -o -name "yarn.lock" -o -name "bun.lockb" -o -name "pyproject.toml" -o -name "requirements.txt" -o -name "Cargo.toml" -o -name "pom.xml" -o -name "build.gradle" -o -name "go.mod" \) 2>/dev/null
find .github/workflows -maxdepth 1 -type f 2>/dev/null || true
```

Detect package manager:

- `pnpm-lock.yaml` -> pnpm
- `package-lock.json` -> npm
- `yarn.lock` -> yarn
- `bun.lockb` -> bun
- `pyproject.toml` or `requirements.txt` -> Python
- `Cargo.toml` -> Rust
- `pom.xml` or `build.gradle` -> Java
- `go.mod` -> Go

For the first version, prioritize Node/Next.js/React projects. For non-Node repos, create `docs/qa/QA_PLAN.md` and a placeholder workflow that calls a repo-provided command, but do not invent a full test framework without user approval.

## Standard files to create or update

Prefer this structure:

```text
.github/workflows/qa.yml
.github/qa/affected-tests.json
docs/qa/QA_PLAN.md
docs/qa/QA_REPORT_TEMPLATE.md
tests/e2e/
tests/api/
```

Only create `tests/e2e` or `tests/api` if the repo is a suitable Node/web repo and no equivalent test folders exist.

## Node repo initialization rules

If `package.json` exists:

1. Detect package manager.
2. Check if `@playwright/test` is installed.
3. If missing, add it as a dev dependency using the detected package manager.
4. Add scripts if missing:

```json
{
  "test:api": "playwright test tests/api",
  "test:e2e:smoke": "playwright test tests/e2e --grep @smoke",
  "test:qa": "npm run test:api --if-present && npm run test:e2e:smoke --if-present"
}
```

Adjust command syntax by package manager:

- pnpm: `pnpm test:api && pnpm test:e2e:smoke`
- npm: `npm run test:api --if-present && npm run test:e2e:smoke --if-present`
- yarn: `yarn test:api && yarn test:e2e:smoke`

For pnpm, prefer:

```json
{
  "packageManager": "pnpm@9.15.4"
}
```

Only add `packageManager` if the repo clearly uses pnpm and the field is missing.

## Minimal Playwright config

If Playwright is not configured and the repo is a web app, create `playwright.config.ts`:

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [['html'], ['list']],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
  ],
});
```

## Minimal smoke tests

If the repo has no E2E smoke test, create `tests/e2e/home.spec.ts`:

```ts
import { test, expect } from '@playwright/test';

test('@smoke home page loads', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveTitle(/.+/);
});
```

If API base URL is available, create `tests/api/health.spec.ts`:

```ts
import { test, expect, request } from '@playwright/test';

test('API health check', async () => {
  const baseURL = process.env.API_BASE_URL;
  test.skip(!baseURL, 'API_BASE_URL is not configured');

  const api = await request.newContext({ baseURL });
  const res = await api.get('/health');
  expect([200, 204, 404]).toContain(res.status());
});
```

If `/health` likely does not exist, do not force this test. Prefer creating a skipped/template test and document that the project should replace it.

## GitHub Actions workflow

Create `.github/workflows/qa.yml` if missing. If one exists, preserve existing behavior and add a new workflow only if needed.

Recommended workflow for Node repos:

```yaml
name: QA Tests

on:
  issue_comment:
    types: [created]

  workflow_dispatch:

jobs:
  qa:
    if: github.event.issue.pull_request && contains(github.event.comment.body, '/qa')
    runs-on: ubuntu-latest
    timeout-minutes: 20
    
    permissions:
      contents: read
      pull-requests: write
      checks: write

    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha || github.ref }}

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: latest

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Install Playwright Browsers
        run: pnpm exec playwright install --with-deps

      - name: Run API Tests
        run: pnpm test:api
        env:
          API_BASE_URL: ${{ secrets.QA_API_BASE_URL }}

      - name: Run E2E Smoke Tests
        run: pnpm test:e2e:smoke
        env:
          BASE_URL: ${{ secrets.QA_WEB_BASE_URL }}

      - name: Upload Playwright Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 14
```

## Affected test mapping

Create `.github/qa/affected-tests.json`:

```json
{
  "app/**": ["tests/e2e/**/*.spec.ts"],
  "pages/**": ["tests/e2e/**/*.spec.ts"],
  "src/app/**": ["tests/e2e/**/*.spec.ts"],
  "src/pages/**": ["tests/e2e/**/*.spec.ts"],
  "components/**": ["tests/e2e/**/*.spec.ts"],
  "app/api/**": ["tests/api/**/*.spec.ts"],
  "pages/api/**": ["tests/api/**/*.spec.ts"],
  "src/app/api/**": ["tests/api/**/*.spec.ts"],
  "api/**": ["tests/api/**/*.spec.ts"],
  "server/**": ["tests/api/**/*.spec.ts"],
  "lib/**": ["tests/api/**/*.spec.ts", "tests/e2e/**/*.spec.ts"],
  "package.json": ["tests/api/**/*.spec.ts", "tests/e2e/**/*.spec.ts"],
  "playwright.config.*": ["tests/api/**/*.spec.ts", "tests/e2e/**/*.spec.ts"]
}
```

This mapping is documentation-first. Do not over-engineer impacted test selection until the repo has enough tests.

## QA plan document

Create or update `docs/qa/QA_PLAN.md` based on current diff:

```md
# QA Plan

## Change scope

- Changed files:
  - `<file>`

## Risk areas

- UI:
- API:
- Auth/permissions:
- Data persistence:
- Regression risk:

## Local QA commands

```bash
<package-manager> test:qa
```

## PR QA command

```text
/qa
```

## Required checks for this PR

- [ ] Lint/typecheck if available
- [ ] API tests for affected endpoints
- [ ] E2E smoke tests for affected user flows
- [ ] Manual check for changed screens if no automated test exists
- [ ] Verify report/artifacts in GitHub Actions

## Notes

- Add project-specific test accounts as GitHub Actions secrets.
- Do not commit secrets.
```

## Local execution

After initialization, run the safest available commands.

For Node repos:

```bash
<pm> install
<pm> test:qa
```

If `test:qa` fails because there are no tests or missing environment variables, document the failure clearly in `docs/qa/QA_PLAN.md` instead of hiding it.

## Commit/PR guidance

Suggested PR comment after local run:

```md
## QA preparation

- Added/updated `/qa` PR workflow.
- Added/updated QA scripts.
- Added/updated QA plan for this change scope.
- Local QA: `<passed/failed/skipped>`

To run PR QA, comment:

```text
/qa
```
```

## Safety rules

- Do not add real credentials.
- Do not disable existing tests.
- Do not loosen branch protection.
- Do not add broad write permissions to GitHub Actions.
- Do not use `pull_request_target` for untrusted PR test execution.
- Keep workflow permissions minimal.
- Avoid changing production deploy workflows unless explicitly requested.
