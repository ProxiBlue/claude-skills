# proxiblue-skills

Specialized skill library for Magento 2 / Mage-OS / Hyvä Themes development, plus general-purpose audit, billing, and diagnostic skills. Authored during an ongoing M1→M2 migration project and refined across multiple DDEV environments.

Skills are reusable, context-aware prompt packs that guide Claude through complex, repeatable tasks. Each skill is a directory containing a `SKILL.md` file with instructions, examples, and rules.

## How these skills get loaded

**These skills are NOT mounted into projects by name.** They are delivered as a **Claude Code plugin** through the standard plugin layer:

- Source on host: `~/claude-plugins-central/seed/marketplaces/proxiblue-skills/skills/`
- Bind-mounted **read-only** into every DDEV container via the existing `plugins-seed` mount at `/var/www/html/.claude/plugins-seed:ro`
- Auto-discovered by Claude Code as plugin `proxiblue-skills@proxiblue-skills` (enabled by default in `~/claude-skills-central/settings.json`)

No per-project mount, no `.claude/skills:ro` overlay. Project-local `.claude/skills/` stays writable.

### Editing skills

The host directory `~/claude-skills-central/skills` is a **symlink** to this plugin's `skills/` directory. Editing `~/claude-skills-central/skills/<skill>/SKILL.md` writes through to the plugin's real location. The existing edit workflow is preserved; the symlink only matters for path resolution.

The plugin's `skills/` is its own git repo (history preserved from the original `claude-skills-central/skills` repo). Commit + push edits from there.

### Per-project opt-out

To disable this plugin in a specific project, override `enabledPlugins.proxiblue-skills@proxiblue-skills` to `false` in that project's settings.json. The plugin layer is always RO by design — no mount surgery needed.

## Skills in this plugin

### Magento / Hyvä development

- `analyze-m1-module-for-migration` — assess legacy M1 modules for M2 migration
- `create-backend-controller` — scaffold an admin (adminhtml) controller with ACL, routing, authorization
- `create-frontend-controller` — scaffold a storefront controller
- `hyva-module-compatibility` — fix M2 module compatibility with Hyvä (block plugins, KO→Alpine, ViewModels)
- `hyva-tailwind-integration` — Tailwind + JS integration in Hyvä themes
- `magento-controller-refactor` — modernize deprecated `Action` controllers to HTTP-verb interfaces (PHP 8.3+)
- `magento2-widget-creation` — build CMS-insertable widget modules
- `page-banner-setup` — page-title templates with/without banner backgrounds
- `email-theme-styling` — Magento transactional email theming

### Audit, diagnostic, and quality

- `audit-loop` — repeated review / fix-up loop on a changeset
- `cache-diagnostic` — diagnose Magento cache misses / corruption
- `code-quality-audit` — PHPCS / PHPStan / PHPMD sweep with prioritized fixes
- `magento-diagnostic` — broad project health diagnostic
- `database-query-analysis` — slow-query / EXPLAIN inspection via MCP
- `security-scan` — basic security review of pending changes
- `server-scan` — host-level health/audit

### Workflow / orchestration

- `agent-teams` — multi-agent orchestration patterns
- `caveman` — terse-output mode for the current session
- `github-analysis` — systematic ticket reproduction (URL substitution, DB sync, asset download)
- `gold-buy-signal` — "should I buy gold now" yes/no + reason, consolidated from 3 public prediction sites via a deterministic host engine (`~/.config/gold-signal/`)
- `publish-test-reports` — publish Playwright reports to per-client GitHub Pages
- `status-page-monitoring` — wire up UptimeRobot / external status feeds
- `sync-vector-db` — refresh context-please / vector index after schema change
- `wiki-docs` — capture custom site functionality in project wiki

### Deployment

- `deploy-check` — sanctioned pre-push gate for `live` (and other deploy branches): tests, type-check, diff review, explicit confirmation before push
- `uat-deploy-verify` — reproduce-push-verify cycle for shipping a fix to `uat`: confirm bug reproduces on UAT, push, confirm the Buddy.works deploy actually ran (via `buddy-mcp`), rerun tests to confirm the fix

### Billing

- `billing-invoice` — Xero invoicing addendum (rules, AI-billing format, deploy-status checks)

## How to use a skill

### Direct invocation

```
Use the hyva-tailwind-integration skill to add custom button styles
```

### Implicit invocation

Claude auto-invokes a skill when the conversation matches its `description` frontmatter.

### Check what's loaded

In a Claude session:
```
list skills
```

## Adding a new skill

1. `mkdir ~/claude-skills-central/skills/my-new-skill` (this resolves through the symlink into the plugin)
2. Create `SKILL.md` with frontmatter:
   ```markdown
   ---
   name: my-new-skill
   description: One-line on when this skill should activate.
   ---
   ```
3. Commit in the plugin's `skills/` git repo.
4. `ddev restart` is **not** required — Claude Code picks up new skills on next session start. (Existing sessions need a restart.)

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Skill doesn't appear in `list skills` | Plugin not enabled in current project | Confirm `proxiblue-skills@proxiblue-skills: true` in settings.json (central or project) |
| Skill loads but doesn't trigger | `description` frontmatter too narrow | Edit the description to cover the user's likely phrasings |
| Edit-in-place not reflected | Path is going through stale path cache | Restart the Claude session; symlink resolution happens at session start |
| Permission error writing into `.claude/skills/` | Some other RO mount overlapping that path | Check `docker-compose.ai.mounts.yaml` for stray `.claude/skills:ro` lines |

## Notes

- Skills are not Magento-specific by file structure — they're just markdown packs. Mix Magento and non-Magento freely under this plugin.
- Some skills reference paths or projects from the dev environment they were authored in (e.g. NTOTanks). Treat those as examples, not gospel.
- Work in progress. PRs / suggestions welcome via the git repo this plugin's `skills/` directory tracks.
