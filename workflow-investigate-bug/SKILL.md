---
name: workflow-investigate-bug
description: "Forensic-only bug investigation workflow. Use when a bug is reported but you don't yet know the root cause. Strictly READ-ONLY: reproduces the bug with Playwright, captures video + screenshots + console + network logs, runs git log/blame to identify candidate regression commits, uses mcp__gitnexus-mageos__impact to enumerate everything that touches the suspect code path, scans var/log/ for related errors, optionally chains workflow-security-audit if security implications suspected. Output: a comprehensive .claude/investigations/<ticket>-<date>.md doc with reproduction steps, log evidence, candidate root causes ranked by likelihood, and a recommended fix-workflow handoff. Does NOT write a fix. After investigation completes, user decides whether to invoke workflow-build-feature (treats fix as a feature plan), do a hotfix, or escalate."
disable-model-invocation: true
---

# /proxiblue-skills:workflow-investigate-bug

Forensic investigation. **Read-only.** Produces an evidence-backed root-cause analysis document; doesn't fix.

Per the mandatory investigation protocol in `~/claude-skills-central/rules/investigation.md`: enumerate own blast radius first, read ALL failure artefacts (not a sample), compare to prior passing state, only THEN form a hypothesis. This workflow makes that protocol concrete.

## Inputs

Ask user for:
- **Ticket number** (optional but recommended for output naming + GH cross-link)
- **Bug description** — what's broken, observed behaviour
- **Expected behaviour** — what should happen instead
- **Reproduction URL / steps** — at least one path that reliably reproduces

## Pre-flight

1. Confirm DDEV web container context (`hostname && pwd`).
2. Verify gitnexus reachable: `curl -sS -o /dev/null -w '%{http_code}\n' -m 3 http://gitnexus:4747/`. If down, fall back to grep but warn output is shallower.
3. Verify Playwright present: `which yarn && test -d tests/m2-hyva-playwright`.
4. Verify clean working tree (`git status --short`). If dirty, refuse — investigation needs a known-good base to diff against.

## Visible todo list

```
1. Capture environmental facts (branch, commit, recent migrations, container uptime)
2. Reproduce the bug with Playwright + record video, screenshots, console, network
3. Scan var/log/ for errors in the reproduction window
4. Identify affected files from the reproduction (network requests, log file paths, error stack traces)
5. For each affected symbol, mcp__gitnexus-mageos__impact → enumerate callers / dependents
6. git log + git blame on the affected files — identify candidate regression commits
7. For each candidate regression commit, summarise the diff + correlation to the bug
8. [OPTIONAL] If security implications suspected: invoke /proxiblue-skills:workflow-security-audit on the affected scope
9. Rank candidate root causes by evidence strength
10. Write the investigation doc to .claude/investigations/<ticket>-<YYYYMMDD>.md
11. Recommend handoff workflow (build-feature / hotfix / escalate)
```

## Step details

### Step 1 — environment snapshot

```bash
echo "Branch:           $(git rev-parse --abbrev-ref HEAD)"
echo "Commit:           $(git rev-parse HEAD) ($(git log -1 --format='%cd' --date=short))"
echo "Composer status:  $(composer diagnose 2>&1 | head -5)"
echo "DDEV uptime:      $(uptime)"
echo "Recent migrations: $(bin/magento setup:db:status 2>&1 | tail -5)"
echo "Cache state:      $(bin/magento cache:status 2>&1)"
```

Save to investigation log. This is the "WHAT I CHANGED THIS SESSION" / context from the investigation protocol.

### Step 2 — Playwright reproduction

**First, review the client's evidence.** If the ticket has a screen recording
(`.mov`/`.mp4`/`.webm`), you cannot skip it — it shows the exact click-path and
failure state. Download it and extract frames with `ffmpeg` (installed in the web
container), then Read the frames — see `github-analysis` Step 1b for the exact
commands (`gh issue view … | grep` the asset URL → `curl` → `ffmpeg -vf fps=1`).
Let the observed path drive the reproduction spec below.

Write a spec at `tests/m2-hyva-playwright/src/apps/<APP_NAME>/tests/_investigations/<ticket>-repro.spec.ts` that:

- Navigates to the reproduction URL
- Captures full-page video (`use video: 'on'`)
- Screenshots at each step (`page.screenshot()`)
- Captures console messages (`page.on('console', msg => ...)`)
- Captures network requests (`page.on('request', ...)` + `page.on('response', ...)`)
- Captures uncaught exceptions (`page.on('pageerror', err => ...)`)
- Asserts the BROKEN state (test should currently fail / show the bug)

Run it: `cd tests/m2-hyva-playwright/src/apps/<APP_NAME>/ && yarn playwright test _investigations/<ticket>-repro.spec.ts`. Capture all artifacts (video, screenshots, trace).

If reproduction fails to fire the bug → flag the bug as "non-reproducible in current env" + document conditions tried. STOP — can't investigate what we can't reproduce.

### Step 3 — log scan

Scan logs for the reproduction window:

```bash
WINDOW_START="$(date -d '5 minutes ago' '+%Y-%m-%d %H:%M:%S')"
for log in var/log/system.log var/log/exception.log var/log/debug.log var/log/payment.log; do
  echo "=== $log ==="
  [ -f "$log" ] && awk -v start="$WINDOW_START" '$0 >= start' "$log" 2>/dev/null | tail -30
done
```

