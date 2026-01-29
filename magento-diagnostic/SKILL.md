---
name: magento-diagnostic
description: Comprehensive Magento 2 system diagnostic skill that gathers cache status, index status, module information, configuration, logs, and performance metrics for rapid troubleshooting.
---

This skill automates comprehensive Magento 2 system diagnostics for troubleshooting and performance analysis.

## What This Skill Does

1. **System Status Check**
   - Deploy mode (developer/production/default)
   - Magento version information
   - PHP version and extensions
   - Database connection and version
   - File system permissions

2. **Cache Analysis**
   - Cache type status (enabled/disabled)
   - Cache backend configuration (Redis/File/Database)
   - Full Page Cache (FPC) status and backend
   - Cache hit/miss rates (if available)
   - Cache invalidation history

3. **Index Status**
   - All indexer status and mode (Update on Save/Schedule)
   - Index backlog and last update times
   - Index lock status
   - Changelog table sizes
   - Indexer performance metrics

4. **Module Information**
   - Enabled/disabled modules list
   - Module version information
   - Module dependencies and conflicts
   - Custom vs vendor modules
   - Recently updated modules

5. **Configuration Analysis**
   - Critical system configurations
   - Performance-related settings
   - Security configurations
   - Multi-store setup details
   - Third-party integration status

6. **Log Analysis**
   - Recent error log entries (var/log/system.log)
   - Exception log review (var/log/exception.log)
   - Debug log analysis (if enabled)
   - Web server error logs
   - Database slow query logs

7. **Performance Metrics**
   - Current system resource usage
   - Queue status and backlog
   - Cron job status and schedule
   - Recent cron execution logs
   - Database table sizes

## Bash Commands Used

```bash
bin/magento deploy:mode:show
bin/magento cache:status
bin/magento indexer:status
bin/magento module:status
bin/magento config:show
bin/magento cron:list
bin/magento queue:consumers:list
tail -n 100 var/log/system.log
tail -n 100 var/log/exception.log
```

## MCP Integration

Leverages:
- **magento2-dev MCP**: Magento-specific commands
- **database MCP**: Database query analysis
- **filesystem**: Log file reading

## Output

The skill provides:
- **Executive Summary**: High-level system health status
- **Critical Issues**: Immediate problems requiring attention
- **Warnings**: Potential issues to monitor
- **Performance Report**: System performance metrics
- **Recommendations**: Prioritized action items

## Report Format

1. **Health Score**: Overall system health (0-100)
2. **Issue Classification**:
   - 🔴 Critical: Immediate action required
   - 🟡 Warning: Monitor and plan resolution
   - 🟢 Info: System operating normally
3. **Detailed Findings**: Per-category analysis
4. **Action Plan**: Prioritized remediation steps

## When to Use

- Initial issue investigation
- Performance troubleshooting
- Before/after deployment validation
- Regular health check monitoring
- Customer-reported issues
- Proactive system maintenance
- Pre-upgrade assessment
