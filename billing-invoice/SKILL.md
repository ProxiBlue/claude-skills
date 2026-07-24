---
name: billing-skill
description: >
  Skill to help with the xero billing generation
disable-model-invocation: true  
---


# Billing Invoice — Addendum


These rules supplement the billing-invoice skill. Apply them whenever invoicing.

READ THIS: http://ddev-billing-web/docs/BILLING-INTEGRATION.md

## Scope

Always use the **current project's GitHub repo** (determined from `gh repo view --json nameWithOwner --jq '.nameWithOwner'`). Never hardcode a repo name.

## Prerequisites — bridge token must be mounted

Only projects that actually invoice clients need this. Most ddev projects DO NOT bill and should not be configured for the bridge. Set this up per-project, on demand, the first time you try to invoice from that project.

### Check whether this project can reach the bridge

Run inside the project's ddev container before invoicing:

```bash
# 1. Token mounted?
test -r /etc/billing-bridge/token && echo "TOKEN_PRESENT" || echo "TOKEN_MISSING"

# 2. Billing container reachable on docker net?
getent hosts ddev-billing-web >/dev/null && echo "BRIDGE_REACHABLE" || echo "BRIDGE_UNREACHABLE"

# 3. Bridge responds?
curl -fsS -o /dev/null -w "HTTP %{http_code}\n" http://ddev-billing-web/internal/cli || echo "BRIDGE_DOWN"
```

If all three pass: proceed with invoicing.

### If TOKEN_MISSING — set up the mount

The bridge mount is per-project, opt-in. Adding it once is a one-time setup per project that needs to invoice.

**Host side (one-time, only if `~/.config/billing-bridge/token` does NOT exist):**

```bash
mkdir -p ~/.config/billing-bridge
openssl rand -hex 32 > ~/.config/billing-bridge/token
chmod 600 ~/.config/billing-bridge/token
```

The billing app uses the same file. Generating it again invalidates existing client projects; generate once, share across.

**Per-project (create `.ddev/docker-compose.billing.yaml` in the project root):**

```yaml
services:
  web:
    volumes:
      - $HOME/.config/billing-bridge/token:/etc/billing-bridge/token:ro
```

Then on the host:

```bash
ddev restart   # in the project directory — required to apply the new volume mount
```

That is the only setup; the bridge URL (`http://ddev-billing-web/internal/cli`) is reachable from every ddev container on the shared docker network once billing is running.

### Reference

For the canonical doc on the bridge, see `~/workspace/proxiblue/billing/ai/handoff.md`. This skill section is the working subset.

## AI-assisted billing format

All work is AI-assisted. Estimate hours as follows for each commit/task:

1. **Human est** — how long this would take a developer without AI
2. **AI actual** — estimated real time with AI assistance
3. **+20min** — add 20 minutes (0.33h) to each item for Lucas's time directing/reviewing AI work
4. **Billed** — AI actual + 20min, rounded to nearest 0.25h, minimum 0.5h

**Rate: determined inside the billing app. can likely query its endpoints if
needed to know

### Line item description format

Invoice line item descriptions MUST include the time breakdown:
```
fix: description here (#ticket)
⏱ Human: Xh | AI: Xh | Billed: Xh
```

### Bundling rule

Only bundle if individual AI time < 0.5h. Bundle related small tasks to reach the 0.5h minimum. Do not bundle items that already meet the minimum individually.

### Proposal table format

When proposing line items, show this table for approval:

| Ticket | Commit | Description | Human est | AI actual | +20min | Billed | Line total |
|--------|--------|-------------|-----------|-----------|--------|--------|------------|

## Invoiced label tracking

### Setup (run once per repo if missing)

Before processing any invoice, determine the repo:

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
```

Check if the `invoiced` label exists:

```bash
gh label list --repo "$REPO" | grep -i invoiced
```

If not found, create it:

```bash
gh label create "invoiced" --description "Ticket has been billed to client" --color "0e8a16" --repo "$REPO"
```

### Eligibility rules

A ticket is eligible for invoicing ONLY if:

1. **Closed by someone other than ProxiBlue (Lucas van Staden)** — tickets closed by ProxiBlue are client-side closures and must NOT be invoiced. Check with:
   ```bash
   gh api "repos/$REPO/issues/NNN/timeline" --jq '[.[] | select(.event=="closed")] | last | .actor.login'
   ```
   If result is `ProxiBlue`, skip the ticket and note it in the proposal as "skipped — closed by client".

2. **Does NOT already have the `invoiced` label** — prevents double-billing. Check with:
   ```bash
   gh issue view NNN --repo "$REPO" --json labels --jq '.labels[].name' | grep -i invoiced
   ```

### After invoice is created

After a successful invoice submission, apply the `invoiced` label to all billed tickets:

```bash
gh issue edit NNN --add-label "invoiced" --repo "$REPO"
```

## Finding unbilled work

When searching for commits related to a ticket:

1. Search `git log --grep="#NNN"` first
2. If nothing found, also search `git log --grep="GITHUB-NNN"` and `git log --grep="GITHUB_NNN"` — commit messages use multiple formats
3. If still nothing, **read the ticket body and comments** for context on what was done — look for root cause analysis, config changes, composer patches
4. Search by keywords from the ticket (function names, module names) as last resort
5. Only conclude "no work found" after exhausting all search patterns

### Finding all uninvoiced tickets

To find all deployed but uninvoiced work:

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
gh issue list --repo "$REPO" --label "has been deployed live" --state closed --json number,title,labels --jq '.[] | select(.labels | map(.name) | index("invoiced") | not)'
```

