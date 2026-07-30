---
name: loki-checkout
description: Build Loki Checkout skins, template overrides, Alpine.js components,
and shipping integrations for Loki Checkout on Magento 2 with Hyvä Theme. Trigger
on any checkout, skin, shipping step, or payment work.
disable-model-invocation: true
---

# Loki Checkout Development Skill

## Always Do First
Before generating any Loki-related code, read the actual source:
- vendor/loki/magento2-checkout/view/frontend/templates/   — overrideable PHMLs
- vendor/loki/magento2-checkout/view/frontend/layout/      — layout XML handles
- vendor/loki/magento2-checkout/view/frontend/web/         — Alpine stores/components
- vendor/loki/magento2-checkout/etc/                       — DI, config, skin reg
- vendor/loki/magento2-checkout/README.md                  — if present

Use these as the canonical override targets, never guess at structure.

## Skin Module Pattern
A Loki skin is a standard Magento 2 module. Reference existing skins:
- vendor/loki/magento2-skin-*/   (or wherever skins live in your vendor)
File structure mirrors standard Magento module layout.

## Alpine Conventions
Loki loads Alpine.js via Loki_Base. All components must be:
- CSP-compatible (no inline mutations, no x-model)
- Registered with Alpine.data() inside alpine:init
- Using $hyvaCsp->registerInlineScript() after every <script> block
Read existing Loki Alpine stores before creating new ones — extend, don't duplicate.

## Our Shipping Module
We are building a shipping integration module for Loki Checkout.
Module: app/code/[Vendor]/[ModuleName]/
This module hooks into Loki's checkout flow at the shipping step.

## Tech Stack Rules
- Hyvä Theme (Hyva/default-csp) + Hyvä UI add-on
- Tailwind CSS only — no custom CSS unless unavoidable
- Alpine.js CSP-safe patterns only
- No RequireJS, jQuery, or Knockout anywhere
- $escaper and $hyvaCsp required in every PHTML
