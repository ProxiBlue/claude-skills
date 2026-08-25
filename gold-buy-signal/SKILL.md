---
name: gold-buy-signal
description: Answer "should I buy gold now" for a short-term micro trade (stop-loss/take-profit style) by consolidating 3 public gold-prediction sites (GoldPriceWatch AI model, LiteFinance daily technical call, BeCoin multi-factor model) into one yes/no call with a short reason. Runs a deterministic host-side engine (no LLM parsing needed) at ~/.config/gold-signal/.
disable-model-invocation: true
---

# Gold Buy Signal Skill

## Overview
Lucas trades gold short-term/micro: small buy/sell positions with a stop-loss
and take-profit, looking for setups where a near-term rise is more likely
than a dip. This skill answers "should I buy gold now?" with a plain yes/no
and a one-line reason, backed by 3 independent public prediction sources.

**All the fetching/scoring logic already exists as a deterministic script.**
Do NOT re-fetch the 3 sites yourself with WebFetch/browser tools and re-derive
a verdict by eyeballing the pages — that's slower, more expensive, and less
consistent than the engine. Only fall back to manual fetching if the engine
itself is broken (see Troubleshooting).

## When to Use This Skill
- User runs `/gold-buy-signal`
- User asks "should I buy gold now/today?", "is gold a buy?", "gold looking
  bullish or bearish?", or similar
- User asks to check/refresh the gold signal, or to see the gold dashboard

## Step 1: Run the engine
```bash
/home/lucas/.config/gold-signal/run.sh
```
This is host-only (not inside a DDEV container) — run it as a plain host
Bash command. It prints one JSON line to stdout (also written to
`~/.config/gold-signal/state/latest.json`) and takes ~15-25s (3 headless
Chromium page loads). If the caller is inside a container and this path
isn't reachable, tell the user and stop — don't try to reimplement the fetch.

If a run is already in flight (flock), the command returns almost instantly
with no output — in that case read `state/latest.json` instead, it'll be
current within ~30s.

## Step 2: Parse the JSON and answer

The payload shape:
```json
{
  "generated_at_utc": "...",
  "overall_score": -1.0..1.0,
  "verdict": "YES" | "NO" | "UNKNOWN",
  "agreement": "unanimous bullish" | "unanimous bearish" | "split" | "mixed/neutral",
  "bullish_count": N, "neutral_count": N, "bearish_count": N,
  "reason": "one-line explanation, already composed",
  "sources": [ {id, label, url, ok, vote, applied_weight, headline, detail, error}, ... ]
}
```

Answer the user directly using `verdict` and `reason` — they're already
composed, don't second-guess or rewrite the logic. Format per the user's
default terse style:

```
[YES/NO]. [reason from payload, tightened to one line].
[1 line per source: label — headline]
```

Example:
```
NO. Split 2 bullish / 1 bearish of 3, weighted score barely positive (+0.01) — no real edge.
GoldPriceWatch: Bullish call but only 30% historically direction-accurate (down-weighted hard)
LiteFinance: analyst's base case is short (bearish) below $4,576, stop $4,612
BeCoin: Bullish technical read, 4/5 signals up
```

If `verdict` is `"UNKNOWN"` (all 3 sources failed to fetch/parse), say so
plainly and suggest retrying — don't guess a verdict.

**Always surface it when `historical_direction_accuracy_pct` on any source
is below 50%** — a source discloses this itself, and it means their own
track record is worse than a coin flip on direction. It's already reflected
in the vote weight, but the user should still see it named, not buried.

This is informational consolidation of 3rd-party model outputs, not
financial advice — don't present it as more certain than it is. A "YES"
here means "the 3 sources lean bullish right now," not a guarantee.

## Step 3 (optional): offer the dashboard
**http://127.0.0.1:8934/** (bookmarked) — served by the `gold-signal-web`
systemd --user unit, always up. Full per-source breakdown (raw parsed
fields, support/resistance levels, stop-loss levels from LiteFinance,
scenario probabilities from BeCoin) plus a **track record** section: each
source graded against its own later-reported price once its call's horizon
passes (1 day for LiteFinance/BeCoin, 7 days for GoldPriceWatch) — green/red
hit-dots per source, so "is this source worth trusting" is visible directly
rather than taken on the source's own word. Mention it exists if the user
wants more than the one-liner; every page load auto-refreshes the data in
the background (see `~/.config/gold-signal/README.md`).

## TradingAgents comparison card
The dashboard also shows a 4th, independent opinion from TradingAgents (a
separate multi-agent LLM framework at `~/workspace/proxiblue/trading-agents`,
local Ollama, no cost) — labeled "comparison only", never folded into the
gold-signal verdict above. It runs on its own daily cron (not the 30-min
one — a run takes ~10-15 min locally), so when answering "should I buy gold
now" you can mention its current Buy/Hold/Sell rating as an aside ("for
what it's worth, TradingAgents' independent take is also X") but the
verdict/reason you lead with is always gold-signal's own weighted score —
don't let the two get conflated into one answer.

To refresh it on request: `~/workspace/proxiblue/trading-agents/venv/bin/python
~/workspace/proxiblue/trading-agents/run_gold_check.py GLD` — warn the user
this takes 10-15 minutes before running it interactively.

## Troubleshooting
- **A source shows `"ok": false`**: that site likely changed its page
  layout — the regex in `~/.config/gold-signal/engine.py` (functions
  `parse_goldpricewatch` / `parse_litefinance` / `parse_becoin`) needs
  updating. Don't hand-patch the JSON; fix the parser. Use the `error`
  field and the site's current rendered text (fetch it via a headless
  browser tool, not curl — all 3 are JS-rendered) to find what changed.
- **All 3 sources fail / `verdict: "UNKNOWN"`**: could be a real fetch
  problem (network, site down) or the host load guard skipped the run
  (check `~/monitor/metrics-*.csv` — the engine logs "skip: load1=... "
  when it bails early). Re-run once load settles.
- **Engine won't run at all** (missing venv, playwright not installed):
  see `~/.config/gold-signal/README.md` for setup — it's a plain
  `python3 -m venv venv && ./venv/bin/pip install -r requirements.txt`
  plus Playwright's Chromium (already cached fleet-wide under
  `~/.cache/ms-playwright` from `pb-watch`).

## Success Criteria
- One yes/no verdict with a short, honest reason
- Each source's own reliability caveats (esp. low disclosed accuracy) surfaced
- No re-derivation of the scoring logic by eye — the engine's `verdict`/`reason` are authoritative
- Clearly labeled as informational, not investment advice