## Troubleshooting — billing container unreachable

The bridge endpoint is the docker DNS name `ddev-billing-web`. The container's IP can change across restarts, but the DNS name stays stable — so "IP changed" is never the real problem. Look for one of the modes below.

### Mode 1: Billing container is stopped

Symptom (inside a sibling container):
```
curl: (6) Could not resolve host: ddev-billing-web
# or
curl: (7) Failed to connect to ddev-billing-web
```

`getent hosts ddev-billing-web` returns nothing.

**From inside the sibling container** you cannot run `ddev`. Options:

- Ask the user to run `ddev start billing` on the host. Tell them exactly that command.
- If the sibling project has the auto-start hook configured, a host-side `ddev restart` of the sibling will trigger billing to start as well (the post-start hook runs `ddev start billing >/dev/null 2>&1 || true`).

**From the host:**
```bash
ddev list | grep '^| billing'              # status of billing
cd ~/workspace/proxiblue/billing && ddev start
# verify it came up
ddev describe -j 2>/dev/null | jq -r '.raw.urls[]' | head
```

After billing is running, retry the sibling check from inside its container:
```bash
getent hosts ddev-billing-web && \
  curl -fsS -o /dev/null -w "bridge HTTP %{http_code}\n" http://ddev-billing-web/internal/cli
```

### Mode 2: Sibling container can't see billing on the docker network

Symptom: billing IS running (verified on host), but inside the sibling container `getent hosts ddev-billing-web` returns nothing.

Cause: DDEV joins sibling projects onto a shared network only when both are up at start time. If billing was started AFTER the sibling, the sibling's network membership may not include billing.

Fix (on host):
```bash
cd ~/workspace/<sibling-project>
ddev restart
```

DDEV re-attaches networks on restart and the sibling will see `ddev-billing-web`.

### Mode 3: TOKEN_PRESENT, BRIDGE_REACHABLE, but bridge returns 401/403

Token mismatch — the file in the sibling container at `/etc/billing-bridge/token` is not the same value the billing app expects (in `/var/www/html/storage/bridge-token`, populated from the same host file).

Fix:
```bash
# Host — compare lengths
wc -c ~/.config/billing-bridge/token   # should be 65 (64 hex chars + newline)
# Sibling container
wc -c /etc/billing-bridge/token        # must match
# Billing container
cd ~/workspace/proxiblue/billing && ddev exec wc -c /var/www/html/storage/bridge-token
```

If they differ: re-mount may be stale. `ddev restart` the project whose token wc-c is off. As a last resort, regenerate the host token (`openssl rand -hex 32 > ~/.config/billing-bridge/token`) and restart **billing + every project that uses the bridge** so all mounts pick up the new value.

### Mode 4: Bridge reachable but `/internal/cli` 404s

Billing app's internal bridge route isn't loaded — usually means the app has crashed or supervisord didn't bring nginx + php-fpm up cleanly.

```bash
cd ~/workspace/proxiblue/billing
ddev logs -s web | tail -50
ddev exec composer mcp:status        # supervisord status, if applicable
ddev restart
```

### Quick one-liner: is billing healthy from where I am?

Inside a sibling container with the token mounted:
```bash
TOKEN=$(cat /etc/billing-bridge/token 2>/dev/null || echo MISSING)
[ "$TOKEN" = MISSING ] && echo "TOKEN_MISSING" && exit 1
curl -fsS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"script":"xero-ping","argv":[]}' \
  http://ddev-billing-web/internal/cli | jq -r '"exit=\(.exit_code) stderr=\(.stderr)"'
```

`exit=0` → bridge + xero auth both healthy. Any other result → walk Modes 1–4 above in order.

## Known issues

- Xero API `StartsWith` filter not supported — billing app's invoice number generation may fail on `create`. Needs fix in billing codebase (`XeroService.php` around line 154).
- `gh issue view --json closedBy` not supported — use timeline API instead to determine who closed a ticket.
