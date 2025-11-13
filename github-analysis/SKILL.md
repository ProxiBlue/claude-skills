---
name: github-analysis
description: Systematically analyze GitHub tickets with proper local reproduction. Automatically substitutes production URLs with local DDEV hosts, ensures database sync, downloads missing assets, and reproduces issues locally before proposing solutions.
---

# GitHub Ticket Analysis Skill

## Overview
This skill provides a comprehensive, systematic approach to analyzing GitHub tickets. It ensures issues are properly reproduced in the local development environment before proposing solutions, preventing wasted effort on cache issues, environment differences, or missing data.

## When to Use This Skill
- User asks to "analyze", "debug", "fix", or "investigate" a GitHub ticket/issue
- User provides a GitHub issue number for investigation
- User asks about a specific bug reported in GitHub
- User requests help with reproducing an issue from a ticket

## Critical First Principle: Always Reproduce Locally
**NEVER propose a solution without first reproducing the issue in the local environment.**

Many apparent bugs are actually:
- Cache issues (clear cache and retry)
- Database differences between environments
- Missing media files
- Environment configuration differences

## GitHub Ticket Analysis Workflow

### Step 1: Read the GitHub Ticket
```bash
# Use the GitHub MCP tools to read the issue
# Capture: title, description, screenshots, URLs, expected vs actual behavior
```

**Extract from the ticket:**
- Issue title and description
- Any URLs mentioned (production/staging/UAT)
- Screenshots or images attached
- Expected behavior vs actual behavior
- Steps to reproduce (if provided)
- Browser/environment details (if mentioned)

### Step 2: Substitute Production URLs with Local DDEV Host
**CRITICAL RULE: Always convert production URLs to local equivalents**

**URL Substitution Patterns:**
```
Production:  https://pvcpipesupplies.com/...
Local DDEV:  https://pvcpipesupplies.ddev.site/...

Production:  https://www.domain.com/...
Local DDEV:  https://{project}.ddev.site/...

Production:  http://domain.com/...
Local DDEV:  https://{project}.ddev.site/...
```

**How to determine DDEV host:**
```bash
# Check DDEV configuration
ddev describe | grep -i "primary url"

# Or read from .ddev/config.yaml
grep "^name:" .ddev/config.yaml
# The URL will be: https://{name}.ddev.site
```

**Example:**
```
Ticket says: "Bug on https://pvcpipesupplies.com/product/test.html"
You test:     https://pvcpipesupplies.ddev.site/product/test.html
```

### Step 3: Verify Local Environment is Running
```bash
# Ensure DDEV is running
ddev describe

# If not running:
ddev start

# Verify services are healthy
ddev exec php -v
ddev exec mysql -e "SELECT 1"
```

### Step 4: Test the URL Locally and Compare with Live
**First attempt: Test with current local data**

1. Open the local URL in your mental model (via WebFetch if needed)
2. Fetch the same URL from production
3. Compare the content/behavior

**If content differs significantly:**
```
STOP: Database sync required

The local environment may have stale data.
Proceed to Step 5 to pull fresh database from production.
```

**If content is similar:**
```
Continue testing - local data appears current
```

### Step 5: Pull Fresh Database from Production (When Needed)
**When to pull database:**
- Local content differs from production
- Ticket mentions specific products/customers/orders not in local DB
- Last DB pull was more than a week ago
- Ticket involves catalog, customer, or order data

**How to pull database:**
```bash
# Project-agnostic path resolution
SNAPSHOT_SCRIPT="./snapshot_live_filtered.sh"

# Check if script exists
if [ -f "$SNAPSHOT_SCRIPT" ]; then
    echo "Found database sync script at project root"

    # Inform user this will take several minutes
    echo "⚠️  This will pull fresh production database (5-10 minutes)"
    echo "⚠️  Your local database will be replaced"

    # Execute the snapshot script
    bash "$SNAPSHOT_SCRIPT"

    # After import, flush caches
    ddev exec bin/magento cache:flush
    ddev exec bin/magento indexer:reindex
else
    echo "❌ Database sync script not found at project root"
    echo "Ask user how to pull production database"
fi
```

**What the snapshot script does:**
1. Connects to production server via SSH
2. Creates filtered database dump (excludes logs, sessions, temp tables)
3. Downloads dump to local machine
4. Drops local database
5. Imports production dump
6. Creates local admin user
7. Generates admin token