Plus `docker exec ddev-<project>-web grep -i "<keyword from bug>" /var/log/php-fpm.log` if needed.

Extract: error messages, stack traces, file:line references, any class names mentioned.

### Step 4 — identify affected files

From the network requests + stack traces, derive a list of PHP files involved. For URL-based: trace through `app/code/*/etc/routes.xml` → controller class. Trace controller → models → blocks → templates.

For each: also check if there are recent commits touching it.

### Step 5 — gitnexus impact propagation

For each affected symbol, run `mcp__gitnexus-mageos__impact <symbol>`. The returned callers/dependents are part of the "blast radius" — bugs are often in callers passing wrong input, not in the called code.

Annotate the investigation: which callers exist; which were recently modified.

### Step 6 — git log + blame on affected files

```bash
git log --oneline -10 -- <affected-file>
git blame <affected-file> | grep -v "^\^"   # filter root commits
```

For files with recent activity → fetch the commits' diffs:

```bash
git show <sha> -- <file>
```

### Step 7 — diff correlation

For each candidate commit identified by blame/log:
- Summarise WHAT changed
- Cross-reference to the bug symptom (does the diff plausibly cause the observed behaviour?)
- Rank: strong / moderate / weak correlation

### Step 8 — security audit (optional chain)

If the bug involves auth bypass, data leakage, unauthorised actions, or unusual log entries:

```
/proxiblue-skills:workflow-security-audit --diff-against=live
```

Captures whether the bug has security implications beyond the reported behaviour.

### Step 9 — rank candidate root causes

For each candidate root cause, score evidence:
- Reproduction confirms behaviour matches hypothesis (high)
- Log entries cite the suspect file/method (high)
- Recent commit on the suspect file changed something related (high)
- Impact analysis shows untouched indirect callers (medium — possible)
- Suspect file unchanged in 6+ months (low — bug is probably environmental/data-side)

### Step 10 — investigation doc

Write to `.claude/investigations/<ticket>-<YYYYMMDD>.md`. Follow the mandated failure-report format from `~/claude-skills-central/rules/investigation.md`:

```markdown
# Investigation: <ticket> — <one-line bug summary>

Date: <YYYY-MM-DD>
Investigator: Claude (workflow-investigate-bug)

## WHAT I CHANGED THIS SESSION (from git diff)
- No production code changes; investigation is read-only.
- Added Playwright reproduction spec: tests/m2-hyva-playwright/src/apps/<APP_NAME>/tests/_investigations/<ticket>-repro.spec.ts

## WHAT THE ARTEFACTS SHOW
- Reproduction video: <path to .webm>
- Reproduction screenshots: <paths>
- Console errors: <verbatim lines>
- Network failures: <verbatim>
- var/log/system.log:NN: <verbatim line>
- var/log/exception.log:NN: <verbatim line>

## COMPARE TO PRIOR RUN
- Bug NOT reproducible on commit <sha> (last passing state per ticket history)
- Reproducible on commit <sha> (current HEAD)
- Diff between those commits: <list of touched files, link to git compare URL>

## HYPOTHESIS (with evidence)
1. **Strongest:** <claim> — supported by <artefact line/key>
2. Alternative: <claim> — supported by <artefact line/key>

## WHAT I HAVE NOT VERIFIED
- <gap>
- (or "nothing, hypothesis is evidence-complete")

## Impact (from gitnexus)
<table or list of affected symbols + their callers>

## Recommended next step
- If the hypothesis is "regression in commit <sha>": invoke /proxiblue-skills:workflow-build-feature for ticket <ticket>; the fix should revert or correct the regression, with tests covering the bug behaviour.
- If "data-side / environmental": stop and escalate (DBA / ops); no code workflow needed.
- If "security implications": investigation findings have been augmented by /proxiblue-skills:workflow-security-audit — see appended section.
```

### Step 11 — handoff recommendation

Print:

```
============================================================
INVESTIGATION COMPLETE — <ticket>

Doc:            .claude/investigations/<ticket>-<YYYYMMDD>.md
Repro spec:     <path>
Repro artifacts: <video / screenshots paths>

Top hypothesis: <one-line summary>
Evidence:       <one-line summary>

Recommended next:
  /proxiblue-skills:workflow-build-feature <ticket>
    (fix as a feature; reproduction spec becomes the regression test in the test plan)

OR for a known-safe one-line fix without full workflow overhead:
  Apply the fix directly, then /deploy-check live

OR if data/environmental:
  Stop — escalate to DBA / ops, no code workflow appropriate
============================================================
```

## What this skill does NOT do

- **Does not modify ANY production code.** Only adds the reproduction spec (which is test code, not prod).
- Does not commit or push.
- Does not guess fixes without evidence — adheres to the investigation protocol's "no claim without citation" rule.
- Does not skip artefact reading. If reproduction fails or logs unavailable, stops and reports limitations.

## Composes well with

- `/proxiblue-skills:workflow-build-feature` — natural next step when the investigation identifies a fix
- `/proxiblue-skills:workflow-security-audit` — auto-chained from step 8 when bug has security implications
- The mandatory investigation protocol at `~/claude-skills-central/rules/investigation.md` — this skill operationalises that protocol
