---
name: verify-feature
description: >
  Execute Playwright verification of a feature by reading a test-plan YAML,
  running each linked spec one at a time with video + trace + screenshot
  capture, tracking progress via TodoWrite, stopping on first failure for
  human review, and on full pass publishing artifacts via
  proxiblue-skills:publish-test-reports and posting a summary ticket comment.
disable-model-invocation: true
---

# Verify Feature Skill

## Overview

Invoked after `hcf:plan-orchestrate` reports completion, or manually via
`/verify-feature <ticket>`. Reads `.claude/test-plans/<ticket>.yml`, runs
each non-manual story through Playwright with full capture enabled, and
surfaces results live via the Claude todo panel.

## Arguments

### Required

| Argument | Description |
|---|---|
| `<ticket>` | GitHub issue / ticket number (e.g. `361`). Resolves to `.claude/test-plans/<ticket>.yml`. |

### Optional flags

| Flag | Description |
|---|---|
| `--story <slug>` | Verify a single story by slug instead of the whole plan. All pre-flight checks still run against the full YAML; only execution is scoped. |
| `--skip-publish` | Keep all captured artifacts local. Skips the publish-test-reports invocation and posts local paths in the ticket comment instead of published URLs. |

## Invocation guard — do NOT run inside plan-orchestrate

This skill MUST refuse to run if invoked inside another agent's session that
already has todos in flight (e.g. inside plan-orchestrate). The `TodoWrite`
call that seeds story progress would clobber the parent agent's todos,
corrupting its task tracking. Always invoke `/verify-feature` from a fresh
top-level session after plan-orchestrate has fully released its todos.

## Pre-flight checks

Pre-flight runs in order. Any failure aborts before executing any test.

### 1. Test plan existence

`.claude/test-plans/<ticket>.yml` must exist. If missing, abort with:

```
ERROR: .claude/test-plans/<ticket>.yml not found.
Run /manual-test-plan <ticket> first to generate the plan.
```

### 2. Spec discovery

Scan all app test directories for `// @story: <slug>` annotations:

```bash
grep -r --include="*.spec.ts" --exclude-dir=.git \
    "// @story: " \
    tests/m2-hyva-playwright/src/apps/*/tests/
```

Parse each match into `(slug, file)` pairs. This scan MUST NOT be hardcoded
to a single app dir — specs live under `src/apps/<X>/tests/` for various
values of `<X>` (e.g. `checkout`, `account`, `catalog`).

**Exclude `.git/`** directories explicitly (`--exclude-dir=.git`). Multiple
apps under `src/apps/` are git submodules with packed objects under
`.git/objects/` that would produce false matches without the exclusion.

#### Duplicate-slug hard error

If the same `@story: <slug>` appears in more than one spec file, abort with:

```
ERROR: Duplicate @story slug "<slug>" found in multiple spec files:
  - tests/m2-hyva-playwright/src/apps/checkout/tests/foo.spec.ts
  - tests/m2-hyva-playwright/src/apps/account/tests/bar.spec.ts
Each @story slug must be unique across all apps. Fix the collision before running verify.
```

#### Missing-slug hard error

For each non-manual story in the YAML, check that its `story_slug` appears in
exactly one discovered spec file. If any slug matches zero spec files, abort
with:

```
ERROR: The following YAML story slugs have no matching @story annotation in any spec file:
  - <slug-a>
  - <slug-b>
Fix the YAML or add the @story comment to the spec before running verify.
```

List all missing slugs before aborting — do not stop at the first one.

### 3. test_name guard

For each story that is NOT `manual_only: true`:

- If the YAML entry omits `test_name` AND the discovered spec file contains
  more than one `test()` / `it()` block, abort with:

```
ERROR: Story "<slug>" maps to <file> which contains multiple tests, but
test_name is not set in the YAML. Set test_name to the exact test title so
-g does not accidentally match unintended tests.
```

### 4. Publish wrapper check (conditional)

