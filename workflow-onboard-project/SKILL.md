---
name: workflow-onboard-project
description: "IN-DDEV onboarding workflow that brings a Mage-OS project's claude-code session into the full ProxiBlue stack. Runs ENTIRELY inside the DDEV web container — does NOT do host-side prep (image build, ddev add-on get, build-mount.sh, ddev restart). Those are hard prerequisites the skill verifies upfront and refuses if missing, with exact host-side commands the user must run first. Once prereqs satisfied: installs the 4 ProxiBlue plugins (pb-hcf, pb-hcf-playwright-tdd, proxiblue-skills, hyva-ai-tools), runs /hcf:project-setup, configures Playwright+pcov TDD via /pb-hcf-playwright-tdd:setup, and wires all per-domain playbooks via /pb-hcf:wire (one wire skill, multi-playbook installer — replaces the per-plugin /pb-gitnexus:wire of v0.x). Visible TodoWrite list throughout. Idempotent — re-runnable, skips already-done steps."
disable-model-invocation: true
---

# /proxiblue-skills:workflow-onboard-project

In-container onboarding for a Mage-OS DDEV project. Assumes all host-side prep is done.

## Host-side prerequisites (NOT done by this skill)

If any of these aren't true, the skill refuses with the exact command to run host-side. Do them first:

1. **gitnexus image built locally** — `docker image inspect mage-os-gitnexus:latest` succeeds on the host
2. **DDEV add-on installed in the project** — `.ddev/docker-compose.gitnexus.yaml` exists in the project root
3. **Index pre-built** — `<project>/.gitnexus/lbug` exists (created by `scripts/build-mount.sh` on the host)
4. **DDEV restarted at least once** since the add-on was installed — so the gitnexus container is running

Host commands the user runs ONCE before invoking this skill:

```bash
# One-time per machine: build the image
cd ~/workspace/proxiblue/mage-os-gitnexus && docker compose build

# Per project, on host:
cd /path/to/project
ddev add-on get ~/workspace/proxiblue/mage-os-gitnexus/ddev-addon
~/workspace/proxiblue/mage-os-gitnexus/scripts/build-mount.sh /path/to/project <short-alias>
ddev restart
# then ddev ssh and re-invoke this skill
```

## Pre-flight (FIRST step inside container)

1. Run `hostname && pwd && ls /var/www/html 2>/dev/null` and confirm you're inside a DDEV web container with `/var/www/html` as the project root. If not → refuse with "this skill runs inside the DDEV web container, run `ddev ssh` from the host first".
2. Verify host-side prereqs via container-visible signals:
   - `.ddev/docker-compose.gitnexus.yaml` exists at `/var/www/html/.ddev/docker-compose.gitnexus.yaml` → if missing, refuse with host command list above
   - `curl -sS -o /dev/null -w '%{http_code}\n' -m 3 http://gitnexus:4747/` returns 200 → if not, refuse with "gitnexus container not reachable — run `ddev restart` on the host"
   - `mcp__gitnexus-mageos__list_repos` returns the project's index → if absent, refuse with "the project's lbug isn't loaded — re-run build-mount.sh on the host then `ddev restart`"
3. Detect Magento: `grep -E '"(mage-os|magento)/(product-community-edition|magento2-base)"' /var/www/html/composer.json` — if no match, refuse (this stack is Mage-OS-only).

## Visible todo list

```
1. Verify in-DDEV context + host-side prereqs satisfied
2. Install pb-hcf plugin (skip if installed)
3. Install pb-hcf-playwright-tdd plugin (skip if installed)
4. Install proxiblue-skills plugin (likely already present — skip if so)
5. Install hyva-ai-tools plugin (skip if installed)
6. /hcf:project-setup (skip if .claude/CLAUDE.md already shows HCF setup)
7. /pb-hcf-playwright-tdd:setup (refuses if pcov absent — surface fix, stop)
8. /pb-hcf:wire (re-run safely if already wired — multi-playbook installer; auto-migrates any legacy <!-- pb-gitnexus:start --> fence)
9. Verify .claude/{CLAUDE.md, testing.md, wires.json} exist + CLAUDE.md has pb-hcf fence + every playbook listed in wires.json has its .claude/<name>.md
10. Print "ready — invoke /proxiblue-skills:workflow-build-feature for your first ticket"
```

