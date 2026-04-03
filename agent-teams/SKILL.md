---
name: agent-teams
description: >-
  Spawn pre-configured teams of Claude Code agents for parallel Magento 2 development. Available teams: issue-resolution, feature-development, module-development, and audit, each with 4 specialized teammates. Use when the user wants to run parallel agents, assemble a dev team, resolve Magento issues with multiple agents, or coordinate parallel module development.
---

# Agent Teams Skill

## Overview
Spawns a coordinated team of Claude Code agents based on pre-built team templates optimized for Magento 2 development workflows. Each team has 4 specialized teammates with defined roles, file ownership boundaries, and coordination strategies.

## When to Use This Skill
- User asks to "swarm", "team up", or "use a team" for a task
- User invokes `/agent-teams` with a team type
- User asks for parallel development on an issue or feature
- Large tasks that benefit from multiple agents working simultaneously

## Prerequisites
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` must be set (already configured in settings)
- Team templates located in `.claude/teams/` (mounted from ~/claude-skills-central/teams/)

## Available Teams

### 1. `issue-resolution` - Bug Fix Team
**Use when**: Fixing a GitHub issue, debugging a bug, resolving a reported problem
**Teammates**: Investigator (opus) + Developer (sonnet) + Code Reviewer (opus) + Test Writer (sonnet)
**Flow**: Investigate → Fix → Review + Test in parallel → Done

### 2. `feature-development` - Feature Build Team
**Use when**: Adding new functionality to an existing module or the project
**Teammates**: Architect (opus) + Backend Dev (sonnet) + Frontend Dev (sonnet) + QA (opus)
**Flow**: Design → Backend + Frontend in parallel → QA → Done

### 3. `module-development` - New Module Team
**Use when**: Building a complete new Magento 2 module from scratch
**Teammates**: Module Architect (opus) + Backend Dev (sonnet) + Frontend/Config Dev (sonnet) + Quality (opus)
**Flow**: Architecture → Backend + Frontend in parallel → Quality verification → Done

### 4. `audit` - Audit & Fix Team
**Use when**: Reviewing code quality, performance, or security of existing code
**Teammates**: Performance Analyst (opus) + Security Analyst (opus) + Code Quality (sonnet) + Fix Implementer (sonnet)
**Flow**: All 3 analysts in parallel → Fix Implementer addresses critical findings → Done

## Execution Steps

### Step 1: Identify the Team Type
Parse the user's request to determine which team template to use. If unclear, ask.

### Step 2: Read the Team Template
Read the appropriate team template file:
- `.claude/teams/issue-resolution.md`
- `.claude/teams/feature-development.md`
- `.claude/teams/module-development.md`
- `.claude/teams/audit-team.md`

### Step 3: Collect Required Variables
Each template has placeholder variables. Gather these from the user or context:

**issue-resolution**:
- `{ISSUE_DESCRIPTION}` - The full issue description or GitHub issue number
- `{ISSUE_NUMBER}` - The GitHub issue number for branch detection

**feature-development**:
- `{FEATURE_DESCRIPTION}` - What the feature should do

**module-development**:
- `{MODULE_DESCRIPTION}` - What the module should do
- `{VENDOR_NAME}` - Vendor namespace (e.g., Uptactics, ProxiBlue, ITTools)
- `{MODULE_NAME}` - Module name (e.g., CustomShipping, ProductLabels)

**audit**:
- `{AUDIT_TARGET}` - Module path or scope to audit (e.g., "app/code/Uptactics/CustomModule" or "all custom modules")

### Step 4: Enter Delegate Mode
Press `Shift+Tab` to enter Delegate Mode. This prevents you (the lead) from writing code directly, keeping you focused on coordination.

### Step 5: Create the Task List
Before spawning teammates, create the initial task structure:
1. Create a parent task for the overall objective
2. Create sub-tasks for each major phase
3. Set up blockedBy relationships per the team template

### Step 6: Spawn Teammates
Spawn each teammate from the template with their personalized prompt (variables substituted).
Use the Task tool to spawn each one. Set the model per the template specification.

For teammates that need plan approval, spawn with `mode: plan`.

Example spawn pattern:
```
Spawn teammate "investigator" with model opus:
[Paste the Investigator spawn prompt with variables replaced]

Spawn teammate "developer" with model sonnet and require plan approval:
[Paste the Developer spawn prompt with variables replaced]

Spawn teammate "reviewer" with model opus:
[Paste the Reviewer spawn prompt with variables replaced]

Spawn teammate "test-writer" with model sonnet:
[Paste the Test Writer spawn prompt with variables replaced]
```

### Step 7: Monitor and Coordinate
As team lead:
1. Monitor task list progress (`Ctrl+T` to toggle task view)
2. Approve plans when developers request it
3. If a teammate is stuck, message them with additional context
4. When the Reviewer requests changes, ensure the Developer picks them up
5. Track file ownership - alert if teammates are editing outside their domain

### Step 8: Final Review
Once all tasks are complete:
1. Review the task list for any remaining items
2. Verify all tests pass
3. Run `bin/magento setup:di:compile` for a final DI check
4. Run `bin/magento cache:flush`
5. Create a summary of all changes made

## Tips for Effective Teams

### Token Management
Agent teams consume 4-15x more tokens than single sessions. To manage costs:
- Use sonnet for implementation teammates (cheaper, fast)
- Use opus for analysis/review teammates (smarter, more thorough)
- Keep spawn prompts focused - don't paste entire agent definitions

### File Conflict Prevention
The templates define file ownership boundaries. As lead, enforce these:
- If two teammates need to edit the same file, have one finish first
- Database schema files should only be edited by one teammate
- di.xml is a common conflict point - assign to Backend Developer only

### When to Use Single Agent Instead
Not every task needs a team:
- Simple bug fixes (single file change) → just fix it directly
- Config changes → single agent
- Small template updates → single agent
- Use teams for: multi-file changes, complex features, thorough audits
