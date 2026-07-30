---
name: manual-test-plan
description: >
  Derive a user-story manual test plan from an implementation plan's Success
  Criteria and task Requirements, post it as a phased comment on the GitHub
  ticket, and write a machine-readable companion YAML at
  .claude/test-plans/<ticket>.yml that links each story to a Playwright spec
  via a @story: slug. Invoke after hcf:plan-create produces an implementation
  plan, or when the user asks to "generate a manual test plan" for a ticket.
disable-model-invocation: true  
---

# Manual Test Plan Skill

## Overview

Mines `_plan.md` Success Criteria and per-task `Requirements (Test Descriptions)`
sections from one or more plan directories, derives user stories, posts a phased
GitHub issue comment, and writes a companion YAML keyed to Playwright specs.

## Coverage tiers — only two

Every story carries one of two marks. There is no third tier.

- **AI** — covered by an automated test that **drives the real UI and passes**. The
  green AI mark *is* the confirmation: it means the actual user journey was exercised
  end to end, so a human does not need to re-verify that story.
- **HI** — a human verifies it manually on UAT.

### AI coverage MUST drive the real UI — never bypass the page

A test earns the **AI** mark only when it reproduces what a customer actually does:
navigate to the real page, set the options/inputs on that page, click the real
controls, submit the real form. A test that shortcuts the UI — POSTing straight to a
controller, fabricating a `form_key`, or hitting an endpoint directly — does **NOT**
count as AI coverage, even when it passes.

Why this is a hard rule (ticket #333): the CartSwap frontend tests POSTed
`product` + `related_product` to `/checkout/cart/add/` and never loaded the product
page. When a display rework stopped the on-page accessory chooser from rendering, all
14 tests stayed green over a broken page, the plan showed **AI ✅**, and the breakage
reached UAT. A green mark over a bypassed page is worse than no mark — it tells a human
*not* to look at something that is in fact broken.

If a behaviour genuinely cannot be driven through the real UI in an automated test
(real payment capture, an external account, a physical action), it is **HI** — not a
UI-bypassing "AI". Never invent a bypass to manufacture an AI tick.

## Arguments

```
hcf:manual-test-plan <ticket> <plan-path>[,<plan-path2>,...]
```

| Arg | Required | Description |
|-----|----------|-------------|
| `ticket` | yes | GitHub issue number (integer) |
| `plan-path` | yes | Path to a plan directory under `.claude/plans/` — OR a comma-separated list of plan dirs when a ticket maps to multiple plans |

**Flags:**

| Flag | Description |
|------|-------------|
| `--skip-comment` | Write the YAML only; do not post or attempt to post a GitHub issue comment. Use when the ticket already has a manual test plan comment posted. |
| `--dry-run` | Print the comment body and YAML to stdout without calling `gh issue comment` or writing the YAML file. Nothing is persisted. |

## Derivation Algorithm

### Step 1 — Load plan sources

For each plan-path in argument order:

1. Read `<plan-path>/_plan.md`. Extract every row from the "Success Criteria"
   table. Each row becomes one **user story candidate**.
2. For each task file `<plan-path>/NNN-<title>.md` (sorted numerically):
   - Extract the task title from the `# Task NNN: <Title>` heading.
   - Extract every line from the "Requirements (Test Descriptions)" section.
     Each bullet becomes one **requirement candidate** belonging to the phase
     named after the task title.

### Step 2 — Merge and de-duplicate (multiple plan-paths only)

When two or more plan-paths are passed, their Success Criteria and Requirements
are merged in argument order. De-duplication uses case-insensitive exact-text
match on the criterion/requirement string. First occurrence wins; duplicates
from later plans are silently discarded.

### Step 3 — Map criteria to stories

Each Success Criterion from `_plan.md` becomes one user story in the output.
The story's phase is determined by which task file the AC maps to (via the task
number referenced in the Success Criteria table). When an AC does not reference
a specific task, it is placed in an introductory "Pre-deployment" phase.

Each task file's Requirements list produces one or more sub-stories within the
phase named after that task's title. A single requirement may expand to multiple
stories when the requirement describes more than one user-observable outcome
(signalled by "AND" or a multi-clause sentence).

### Step 4 — Assign slugs

