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
2. `report-dir` — path containing `hyva-*-reports/` directories
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

## Expected report directory structure

The publish script expects Playwright's output layout:

```
<report-dir>/
├── hyva-admin-reports/
│   ├── playwright-report/index.html   # HTML report
│   └── json-reports/json-report.json  # Stats source
├── hyva-hyva-reports/
│   └── ...
└── hyva-default-reports/
    └── ...
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Not on live branch" | `git checkout live` before publishing |
| "Test failures detected" | Fix tests, rerun, then publish |
| No JSON reports | Ensure Playwright config includes JSON reporter |
| Push rejected | `cd tests/test-reports && git pull --rebase origin main` |
| Pages 404 | Check Settings > Pages in the test-reports repo |
| Repo doesn't exist | Run setup steps above to create from template |