**After database import:**
```bash
# Clear all caches
ddev exec bin/magento cache:flush

# Reindex if needed
ddev exec bin/magento indexer:reindex

# Verify data
ddev exec bin/magento admin:user:list
```

**Then retry the URL and verify content matches production**

### Step 6: Download Missing Media Assets (If Needed)
**When to download media:**
- Product images return 404
- Screenshots show images that don't display locally
- Media-heavy pages have broken images

**How to download missing images:**

**Option A: Download specific image**
```bash
# Determine production URL
PROD_URL="https://production-site.com"

# Determine image path from HTML/error
IMAGE_PATH="/media/catalog/product/x/y/xyz.jpg"

# Download to correct local path
LOCAL_MEDIA="pub/media"
mkdir -p "$(dirname "$LOCAL_MEDIA${IMAGE_PATH#/media}")"

# Use wget or curl to download
wget -P "$(dirname "$LOCAL_MEDIA${IMAGE_PATH#/media}")" \
     "${PROD_URL}${IMAGE_PATH}"

# Or use curl
curl -o "${LOCAL_MEDIA}${IMAGE_PATH#/media}" \
     "${PROD_URL}${IMAGE_PATH}"
```

**Option B: Sync entire media directory (large operation)**
```bash
# If many images are missing, consider rsync from production
# This requires SSH access to production

# Example (project-agnostic):
rsync -avz --progress \
      production-server:/path/to/production/pub/media/catalog/product/ \
      ./pub/media/catalog/product/
```

**Option C: Download via WebFetch**
```bash
# For specific images, use WebFetch tool to access the production URL
# Then save the image content to the correct local path
```

**Verify media exists:**
```bash
# Check if image exists locally
ls -la pub/media/catalog/product/path/to/image.jpg

# Verify permissions
chmod 644 pub/media/catalog/product/path/to/image.jpg
```

### Step 7: Reproduce the Issue Locally
**Now that environment is synced, reproduce the exact issue:**

1. **Follow ticket reproduction steps exactly**
   - Use the same URL (converted to local)
   - Use the same browser actions
   - Use the same user role if specified

2. **Document what you observe**
   - Does the issue reproduce? (Yes/No)
   - What is the actual behavior?
   - Any console errors?
   - Any PHP errors in logs?

3. **If issue does NOT reproduce locally:**
   ```
   Possible reasons:
   - Cache issue (most common!)
   - Environment-specific configuration
   - Production-only modules/settings
   - External service integration

   Action: Report back to user that issue does not reproduce locally
   Suggest: Clear cache on production and retry
   ```

4. **If issue DOES reproduce locally:**
   ```
   Great! Now you can investigate the root cause
   Proceed to Step 8
   ```

### Step 8: Investigate Root Cause
**Only after reproducing locally, begin investigation:**

1. **Check browser console for errors**
   ```bash
   # If WebFetch used, look for JavaScript errors in output
   # If testing manually, check browser developer tools
   ```

2. **Check PHP error logs**
   ```bash
   # System logs
   ddev exec tail -f var/log/system.log

   # Exception logs
   ddev exec tail -f var/log/exception.log

   # Debug logs (if enabled)
   ddev exec tail -f var/log/debug.log
   ```

3. **Identify the code path**
   - Find templates involved
   - Find controllers/blocks/viewmodels
   - Find relevant configuration

4. **Use debugging tools**
   ```bash
   # Enable Xdebug if needed
   ddev xdebug on

   # Check configuration
   ddev exec bin/magento config:show | grep -i "relevant_section"

   # Check module status
   ddev exec bin/magento module:status | grep -i "ModuleName"
   ```

### Step 9: Test the Fix Locally
**Before proposing solution, test it works:**

1. **Implement the fix in local environment**
2. **Clear caches**
   ```bash
   ddev exec bin/magento cache:flush
   ```
3. **Test the original reproduction steps**
4. **Verify issue is resolved**
5. **Test related functionality (regression testing)**

### Step 10: Document the Solution
**Provide comprehensive solution documentation:**

```markdown
## Issue Analysis: [Ticket Number]

### Issue Reproduced: [Yes/No]

### Root Cause:
[Explain what was wrong]

### Solution:
[Explain the fix]

### Files Modified:
- path/to/file1.php:123
- path/to/file2.phtml:45

### Testing Performed:
- [X] Original issue resolved
- [X] No console errors
- [X] Related features still work
- [X] Caches cleared and retested

### Deployment Notes:
[Any special deployment considerations]

### Regression Risk:
[Low/Medium/High - explain why]
```

