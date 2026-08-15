---
name: publish-test-reports
description: >
  Set up and publish Playwright test report artifacts to GitHub Pages via a
  per-client test-reports repo (forked from ProxiBlue/test-reports template).
  Use when the user asks to publish test results, set up test reporting, wire
  up report publishing, or check published reports. Also use after running a
  full test suite on the live branch.
disable-model-invocation: true  
---

# Publish Test Reports Skill

## Overview

Playwright HTML test reports are published to a per-client GitHub Pages site
so clients can view test results visually. Only **fully successful runs on
the live branch** are published.

- **Template repo:** `ProxiBlue/test-reports` (GitHub template)
- **Per-client repos:** e.g. `ITToolsAU/test-reports`, `ClientOrg/test-reports`
- **Publish script:** `publish-report.sh` lives inside each client's test-reports repo

## Setup in a new environment

### Step 1: Determine the client's GitHub org

```bash
# Read from CLAUDE.md or git remote
ORG=$(git remote get-url origin | sed 's|.*github.com[:/]\([^/]*\)/.*|\1|')
echo "Org: $ORG"
```

### Step 2: Check if client test-reports repo exists

```bash
gh repo view "$ORG/test-reports" --json name 2>/dev/null && echo "EXISTS" || echo "NEEDS CREATION"
```

### Step 3: Create from template if needed

```bash
# Create private repo from ProxiBlue template
gh repo create "$ORG/test-reports" --private \
    --description "Playwright test report artifacts — viewable via GitHub Pages"

# Clone template content
TMPDIR=$(mktemp -d)
git clone git@github.com:ProxiBlue/test-reports.git "$TMPDIR"
cd "$TMPDIR"
git remote set-url origin "git@github.com:$ORG/test-reports.git"
git push -u origin main

# Enable GitHub Pages
gh api "repos/$ORG/test-reports/pages" -X POST \
    -f "source[branch]=main" -f "source[path]=/"
```

### Step 4: Clone into the project

Standard location is `tests/test-reports/`:

```bash
REPORTS_DIR="tests/test-reports"
if [ ! -d "$REPORTS_DIR" ]; then
    git clone "git@github.com:$ORG/test-reports.git" "$REPORTS_DIR"
fi
```

Add to `.gitignore` if not already:
```
tests/test-reports/
```

### Step 5: Verify setup

```bash
ls -la tests/test-reports/publish-report.sh
# Should exist and be executable
```

## Publishing reports

### When to publish

After a **full test suite run on the live branch** where all tests pass.

### Command

```bash
tests/test-reports/publish-report.sh \
    <site-name> \
    <path-to-test-results> \
    <project-root>
```

**Arguments:**
1. `site-name` — identifier for this project (e.g. `lcd`, `store-name`)
2. `report-dir` — path containing one or more `*-reports/` directories (see "Expected report directory structure" below — any suffix works, not just `hyva-`)
3. `project-dir` — project git root (for branch check + commit ref)

**Example (LCD site):**
```bash
tests/test-reports/publish-report.sh \
    lcd \
    tests/m2-hyva-playwright/test-results/lcd \
    /var/www/html
```

### What the script does

1. **Branch gate:** Checks project is on `live` — skips otherwise
2. **Pass gate:** Scans all JSON reports — aborts if ANY test failed
3. **Copies** HTML reports into `site/date/suite/` structure
4. **Updates** `reports.json` index
5. **Commits + pushes** → GitHub Pages auto-deploys
6. **Prunes** to last 20 runs per site

### Viewing reports

```
https://<org>.github.io/test-reports/              # Index of all runs
https://<org>.github.io/test-reports/<site>/<date>/ # Specific run
```

**Example:** `https://ittoolsau.github.io/test-reports/lcd/2026-04-27_0830/`

### Referencing in tickets/PRs

```markdown
Test results: https://ittoolsau.github.io/test-reports/lcd/2026-04-27_0830/
```

## Known extension pattern — auto-deploy-on-pass (LCD only, NOT template default)

LCD's fork (`ITToolsAU/test-reports`) diverges from the `ProxiBlue/test-reports`
template: after a successful publish, its `publish-report.sh` ALSO does an
`git commit --allow-empty` + `git push origin live` in the project repo, then
posts a "Scheduled to Deploy Live" comment and swaps labels on the GitHub
issue extracted from recent commit messages. In effect, a green full-suite
report on `live` becomes an automated production-deploy trigger, not just a
report publish.

**This is intentionally NOT part of the template or this skill's default
setup steps.** A green test run does not imply authorization to push to
production on every project — e.g. pvcpipesupplies's own `CLAUDE.md` states
"Never push or create a PR without the user's explicit permission", which
this pattern would violate if blindly copied there. Treat it as a reference
implementation for a specific automation LCD's owner opted into, not
something `/pb-skills:publish-test-reports` sets up by default.

If a project genuinely wants this: fork the auto-deploy block from LCD's
`publish-report.sh` (the part after "Commit and push" in the report-repo
section) explicitly, confirm it matches that project's own deploy-approval
rules, and document the opt-in in that project's own CLAUDE.md — don't carry
it over silently as part of routine template setup.

## Fleet status

As of 2026-08-15, LCD (`ITToolsAU/test-reports`) is the only project actually
using this — last published run was 2026-05-15 (stale). No other project
(including pvcpipesupplies) has forked the template or wired this up yet.
The mechanism and template are fleet-ready; rollout to additional projects is
a separate, per-project decision (fork the template, clone into
`tests/test-reports/`, wire a publish step — see Setup above).

## Expected report directory structure

The publish script discovers ANY subdirectory of `<report-dir>` ending in
`-reports` — the prefix is whatever your Playwright configs name their
output folders (`hyva-`, `admin-`, `checkout-`, a site-specific app name like
`pps-`, etc.), not hardcoded to a specific project's naming:

```
<report-dir>/
├── hyva-admin-reports/
│   ├── playwright-report/index.html   # HTML report
│   └── json-reports/json-report.json  # Stats source
├── admin-default-reports/
│   └── ...
├── pps-admin-reports/                 # multi-bucket projects: one -reports dir per bucket
│   └── ...
└── checkout-reports/
    └── ...
```

The published suite label is the directory name with the trailing
`-reports` stripped (e.g. `hyva-admin-reports` → `hyva-admin`) — it is NOT
assumed to start with `hyva-`. Point `report-dir` at whatever directory
actually contains your project's `*-reports/` folders (for m2-hyva-playwright
projects this is usually `tests/m2-hyva-playwright/test-results/<APP_NAME>`).

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Not on live branch" | `git checkout live` before publishing |
| "Test failures detected" | Fix tests, rerun, then publish |
| No JSON reports | Ensure Playwright config includes JSON reporter |
| Push rejected | `cd tests/test-reports && git pull --rebase origin main` |
| Pages 404 | Check Settings > Pages in the test-reports repo |
| Repo doesn't exist | Run setup steps above to create from template |
