---
name: status-page-monitoring
description: Check status pages for down monitors and alert immediately. Supports UptimeRobot, PayPal, and other status page providers via direct API fetch or MCP.
---

# Status Page Monitoring Skill

## Overview
This skill checks the status of all monitored services and alerts immediately if any monitor is showing DOWN status. Supports multiple status page providers via direct API endpoints or MCP tools.

## When to Use This Skill
- User runs `/status-page-monitoring`
- User asks "check status" or "are sites up?"
- User wants to know if any monitors are down
- User asks about uptime or service status

## Monitoring Workflow

### Step 1: Check Monitor Status

#### Option A: Direct API Fetch (Preferred when MCP unavailable)
Many status pages load data dynamically via JavaScript. If fetching the HTML page returns empty/loading state, try the API endpoint directly.

**Workflow:**
1. Try fetching the status page URL first
2. If data is missing/loading, use the known API pattern for that provider
3. Fetch the API endpoint directly to get JSON data

### Known API Patterns by Provider

#### UptimeRobot
```
Status page: https://stats.uptimerobot.com/{PAGE_ID}
API endpoint: https://stats.uptimerobot.com/api/getMonitorList/{PAGE_ID}
```
Example: `https://stats.uptimerobot.com/api/getMonitorList/Zk2EbUnM73`

#### PayPal Status
```
Status page: https://paypal-status.com/product/production
API endpoint: https://paypal-status.com/api/v1/components
```
Returns all PayPal services: Checkout, APIs, Account Management, Braintree, Venmo, etc.

#### Atlassian Statuspage (Generic)
Many services use Atlassian Statuspage. Try these endpoints:
```
/api/v2/summary.json
/api/v2/components.json
/api/v2/status.json
```

#### AWS Health Dashboard (Requires Playwright)
```
Status page: https://health.aws.amazon.com/health/status
API endpoint: NONE - JavaScript SPA, requires browser rendering
```
AWS Health Dashboard has no public API. Must use Playwright MCP (see Option C below).

**Key indicators in page snapshot:**
- `Open and recent issues (0)` = No current issues
- `No recent issues` heading = All systems operational
- Look for `disruption` or `degraded` in service rows

#### Unknown Provider
If provider is unknown, try these common patterns in order:
1. `/api/v1/components`
2. `/api/v2/components.json`
3. `/api/v2/summary.json`
4. `/api/status`

If all API patterns fail, use Playwright (Option C).

#### Option B: UptimeRobot MCP Tools
If MCP is available, get the schema then call the list-monitors tool:
```bash
mcp-cli info uptimerobot/list-monitors
mcp-cli call uptimerobot/list-monitors '{"limit": 100}'
```

#### Option C: Playwright MCP (For JavaScript-heavy pages)
Use Playwright when status pages require JavaScript rendering (e.g., AWS Health Dashboard).

**Workflow:**
```bash
# 1. Check schemas first
mcp-cli info playwright/browser_navigate
mcp-cli info playwright/browser_snapshot

# 2. Navigate to status page
mcp-cli call playwright/browser_navigate '{"url": "https://health.aws.amazon.com/health/status"}'

# 3. Wait for content to load
mcp-cli call playwright/browser_wait_for '{"time": 3}'

# 4. The snapshot is returned automatically - search for status indicators
# Look for: "Open and recent issues", "disruption", "degraded", "No recent issues"

# 5. Close browser when done
mcp-cli call playwright/browser_close '{}'
```

**Note:** If Playwright browser is not installed, run:
```bash
npx playwright install chrome
```

### Step 2: Identify Down Monitors
From the JSON response, look for monitors with `"status": "DOWN"`.

Extract for each monitor:
- Monitor name
- Monitor URL
- Current status
- How long it's been in current state (`currentStateDuration`)
- Last incident ID

### Step 3: Alert Format

#### If ALL monitors are UP:
```
STATUS: ALL SYSTEMS OPERATIONAL

[count] monitors checked - all UP
[count] monitors PAUSED (intentional)
Last check: [timestamp]
```

#### If ANY monitor is DOWN:
```
ALERT: MONITOR(S) DOWN

DOWN:
  - [Monitor Name] - [URL]
    Status: DOWN for [duration]
    Incident ID: [id]

UP: [count] monitors operational
PAUSED: [count] monitors paused

Recommended Actions:
1. Check the affected URL directly
2. SSH to server and check services
3. Review server logs
```

