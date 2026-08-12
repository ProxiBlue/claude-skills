---
name: magecli-skill
description: >
  Query a live Magento 2 store's REST API (catalog, orders, customers, CMS, config)
  from the command line — no SSH, no DB access, no local booted app needed.
disable-model-invocation: true
---

# magecli — live-store REST queries

`magecli` (atlanticbt/magecli, MIT, Go binary) queries any Magento 2 store's REST
API directly — product/category/order/customer/CMS/config lookups — using an
Integration Access Token. It complements, doesn't replace, the other Magento
tools already wired:

| Need | Reach for |
|---|---|
| What's *actually* in the catalog/orders on a **remote** env (staging, prod) right now | **magecli** — pure REST, no SSH/DB/booted-app needed |
| What a class/plugin/preference resolves to in **this container's** booted app | `bricklayer` MCP |
| Static structure / blast radius across the whole codebase | `pb-codegraph` MCP |
| Direct SQL against **this container's own** DB | `database` MCP (mariadb) |

Binary lives at the fleet-mounted `~/claude-skills-central/scripts/bin/magecli`,
visible in every DDEV container at `/var/www/html/.claude/scripts/bin/magecli`
(same live-mount as `bricklayer-mcp.sh` etc — installed 2026-08-12, checksum
verified against the GitHub release). Nothing to install per project.

## When to use it

Opt-in, per project, on demand — same convention as the billing-invoice skill.
Most projects never need this; reach for it only when a task genuinely needs
live REST data from a store you can't otherwise query (no SSH to prod, or the
question is about a *different* environment than the one you're sitting in).

## One-time setup (per store, run by the user — not the agent)

magecli needs a Magento Integration Access Token. **This step requires Admin
access and must be done by Lucas, not the agent** — creating the token is a
credential-issuing action.

1. Magento Admin → **System → Extensions → Integrations** → Add New Integration
2. Name it `magecli`, leave callback/identity URLs blank
3. API tab: scope resource access to what's needed — **Sales**/**Customers**
   scopes gate the `sales`/`customer` commands; skip them if the agent
   shouldn't see order/customer data (those commands then fail with a clear
   403, not a silent leak)
4. Save → Activate → confirm in the popup → copy the **Access Token** value
   (not Consumer Key/Secret)
5. Run inside the target container (or host, if it has network access to the
   store):
   ```bash
   /var/www/html/.claude/scripts/bin/magecli auth login https://store.example.com
   ```
   Prompts for the token at a hidden prompt — never pass `--token` interactively,
   it leaks into shell history. DDEV containers usually have no OS keyring; if
   `auth login` fails to find one, use the encrypted file-store fallback:
   ```bash
   /var/www/html/.claude/scripts/bin/magecli auth login https://store.example.com --allow-insecure-store
   ```

Once logged in, the token persists in that container's own storage — isolated
per project/container, no cross-project leakage since each DDEV container has
its own `$HOME`.

## Usage

Read-only by default. All commands support `--json`, `--yaml`, `--jq '<filter>'`,
`--filter "field op value"` (operators: eq, neq, gt, gteq, lt, lteq, like, nlike,
in, nin, null, notnull, from, to, finset), `--sort`, `--limit`.

```bash
magecli auth status                                          # confirm logged in / which store

magecli product list --filter "name like %shirt%" --json
magecli product view SKU123 --json --jq '.name'
magecli category tree --json
magecli inventory status SKU123 --json

magecli sales order list --filter "status eq processing" --limit 20   # needs Sales scope
magecli customer search --filter "email like %@example.com" --json   # needs Customers scope

magecli config get catalog/price/scope
magecli cms page list --json
```

`magecli api <method> <path>` is a raw REST escape hatch — read-only by default.
**Never pass `--allow-writes`** unless the user explicitly asked for a write
operation in this exact conversation turn; it's an irreversible-class action
(mutates the live store) and needs the same confirmation discipline as any
other destructive/hard-to-reverse action.

## Updating

```bash
/var/www/html/.claude/scripts/bin/magecli update
```

Downloads the latest release from GitHub, verifies the SHA-256 checksum,
replaces the binary in place. Run this on the host copy
(`~/claude-skills-central/scripts/bin/magecli`) so the update propagates to
every container via the shared mount — don't update per-container copies that
don't exist.
