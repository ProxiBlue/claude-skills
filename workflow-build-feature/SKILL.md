---
name: workflow-build-feature
description: "PER-TICKET BUILD workflow for shipping a Magento feature through plan + implement + review + verify + quality-gate stages — but STOPS at 'ready to deploy'. Does NOT push or deploy. The user runs /deploy-check separately when they're ready. Flow: prereq check (wires.json registry-driven, scales to N per-domain playbooks) → branch creation → /hcf:plan-create (devils-advocate consults wired playbooks: gitnexus, graphiti, …) → /manual-test-plan posts GitHub ticket comment → /hcf:plan-orchestrate (HCF native — gitnexus-reviewer runs as a pipeline.md post-implementation agent rather than per-task wrapper gate, after pb-gitnexus wrapper was retired in favour of pb-hcf wires) → handoff to /verify-feature (in a fresh thread per skill convention) → CHAINS /proxiblue-skills:workflow-security-audit as a quality gate → completion summary listing all gates passed/failed. Maintains visible TodoWrite list throughout. Pauses at every user-decision boundary. Assumes /proxiblue-skills:workflow-onboard-project has already run. Composable: future quality workflows (perf-audit, dependency-check) can be chained in by editing the QUALITY_GATES list in this skill."
disable-model-invocation: true
---

# /proxiblue-skills:workflow-build-feature

End-to-end build workflow that lands a feature in a ready-to-deploy state. **Does not push.** Separation of concerns: build verifies the change is correct; `/deploy-check` (separate skill) handles the push decision.

Replaces ~5 sequential commands documented in Phase 1 of `~/claude-plugins-central/WORKFLOW.md`, plus adds chained quality gates that the workflow doc only described in prose.

## Pre-flight (FIRST)

1. Run `hostname && pwd && ls /var/www/html 2>/dev/null` to confirm context. Refuse if not inside a DDEV web container.
2. Verify onboarding artifacts present:
   - `.claude/CLAUDE.md` (HCF setup)
   - `.claude/testing.md` (pb-hcf-playwright-tdd setup)
   - `.claude/wires.json` (pb-hcf:wire registry; lists every per-domain playbook installed)
   - `<!-- pb-hcf:start -->` marker inside `.claude/CLAUDE.md`
   If any missing → refuse and tell user to run `/proxiblue-skills:workflow-onboard-project` first.
   **Legacy fallback:** if `.claude/wires.json` is absent but `.claude/gitnexus.md` + `<!-- pb-gitnexus:start -->` are present, tell user to re-run `/pb-hcf:wire` to migrate the legacy fence (the wire skill auto-migrates) and then re-invoke this skill. Don't proceed on the legacy state.
3. **Verify every wired playbook's reachability**, registry-driven:
   ```bash
   # For each entry in .claude/wires.json -> playbooks[]:
   #   run its `probe` command (or call its `probe` MCP tool name)
   #   compare against the expected response
   #   on failure: surface the entry's `name`, the probe, and the recommended fix from pb-hcf docs
   ```
   This scales — when pb-hcf adds a new playbook (security.md, playwright.md, …), the pre-flight loop picks it up automatically without editing this skill. If ANY probe fails → STOP, print the failures + fixes, refuse to proceed (running a plan against a half-up stack produces unreliable output).
4. Ask the user for: ticket number, branch name (default: `feature/<ticket>-<short-desc>`), one-paragraph feature description.

## Quality gates (composable)

Configured set the workflow chains AFTER verify-feature. Edit this list to add more:

```
QUALITY_GATES = [
  "workflow-security-audit",   # required by default — security review of the diff
  # "workflow-add-test-coverage",   # uncomment if coverage fill is wanted post-build
  # "workflow-perf-audit",           # available when hcf-xhgui ships
]
```

Each gate runs in sequence. If a gate produces blocking findings, the workflow surfaces them and stops before the completion summary. Non-blocking findings get listed in the completion summary as "review before deploy".

## Visible todo list

```
1. Verify onboarding artifacts present
2. Verify every wired playbook reachable (loop .claude/wires.json registry)
3. Create feature branch from live (after confirming clean working tree)
4. /hcf:plan-create — generate plan + devils-advocate review (consults wired playbooks: gitnexus, graphiti, …)
5. [USER REVIEW] Plan + devil's advocate findings — proceed/refine
6. /manual-test-plan <ticket> <plan-path> — GH ticket comment + YAML
7. [USER REVIEW] GH ticket comment posted — confirm scope
8. /hcf:plan-orchestrate — HCF native orchestration; gitnexus-reviewer + workflow-security-audit run as pipeline.md post-implementation agents (no wrapper)
9. [PAUSE] Wait for orchestrator + summary
10. [USER ACTION] Open a FRESH thread (skill convention) and invoke /verify-feature <ticket>
11. [USER REVIEW] Verify-feature results — confirm all stories passing
12. Chain quality gates (default: workflow-security-audit — also runnable inside pipeline.md if preferred)
13. Aggregate gate results + print build-complete summary
14. Tell user: "Ready to deploy. Run /deploy-check <target> when ready."
```

## Step details

### Step 3 — branch creation

```bash
git status --short                          # MUST be clean
git fetch origin live
git checkout -b feature/<ticket>-<desc> origin/live
```

If working tree dirty → STOP, tell user to stash or commit.

### Step 4 — plan-create

```
/hcf:plan-create
```

HCF asks its own questions about scope, tasks, etc. Pass through. The devils-advocate that runs at end of plan-create automatically consults every wired playbook via `.claude/CLAUDE.md` context — gitnexus for code-graph impact, graphiti for related decisions / discussed-but-not-built features in the same domain, and any future wires (security, playwright) as they ship.

