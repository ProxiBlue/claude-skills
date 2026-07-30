# Test Plan YAML Schema

Authoritative specification for YAML files produced by `/manual-test-plan` and consumed by `/verify-feature`.

- **Producers**: `/manual-test-plan` skill — writes files matching this schema.
- **Consumers**: `/verify-feature` skill — reads files matching this schema.
- **Location**: `.claude/test-plans/<ticket>.yml` (e.g. `.claude/test-plans/361.yml`)
- **Field naming**: snake_case throughout.

---

## Directory

`.claude/test-plans/` MUST exist. A placeholder `.gitkeep` is acceptable (and present in this repo) even though the parent `.claude/` is gitignored — the `.gitkeep` documents intent and keeps the directory tracked. Real plan files committed alongside feature work are preferred over `.gitkeep` alone.

---

## Annotated Single-Plan Example

```yaml
# REQUIRED. GitHub issue number. Integer.
ticket: 361

# REQUIRED. References the implementation plan directory at .claude/plans/<name>/.
# Accepts a single string OR a list of strings (multi-plan — see below).
plan_name: auto-exempt-checkout

# REQUIRED. ISO-8601 timestamp — when this file was generated.
# Producers must write UTC: 2026-05-29T14:00:00Z
generated_at: "2026-05-29T14:00:00Z"

# REQUIRED. List of one or more phase objects.
# Phases group stories by deployment stage or concern (e.g. pre-deployment UAT,
# feature under test, live smoke test).
phases:
  - # REQUIRED string. Human-readable phase label.
    name: Pre-deployment (UAT)

    # OPTIONAL string. Extra context about what this phase covers.
    description: >
      Verify the AvaTax certificate workflow still functions before
      the custom tax-exempt logic is deployed.

    # REQUIRED list. One or more story objects.
    stories:
      - # REQUIRED. kebab-case identifier. Must be globally unique within this
        # ticket YAML AND must not collide with @story annotations declared in
        # other spec files — slugs are used as cross-reference keys.
        slug: avatax-cert-modal-opens

        # REQUIRED string. What this story tests in plain language.
        description: AvaTax "Manage Certificates" modal opens from checkout

        # REQUIRED string. What success looks like.
        expected: Modal renders with "Add Certificate" button visible

        # REQUIRED-unless-manual-only. Path relative to the project root.
        # The file may live inside a git submodule — that is supported.
        # Example: tests/m2-hyva-playwright/src/apps/checkout/tests/avatax.spec.ts
        spec_file: tests/m2-hyva-playwright/src/apps/checkout/tests/avatax-cert.spec.ts

        # REQUIRED-unless-manual-only. The EXACT title string of the Playwright
        # test() block — matched via the -g flag. Must be the complete title, not
        # a substring, to avoid matching multiple tests. The verify-feature skill
        # aborts the story (marks it errored, does not run) when test_name is
        # missing and the spec file contains more than one test().
        test_name: "AvaTax cert modal opens at checkout"

        # OPTIONAL string. Playwright app workspace name. Defaults to "pps".
        app_name: pps

        # OPTIONAL string. Derived from spec_file path when omitted.
        # Pattern: src/apps/<X>/tests/... → X
        # So tests/m2-hyva-playwright/src/apps/checkout/tests/avatax-cert.spec.ts
        # → test_base: checkout
        # Only set this explicitly when the path does not follow that pattern.
        test_base: checkout

        # OPTIONAL list. Named screenshot capture points.
        # Each name MUST correspond exactly to a testInfo.attach('<name>', ...)
        # call in the spec. verify-feature does not regenerate the captured set —
        # it resolves existing attachment paths by these names.
        screenshot_points:
          - modal-open
          - add-certificate-button-visible

        # OPTIONAL boolean. Default: false.
        # When true, spec_file and test_name are not required and verify-feature
        # skips automated execution — the story is recorded as manual-only.
        manual_only: false

        # OPTIONAL string. When set, a test failure for this story is recorded
        # as a known issue rather than a hard failure. Suppresses failure in the
        # verify-feature report but still logs the result.
        # known_issue: "AvaTax sandbox cert API intermittent — tracked in #400"

  - name: Auto-exempt checkout
    description: Tax-exempt group customers bypass the certificate modal entirely.
    stories:
      - slug: tax-exempt-group-skips-cert-modal
        description: Customer in "Tax Exempt" group sees no cert modal at checkout
        expected: Checkout proceeds to payment step without certificate prompt
        spec_file: tests/m2-hyva-playwright/src/apps/checkout/tests/tax-exempt.spec.ts
        test_name: "Tax exempt group customer skips AvaTax cert modal"
        screenshot_points:
          - payment-step-reached

      - slug: non-exempt-group-sees-cert-modal
        description: Regular customer still sees cert modal when AvaTax requires it
        expected: Certificate modal appears before payment step
        spec_file: tests/m2-hyva-playwright/src/apps/checkout/tests/tax-exempt.spec.ts
        test_name: "Non-exempt customer sees AvaTax cert modal"

  - name: Live deployment smoke
    description: Post-deploy sanity check on production.
    stories:
      - slug: live-checkout-loads
        description: Checkout page loads without JS errors after deploy
        expected: Checkout renders; browser console has no critical errors
        spec_file: tests/m2-hyva-playwright/src/apps/checkout/tests/smoke.spec.ts
        test_name: "Checkout page loads"
        app_name: pps

      - slug: live-cert-modal-manual-verify
        description: Manually confirm cert modal still works on live for a test account
        expected: Modal opens, certificate can be added
        manual_only: true
        known_issue: "Cannot automate live cert creation without real AvaTax account"
```

