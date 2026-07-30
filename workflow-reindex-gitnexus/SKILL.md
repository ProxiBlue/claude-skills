---
name: workflow-reindex-gitnexus
description: "Maintenance workflow to re-index the per-project GitNexus knowledge graph after a major refactor, dependency update, or when impact/find_symbol queries stop matching reality. Use when: a recent merge added new modules, you ran composer update with significant version bumps, the gitnexus-reviewer is reporting symbols that don't match the actual code, or it's been a long time since the last reindex (index is a snapshot — drift accumulates). Workflow: detects the project, derives the correct mount alias from the .ddev/docker-compose.gitnexus.yaml, clears the existing lbug, runs build-mount.sh on the host (with crash auto-recovery if a tree-sitter Napi error fires), restarts the per-project gitnexus container, verifies the new index via mcp__gitnexus-mageos__list_repos. Crosses host/container boundary: host steps are explicitly handed off with copy-paste commands when invoked from inside the DDEV web container."
disable-model-invocation: true
---

# /proxiblue-skills:workflow-reindex-gitnexus

Re-index the per-project gitnexus knowledge graph. Run when the index has drifted from reality (after refactor, composer update, or long uptime).

## Pre-flight

1. Run `hostname && pwd && ls /var/www/html 2>/dev/null` to determine context.
2. If running from HOST shell: proceed directly with all steps.
3. If running from DDEV web container: ALL re-index steps must run from the host. Switch to host shell first. Print exact commands and stop.

## Visible todo list

```
1. Detect project root + DDEV project name
2. Derive mount alias from .ddev/docker-compose.gitnexus.yaml volumes line
3. Show current index stats (gitnexus list before)
4. [USER CONFIRM] Proceed with re-index (destructive — clears existing lbug)
5. Stop the gitnexus container (so no file lock on lbug)
6. rm -rf <project>/.gitnexus
7. build-mount.sh <project-path> <alias>
   - On Napi::Error: surface the auto-detected culprit file + the .gitnexusignore fix one-liner, ask user to apply, retry
8. Start the gitnexus container
9. Verify new index (gitnexus list after — file/node/edge deltas)
10. [USER ACTION inside container] /pb-gitnexus:wire --reprobe (or full re-wire if signature changed)
11. Print completion summary with delta
```

## Step details

### Step 1 — detect project

Find the DDEV project root. If invoked with no args and cwd is inside a project: use that. Otherwise ask user for the project path.

```bash
DDEV_PROJ=$(cd /path/to/project && ddev describe -j 2>/dev/null | jq -r '.raw.name')
```

### Step 2 — derive alias

```bash
ALIAS=$(grep -E "${DDEV_APPROOT}:/mounts/" /path/to/project/.ddev/docker-compose.gitnexus.yaml | sed -E 's|.*:/mounts/([^"]+).*|\1|')
```

If the file doesn't exist or alias not derivable: refuse — project hasn't been onboarded with the gitnexus add-on.

### Step 3 — current stats

```bash
docker exec ddev-${DDEV_PROJ}-gitnexus gitnexus list 2>&1 | grep -A 5 "${ALIAS}\|/mounts/${ALIAS}"
```

Capture for the before/after delta.

### Step 4 — destructive confirm

```
About to re-index: <project> (alias: <alias>)

This will:
  - Stop the gitnexus container
  - Delete <project-path>/.gitnexus/ (lbug + meta + parse-cache)
  - Re-run build-mount.sh from scratch (minutes, depends on code size + .gitnexusignore filter)
  - Restart the gitnexus container

Reply 'go' to proceed, anything else to abort.
```

### Step 5 — stop container

```bash
docker stop ddev-${DDEV_PROJ}-gitnexus 2>&1
```

### Step 6 — clear

```bash
rm -rf /path/to/project/.gitnexus
```

### Step 7 — re-build

```bash
~/workspace/proxiblue/mage-os-gitnexus/scripts/build-mount.sh /path/to/project <alias>
```

On `Napi::Error` / `terminate called` — the script auto-detects the culprit file and prints:

```
Tree-sitter crashed parsing:
  app/code/Vendor/Module/some-file.js

To exclude this exact file, append to:
  /path/to/project/.gitnexusignore
this line:
  app/code/Vendor/Module/some-file.js

Broader pattern (excludes any file with this name):
  **/some-file.js

One-liner — append exact file + retry:
  echo 'app/code/Vendor/Module/some-file.js' >> /path/to/project/.gitnexusignore && \
    rm -rf /path/to/project/.gitnexus && \
    /path/to/build-mount.sh /path/to/project <alias>
```

Show this to the user, wait for them to apply (or to authorise the one-liner). Re-run. Loop until success.

### Step 8 — start container

```bash
docker start ddev-${DDEV_PROJ}-gitnexus 2>&1
sleep 3
```

### Step 9 — verify new index

```bash
docker exec ddev-${DDEV_PROJ}-gitnexus gitnexus list 2>&1 | grep -A 5 "${ALIAS}\|/mounts/${ALIAS}"
```

Capture for delta.

Expected entries:
- `mageos` (bundled)
- `hyva` (bundled)
- `deps` (bundled)
- `<alias>` or git-remote-name (custom) — should show new commit sha + updated file/node/edge counts

If the custom alias is missing → registry didn't land. Surface the container's startup logs (`ddev logs gitnexus 2>&1 | tail -30`).

### Step 10 — re-wire (handoff to in-container session)

Tell the user:

```
Re-index complete. Open a claude-code session inside the DDEV container
and run:

  /pb-gitnexus:wire --reprobe

This refreshes .claude/gitnexus.json with the latest reachability + repos list
without rewriting the wire docs.

If the project's index alias CHANGED (e.g. you set up a new git remote),
do the full re-wire instead:

  /pb-gitnexus:wire
```

### Step 11 — completion summary

```
============================================================
REINDEX COMPLETE — <project>

Alias:          <alias>
Files:          <before> → <after>   (Δ <signed>)
Nodes:          <before> → <after>   (Δ <signed>)
Edges:          <before> → <after>   (Δ <signed>)
Commit:         <before-sha> → <after-sha>
Indexed at:     <new timestamp>

Next: re-wire from inside the container (see handoff above), then run
your usual workflows. gitnexus-reviewer and devils-advocate will now see
the fresh code-graph state.
============================================================
```

## What this skill does NOT do

- Does not push or commit anything.
- Does not modify `.gitnexusignore` automatically (user authorises any addition).
- Does not touch other projects' indexes — strictly scoped to the one project.
- Does not rebuild the gitnexus image itself (that's a separate, ~minutes-long operation).

## When to invoke

- After `composer update` with major version bumps (vendor tree restructures)
- After a large merge to live (many new modules / classes)
- When `mcp__gitnexus-mageos__find_symbol` reports symbols you can see in the code but the graph doesn't have
- When `gitnexus-reviewer` is flagging "Critical: symbol not found" for symbols that clearly exist (index drift, not real bug)
- Periodically (e.g., monthly) for actively-developed projects
