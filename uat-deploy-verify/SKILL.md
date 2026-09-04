---
name: uat-deploy-verify
description: Reproduce-push-verify cycle for shipping a fix to UAT — confirms the bug still reproduces on UAT (reusing the local test that proves the fix, pointed at the UAT host), merges + pushes the feature branch to `uat` (structurally authorized to clear push-guard.sh's uat block for that one push), confirms via the buddy-mcp Buddy.works MCP server that the deploy pipeline actually completed, then reruns the same test against UAT to confirm the bug is gone. Invoke explicitly (e.g. "/uat-deploy-verify") — never auto-triggered, since it performs an authorized push.
disable-model-invocation: true
---

# /uat-deploy-verify

Moves an already-implemented, already-tested fix from a feature branch onto
`uat`, and proves — with evidence, not assumption — that the deploy actually
landed and the bug is actually gone there. This skill does **not** write the
fix. It runs after the normal TDD/dev cycle, once a feature branch already has
a passing local test for the bug.

Companion to `/deploy-check` (the `live` push gate) — this one is `uat`-only
and adds the reproduce-before / verify-after cycle deploy-check doesn't do.
Never touches `live`; that stays exactly as blocked as always.

## Preconditions (check before Step 0)

```bash
git status --short          # must be clean
git rev-parse --abbrev-ref HEAD   # must be the feature branch, not uat/live
```

If the tree is dirty, or you're sitting on `uat`/`live` already, **STOP** and
tell the user. You also need to already know which local test proves the fix
(from the ticket's `.claude/test-plans/<ticket>.yml` if `manual-test-plan` /
`verify-feature` produced one, or a specific spec/test the user names). If
none is identified, ask before continuing — Steps 1 and 4 both depend on it
being the exact same test run against two different hosts.

## Step 0 — Confirm buddy-mcp is wired

Step 3 is not optional, so this has to work before you push anything.

Buddy is fleet-seeded, not per-project-installed: the `buddy` server lives
in `~/claude-skills-central/mcps/.mcp.json` (same place as `graphiti` /
`chatroom`), mounted read-only into every DDEV project at
`.ddev/claude-code/.claude/mcp.json`, and `BUDDY_TOKEN` is wired into every
project's `.ddev/docker-compose.ai.mounts.yaml` `environment:` block
(sourced from the host's `~/.config/secrets.env`). There is no
`claude plugin install` step and no `enabledPlugins` entry — nothing to set
up per project. (`~/claude-plugins-central/seed/marketplaces/buddy-mcp` is a
superseded, do-not-install reference copy only — see its CHANGELOG.)

1. Sanity-check the tool actually responds (list pipelines for this
   project's Buddy workspace). If it doesn't:
   - Confirm the mount reached the container: `cat /var/www/html/.ddev/claude-code/.claude/mcp.json | grep buddy` inside the container. If absent, the project's `docker-compose.ai.mounts.yaml` predates this wiring — add the `BUDDY_TOKEN=$BUDDY_TOKEN` environment line (see the other 11 fleet projects for the pattern) and `ddev restart`.
   - Confirm `BUDDY_TOKEN` reaches the container (`echo $BUDDY_TOKEN` inside it). If empty, the host-side `~/.config/secrets.env` export didn't reach `ddev start` — check it's sourced in the shell that ran `ddev start`.
2. Identify **which pipeline** deploys to UAT for this project. Ask the user
   if it's not obvious from the project name/history.

If buddy-mcp can't be reached after this, **STOP**. Do not fall back to
"assume the push means it deployed" — a git push landing on `uat` and a
Buddy pipeline finishing are two different events, and the entire point of
this skill is not conflating them. Report the blocker and wait.

## Step 1 — Confirm the bug still reproduces on UAT

The fix already passes locally. Prove UAT still has the *old* behavior —
otherwise there's nothing to deploy for.

1. Determine the UAT host. Check the project's CLAUDE.md / `.claude/testing.md`
   for the documented convention (commonly `uat.<primary-domain>`, e.g.
   `uat.laptoplcdscreen.com.au`). Ask if it isn't documented anywhere.
2. Point the *one* test that proves the fix at the UAT host instead of local,
   changing nothing else:
   - **m2-hyva-playwright** projects: `config.init.ts` resolves
     `process.env.url = privateData.url || jsonData.url` from
     `src/apps/<app>/config.private.json`. Temporarily set that file's `url`
     to the UAT host for this one run, then **restore it immediately after**
     the run (Step 1 and Step 4 both need this — never leave a UAT URL sitting
     in a file meant for local dev, and never commit it).
   - Other stacks: look for a `BASE_URL` / `TEST_HOST`-style env var override
     before assuming a config-file edit is needed.
   - Run only that one test (`--grep "<exact test title>"`, or PHPUnit
     `--filter`) — this is a targeted check, not a regression sweep.
3. **Test passes against UAT** (bug does NOT reproduce there): **STOP**. UAT
   may already have the fix, the bug may be local-only, or UAT's data/cache
   differs from what the ticket assumed. Report this to the user and ask how
   to proceed — do not push.
4. **Test fails against UAT** (bug reproduces, same as pre-fix local
   behavior): continue to Step 2.

## Step 2 — Push the fix to UAT

1. Re-confirm the tree is still clean and the fix is fully committed.
2. Print an explicit confirmation block (same spirit as `/deploy-check` step
   5) and **wait for an unambiguous "yes"/"go"/"push" in the current turn**
   — no acting on an earlier-turn confirmation:
   ```
   =================================================
   READY TO PUSH TO UAT
   Branch:   <feature-branch> -> uat  (regular merge, no squash)
   Commits:  <N>
   Reproduced on UAT: yes (see Step 1)
   Proceed?
   =================================================
   ```
3. On explicit go, merge the feature branch into `uat` with a regular merge
   (never squash, per this fleet's `uat` convention — it's a testing
   superset, not a deploy artifact):
   ```bash
   git fetch origin uat
   git checkout uat && git pull origin uat
   git merge --no-ff <feature-branch>
   ```
4. Authorize and execute exactly one push:
   ```bash
   date +%s > "$(git rev-parse --git-dir)/.claude-uat-push-authorized"
   git push origin uat
   ```
   `push-guard.sh` blocks `git push … uat` by default. This marker is the
   sanctioned, single-use exception — written only here, only after the
   explicit confirmation above, consumed (deleted) by the hook on first read
   whether valid or not, and expires after 10 minutes if unused. It
   authorizes **uat only**; `live`, `--force`, `ddev push`, and SSH-to-live
   stay hard-blocked with no change. Never write this marker speculatively
   or ahead of the confirmed push it's for.
5. Return to the feature branch — never leave the session sitting on `uat`:
   ```bash
   git checkout <feature-branch>
   ```

## Step 3 — Confirm the deploy actually ran (Buddy)

A successful `git push origin uat` only means the branch moved — it says
nothing about whether the UAT server updated.

1. Using the `buddy-mcp` tools identified in Step 0, find the latest
   execution of the UAT deploy pipeline triggered by this push and poll it
   until it reports a terminal state (succeeded / failed) — not just
   "started" or "queued."
2. **Pipeline failed or errored**: **STOP**. Report the failure detail from
   Buddy's execution log. Do not proceed to Step 4 — retesting a UAT that
   never actually updated would produce a misleading "still broken" result.
3. **Pipeline succeeded**: continue to Step 4.
4. If buddy-mcp becomes unreachable mid-check: do not silently assume
   success. Ask the user to confirm the deploy completed by another means
   (Buddy dashboard, deploy log) before continuing.

## Step 4 — Rerun the test against UAT to confirm the fix

1. Reapply the same host override from Step 1 (same file/env mechanism),
   restore it after.
2. Run the exact same test from Step 1 against UAT.
3. **Passes**: report success — the bug/ticket, the commit(s) now on `uat`,
   the Buddy pipeline run link, and the test that now passes.
4. **Still fails**: report clearly that Buddy confirmed the deploy but the
   bug persists (check for cache/CDN/config drift on UAT before assuming the
   fix itself is wrong) — do not declare success.

## What this skill does NOT do

- Does not implement the fix — that happens before this skill runs.
- Does not deploy to `live`. No bypass in this skill reaches the `live`
  block; that stays exactly as gated as it is today.
- Does not skip Step 1 (proving the push is needed) or Step 3 (proving the
  deploy landed) under time pressure — either skip defeats the point of the
  skill.
- Does not merge `uat` into anything else, and does not open a uat→live
  path. `uat` stays one-way: feature branches in, tested, never merged back
  out. Live only ever receives code from a feature branch via squash merge
  through `/deploy-check`.
- Does not write or extend the push-guard.sh marker mechanism for anything
  other than the single push in Step 2.

## Related skills

- `/deploy-check` — the `live` push gate. Separate skill, separate bypass,
  untouched by this one.
- `/manual-test-plan` + `/verify-feature` — if `.claude/test-plans/<ticket>.yml`
  already exists for this bug, Steps 1 and 4 should run through those
  tracked tests rather than improvising a new one.