---

## Multi-Plan Example

When a ticket spans multiple implementation plans, `plan_name` is a list. Each entry references a separate `.claude/plans/<name>/` directory.

```yaml
ticket: 362
plan_name:
  - auto-exempt-checkout
  - avatax-cert-migration
generated_at: "2026-05-29T15:00:00Z"
phases:
  - name: Phase A
    stories:
      - slug: plan-a-story-one
        description: Something from plan auto-exempt-checkout
        expected: It works
        spec_file: tests/m2-hyva-playwright/src/apps/checkout/tests/plan-a.spec.ts
        test_name: "Plan A story one"
  - name: Phase B
    stories:
      - slug: plan-b-story-one
        description: Something from plan avatax-cert-migration
        expected: Migration applied cleanly
        manual_only: true
```

---

## Field Reference

### Top-Level Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `ticket` | integer | yes | GitHub issue number |
| `plan_name` | string OR list of strings | yes | Each string references `.claude/plans/<name>/` |
| `generated_at` | string | yes | ISO-8601 timestamp (UTC preferred) |
| `phases` | list | yes | One or more phase objects |

### Phase Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Human-readable label |
| `description` | string | no | Extra context |
| `stories` | list | yes | One or more story objects |

### Story Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `slug` | string (kebab-case) | yes | Globally unique within this YAML; must not collide with `@story` annotations in other spec files |
| `description` | string | yes | Plain-language description of what is tested |
| `expected` | string | yes | Success criterion |
| `spec_file` | string | required-unless-manual-only | Path relative to project root; git submodules supported |
| `test_name` | string | required-unless-manual-only | Exact Playwright `test()` title; see slug/test_name rules below |
| `app_name` | string | no | Playwright app workspace; defaults to `"pps"` |
| `test_base` | string | no | Playwright suite name; derived from `spec_file` when omitted |
| `screenshot_points` | list of strings | no | Named attachment points; see screenshot rules below |
| `manual_only` | boolean | no | Default `false`; skips automated execution |
| `known_issue` | string | no | Suppresses hard failure; logs result as known |

---

## Rules

### Slug Uniqueness

Slugs must be globally unique within a single ticket YAML. They must also not collide with `@story` annotations declared in other spec files — slugs serve as cross-reference keys across the verification system.

### `spec_file` Path

`spec_file` is a path relative to the project root (e.g. `tests/m2-hyva-playwright/src/apps/checkout/tests/foo.spec.ts`). The file may live inside a git submodule — that is supported. The path must be resolvable from the project root at verification time.

### `test_name` Matching

`test_name` is matched via Playwright's `-g` flag. It must be the EXACT `test()` title string — not a substring — to avoid matching multiple tests in the same file. The `verify-feature` skill aborts a story (marks it errored, does not execute) when `test_name` is missing and the spec file contains more than one `test()`.

### `app_name` + `test_base` Invocation Envelope

`app_name` and `test_base` together form the Playwright invocation envelope. When either is omitted from YAML, `verify-feature` derives the values:

- `test_base`: extracted from `spec_file` path — `src/apps/<X>/tests/...` → `X`
- `app_name`: defaults to `"pps"`

Set them explicitly only when the spec file path does not follow the `src/apps/<X>/tests/` convention.

### `screenshot_points` Naming

Each name in `screenshot_points` must correspond exactly to a `testInfo.attach('<name>', ...)` call in the spec. `verify-feature` does not regenerate the captured screenshot set — it resolves existing attachment paths by these names. Mismatch between YAML names and `attach()` call names causes path resolution failures.
