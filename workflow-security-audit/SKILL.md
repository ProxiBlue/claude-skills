---
name: workflow-security-audit
description: "Security audit workflow for Magento custom code. Invokable standalone (audit the whole project's custom code) OR chained from /proxiblue-skills:workflow-build-feature as a quality gate (audit only the current branch's diff). Checks for: SQL injection (raw query strings without binding, $resource->getConnection()->query() with concatenation), XSS (unescaped output in .phtml, missing escapeHtml/escapeUrl), CSRF (form keys, missing form_key inputs), authentication bypass (admin controllers missing _isAllowed), file upload risks, command injection (shell_exec / exec / system / passthru / `backticks` with user input), insecure deserialization (unserialize on user data), hardcoded secrets/credentials, insecure direct object reference (route params reaching $resource->load without ACL), CSP violations, weak crypto. Returns structured STATUS (PASS / PASS-WITH-NOTES / BLOCKING) + findings list with file:line + severity + suggested fix. Uses gitnexus impact() to chase callers of risky symbols across the project. Composes the existing security-scan skill (basic patterns) + new gitnexus-aware checks."
disable-model-invocation: true
---

# /proxiblue-skills:workflow-security-audit

Security audit with Magento-specific patterns and gitnexus-aware impact propagation.

## Invocation modes

### Standalone — audit the full custom-code surface

```
/proxiblue-skills:workflow-security-audit
```

Audits all of `app/code/<your-namespaces>/` + custom `vendor/<your-namespaces>/`. Produces a findings doc at `.claude/security-audit/<timestamp>.md`.

### Chained — audit the current branch's diff (called from workflow-build-feature)

```
/proxiblue-skills:workflow-security-audit --diff-against=live
```

Audits ONLY files changed in the current branch vs `live`. Returns structured STATUS + findings (no doc written; output is the chain handoff).

## Pre-flight

