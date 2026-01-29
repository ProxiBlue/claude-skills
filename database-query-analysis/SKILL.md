---
name: database-query-analysis
description: Rapid database query execution skill for common Magento 2 data analysis tasks. Leverages database and magento2-dev MCP servers for catalog, order, configuration, and system data retrieval.
---

This skill provides efficient database query execution for Magento 2 data analysis.

## What This Skill Does

1. **Catalog Data Queries**
   - Product counts by type, status, visibility
   - Category hierarchy and product assignments
   - Inventory levels and stock status
   - Attribute usage and EAV data analysis
   - Price analysis and tier pricing

2. **Order & Sales Queries**
   - Order volume and revenue analysis by period
   - Payment method distribution and success rates
   - Shipping method usage and costs
   - Customer purchase patterns and lifetime value
   - Refund and credit memo analysis

3. **Configuration Queries**
   - System configuration by scope (default/website/store)
   - Module status and version information
   - Store view hierarchy and relationships
   - Admin user and role analysis
   - Cron job status and history

4. **Performance Queries**
   - Index status and update times
   - Cache tag analysis
   - Database table sizes and growth
   - Slow query identification
   - Customer session analysis

## MCP Integration

Utilizes:
- **database MCP**: Direct SQL query execution
- **magento2-dev MCP**: Magento-specific data retrieval functions
  - `mcp__magento2-dev__db-query`
  - `mcp__magento2-dev__sys-store-list`
  - `mcp__magento2-dev__config-show`

## Usage

When invoked, this skill can:

1. Execute predefined common queries for rapid insights
2. Build dynamic queries based on analysis requirements
3. Format results for business stakeholder reporting
4. Provide data export in CSV or JSON format
5. Generate comparative analysis across time periods

## Output

The skill provides:
- Structured data results with clear labeling
- Statistical summaries and aggregations
- Trend analysis and period comparisons
- Actionable insights and recommendations
- Executive summaries for business stakeholders

## When to Use

- Initial catalog health assessment
- Order performance analysis
- Configuration audits and validation
- Database optimization planning
- Business intelligence reporting
- Troubleshooting data-related issues
