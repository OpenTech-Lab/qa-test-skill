# Repo QA Initializer Skill

A lightweight Claude Code skill that initializes a repository-level QA workflow for small projects. It helps an AI coding agent inspect the current repo, add a reusable `/qa` GitHub PR workflow when missing, create a QA plan from the current diff, run local QA once, and prepare the PR so CI QA can be triggered and recorded.

This skill is designed for solo developers, small client projects, and AI-assisted development workflows where you want practical pre-merge QA without adopting a large QA platform.

## What it does

The skill guides an AI coding agent through this flow:

```text
Inspect current repository
  -> Detect project type and package manager
  -> Add or update lightweight QA scripts if missing
  -> Add or update GitHub Actions /qa workflow if missing
  -> Create docs/qa/QA_PLAN.md based on current changes
  -> Create or update affected-test mapping
  -> Run local QA once when possible
  -> Let the PR owner comment /qa to run CI QA and keep a record
```

The goal is to standardize every project around a simple command such as:

```bash
pnpm test:qa
```

or the equivalent command for the repo's package manager.

## Repository contents

This repository contains:

```text
repo-qa-initializer-skill/
  SKILL.md
  README.md
  .github/
    workflows/
      qa-test.yml
```

`SKILL.md` is the actual Claude Code skill. `qa-test.yml` is an optional workflow for testing or validating this skill repository itself.

## Requirements

You need:

```text
Claude Code or another agent that supports local skills
Git
A target repository where you want to initialize QA
Optional: Node.js / pnpm / npm / yarn if the target repo is a web or Node project
Optional: GitHub Actions if you want PR comment /qa automation
```

The skill can inspect non-Node repos, but the first practical implementation is most useful for Node, Next.js, React, and web/API projects.

## Installation

Clone this repository:

```bash
git clone https://github.com/YOUR_USER_OR_ORG/repo-qa-initializer-skill.git
cd repo-qa-initializer-skill
```

Copy the skill into your Claude skills directory:

```bash
mkdir -p ~/.claude/skills/repo-qa-initializer
cp SKILL.md ~/.claude/skills/repo-qa-initializer/SKILL.md
```

After that, open your target project with Claude Code and ask it to use the skill.

## Usage

From inside the target repository, ask your AI coding agent:

```text
Use repo-qa-initializer. Initialize QA for this repo. If missing, add a /qa GitHub workflow, test:qa scripts, minimal Playwright setup, affected test mapping, and docs/qa/QA_PLAN.md based on the current diff. Run local QA once if possible. Do not change unrelated files.
```

A shorter version:

```text
Use repo-qa-initializer. Initialize QA for this repo, create a QA plan for the current changes, run local QA once, and prepare the PR so I can comment /qa to run CI QA.
```

For an existing PR workflow:

```text
Use repo-qa-initializer. Review this PR change scope, update docs/qa/QA_PLAN.md, run local QA once, and make sure /qa can be used in the PR to run CI QA.
```

## Expected generated files in the target repo

Depending on what already exists, the skill may create or update files like these:

```text
.github/workflows/qa.yml
.github/qa/affected-tests.json
docs/qa/QA_PLAN.md
docs/qa/QA_REPORT_TEMPLATE.md
playwright.config.ts
tests/e2e/home.spec.ts
tests/api/health.spec.ts
package.json
```

The skill should reuse existing test folders, scripts, and workflows when possible. It should not replace a mature test setup with a new one.

## How `/qa` works in a PR

After the target repo has a QA workflow, open a pull request and comment:

```text
/qa
```

The GitHub Actions workflow should:

```text
Read the PR information
Checkout the PR head SHA
Install dependencies
Run the repo's test:qa command
Upload reports as workflow artifacts
Comment the result back to the PR
```

You can also use a more descriptive comment:

```text
/qa Please test affected API paths, core E2E smoke flows, and related changed screens.
```

The first version may still run a fixed QA command. The useful part is that every PR keeps a visible QA record.

