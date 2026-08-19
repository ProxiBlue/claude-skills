---
name: debt-ledger
description: >
  Harvest deferred-shortcut markers (`DEBT: <reason>`) left in code across the current
  repo into a tracked ledger at `.claude/debt-ledger.md`, so a scope cut made to keep a
  change minimal doesn't silently become permanent. Use when the user says "harvest debt
  markers", "check deferred debt", "what shortcuts did we defer", "update debt ledger",
  or invokes /debt-ledger. Language-agnostic (matches the marker text, not any one
  comment syntax) — works across PHP, JS, Python, shell, etc.
---

Any time a change deliberately skips scope to stay minimal (skip a case that can't
happen yet, defer a generalization, leave a rough edge for a known reason), leave a
one-line marker at the spot instead of just remembering it:

```
// DEBT: no pagination yet — fine while catalog is <500 items, revisit if it grows
# DEBT: hardcoded 3-retry limit, should read from config once multi-tenant lands
```

`DEBT: <reason>` — any comment syntax, always that exact token + colon + a reason. This
skill finds them; nothing else about how you write code needs to change.

## What this skill does

1. **Find markers.** In a git repo: `git grep -n "DEBT:"` (respects `.gitignore`,
   skips `vendor/`, `node_modules/`, binary files automatically). Outside a git repo:
   `grep -rn "DEBT:" --exclude-dir={vendor,node_modules,.git}`.
2. **Attribute each hit.** For tracked files, `git blame -L <line>,<line> --porcelain <file>`
   to get commit sha, author, and date — best effort, skip silently if blame fails
   (uncommitted file, binary, etc.).
3. **Diff against the existing ledger** at `.claude/debt-ledger.md` (create if absent).
   Match prior entries by `file:line` + marker text:
   - **NEW** — marker found now, not in the previous ledger.
   - **STILL OPEN** — marker found now and was in the previous ledger (carry its
     original "first seen" date forward, don't reset it).
   - **RESOLVED** — was in the previous ledger, marker no longer found in the code
     (comment removed or line changed) — move it to a "Resolved" section with today's
     date rather than deleting it, so the history of what got paid down is visible.
4. **Write the ledger.** Overwrite `.claude/debt-ledger.md` with:
   - A one-line summary count (`N open, M resolved`) at the top.
   - An "Open" table sorted oldest-first: `file:line | reason | first seen | commit`.
   - A "Resolved" table (most recent first, capped at the last 20 — older resolved
     entries can be found via git history on the ledger file itself, don't let this
     section grow unbounded).
5. **Report.** Print the same summary counts to the conversation, and call out any
   entry open for a long time (no fixed threshold — use judgment: something first-seen
   months ago with no movement is worth flagging explicitly, not just left in the table).

## Notes

- This is a harvesting/reporting skill only — it never edits the DEBT-marked code
  itself or removes markers. Paying down the debt is a separate, deliberate task.
- If `.claude/debt-ledger.md` doesn't exist yet and no markers are found either, say so
  plainly and stop — don't create an empty ledger file for a repo that isn't using the
  convention yet.