### Step 5 — pause for plan review

Print:

```
Plan generated at: .claude/plans/<plan-name>/_plan.md
Devil's advocate findings: .claude/plans/<plan-name>/_devils_advocate.md

Read both. Reply 'continue' to proceed to manual-test-plan, or specify changes you want first.
```

Wait for explicit "continue" or equivalent.

### Step 6 — manual-test-plan

```
/manual-test-plan <ticket> .claude/plans/<plan-name>/_plan.md
```

Posts phased comment to GH ticket. Writes `.claude/test-plans/<ticket>.yml`.

### Step 7 — pause for ticket comment review

```
Manual test plan posted to GH ticket <ticket>.
YAML written to .claude/test-plans/<ticket>.yml.

Open the ticket in your browser. Review the manual test plan comment.
Reply 'continue' if the scope matches your intent, or 'refine' to adjust.
```

### Step 8 — orchestrate (HCF native, no wrapper)

```
/hcf:plan-orchestrate
```

Pass plan name. HCF orchestrates the full plan natively. The pb-gitnexus wrapper that previously gated per-task with the gitnexus-reviewer has been retired (it was creating parallel-dev drift with HCF upstream). The reviewer is now invoked via the project's `pipeline.md` `## post-implementation` slot instead — runs over the whole batch's diff at plan-end, no wrapping.

If the project's `pipeline.md` lists `gitnexus-reviewer` (and/or `workflow-security-audit`) under `## post-implementation`, HCF picks them up natively in Phase 6 of plan-orchestrate. Pre-task gating is lost; trade-off accepted in exchange for not wrapping HCF.

Wait for completion. Surface any pipeline.md agent findings from the orchestration output.

### Step 10 — verify-feature handoff

`/verify-feature` per skill convention runs in a FRESH thread (so the orchestration thread's todos don't interfere with verify's own TodoWrite tracking). Tell the user:

```
Orchestration complete. To run verification:

  1. Open a NEW Claude Code thread (Ctrl-N or new terminal)
  2. In that thread: /verify-feature <ticket>
  3. When verify-feature finishes (all stories passing OR stops on a failure):
     come back to THIS thread and reply 'verified' (or 'failed' with details)
```

Wait for user to come back.

### Step 11 — pause for verify-feature review

If user replies 'verified': proceed to quality gates.
If 'failed': stop. Surface failure details. Do NOT chain quality gates on a failed build — fix tests first, re-invoke.

### Step 12 — chain quality gates

For each gate in QUALITY_GATES (in order), invoke it on the current branch's diff vs live:

```
/proxiblue-skills:<gate-skill-name>
```

Each gate skill must follow the convention: return either `STATUS: PASS`, `STATUS: PASS-WITH-NOTES`, or `STATUS: BLOCKING` plus structured findings. Aggregate to a single results object:

```python
gates = {
  "security-audit": {"status": "PASS-WITH-NOTES", "notes": [...]},
}
```

Default chain has just `workflow-security-audit`. Future: `workflow-add-test-coverage` (if pre-existing gaps in touched files), `workflow-perf-audit` (when hcf-xhgui ships).

### Step 13 — aggregate + summarize

```
============================================================
BUILD COMPLETE — ticket <ticket>

Plan:               .claude/plans/<plan-name>/_plan.md
Devils-advocate:    .claude/plans/<plan-name>/_devils_advocate.md
Per-task reviews:   <see plan task files; review-outcomes summary above>
Test plan + YAML:   .claude/test-plans/<ticket>.yml
GH ticket:          <link>
Verify run:         <link to verify-feature artifact comment>

Quality gates:
  workflow-security-audit:  PASS-WITH-NOTES (2 informational findings — see notes)

============================================================
READY TO DEPLOY

Branch:  <current> (HEAD: <sha>, N commits ahead of live)

To deploy:  /deploy-check live
============================================================
```

If any gate returns `BLOCKING`: stop here. Print the blocking findings. Do not say "ready to deploy". Tell user to address findings + re-invoke build-feature (or skip to the failing gate with `/proxiblue-skills:workflow-security-audit` directly + re-run quality gates).

### Step 14 — NOT push

Explicitly do not invoke `/deploy-check`. The user decides when to deploy. This skill ends at "ready to deploy".

## What this skill does NOT do

- **Does not push or deploy.** Build vs deploy is a deliberate separation — different decisions, different timings.
- Does not skip the manual review pauses — those are intentional gates.
- Does not run `/verify-feature` itself (skill convention requires a fresh thread).
- Does not handle hotfixes / out-of-band changes — those are a different (shorter) workflow.

## Recovery

If interrupted: re-invoke and tell the skill what step you were at. The skill should resume from there. Most steps are idempotent.

## Composability — adding a new quality gate

To add (e.g.) a perf audit gate when `hcf-xhgui` ships:

1. Build `/proxiblue-skills:workflow-perf-audit` following the same return-convention (`PASS` / `PASS-WITH-NOTES` / `BLOCKING`)
2. Edit this skill's `QUALITY_GATES` list, append the new gate name
3. Done — every subsequent `/workflow-build-feature` invocation runs the new gate as part of step 12

## Skip cases (when NOT to use)

- Tiny single-line changes — overhead > value. Just do it + `/deploy-check`.
- Hotfix from a known-good diagnosis — skip plan-create + orchestration. Use `/proxiblue-skills:workflow-investigate-bug` first if root cause unclear.
- Exploration / spike — no plan to orchestrate; freeform until you have a plan.