Each story gets a slug:

- Format: `kebab-case`, `lowercase`, starts with a verb or noun describing the
  **user-observable behavior** (e.g. `guest-checkout-tax-exempt`,
  `migrate-avatax-certs-dry-run`).
- Must be **globally unique** within the YAML being written.
- Must **not collide** with `@story:` slugs already present in any Playwright
  spec file.

**Pre-flight collision check** — use the project's Playwright spec location:
`tests/m2-hyva-playwright/src/apps/*/tests/*.spec.ts` on M2-Hyvä projects,
`dev/tests/playwright/tests/*.spec.ts` on OpenMage/M1 projects (e.g. ntotank):

```bash
grep -rh "@story:" \
  tests/m2-hyva-playwright/src/apps/*/tests/*.spec.ts \
  dev/tests/playwright/tests/*.spec.ts 2>/dev/null \
  | sed 's/.*@story: *//' | sort -u
```

If any proposed slug matches an existing `@story:` tag, emit a warning line per
collision before finalizing:

```
WARN: slug "guest-checkout-tax-exempt" collides with an existing @story: tag in
      tests/m2-hyva-playwright/src/apps/pps/tests/checkout.spec.ts — rename it.
```

Rename colliding slugs by appending `-<ticket>` suffix, then warn the operator
to update the spec file to match.

### Step 5 — Classify manual_only stories

A story is `manual_only: true` only when the behaviour **cannot be driven through
the real UI by an automated test** — e.g. it needs real payment capture, an external
account, or a physical action. Then it requires a human tester every release.