## Local QA first, PR QA second

For small projects, local QA is often enough during development. The recommended workflow is:

```text
1. Make changes locally.
2. Ask the AI agent to update the QA plan from the current diff.
3. Run local QA once.
4. Open or update the PR.
5. Comment /qa in the PR.
6. GitHub Actions runs QA again and keeps the report.
```

This gives you both speed and traceability.

## Example target repo scripts

For a pnpm web project:

```json
{
  "scripts": {
    "test:api": "playwright test tests/api",
    "test:e2e:smoke": "playwright test tests/e2e --grep @smoke",
    "test:qa": "pnpm test:api && pnpm test:e2e:smoke"
  },
  "devDependencies": {
    "@playwright/test": "^1.0.0"
  }
}
```

For npm projects:

```json
{
  "scripts": {
    "test:api": "playwright test tests/api",
    "test:e2e:smoke": "playwright test tests/e2e --grep @smoke",
    "test:qa": "npm run test:api --if-present && npm run test:e2e:smoke --if-present"
  }
}
```

## Example target repo workflow

The skill may create a workflow similar to this in the target repo:

```yaml
name: QA

on:
  issue_comment:
    types: [created]
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: read
  issues: write
  actions: read

jobs:
  qa:
    if: >-
      github.event_name == 'workflow_dispatch' ||
      (github.event.issue.pull_request && contains(github.event.comment.body, '/qa'))
    runs-on: ubuntu-latest

    steps:
      - name: Get PR info
        if: github.event_name == 'issue_comment'
        uses: actions/github-script@v7
        id: pr
        with:
          script: |
            const pr = await github.rest.pulls.get({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number,
            });
            core.setOutput('head_sha', pr.data.head.sha);

      - name: Checkout PR branch
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event_name == 'issue_comment' && steps.pr.outputs.head_sha || github.sha }}

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 9

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Install Playwright browsers
        run: pnpm exec playwright install --with-deps

      - name: Run QA
        run: pnpm test:qa
        env:
          BASE_URL: ${{ secrets.QA_WEB_BASE_URL }}
          API_BASE_URL: ${{ secrets.QA_API_BASE_URL }}

      - name: Upload Playwright report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/

      - name: Comment result
        if: always() && github.event_name == 'issue_comment'
        uses: actions/github-script@v7
        with:
          script: |
            const result = '${{ job.status }}';
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `QA finished: **${result}**\n\nCheck the workflow artifact for the Playwright report.`
            });
```

Adjust this workflow for npm, yarn, bun, Python, Rust, Java, or other project types.

## GitHub secrets

Some projects need environment variables for QA:

```text
QA_WEB_BASE_URL
QA_API_BASE_URL
QA_ADMIN_EMAIL
QA_ADMIN_PASSWORD
```

Do not commit real credentials. Store them in GitHub Actions secrets and reference only the secret names in workflows or documentation.

## Safety rules for the AI agent

The skill tells the agent to follow these rules:

```text
Inspect before editing.
Do not assume every repo is Node.js.
Prefer minimal changes.
Reuse existing tests and workflows.
Do not change unrelated files.
Do not commit secrets.
Checkout the PR head SHA for PR comment workflows.
Document tested, skipped, and manual-review items.
Run local QA when the environment allows it.
```

## Recommended scope

For personal or small client projects:

```text
Local:
  lint + typecheck + unit + targeted QA

PR /qa:
  API smoke + E2E smoke + affected tests

Release:
  broader E2E + optional Lighthouse/ZAP/security checks
```

Do not run a huge full regression on every PR unless the project really needs it.

## Publishing checklist

Before making this repository public, check:

```text
README.md explains how to install and use the skill.
SKILL.md does not contain private repo names, tokens, or secrets.
.github/workflows/qa-test.yml does not require private secrets.
The repository has a LICENSE file if you want open-source reuse.
Example commands use placeholders such as YOUR_USER_OR_ORG.
```

## License

MIT