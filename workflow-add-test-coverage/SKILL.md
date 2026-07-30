---
name: workflow-add-test-coverage
description: "Coverage-fill workflow for untested code. Use when you have legacy/inherited Magento custom code that has no tests (or low coverage) and you want to systematically add unit + integration tests before refactoring or extending it. Process: identify target scope (a module, a class, or coverage gaps from var/coverage.xml), use mcp__gitnexus-mageos__find_symbol + impact to map all public methods needing coverage, build a mini-plan of test tasks per uncovered method, then invoke /pb-gitnexus:plan-orchestrate to drive tdd-worker through writing characterisation tests (RED-GREEN, no behaviour change), then re-run coverage to verify. Stops at 'tests written + green + coverage improved' — does NOT touch the production code being tested. Composable: can be chained from workflow-build-feature when the build touches a file with coverage < threshold."
disable-model-invocation: true
---

# /proxiblue-skills:workflow-add-test-coverage

Add unit/integration tests to existing Magento code that lacks them. Characterisation testing — captures CURRENT behaviour so future refactors have a safety net, doesn't fix bugs.

## Inputs

The skill needs ONE of these to define scope:

1. **A specific class/method** — `MyVendor\MyModule\Model\Foo::bar`
2. **A module path** — `app/code/MyVendor/MyModule/`
3. **Coverage gap report** — read existing `var/coverage.xml`, target files under <X>% coverage
4. **--from-build-diff** — when chained from workflow-build-feature: target only files touched in the current branch's diff vs `live`

Ask the user which mode if not specified.

## Pre-flight

1. Run `hostname && pwd && ls /var/www/html 2>/dev/null` to confirm DDEV web container.
2. Verify pcov + PHPUnit working: `php -m | grep -qi pcov && vendor/bin/phpunit --version`.
3. Verify gitnexus reachable: `curl -sS -o /dev/null -w '%{http_code}\n' -m 3 http://gitnexus:4747/`. If unreachable, refuse — this skill heavily depends on impact + find_symbol.
4. Verify `.claude/testing.md` documents the project's PHPUnit invocation conventions.

## Visible todo list

```
1. Determine scope (interactive if not provided)
2. Enumerate target classes / methods via gitnexus find_symbol
3. For each method, query existing coverage (var/coverage.xml if present, else assume 0)
4. Filter to UNCOVERED public methods (skip private — exercise via public surface)
5. For each uncovered method, query impact() to understand call surface (helps design test inputs)
6. Build a plan: one task per uncovered method, RED-GREEN characterisation
7. [USER REVIEW] Plan summary — proceed/refine scope/skip-trivial
8. Write plan to .claude/plans/coverage-<scope-slug>/_plan.md (HCF format)
9. /pb-gitnexus:plan-orchestrate — drive tdd-worker through the tasks
10. After completion: re-run PHPUnit with --coverage-clover; compare before/after
11. Print coverage delta summary + remaining gaps
```

## Step details

### Step 1 — scope determination

If user didn't specify:

```
What's the scope for coverage fill?

  (1) Specific class — e.g. "MyVendor\MyModule\Model\Foo"
  (2) Module directory — e.g. "app/code/MyVendor/MyModule"
  (3) Coverage gaps — auto-find files under <X>% (default 50%) from var/coverage.xml
  (4) Build diff — files touched in current branch (chained mode)

Reply with the number + arg.
```

### Step 2 — enumerate symbols

For mode 1/2: `mcp__gitnexus-mageos__find_symbol` on the class/module, get all public methods.
For mode 3: parse `var/coverage.xml`, find files where `<class>` coverage percent < threshold.
For mode 4: `git diff --name-only origin/live | xargs <find_symbol per file>`.

### Step 3 — coverage filter

For each candidate method, check `var/coverage.xml` (Clover format) — find the `<class name="...">` / `<method name="...">` entry. If covered line count > 0, deprioritise (still candidate for "coverage drop" tests but lower priority).

### Step 4 — filter to public

Only public methods. Private methods are characterised through their public callers.

### Step 5 — impact analysis per method

`mcp__gitnexus-mageos__impact` on each uncovered public method. Output informs test design:

- Methods with many indirect callers → write tests that exercise the callers (broader characterisation)
- Methods called only from a single place → write tests against the method directly with realistic input the caller would pass

### Step 6-8 — build + write plan

Format the plan as HCF-compatible:

```markdown
# Plan: coverage-<scope-slug>

## Success Criteria
- Coverage of <scope> rises from <X>% to >=<Y>% (default Y = 80)
- All new tests pass against current (unchanged) behaviour
- No production code changed — characterisation only

## Tasks

### 001 — characterise Foo::bar
Requirements: Write PHPUnit unit test covering all public branches of Foo::bar. Test input shapes informed by impact() callers: <list>. Do NOT modify Foo::bar.

### 002 — characterise Foo::baz
...
```

Write to `.claude/plans/coverage-<scope-slug>/_plan.md` + per-task files.

### Step 9 — orchestrate

```
/pb-gitnexus:plan-orchestrate
```

Pass the plan name. tdd-worker writes the tests; gitnexus-reviewer at each task verifies the test actually exercises the targeted code (coverage report gate catches over-mocked tests).

### Step 10 — coverage re-run

After all tasks pass:

```bash
./vendor/bin/phpunit -c dev/tests/unit/phpunit.xml \
  --coverage-clover=var/coverage.xml \
  --coverage-text=var/coverage.txt
```

Compare to baseline (was `var/coverage.xml.baseline` if saved; otherwise the previous run's `var/coverage.xml`).

### Step 11 — completion summary

```
============================================================
COVERAGE FILL COMPLETE — <scope>

Before:  <X>%   After:  <Y>%   Delta:  +<Y-X>%
Tests added:    <N>
Methods now covered:  <list>
Methods STILL uncovered (skipped — reason):
  - Foo::quux — private, exercised via Foo::bar transitively
  - Foo::deprecated — flagged for removal, not worth testing

No production code modified. Diff: 0 PHP lines changed, <N> test lines added.

Next: /deploy-check live (if you want to merge the test-additions to live)
   or: /proxiblue-skills:workflow-build-feature (proceed to the actual feature work — now with safety net)
============================================================
```

## What this skill does NOT do

- Does not refactor or fix the code being tested. Characterisation only — captures current behaviour, including its bugs. Bug fixes are a separate workflow.
- Does not generate trivial tests (one-line getters, simple constructors). Filters those.
- Does not chase 100% coverage — diminishing returns. Default target 80%; user can override.
- Does not push or commit. Build → test additions → user runs deploy-check separately.

## Chain mode (when invoked by workflow-build-feature)

If invoked with `--from-build-diff`:
- Scope = files in current branch's diff vs `live`
- Only flag if coverage of the touched files is below threshold AFTER the build's own tests landed
- Return `STATUS: PASS` if no gaps found, `STATUS: PASS-WITH-NOTES` if gaps found but non-critical (e.g., new private helpers), `STATUS: BLOCKING` if a public method was added/modified without ANY test coverage

## Composes well with

- `/proxiblue-skills:workflow-build-feature` (optional chain — uncomment in QUALITY_GATES to enable)
- Standalone before a refactor — get the safety net in first
- Quarterly fleet review — pick the lowest-coverage modules and fill them