## Common Pitfalls to Avoid

### ❌ DON'T: Analyze Without Reproducing
**Bad:** Read ticket → Guess at solution → Propose fix
**Good:** Read ticket → Reproduce locally → Investigate → Test fix → Propose solution

### ❌ DON'T: Use Production URLs in Testing
**Bad:** Test `https://production.com/page.html`
**Good:** Test `https://project.ddev.site/page.html`

### ❌ DON'T: Skip Database Sync When Needed
**Bad:** "Can't find product in local" → Give up
**Good:** "Can't find product" → Pull fresh DB → Continue

### ❌ DON'T: Ignore Cache as Potential Cause
**Bad:** Immediately dive into code analysis
**Good:** First reproduce, then check if cache clear resolves it

### ❌ DON'T: Propose Solutions for Non-Reproducible Issues
**Bad:** Can't reproduce but suggest code changes anyway
**Good:** Can't reproduce → Report this → Suggest cache clear on production

## Project-Agnostic Path Patterns

**Use these patterns to work in any project:**

```bash
# DDEV config
CONFIG_FILE="./.ddev/config.yaml"

# Database snapshot script (at project root)
DB_SNAPSHOT="./snapshot_live_filtered.sh"

# Magento directories
MAGENTO_ROOT="."
BIN_MAGENTO="./bin/magento"
PUB_MEDIA="./pub/media"
VAR_LOG="./var/log"

# Custom code locations
APP_CODE="./app/code"
APP_DESIGN="./app/design"

# Execute Magento CLI
ddev exec bin/magento [command]

# Check logs
ddev exec tail -f var/log/system.log
```

## Quick Reference: Common Commands

```bash
# Start environment
ddev start

# Get DDEV URL
ddev describe | grep "primary url"

# Pull production database
bash ./snapshot_live_filtered.sh

# Clear caches
ddev exec bin/magento cache:flush

# Check logs
ddev exec tail -100 var/log/system.log
ddev exec tail -100 var/log/exception.log

# Download missing image
IMAGE_URL="https://production.com/media/path/to/image.jpg"
curl -o pub/media/path/to/image.jpg "$IMAGE_URL"

# Enable Xdebug
ddev xdebug on

# Check module status
ddev exec bin/magento module:status
```

## Example Workflow

**Ticket: "Product page shows wrong price"**

1. ✅ Read GitHub issue #123
2. ✅ Extract URL: `https://production.com/product/abc.html`
3. ✅ Convert to local: `https://project.ddev.site/product/abc.html`
4. ✅ Test local URL → Shows old price ($10 instead of $15)
5. ✅ Test production URL → Shows correct price ($15)
6. ✅ **Database sync needed** → Run `./snapshot_live_filtered.sh`
7. ✅ Retest local URL → Now shows $15 ✓
8. ✅ Issue does NOT reproduce with fresh data
9. ✅ **Conclusion: Stale local data, not a bug**
10. ✅ Recommend: Clear cache on production and verify

**Result: Prevented wasted debugging effort on non-existent bug**

## Environment-Specific Notes

**For Magento 2 / Mage-OS projects:**
- Database snapshot script uses `n98-magerun2`
- Admin user creation included in snapshot script
- Cache flush required after DB import
- Indexing may be required for catalog changes

**For Hyvä Themes projects:**
- Static content may need regeneration
- Tailwind CSS may need rebuild
- ViewModels may have cached data

**For DDEV environments:**
- Always use `ddev exec` prefix for commands
- DDEV hostname format: `{project-name}.ddev.site`
- Services accessible: web, db, mailhog, etc.

## Success Criteria

✅ **Issue was reproduced in local environment**
✅ **Root cause identified with evidence**
✅ **Fix implemented and tested locally**
✅ **Related functionality verified (no regressions)**
✅ **Solution documented with file paths and line numbers**
✅ **Deployment steps clearly outlined**

## When Not to Use This Skill

- Production-only issues (no local access)
- Infrastructure issues (server, networking, DNS)
- Third-party service issues (payment gateway, shipping API)
- Issues already confirmed as cache-related by user

In these cases, document the limitations and work with available information.
