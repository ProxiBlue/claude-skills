---
name: security-scan
description: Comprehensive Magento 2 security scanning skill that checks for vulnerabilities, misconfigurations, outdated dependencies, security patches, and compliance with security best practices.
---

This skill automates security auditing and vulnerability scanning for Magento 2 applications.

## What This Skill Does

1. **Dependency Vulnerability Scan**
   - Scan composer dependencies for known CVEs
   - Check for outdated Magento core version
   - Identify vulnerable third-party modules
   - Review security patch status
   - Validate PHP version security support

2. **Configuration Security Audit**
   - Admin panel security settings
   - Two-factor authentication status
   - Session configuration and timeout
   - Cookie security settings
   - HTTPS enforcement validation
   - Secret key usage in admin URLs

3. **File System Security**
   - File and directory permissions (should be 644/755)
   - Sensitive file exposure checks (.git, .env, etc.)
   - var/log accessibility
   - pub/media upload validation
   - Validate restricted file extensions

4. **Code Security Analysis**
   - SQL injection vulnerability scan
   - XSS prevention validation (escaper usage)
   - CSRF protection (form key validation)
   - Input validation and sanitization
   - Insecure deserialization checks
   - Hardcoded credentials detection

5. **Access Control Validation**
   - Admin user audit (strong passwords, MFA)
   - Role and permission configuration
   - API authentication security
   - Customer password policy
   - Failed login attempt monitoring

6. **Compliance Checks**
   - PCI DSS configuration validation
   - GDPR compliance settings
   - Security headers (CSP, HSTS, X-Frame-Options)
   - Cookie consent and privacy settings
   - Data encryption validation

## Security Tools Used

```bash
# Composer security check
composer audit

# Magento security scan
bin/magento security:check:now

# File permission check
find . -type f ! -perm 644 -o -type d ! -perm 755

# Search for potential vulnerabilities
grep -r "eval\|exec\|system\|passthru" app/code/
grep -r "unserialize" app/code/

# Check for exposed sensitive files
curl -I https://example.com/.git/config
curl -I https://example.com/.env
curl -I https://example.com/var/log/system.log
```

## MCP Integration

Uses:
- **filesystem**: File scanning and permission checking
- **magento2-dev**: Configuration validation
- **database**: Security-related configuration queries

## Scan Output

### Risk Classification
- **Critical**: Immediate security threat requiring urgent action
- **High**: Significant vulnerability, prioritize remediation
- **Medium**: Security weakness, schedule fix
- **Low**: Best practice improvement, low risk
- **Info**: Security information, no immediate action needed

### Report Sections
1. **Executive Summary**
   - Overall security score (0-100)
   - Critical findings count
   - Compliance status

2. **Vulnerability Details**
   - CVE IDs and severity
   - Affected components and versions
   - Exploitation difficulty
   - Remediation steps

3. **Configuration Issues**
   - Misconfigured security settings
   - Weak authentication configurations
   - Missing security headers
   - Recommended configurations

4. **Compliance Status**
   - PCI DSS requirements status
   - GDPR compliance gaps
   - Industry best practices adherence

5. **Remediation Plan**
   - Prioritized action items
   - Implementation steps
   - Testing recommendations
   - Validation methods

## When to Use

- Regular security audits (monthly/quarterly)
- Before production deployments
- After installing new modules
- Post-security incident analysis
- Compliance audit preparation
- Customer security requirement validation
- Pre-acquisition due diligence
