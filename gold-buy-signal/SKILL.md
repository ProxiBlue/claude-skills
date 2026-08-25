---
name: gold-buy-signal
description: Answer "should I buy gold now" for a short-term micro trade (stop-loss/take-profit style, either direction) with a BUY/SELL/HOLD call and a short reason, weighted across 4 independent sources — GoldPriceWatch AI model, LiteFinance daily technical call, BeCoin multi-factor model, and TradingAgents (local multi-agent LLM debate). Runs a deterministic host-side engine (no LLM parsing needed for the 3 scraped sources) at ~/.config/gold-signal/.
disable-model-invocation: true
---

# Gold Buy Signal Skill

## Overview
Lucas trades gold short-term/micro: small positions with a stop-loss and
take-profit, in either direction — not a buy-and-hold play. This skill
answers "should I buy gold now?" with a plain **BUY / SELL / HOLD** call and
a one-line reason, weighted across 4 independent sources:
- 3 scraped prediction sites (deterministic, ~20s/run, on a 30-min cron)
- TradingAgents, a separate multi-agent LLM framework (non-deterministic,
  ~10-15 min/run, on its own daily cron) — see the dedicated section below

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
  "verdict": "BUY" | "SELL" | "HOLD" | "UNKNOWN",
  "agreement": "unanimous bullish" | "unanimous bearish" | "split" | "mixed/neutral",
  "bullish_count": N, "neutral_count": N, "bearish_count": N,
  "reason": "one-line explanation, already composed",
  "sources": [ {id, label, url, ok, vote, applied_weight, headline, detail, error}, ... ]
}
```
`sources` always has 4 entries: `goldpricewatch`, `litefinance`, `becoin`,
`tradingagents` — same shape, same weighted-vote participation for all 4.
`tradingagents`'s `detail.full_reasoning` carries its full debate writeup
when present; the other three don't have that field.

Answer the user directly using `verdict` and `reason` — they're already
composed, don't second-guess or rewrite the logic. Format per the user's
default terse style:

```
[BUY/SELL/HOLD]. [reason from payload, tightened to one line].
[1 line per source: label — headline]
```

Example:
```
HOLD. Split 2 bullish / 1 neutral / 1 bearish of 4, weighted score barely positive (+0.01) — no real edge.
GoldPriceWatch: Bullish call but only 30% historically direction-accurate (down-weighted hard)
LiteFinance: analyst's base case is short (bearish) below $4,576, stop $4,612
BeCoin: Bullish technical read, 4/5 signals up
TradingAgents: Hold on GLD — overbought technicals vs. medium-term support, sitting out
```

If `verdict` is `"UNKNOWN"` (all sources failed to fetch/parse), say so
plainly and suggest retrying — don't guess a verdict.

**Always surface it when `historical_direction_accuracy_pct` on any source
is below 50%** — a source discloses this itself, and it means their own
track record is worse than a coin flip on direction. It's already reflected
in the vote weight, but the user should still see it named, not buried.

This is informational consolidation of 3rd-party model outputs, not
financial advice — don't present it as more certain than it is. A "BUY"
or "SELL" here means "the 4 sources lean that way right now," not a guarantee.

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

## TradingAgents — the 4th source
[TradingAgents](https://github.com/TauricResearch/TradingAgents) is a
separate multi-agent LLM trading framework, installed at
`~/workspace/proxiblue/trading-agents`, running fully local via Ollama
(`qwen2.5:7b-instruct` — see that project's `.env` for why not the other
locally-pulled models). It analyzes `GLD` (SPDR Gold ETF, the closest
liquid Yahoo-Finance-backed proxy TradingAgents' equity/ETF-shaped
analysts can work with — not spot XAU/USD like the other 3 sources, so
expect some tracking difference) and produces a 5-tier rating (Buy /
Overweight / Hold / Underweight / Sell) via a full analyst-debate-risk
pipeline.

It's now a genuine 4th vote in `sources` — `engine.py`'s
`load_tradingagents_source()` maps its rating to the same -1..+1 scale
(Buy=+1, Overweight=+0.5, Hold=0, Underweight=-0.5, Sell=-1) and folds it
into the weighted score with its own configurable weight (`tradingagents.weight`
in `config.yaml`, default 0.5 — no self-disclosed accuracy to auto-discount
by yet, unlike GoldPriceWatch).

**It does NOT run on the 30-min cron** — a run takes ~10-15 min locally, far
too slow for that cadence and too much repeated GPU load to boot. It runs on
its own separate daily cron instead
(`~/workspace/proxiblue/trading-agents/run_gold_check.sh`), writing
`~/workspace/proxiblue/trading-agents/state/gold_check.json`, which
gold-signal reads on every one of ITS runs — so the TradingAgents vote in
any given gold-signal payload can be up to ~24h old (marked `stale: true`
and excluded from the score past `stale_hours`, same graceful-degradation
pattern as any other source going quiet).

To refresh it on request: `~/workspace/proxiblue/trading-agents/venv/bin/python
~/workspace/proxiblue/trading-agents/run_gold_check.py GLD` — warn the user
this takes 10-15 minutes before running it interactively. gold-signal won't
see the fresh result until its own next run afterward.

## Troubleshooting
- **A source shows `"ok": false`**: that site likely changed its page
  layout — the regex in `~/.config/gold-signal/engine.py` (functions
  `parse_goldpricewatch` / `parse_litefinance` / `parse_becoin`) needs
  updating. Don't hand-patch the JSON; fix the parser. Use the `error`
  field and the site's current rendered text (fetch it via a headless
  browser tool, not curl — all 3 are JS-rendered) to find what changed.
- **All sources fail / `verdict: "UNKNOWN"`**: could be a real fetch
  problem (network, site down) or the host load guard skipped the run
  (check `~/monitor/metrics-*.csv` — the engine logs "skip: load1=... "
  when it bails early). Re-run once load settles.
- **`tradingagents` source shows `"ok": false`**: either no run has
  happened yet (`state/gold_check.json` missing), or its last run errored
  (`error` field carries the reason — often a market-data hiccup or the
  local model hallucinating a bad ticker suffix; see that project's
  `run_gold_check.py`, which already retries once). Not gold-signal's
  code to fix — go to `~/workspace/proxiblue/trading-agents`.
- **Engine won't run at all** (missing venv, playwright not installed):
  see `~/.config/gold-signal/README.md` for setup — it's a plain
  `python3 -m venv venv && ./venv/bin/pip install -r requirements.txt`
  plus Playwright's Chromium (already cached fleet-wide under
  `~/.cache/ms-playwright` from `pb-watch`).

## Success Criteria
- One BUY/SELL/HOLD verdict with a short, honest reason
- Each source's own reliability caveats (esp. low disclosed accuracy, or a
  stale/failed TradingAgents run) surfaced
- No re-derivation of the scoring logic by eye — the engine's `verdict`/`reason` are authoritative
- Clearly labeled as informational, not investment advice