### Step 4: Quick Status Filters
To check only DOWN monitors:
```bash
mcp-cli call uptimerobot/list-monitors '{"filter": ["DOWN"], "limit": 100}'
```

## Alert Severity Levels

| Status | Severity | Action |
|--------|----------|--------|
| DOWN | CRITICAL | Immediate alert, investigate now |
| PAUSED | INFO | Informational, monitor intentionally stopped |
| UP | OK | No action needed |

## Example Output

### All Systems OK
```
STATUS CHECK COMPLETE

All 22 monitors are UP

Sites monitored:
- NTO Tank (ntotank.com)
- Protank (protank.com)
- PVC Pipe Supplies (pvcpipesupplies.com)
- IBC Tanks (ibctanks.com)

1 monitor PAUSED (Protank & Equipment - intentional)

No action required.
```

### System Down
```
CRITICAL ALERT

1 MONITOR DOWN:

  NTO Tank
  URL: https://www.ntotank.com
  Status: DOWN
  Duration: 5 minutes
  Incident: #269634131363671187

IMMEDIATE ACTIONS:
1. Check https://www.ntotank.com directly
2. SSH to server and check services
3. Review error logs
4. Check DNS/SSL status

21 other monitors operational.
```

## Claude Code Status Bar Integration

Display monitor status directly in the Claude Code status bar for always-visible monitoring.

### How It Works

```
┌─────────────────────────────────────────────────────────────┐
│ [Opus] $0.12 | 21 OK                    <- All systems up   │
│ [Opus] $0.12 | 21 1 (NTO HomePage)      <- 1 monitor down   │
│ [Opus] $0.12 | ? stale                  <- Cache outdated   │
└─────────────────────────────────────────────────────────────┘
```

**Architecture:**
1. **Cron job** runs periodically, invokes Claude CLI with this skill
2. **Claude** uses the skill to fetch and parse status from any supported provider
3. **Bash script** captures Claude's JSON output, writes to cache file
4. **Status bar script** reads cache file, displays in Claude Code status line

This approach leverages Claude's ability to parse multiple status page providers using the skill's documented patterns.

### Setup Instructions

#### Step 1: Scripts Location

The scripts are located in your project:
```
.claude/scripts/status-monitor-cron.sh   # Cron job - fetches & caches status
.claude/scripts/statusline-with-monitors.sh  # Status bar - displays cached status
```

#### Step 2: Configure Cron Job

Add to crontab to run the status check periodically:
```bash
crontab -e
```

Add this line (adjust path to your project):
```bash
# Every 5 minutes (recommended - balances freshness vs API costs)
*/5 * * * * /var/www/html/.claude/scripts/status-monitor-cron.sh

# Every 15 minutes (lower cost)
*/15 * * * * /var/www/html/.claude/scripts/status-monitor-cron.sh
```

**Note:** This invokes Claude CLI which uses API credits. Choose interval based on:
- How critical real-time status is for you
- Your API usage budget
- The skill handles any provider (UptimeRobot, PayPal, AWS, etc.) automatically

> **⚠️ COST ALERT: Review the cost estimates below before configuring cron frequency!**

### Cron Job Cost Estimates

Running this skill via cron invokes Claude CLI, which consumes API credits. Plan your monitoring frequency accordingly.

#### Token Usage Per Check (Estimated)

| Component              | Tokens       |
|------------------------|--------------|
| Prompt + skill context | ~2,000-4,000 |
| WebFetch tool call     | ~100         |
| WebFetch result        | ~300-500     |
| JSON output            | ~50          |
| **Total**              | **~3,000-5,000** |

#### Cost Per Check (Claude Sonnet Pricing)

| Type   | Tokens | Rate   | Cost    |
|--------|--------|--------|---------|
| Input  | ~4,000 | $3/1M  | $0.012  |
| Output | ~200   | $15/1M | $0.003  |
| **Total** |     |        | **~$0.015** |

#### Daily/Monthly Cost by Interval

| Interval     | Checks/day | Cost/day | Cost/month |
|--------------|------------|----------|------------|
| Every 1 min  | 1,440      | ~$21.60  | ~$648      |
| Every 5 min  | 288        | ~$4.32   | ~$130      |
| Every 15 min | 96         | ~$1.44   | ~$43       |
| Every 30 min | 48         | ~$0.72   | ~$22       |
| Every 1 hour | 24         | ~$0.36   | ~$11       |

#### Recommendations