## Step details

### Steps 2-5 — plugin installs (idempotent)

For each:

```
/plugin install pb-hcf@pb-hcf
/plugin install pb-hcf-playwright-tdd@pb-hcf-playwright-tdd
/plugin install proxiblue-skills@proxiblue-skills
/plugin install hyva-ai-tools@hyva-ai-tools
```

If a legacy `pb-gitnexus` install is present, also `/plugin uninstall pb-gitnexus` after `/pb-hcf:wire` runs successfully — wire migrates the fence + playbook so the legacy plugin is no longer needed.

For any that report "already installed" → mark task completed and proceed (not an error).

### Step 6 — HCF project setup

```
/hcf:project-setup
```

HCF will ask its own questions about scope, plan dir, etc. Pass through. If `.claude/CLAUDE.md` already exists with HCF markers → ask user if they want to re-run (default: skip).

### Step 7 — Playwright + pcov TDD setup

```
/pb-hcf-playwright-tdd:setup
```

HARD requirement: pcov must be loaded in the container. If the skill refuses with "pcov absent", relay the exact `.ddev/config.yaml` fix it suggests:

```yaml
webimage_extra_packages: [..., php<X.Y>-pcov]
xdebug_enabled: false
```

Then `ddev restart` from the host, then `ddev ssh` back in, then re-invoke this skill (will skip already-completed earlier steps).

### Step 8 — wire all per-domain playbooks

```
/pb-hcf:wire
```

Multi-playbook installer: discovers every `*.md` under pb-hcf's `templates/playbooks/`, installs each as `.claude/<name>.md`, appends a single fenced section to `.claude/CLAUDE.md` listing pointers to all of them, runs per-domain reachability probes, and writes `.claude/wires.json` as the wire registry.

If the project was previously wired with the legacy `/pb-gitnexus:wire`, this skill auto-detects the `<!-- pb-gitnexus:start -->` fence in `.claude/CLAUDE.md` and replaces it with `<!-- pb-hcf:start -->`. The legacy `.claude/gitnexus.json` is superseded by `.claude/wires.json`. After verifying `wires.json` is populated, `/plugin uninstall pb-gitnexus`.

Idempotent — re-running refreshes the wire registry timestamp + re-probes reachability. Pass `--reprobe` to update only reachability state.

### Step 9 — final verification

```bash
test -f /var/www/html/.claude/CLAUDE.md && echo "CLAUDE.md ok"
test -f /var/www/html/.claude/testing.md && echo "testing.md ok"
test -f /var/www/html/.claude/gitnexus.md && echo "gitnexus.md ok"
grep -q "<!-- pb-gitnexus:start -->" /var/www/html/.claude/CLAUDE.md && echo "gitnexus fence ok"
```

All four should report ok. If any missing → surface which step's output the user should re-check.

### Step 10 — completion

```
============================================================
ONBOARDING COMPLETE — /var/www/html

Plugins installed:   pb-gitnexus, pb-hcf-playwright-tdd, proxiblue-skills, hyva-ai-tools
HCF setup:           done (.claude/CLAUDE.md present)
TDD config:          done (.claude/testing.md present, pcov verified)
GitNexus wire:       done (.claude/gitnexus.md + CLAUDE.md fence present)

Next: /proxiblue-skills:workflow-build-feature  (for your first ticket)
Reference: ~/claude-plugins-central/WORKFLOW.md
============================================================
```

## What this skill does NOT do

- **Does not do any host-side work.** No image build, no `ddev add-on`, no `build-mount.sh`, no `ddev restart`. Refuses upfront if the host-side prep isn't done.
- Does not modify `.ddev/config.yaml` (e.g., to add pcov). If pcov absent, prints the YAML the user must add host-side, stops.
- Does not push or commit anything.
- Does not run any feature tasks — just bootstrap.

## Recovery

Re-invoke anytime. Each step is idempotent:
- Plugin installs: "already installed" is OK
- /hcf:project-setup: asks before overwriting
- /pb-hcf-playwright-tdd:setup: idempotent re-runs, asks before overwrite
- /pb-hcf:wire: refreshes .claude/wires.json registry, replaces fenced CLAUDE.md section, re-probes per-domain reachability

The TaskCreate list re-seeds; completed steps just verify+skip on re-runs.
