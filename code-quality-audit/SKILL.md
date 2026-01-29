---
name: code-quality-audit
description: Automated code quality audit skill that runs PSR-12 compliance checks, phpstan static analysis, phpcs code style validation, and phpmd mess detection for Magento 2 code.
---

This skill automates comprehensive code quality checks for Magento 2 projects.

## What This Skill Does

1. **PSR-12 Compliance Check**
   - Validates strict PSR-12 adherence
   - Checks opening braces placement
   - Validates spacing and indentation
   - Verifies proper use statement ordering

2. **PHPStan Static Analysis**
   - Type checking at maximum level
   - Identifies potential bugs and errors
   - Validates method signatures and return types
   - Checks for undefined variables and methods

3. **PHP_CodeSniffer (Magento2 Standard)**
   - Enforces Magento2 coding standards
   - Validates dependency injection patterns
   - Checks service contract usage
   - Verifies plugin and observer implementation

4. **PHP Mess Detector**
   - Identifies code complexity issues
   - Detects unused code and parameters
   - Highlights potential design problems
   - Suggests refactoring opportunities

## Usage

When invoked, this skill will:

1. Run `vendor/bin/php-cs-fixer fix --dry-run --diff` to check PSR-12 compliance
2. Execute `vendor/bin/phpstan analyse -c phpstan.neon` for static analysis
3. Run `vendor/bin/phpcs --standard=Magento2 app/code/` for Magento standards
4. Execute `vendor/bin/phpmd app/code/ text cleancode,codesize,design,naming,unusedcode`

## Output

The skill provides:
- List of violations with file paths and line numbers
- Severity classification (Critical, High, Medium, Low)
- Specific recommendations for fixes
- Code examples of proper implementations

## When to Use

- Before code review submissions
- After implementing new features or modules
- During pull request validation
- Before production deployments
- As part of CI/CD pipeline validation