`manual_only` is NOT a home for behaviour that is merely awkward to automate. If a
story has no `@story` tag yet but *could* be exercised through the real storefront,
the correct action is to **write that UI-driven spec**, not to mark it manual. And
never award the **AI** mark on the back of a UI-bypassing test (a direct controller
POST, a fabricated `form_key`): that paints a green tick over a possibly-broken page
(see #333) and tells the human to skip the one thing that needs looking at.

After classification, if **any** story is `manual_only: true`, print a warning
block before posting the comment or writing the YAML:

```
WARNING: The following stories have no Playwright spec coverage and are marked
manual_only: true — a human must verify these on every deploy:

  - migrate-avatax-certs-idempotency
  - live-deploy-cron-no-demotions

Consider writing Playwright specs for these or accept the manual testing burden.
Proceed? [y/N]
```

Pause for user confirmation. If the user answers `N` or the response times out,
abort without posting or writing.

## Pre-Existing Plan Detection

Before posting a comment, fetch the issue's existing comments:

```bash
gh issue view <ticket> --repo <org>/<repo> --comments
```

Pipe the output through a case-insensitive grep for the standalone phrase
`manual test plan`:

```bash
| grep -qi "manual test plan"
```

If the phrase is found in any comment block, **abort** with:

```
ABORT: A manual test plan comment already exists on issue #<ticket>.
Use --skip-comment to write the YAML only, or review the existing plan first.
```

Do **not** rely on any specific heading string. The detection phrase
`manual test plan` covers all variations.

## GitHub Comment Output Format

The posted comment is a phased Markdown block matching the format the
operator chose for this client. Default is a numbered, phased checklist
with one expected result per step:

```markdown
## Manual Test Plan — <short title derived from plan name(s)>

### <Phase 1 name>

**<Sub-section heading>**
- [ ] <step> — <one-line expected result>
- [ ] <step> — <one-line expected result>

**<Sub-section heading>**
- [ ] <step> — <one-line expected result>

---

### <Phase 2 name>

**<Sub-section heading>**
- [ ] <step> — <one-line expected result>

---

### <Phase N name>
...
```

Rules:
- Phase names come from task titles in the plan files.
- Sub-section headings come from the grouping label of each requirement cluster.
  When requirements don't cluster naturally, use the task title as the only
  sub-section.
- Each checkbox line is: `- [ ] <what the tester does> — <expected result>`.
- Horizontal rules (`---`) separate phases.
- No trailing whitespace on any line.

## YAML Companion Output

Written to `.claude/test-plans/<ticket>.yml`. See `SCHEMA.md` (this skill's
directory) for the authoritative YAML schema and all valid field names. The
skill MUST validate its output against the schema before writing.

Minimal example shape (field names from SCHEMA.md govern):

```yaml
ticket: 361
title: "Migration + Re-enable + Auto-Exempt"
plans:
  - .claude/plans/avatax-cert-migration-361
  - .claude/plans/361-auto-exempt-checkout
phases:
  - name: "Pre-deployment (UAT)"
    stories:
      - slug: migrate-avatax-certs-dry-run
        title: "Dry-run outputs one progress line per customer"
        manual_only: false
        spec: tests/m2-hyva-playwright/src/apps/pps/tests/tax-exempt-checkout.spec.ts
        story_tag: "@story: migrate-avatax-certs-dry-run"
      - slug: live-deploy-cron-no-demotions
        title: "Cron does not demote Tax Exempt customers after live migration"
        manual_only: true
        spec: ~
        story_tag: ~
  - name: "Auto-exempt checkout"
    stories:
      - slug: tax-exempt-checkbox-hidden-for-exempt-group
        title: "Tax-exempt checkbox is hidden for Tax Exempt group customers"
        manual_only: false
        spec: tests/m2-hyva-playwright/src/apps/pps/tests/tax-exempt-checkout.spec.ts
        story_tag: "@story: tax-exempt-checkbox-hidden-for-exempt-group"
```

### manual_only: true policy

When a story maps to zero Playwright tests (no matching `@story:` tag found in
any spec file), set `manual_only: true` and set `spec` and `story_tag` to `~`
(YAML null). This signals to reporting tools that this story requires a human
tester on every release. Reach this state only after confirming the behaviour
cannot be driven through the real UI (see Step 5) — a missing spec for a
UI-reachable behaviour means "write the spec", not "mark it manual".

## Gitignored Output Notice

`.claude/` is gitignored at the project root. When the skill writes the YAML
for the first time, it MUST print a one-line notice:

```
NOTICE: .claude/test-plans/<ticket>.yml written to a gitignored path — not tracked by git.
```

This ensures the operator is not surprised when the file does not appear in
`git status`. Operators who want the test plan shared across the team must
either lift the `.gitignore` entry for `.claude/test-plans/` or copy the file
out of band.

The notice is printed once per file creation, not on subsequent overwrites.

## Execution Steps (in order)

1. Parse arguments: `ticket`, `plan-path` list, flags.
2. Detect pre-existing plan comment (`gh issue view --comments` + grep). Abort
   if found (unless `--skip-comment` is set, in which case skip step 8 only —
   do not abort).
3. Load `_plan.md` and task files for each plan-path.
4. Merge and de-duplicate (if multiple plans).
5. Map criteria to stories and assign phases.
6. Assign slugs. Run pre-flight collision grep. Warn on collisions.
7. Classify `manual_only` stories. Warn operator if any found; pause for
   confirmation (skip pause on `--dry-run`).
8. Build comment Markdown. If `--dry-run`, print to stdout and stop.
9. Unless `--skip-comment`: post comment with `gh issue comment <ticket> --body "..."`.
10. Write YAML to `.claude/test-plans/<ticket>.yml`. Print gitignored-path notice
    on first write.

## Example Invocations

```bash
# Single plan
hcf:manual-test-plan 361 .claude/plans/361-auto-exempt-checkout

# Multiple plans (merged)
hcf:manual-test-plan 361 .claude/plans/avatax-cert-migration-361,.claude/plans/361-auto-exempt-checkout

# Dry run — preview comment without posting
hcf:manual-test-plan 361 .claude/plans/361-auto-exempt-checkout --dry-run

# YAML only — ticket already has a comment
hcf:manual-test-plan 361 .claude/plans/361-auto-exempt-checkout --skip-comment
```

## References

- Schema for YAML output: `SCHEMA.md` (this skill's directory)
- Ticket manual test plan (format reference): `gh issue view <ticket> --comments`
- Playwright specs: `tests/m2-hyva-playwright/src/apps/*/tests/*.spec.ts` (M2-Hyvä),
  or `dev/tests/playwright/tests/*.spec.ts` (OpenMage/M1, e.g. ntotank)
- Plan source format: `.claude/plans/<name>/_plan.md` and `.claude/plans/<name>/NNN-*.md`
