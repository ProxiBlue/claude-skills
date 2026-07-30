---
name: deploy-check
description: The sanctioned pre-push verification gate. Runs tests, type-check, shows diff stat against the target branch, and EXPLICITLY asks for user confirmation before any push. Required by CLAUDE.md's Deployment & Git Safety cardinal rules — the push-guard.sh hook blocks direct pushes to live/uat without this skill (or an explicit CLAUDE_PUSH_ALLOWED=1 session env var).
disable-model-invocation: false
---

# /deploy-check

Sanctioned pre-push verification gate. Invoke before any push to a deployment branch (`live`, `uat`, etc.) OR any push the user has flagged as needing the gate.

This skill is **the only authorised path** to push code under the central CLAUDE.md cardinal rules. Direct `git push live` / `git push uat` calls are blocked by the `push-guard.sh` PreToolUse hook.

## Inputs

- `<target-branch>` — the branch you intend to push to (default: `live` if not specified)
- Optionally, the current branch will be compared against the target

## Process

### Step 1 — Context check

Confirm working dir + branch state:

```bash
pwd
git rev-parse --abbrev-ref HEAD       # current branch
git status --short                     # uncommitted changes
```

If there are uncommitted changes → **STOP** and tell the user. Pushing with dirty tree is never right.

### Step 2 — Diff stat against target

```bash
git diff --stat origin/${TARGET_BRANCH:-live}...HEAD
git log --oneline origin/${TARGET_BRANCH:-live}..HEAD     # commits going out
```

Print both to the user. Make it visually obvious what's about to ship.

### Step 3 — Type check (if TS/TSX touched)

If any `*.ts` or `*.tsx` files are in the diff:

```bash
# inside DDEV web container OR project root with tsc available
tsc --noEmit
```

Report errors. If non-zero exit → **STOP** and tell the user.

### Step 4 — Tests

Run the project's primary test suite. Common patterns:

```bash
# Playwright (DDEV / Hyva projects)
yarn test:hyva 2>&1 | tail -30

# OR PHPUnit
./vendor/bin/phpunit -c dev/tests/unit/phpunit.xml 2>&1 | tail -30
```

Pick the right suite based on what the project's `.claude/testing.md` says. If unsure, ask the user which suite covers this change. If tests fail → **STOP**.

### Step 5 — Explicit confirmation (NON-NEGOTIABLE)

Print a single, clear question to the user:

```
=================================================
READY TO PUSH

Branch:   <current> → <target>
Commits:  <N>
Files:    <N>  (+<lines> / -<lines>)
Tests:    <suite> ✓ <count> passed
TSC:      <pass/skipped/N issues>

Proceed with `git push origin <target>`?
=================================================
```

**Wait for an explicit "yes" / "go" / "push" reply in the current turn.** Do NOT proceed on previous-turn confirmations, ambient context, or assumed consent. Re-prompt if the user's reply is anything other than an unambiguous go.

### Step 6 — Push (only on explicit confirmation)

Set the bypass env var transiently so push-guard.sh doesn't block:

```bash
CLAUDE_PUSH_ALLOWED=1 git push origin <target>
```

Print the push output. Confirm the commit SHA that's now on the remote.

### Step 7 — Post-push verification

```bash
git rev-parse origin/<target>          # confirm what's on remote
git log -1 origin/<target> --oneline
```

Report success or any anomalies.

## What this skill does NOT do

- **Does not skip tests** even if "everything looks fine". Run them.
- **Does not push --force.** Never. If a force-push is genuinely needed, the user does it manually outside this skill.
- **Does not assume a confirmation from earlier turns counts.** Each push gets its own explicit confirm prompt in the current turn.
- **Does not silence test or TSC output.** Always surface the result so the user sees what passed.
- **Does not merge or tag or run release scripts.** It only handles the push step.

## Why this exists

Per the insights report and prior session history, autonomous Claude has pushed to UAT without authorization and planned direct SSH-to-live edits. The push-guard.sh hook now blocks the most dangerous variants at the tool layer; this skill is the safe, documented alternative for legitimate pushes. The combination = structural enforcement, not just per-session reminders.
