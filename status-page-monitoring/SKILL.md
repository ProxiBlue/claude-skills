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

## Background Monitoring Script

A companion background monitoring script is available at:
```
.claude/scripts/status-monitor.sh
```

### Usage
```bash
# Run once
.claude/scripts/status-monitor.sh

# Run continuously (checks every 60 seconds)
.claude/scripts/status-monitor.sh --loop

# Run continuously with custom interval (30 seconds)
.claude/scripts/status-monitor.sh --loop 30
```

### Environment Variables
```bash
# Required: Set your status page URL
export STATUS_PAGE="https://stats.uptimerobot.com/your-page-id"
# Or for PayPal:
export STATUS_PAGE="https://paypal-status.com/product/production"
# Or for AWS (requires Playwright):
export STATUS_PAGE="https://health.aws.amazon.com/health/status"

# Optional: Check interval in seconds (default: 60)
export STATUS_CHECK_INTERVAL=60

# Optional: Play sound on alert (default: true)
export STATUS_ALERT_SOUND=true

# Optional: Send desktop notification (default: true)
export STATUS_NOTIFY=true
```

### Running in Background
```bash
# Run in tmux session
tmux new-session -d -s status-monitor '.claude/scripts/status-monitor.sh --loop'

# Attach to see output
tmux attach -t status-monitor

# Detach: Ctrl+B, then D
```

## Claude Code Hook Integration

Automatically alert Claude when monitors are down at conversation start.

### Setup Instructions

#### Step 1: Copy Scripts to Your Project

Copy these files to your project's `.claude/scripts/` directory:
```bash
mkdir -p .claude/scripts

# Copy the scripts (adjust source path as needed)
cp path/to/status-monitor.sh .claude/scripts/
cp path/to/status-check-hook.sh .claude/scripts/

# Make executable
chmod +x .claude/scripts/status-monitor.sh
chmod +x .claude/scripts/status-check-hook.sh
```

#### Step 2: Set Environment Variable

Add to your shell profile (`~/.bashrc`, `~/.zshrc`, etc.):
```bash
export STATUS_PAGE="https://stats.uptimerobot.com/your-page-id"
```

Or for other providers:
```bash
# PayPal
export STATUS_PAGE="https://paypal-status.com/product/production"

# Google Workspace
export STATUS_PAGE="https://www.google.com/appsstatus/dashboard/"
```

#### Step 3: Configure Claude Code Hook

Add to your `.claude/settings.local.json` (or global `~/.claude/settings.json`):
```json
{
  "hooks": {
    "user-prompt-submit": [
      {
        "command": ".claude/scripts/status-check-hook.sh",
        "timeout": 5000
      }
    ]
  }
}
```

#### Step 4: Verify Setup

Start a new Claude Code conversation. If any monitors are down, you'll see:
```
==========================================
STATUS ALERT: 1 monitor(s) DOWN
Down: NTO HomePage
Status page: https://stats.uptimerobot.com/Zk2EbUnM73
==========================================
```

### How It Works

1. **On each prompt submit**, the hook script runs
2. **Fetches status** from the configured status page API
3. **If monitors are down**, outputs an alert message
4. **Claude sees the alert** and can proactively inform you or investigate

### Supported Providers

| Provider | API Auto-Detection | Requires |
|----------|-------------------|----------|
| UptimeRobot | Yes | `jq` for accurate parsing |
| PayPal | Yes | - |
| Google Workspace | Yes | - |
| AWS Health | No | Playwright (JavaScript SPA) |

### Troubleshooting

**Hook not running:**
- Check script is executable: `chmod +x .claude/scripts/status-check-hook.sh`
- Verify `STATUS_PAGE` is set: `echo $STATUS_PAGE`
- Test manually: `.claude/scripts/status-check-hook.sh`

**No output even when monitors are down:**
- Ensure `jq` is installed for UptimeRobot: `apt install jq` or `brew install jq`
- Check API endpoint works: `curl -s "$(get_api_url $STATUS_PAGE)" | head`

**AWS not working:**
- AWS requires Playwright (JavaScript SPA) - hook doesn't support it
- Use the skill manually: `/status-page-monitoring`

## Success Criteria

- All monitors checked quickly
- DOWN status clearly highlighted
- Actionable next steps provided
- No false positives
- Historical context included (incident duration)