Publishing is in effect when **both** conditions hold:
- `--skip-publish` was NOT passed, AND
- the current git branch IS `live`

When publishing is in effect, `tests/test-reports/publish-report.sh` must
exist and be executable. If it does not:

```
ERROR: tests/test-reports/publish-report.sh not found.
Publishing requires the proxiblue-skills:publish-test-reports setup.
Run /publish-test-reports to complete setup, then re-run verify.
Alternatively, pass --skip-publish to skip publishing for this run.
```

**When `--skip-publish` is passed OR the current branch is not `live`**, this
wrapper check is SKIPPED entirely so dev iteration works without the publish
infrastructure in place.

### 5. Non-live branch auto-fallback

If the current git branch is NOT `live`, the skill automatically behaves as if
`--skip-publish` were passed. Emit one notice line before execution begins:

```
NOTICE: Not on live branch — publishing skipped. Artifacts will be local only.
```

No further action required from the user.

## Todo tracking

This skill uses the `TodoWrite` tool (Claude Code's built-in todo mechanism).
`TaskCreate` and `TaskUpdate` do NOT exist in Claude Code — never reference them.

### Seeding

Before the first Playwright invocation, issue a single `TodoWrite` call with
one entry per non-manual story. Each entry:

```json
{
  "id": "<story-slug>",
  "content": "<story title from YAML>",
  "status": "pending",
  "priority": "medium"
}
```

If `--story <slug>` was passed, seed only that one entry.

### Live updates

- Before running each story: re-issue `TodoWrite` flipping that story's
  `status` to `"in_progress"`.
- After a story passes: re-issue `TodoWrite` flipping to `"completed"`.
- On failure: leave the failing story at `"in_progress"` and pause for human
  input (see Fail-fast behavior below). Do not flip remaining stories.

## Per-story Playwright invocation

Run from inside `tests/m2-hyva-playwright/src/apps/<APP_NAME>/` (e.g.
`src/apps/pps/`). Execute ONE story at a time — never batch.

**Why the cwd matters:** the pps `playwright.config.ts` references reporters
with relative paths (e.g. `"./markdown-reporter.ts"`). Playwright resolves
these relative to the cwd at config-load time, not to the config file's
directory. Running from anywhere else fails with
`Cannot find module './markdown-reporter.ts'`. The yarn workspace scripts
(`yarn workspace pps test:checkout`) handle this by cd'ing automatically; if
invoking `npx playwright` directly, you must cd in first.

### Deriving APP_NAME and TEST_BASE

- `TEST_BASE` derives from the spec_file path:
  `tests/m2-hyva-playwright/src/apps/<X>/tests/...` → `TEST_BASE=<X>`
- `APP_NAME` defaults to `pps` (this project's Playwright config).
- Either value can be overridden per-story in the YAML via optional `app_name`
  and `test_base` fields.

### Output directory — important: Playwright IGNORES `--output=` here

The per-app `playwright.config.ts` files hardcode `outputDir` (e.g. pps uses
`outputDir: path.join(process.cwd(), '../../../test-results', 'pps', 'pps-${TEST_BASE}')`).
Playwright's `--output=<dir>` CLI flag does NOT override this config value in
the m2-hyva-playwright harness (verified empirically — passing `--output=` to
a verify run produced empty dirs while real artifacts landed at the config
path).

**Real artifact location** for an `APP_NAME=pps TEST_BASE=<X>` run:

```
tests/m2-hyva-playwright/test-results/pps/pps-<X>/<spec-file>-<test-title-slug>-<project>/
```

For `APP_NAME=pps TEST_BASE=checkout` running the tax-exempt spec, paths
become:

```
tests/m2-hyva-playwright/test-results/pps/pps-checkout/tax-exempt-checkout-<test-title-slug>-chromium/
```

verify-feature MUST NOT pass `--output=` (it does nothing). To get a per-run
isolated location, the implementation either:
- Copies the per-test artifact dirs from the config outputDir into
  `tests/m2-hyva-playwright/test-results/verify/<ticket>/<ISO-timestamp>/<story-slug>/`
  after each Playwright invocation completes, OR
- Resolves paths directly under the config outputDir and includes them in
  the final ticket comment.

The ISO-timestamp component (when used for snapshotting) is generated once at
verify startup and reused across all stories in the run.

### Command template

```bash
cd tests/m2-hyva-playwright/src/apps/<APP_NAME> && \
NODE_TLS_REJECT_UNAUTHORIZED=0 \
APP_NAME=<app_name> TEST_BASE=<test_base> \
npx playwright test <spec_file_basename> \
  -g "<test_name>" \
  --project=chromium \
  --retries=0 \
  --trace=on
```

The `-g` value MUST be wrapped in double quotes. It is a regex substring
match, so use the full exact test title from `test_name` to prevent
unintended tests from matching.

`--trace=on` overrides the per-app config default (`retain-on-failure`) so
the trace is preserved on passing runs.

**Do NOT pass `--video=on` or `--screenshot=on` as CLI flags** — Playwright's
CLI in this version does not accept them. Video and screenshot behavior is
controlled by the per-app config (`video: "on"` and
`screenshot: "only-on-failure"` for pps). If video is missing, the per-app
config needs `video: "on"` set there.

### What artifacts get preserved per story (with `preserveOutput: 'always'`)

Inside `test-results/pps/pps-<TEST_BASE>/<spec-test-slug>-<project>/`:

- `trace.zip` — full Playwright trace with screenshots, network, console,
  AND every `testInfo.attach()` named attachment embedded. View via
  `npx playwright show-trace <path>`.
- One or more `.webm` files — full session video (one per browser context
  Playwright opens).

**Named `testInfo.attach()` screenshots are NOT standalone `.png` files** —
they live inside `trace.zip` as attachments visible in the trace viewer.
verify-feature should reference `trace.zip` as the entry point for named
captures, not enumerate individual `.png` paths.

## Fail-fast behavior

This skill MUST NOT loop or auto-retry. First failure stops execution.

On first non-zero Playwright exit code:

1. Resolve artifact paths under the config outputDir (NOT `--output=`, which
   Playwright ignores in this harness):
   ```
   tests/m2-hyva-playwright/test-results/pps/pps-<TEST_BASE>/<spec-test-slug>-<project>/
   ```
   Inside that directory:
   - Trace: `trace.zip` (contains all `testInfo.attach()` named captures)
   - Video(s): `*.webm` (one per browser context)
   - Failure screenshot (only if test failed): `*.png` written by Playwright's
     `screenshot: 'only-on-failure'` config setting

2. Strip ANSI color codes from stdout/stderr, display the last 50 lines.

3. Ask the user to choose one action:

```
FAILURE in story "<slug>" — <spec file>
  Trace:           <path-to-trace.zip>  (view: npx playwright show-trace <path>)
  Video(s):        <path-to-*.webm>
  Failure screenshot: <path-to-*.png>  (only present if Playwright captured one)

--- last 50 lines of output ---
<stripped output>
---

Choose an action:
  [R] Retry this story alone (NOT the whole suite)
  [K] Mark as known issue (enter reason, continue to next story)
  [A] Abort verification run
```

### Retry

Re-run the FAILED STORY ALONE — not the full suite. Emit the same artifact
paths and output tail on the second failure. If the retry also fails, stop and
present the same three choices again.

### Mark known-issue

Update `.claude/test-plans/<ticket>.yml` to add `known_issue: <reason>` to
that story entry. Then continue execution with the next story. The story
remains in the `"in_progress"` todo state until the run concludes.

### Abort

Stop immediately. Do not publish. Report local artifact paths for the failed
story.

## Success path and publishing

After all non-manual stories complete without failure (or after known-issues
are recorded), evaluate the publish gate:

**Publish when ALL of the following hold:**
- Current branch IS `live`
- `--skip-publish` was NOT passed
- `tests/test-reports/publish-report.sh` exists

### Report directory renaming

`publish-test-reports` expects directories matching `hyva-*-reports/` inside
the report path it is given. The verify output is at
`test-results/verify/<ticket>/<timestamp>/` and contains
`<story-slug>/` subdirectories — not `hyva-*-reports/`.

Before invoking publish, create a staging directory:

```
test-results/verify/<ticket>/<timestamp>/hyva-verify-<ticket>-reports/
```

Copy or symlink each story's output into that staging directory so the
`hyva-verify-<ticket>-reports/` directory contains the Playwright HTML
report artifacts. If this rename/staging step cannot be performed safely
(e.g., naming collisions, insufficient permissions), skip publishing and
include a clear notice in the final ticket comment:

```
NOTE: Artifact rename for hyva-*-reports/ layout could not be completed.
Publishing skipped. Artifacts available locally at: <path>
```

### Invoking publish

```
invoke proxiblue-skills:publish-test-reports
```

Pass:
- `site-name`: derived from the project CLAUDE.md or git remote org slug
- `report-dir`: `tests/m2-hyva-playwright/test-results/verify/<ticket>/<timestamp>/`
- `project-dir`: project git root

Capture the published URL returned by the skill.

## Final ticket comment

**Important constraint:** GitHub issue comments posted via `gh issue comment`
cannot upload video or binary attachments. Image uploads via the web UI work
(drag/drop), but not via the CLI. The skill therefore posts a **skeleton
comment with placeholders** that the operator manually fills in by
drag-dropping screenshots into the web UI after viewing them via
`npx playwright show-trace <trace.zip>`.

The skill does NOT auto-upload artifacts. It does NOT generate hosted URLs
unless the publish step ran on the live branch. Local-only runs produce a
template the operator completes manually.

Post one comment to the GitHub issue `<ticket>` with this format:

```markdown
## Verification run — <ISO timestamp>

Branch: `<branch-name>` · Publish: <published|skipped: <reason>>

### Passed stories

#### <story title>
- Trace: `<local-path-to-trace.zip>` · view via `npx playwright show-trace <path>`
- Video: `<local-path-to-video.webm>`
- Named captures (embedded in trace.zip): <comma-separated screenshot_points names>

<!-- PASTE SCREENSHOT(S) HERE — drag-drop from trace viewer into the web UI -->

---

#### <next story title>
- Trace: `<local-path-to-trace.zip>` · view via `npx playwright show-trace <path>`
- Video: `<local-path-to-video.webm>`
- Named captures (embedded in trace.zip): <comma-separated screenshot_points names>

<!-- PASTE SCREENSHOT(S) HERE -->

---

### Manual verification required

- [ ] **<story title>** — verify manually per acceptance criteria in the test plan

### Known issues

- **<story title>** — <known_issue reason>

---

Full report: <published-reports-site-url>
<!-- OR if --skip-publish or non-live branch: -->
Local artifact root: tests/m2-hyva-playwright/test-results/pps/pps-<TEST_BASE>/
```

Rules:
- All `<local-path-...>` placeholders are RELATIVE paths from project root.
  When the publish step ran, replace each local path with the published URL
  for that artifact and drop the `npx playwright show-trace` hint.
- Each story's section ends with a `<!-- PASTE SCREENSHOT(S) HERE -->`
  placeholder comment. The operator opens the trace viewer
  (`npx playwright show-trace <path>`), takes screenshots of the relevant
  named captures, then drag-drops them into the GitHub web UI between the
  placeholder line and the `---` separator below.
- "Manual verification required" section omitted if there are no `manual_only`
  stories.
- "Known issues" section omitted if there are no `known_issue` entries.
- Footer points to the published reports site when published, or to the
  per-app artifact root when local-only.
- Use `gh issue comment <ticket> --body-file <path-to-skeleton.md>` to post.
  Then the operator opens the issue in the browser to drop in images.