1. Run `hostname && pwd` — confirm in DDEV web container.
2. Run `git status --short` — note any uncommitted changes (don't refuse; audit-only is read-only).
3. Verify gitnexus reachable: `curl -sS -o /dev/null -w '%{http_code}\n' -m 3 http://gitnexus:4747/`. If unreachable: continue with grep-only mode (skip impact checks).

## Visible todo list

```
1. Determine scope (full custom-code OR diff vs live, based on --diff-against arg)
2. Enumerate files in scope
3. SQL injection patterns
4. XSS / output-escape patterns
5. CSRF / form_key patterns
6. Authentication / ACL patterns (admin controllers)
7. File upload patterns
8. Command injection patterns
9. Insecure deserialization patterns
10. Hardcoded secrets patterns
11. CSP / inline script patterns
12. Crypto pattern check
13. For each finding's risky symbol, gitnexus impact() to surface affected callers
14. Aggregate findings by severity, build STATUS verdict
15. Write findings doc (standalone mode) OR return chain handoff (diff mode)
```

## Check details

### Step 3 — SQL injection

Patterns to grep:

```
\$resource->getConnection\(\)->query\(.*\$
\->fetchAll\(.*\$
\->fetchRow\(.*\$
\->fetchOne\(.*\$
"SELECT[^"]*"\s*\.\s*\$
->query\([^)]*\.[^)]*\$
```

For each hit: read the surrounding context. Flag as **Critical** if the concatenated variable comes from request input. **Important** if from a model field that could be tampered with. **Minor** if from a configured constant.

Acceptable patterns (don't flag): `->select()->where('col = ?', $val)`, `->bindValue(...)`, parameterised queries via `\Zend_Db_Expr`.

### Step 4 — XSS / unescaped output

In `*.phtml`:

```
<?=\s*\$
<\?php\s+echo\s+\$
echo\s+\$.*->getData\(
```

If the echo is NOT wrapped in `$escaper->escapeHtml()` / `escapeUrl()` / `escapeJs()` / `escapeCss()` / `escapeHtmlAttr()` → flag. Severity:
- Output of request data → **Critical**
- Output of admin-configured data → **Important**
- Output of static / system constants → **Minor**

Acceptable: `<?= $escaper->escapeHtml($block->getFoo()) ?>`, `<?= $escaper->escapeUrl($block->getUrl()) ?>`.

### Step 5 — CSRF

For every admin controller's `execute()` that handles POST:
- Magento 2.4+ enforces form_key automatically for admin POST. Check `$this->getRequest()->isPost()` followed by data mutation without `formKeyValidator->validate()` only on frontend controllers.
- Custom REST/AJAX endpoints under `app/code/*/etc/webapi.xml` — flag if `<resource>anonymous</resource>` without explicit comment justifying.

### Step 6 — Auth / ACL

Admin controllers — find every controller under `app/code/*/Controller/Adminhtml/`:

```bash
grep -rln "extends.*\\\\Backend\\\\App\\\\Action" app/code/<namespaces>/
```

For each:
- Must implement `_isAllowed()` returning `$this->_authorization->isAllowed('SomeAcl::resource')`.
- If `_isAllowed()` returns `true` unconditionally → **Critical** (ACL bypass).
- If absent → **Critical** (default-deny isn't guaranteed; depends on parent class).

### Step 7 — File upload

```
->save\(\$_FILES
move_uploaded_file
\\\\Magento\\\\Framework\\\\File\\\\Uploader
```

For each: confirm:
- `setAllowedExtensions(['png', 'jpg', ...])` is called BEFORE save
- `setAllowRenameFiles(true)` + `setFilesDispersion(true)` set
- Destination NOT inside `pub/` or anywhere directly web-accessible
Missing any → **Critical**.

### Step 8 — Command injection

```
shell_exec\s*\(
exec\s*\(
system\s*\(
passthru\s*\(
proc_open\s*\(
`[^`]*\$
```

Any hit involving variable interpolation → **Critical**. Static commands → **Minor** (still review; usually replaceable with native PHP).

### Step 9 — Insecure deserialization

```
unserialize\s*\(
\\\\Magento\\\\Framework\\\\Unserialize
```

If the input comes from request / cookie / db field of unknown source → **Critical**. Use `Magento\Framework\Serialize\Serializer\Json` instead.

### Step 10 — Hardcoded secrets

```
(api_?key|secret|password|token|bearer)\s*=\s*["'][^"']{12,}
```

Any hit not loaded from `env.php` / `config.php` / `Magento\Framework\Encryption\EncryptorInterface` → **Critical**.

### Step 11 — CSP / inline scripts

In Hyva projects: inline `<script>` blocks in `.phtml` files violate the default CSP policy. Flag with **Important** + suggest moving to `view/frontend/web/js/<name>.js` with proper RequireJS / Alpine registration.

### Step 12 — Weak crypto

```
md5\s*\(
sha1\s*\(
DES|3DES|ECB
```

For password / token hashing usage → **Critical**. For non-security uses (e.g., cache key generation) → no finding.

### Step 13 — Gitnexus impact propagation

For each Critical/Important finding's symbol, run `mcp__gitnexus-mageos__impact` to identify callers. Annotate each finding with:

```
File: app/code/Foo/Bar/Controller/Index.php:42
Severity: Critical
Issue: SQL injection — concatenation in fetchAll() of request param
Impact (callers that reach this code path):
  - app/code/Foo/Bar/Block/Result.php:18 (renders the controller's output unescaped — XSS chain risk)
  - vendor/proxiblue/foo-bar/Plugin/AfterDispatch.php:11
```

This turns single findings into chain-of-impact findings — high-leverage signal.

### Step 14 — STATUS verdict

```python
critical_count = len([f for f in findings if f.severity == "Critical"])
important_count = len([f for f in findings if f.severity == "Important"])

if critical_count > 0:
  status = "BLOCKING"     # build-feature stops here
elif important_count > 0:
  status = "PASS-WITH-NOTES"
else:
  status = "PASS"
```

### Step 15 — Output

#### Standalone mode

Write `.claude/security-audit/<YYYYMMDD-HHMM>.md`:

```markdown
# Security Audit — <date>

Scope: <full | diff against live>
Files scanned: <N>
Gitnexus available: <yes/no>

## Critical (N)
1. <file:line> — <issue>
   Impact (callers): <list>
   Fix: <suggestion>

## Important (N)
...

## Minor (N)
...

## Verdict: <STATUS>
```

#### Chain mode (--diff-against=live)

Print exactly this format to stdout for the parent workflow to parse:

```
STATUS: BLOCKING

Findings:

1. Critical — app/code/Foo/Bar/Controller/Index.php:42
   SQL injection via concatenation of $this->getRequest()->getParam('q') into fetchAll().
   Impact: 2 callers in app/code/Foo/Bar/Block + vendor/proxiblue/foo-bar
   Fix: Use parameterised query — $this->_resource->getConnection()->fetchAll($query, [':q' => $param]).
```

## What this skill does NOT do

- **Does not fix anything automatically.** Findings are read-only. The user (or a tdd-worker invocation against the findings doc) applies fixes.
- Does not run static analyzers (phpstan, psalm). Composes well with them but doesn't replace them.
- Does not check infra (DDEV config, php.ini hardening, server config). This is application-layer only.
- Does not modify any file.

## Composes well with

- `/proxiblue-skills:workflow-build-feature` chains this as default quality gate
- `/proxiblue-skills:workflow-investigate-bug` invokes this when bug suspected to have security implications
- Standalone runs valuable as a quarterly review across the full codebase
