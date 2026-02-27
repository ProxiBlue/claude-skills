---
name: audit-loop
description: Iterative audit-fix loop using Ralph Wiggum. Spawns parallel performance, security, and code quality audits on currently edited files, fixes Critical/High issues, and re-audits until clean. Invoke with /audit-loop.
---

# Audit Loop Skill

Automated iterative audit-fix cycle using the Ralph Wiggum loop plugin. Detects your currently edited files via `git diff`, spawns 3 parallel audit agents (performance, security, code quality), fixes any Critical or High findings, then re-audits until clean.

## When to Use

- After implementing a feature or refactoring, before committing
- When you want hands-off audit-fix iteration
- Invoke with: `/audit-loop`

## Execution Steps

### Step 1: Detect Changed Files

Run `git diff --name-only HEAD` and `git diff --name-only --cached` to get all modified and staged files. Filter to only `.php`, `.phtml`, and `.xml` files (auditable code). If no changed files are found, inform the user and stop.

### Step 2: Start the Ralph Loop

Use the `/ralph-loop` command with the following prompt template. Replace `{FILE_LIST}` with the actual file list from Step 1.

```
/ralph-loop "Audit the following recently edited files for performance, security, and code quality issues.

## Files to Audit
{FILE_LIST}

## Process

1. **Detect files**: Run `git diff --name-only HEAD` and `git diff --name-only --cached` to get current changed files (only .php, .phtml, .xml). Use these as the audit target.

2. **Run 3 parallel audits** using the Task tool:
   - **Performance** (subagent_type: magento-performance-analyst, model: opus): Check for N+1 queries, redundant DB loads, missing collection attribute preloading, cache concerns, memory issues.
   - **Security** (subagent_type: magento-security-analyst, model: opus): Check for XSS, CSS/SQL injection, output escaping, input validation, CSRF.
   - **Code Quality** (subagent_type: magento-code-reviewer, model: opus): Check PSR-12, type hints, PHPDoc, use statements, Magento patterns. Do NOT flag missing copyright headers (project standard prohibits them).

3. **Collect findings** with severity: Critical / High / Medium / Low.

4. **If Critical or High findings exist**:
   - Fix them in the code immediately
   - Run `bin/magento setup:di:compile` if any PHP files were changed, to verify compilation
   - Run `bin/magento cache:flush`
   - Exit (Ralph loop will feed this prompt back for re-audit)

5. **If NO Critical or High findings remain**:
   - Print the final audit summary table
   - Output: <promise>AUDIT CLEAN</promise>

## Rules
- Do NOT add copyright headers (project CLAUDE.md says: do not add copyright headers)
- Do NOT flag deprecated Registry usage as Critical/High (deferred by design)
- Do NOT flag pre-existing issues in files you did not modify
- Focus audit on the CHANGED code, not the entire codebase
- Always verify PHP changes compile before declaring clean" --completion-promise "AUDIT CLEAN" --max-iterations 5
```

### Step 3: Monitor

The Ralph loop handles everything automatically:
- **Iteration 1**: Audit → find issues → fix → exit → loop restarts
- **Iteration N**: Re-audit → if clean → outputs `<promise>AUDIT CLEAN</promise>` → loop ends
- **Safety**: `--max-iterations 5` prevents runaway token burn (5 full audit-fix cycles max)

## Example Usage

```
/audit-loop
```

That's it. Claude will:
1. Detect your changed files
2. Start the ralph loop
3. Audit → fix → re-audit automatically
4. Stop when clean or after 5 iterations

## Cost Estimate

Each iteration spawns 3 opus agents (~30-40k tokens each). Expect:
- **Clean code**: ~120k tokens (1 iteration, exits immediately)
- **1-2 fix cycles**: ~250-400k tokens
- **Max 5 iterations**: ~600-800k tokens (worst case)