| Use Case | Recommended Interval | Estimated Cost |
|----------|---------------------|----------------|
| Production-critical sites | Every 5-15 min | $1.50-4.50/day |
| General awareness | Every 15-30 min | $0.70-1.50/day |
| Low-priority/dev sites | Every 30-60 min | $0.35-0.70/day |

**Cost-saving tips:**
- Use longer intervals (15-30 min) for non-critical monitoring
- Consider using free UptimeRobot email alerts as primary notification
- Use this cron integration for status bar visibility, not primary alerting
- Playwright-based checks (AWS, etc.) cost more due to additional tool calls

#### Step 3: Configure Claude Code Status Line

Add to your Claude Code settings file.

**Project-level** (`.claude/settings.local.json`):
```json
{
  "statusLine": {
    "type": "command",
    "command": "/var/www/html/.claude/scripts/statusline-with-monitors.sh"
  }
}
```

**Or user-level** (`~/.claude/settings.json`):
```json
{
  "statusLine": {
    "type": "command",
    "command": "/path/to/your/project/.claude/scripts/statusline-with-monitors.sh"
  }
}
```

#### Step 4: Verify Setup

1. **Test cron script manually** (this will invoke Claude CLI):
```bash
.claude/scripts/status-monitor-cron.sh
cat /tmp/monitor-status.json
```

Expected output:
```json
{"up": 21, "down": 1, "paused": 1, "down_names": "NTO HomePage", "timestamp": "..."}
```

2. **Test status line script:**
```bash
echo '{"model":{"display_name":"Test"},"cost":{"total_cost_usd":0.05}}' | .claude/scripts/statusline-with-monitors.sh
```

3. **Restart Claude Code** to load the new status line configuration.

### Status Bar Display

| Display | Meaning |
|---------|---------|
| `21 OK` | All 21 monitors are UP (green) |
| `21 1 (Name)` | 21 UP, 1 DOWN - shows name of down monitor |
| `21 3` | 21 UP, 3 DOWN (red) |
| `? stale` | Cache file older than 5 minutes (yellow) |
| `! error` | API fetch failed (yellow) |
| `-` | No cache file found |

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `STATUS_CACHE_FILE` | `/tmp/monitor-status.json` | Where to store cached status |
| `CLAUDE_CMD` | `claude` | Path to Claude CLI executable |

### Cache File Format

The cron job writes JSON to the cache file:
```json
{
    "up": 21,
    "down": 1,
    "paused": 1,
    "down_names": "NTO HomePage",
    "timestamp": "2026-01-15T22:30:00+00:00",
    "error": false
}
```

### Troubleshooting

**Status bar shows `-`:**
- Cron job hasn't run yet, or cache file doesn't exist
- Run manually: `.claude/scripts/status-monitor-cron.sh`
- Check cache: `cat /tmp/monitor-status.json`

**Status bar shows `? stale`:**
- Cron job not running or failing
- Check cron logs: `grep CRON /var/log/syslog`
- Verify crontab: `crontab -l`

**Status bar shows `! error`:**
- Claude CLI failed or returned invalid output
- Test manually: `.claude/scripts/status-monitor-cron.sh && cat /tmp/monitor-status.json`
- Check Claude CLI is installed: `which claude`
- Check Claude CLI auth: `claude --version`

**Status bar not updating:**
- Claude Code status line updates every 300ms max
- Restart Claude Code if settings changed
- Check script is executable: `chmod +x .claude/scripts/statusline-with-monitors.sh`

**Claude CLI not found:**
- Ensure Claude CLI is in PATH for cron
- Use full path in cron: `CLAUDE_CMD="/path/to/claude" /path/to/status-monitor-cron.sh`

**jq not found:**
- Install jq: `apt install jq` or `brew install jq`

### Customization

**Change cache location:**
```bash
# In crontab
*/5 * * * * STATUS_CACHE_FILE="/custom/path/status.json" /path/to/status-monitor-cron.sh
```

**Specify Claude CLI path:**
```bash
# In crontab (if claude not in PATH)
*/5 * * * * CLAUDE_CMD="/usr/local/bin/claude" /path/to/status-monitor-cron.sh
```

**Extend status bar script** to show additional info by editing `.claude/scripts/statusline-with-monitors.sh`.

## Success Criteria

- Monitor status always visible in status bar
- DOWN monitors highlighted in red
- No latency impact (reads from cache, not API)
- Stale/error states clearly indicated
- Works across Claude Code sessions
