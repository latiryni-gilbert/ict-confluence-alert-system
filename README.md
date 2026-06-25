# ICT Confluence Alert System — Case Study

## The Problem

Discretionary ICT-style trading requires watching three independent conditions
align in real time: a liquidity sweep, a Fair Value Gap retest, and a specific
time window (the NY killzone, 7–10am EST). Manually tracking all three across
multiple timeframes (1m/5m) is cognitively expensive and error-prone — most
missed or mistimed entries come from a trader being late to notice one of the
three conditions, not from the strategy itself being wrong.

The goal was not to build a "holy grail" indicator. It was to remove the
manual surveillance burden and let the platform flag *only* the moments
where all three conditions are mechanically true, so the trader's attention
is spent on execution decisions, not pattern-scanning.

## Stack & Why

- **Pine Script v5 (TradingView)** — chosen because it's the execution
  environment the strategy is actually traded from. No value in building this
  as a standalone Python backtester first; the constraint that matters is
  "does this run live on the chart I trade from."
- **No external API / backend** — deliberate. Indicator logic stays inside
  TradingView's sandbox. Zero credentials, zero infrastructure, zero attack
  surface. Simplicity was a feature, not a shortcut.
- **Stateful array-based tracking for FVGs** — rather than redrawing every
  gap on every bar (expensive, and produces visual clutter from long-dead
  gaps), the script tracks live FVGs in arrays paired with box object
  references, and explicitly invalidates/deletes them once price closes
  through the gap. This is the same pattern as garbage collection on a
  rolling buffer — don't let dead state linger and degrade signal quality.

## Engineering Challenges (the actual interesting part)

1. **No true lookahead-free liquidity sweep detection.** Pine Script
   evaluates on bar close, so a "sweep" can only be confirmed retroactively —
   the wick that took liquidity and the close that confirms reversal happen
   in the same bar's data, by definition one bar after the fact. This is a
   structural limitation of the platform, not a bug. Documenting this
   matters more than hiding it.
2. **State management across sessions.** Asia/London session highs and lows
   had to be tracked with `var` persistent variables and explicitly reset at
   a fixed UTC-aligned time, since Pine has no native "session range" object
   for arbitrary custom windows.
3. **Mobile-to-desktop Pine Editor incompatibility.** Special characters
   (em-dashes, unicode arrows) in source comments silently broke indentation
   parsing on TradingView's mobile editor. Rewrote all formatting using only
   ASCII to guarantee portability across input methods.
4. **Object lifecycle management.** Initial version redrew FVG boxes
   indefinitely, causing unbounded clutter. Fixed by storing `box` object
   references in parallel arrays and calling `box.delete()` the moment a
   gap is invalidated by price — same principle as releasing a resource
   handle once it's no longer needed.

## Current Status

Visual validation only. The script has been confirmed to:
- Correctly flag the NY killzone window
- Correctly detect and label liquidity sweeps within that window
- Correctly draw and auto-invalidate FVG zones
- Correctly fire combined LONG/SHORT alerts only when all three conditions
  align on the same bar

**It has not yet been forward-tested against logged trade outcomes.** That
is the explicit next step, not a finished result. Any version of this case
study that claims a win rate without that data would be fabricated, so none
is included here.

## What I'd Improve Next

1. **Forward-test logging.** Every alert fire should be timestamped and
   logged automatically (via `alert()` payloads to a webhook) into a
   database, so the signal-to-outcome correlation can be measured instead
   of eyeballed from chart screenshots.
2. **Backtest with proper bar-replay**, not just visual inspection, to get a
   first-pass quantitative read before live forward-testing.
3. **Parameterize the killzone and session windows** as user inputs instead
   of hardcoded times, so the tool generalizes beyond one trader's schedule.
4. **If this grows into broker-integrated execution** (e.g. via Tradovate's
   API), that's the point real secrets enter the picture — API keys, OAuth
   tokens — and they'd need to live in environment variables, never in
   source control, with a `.env.example` committed instead of the real file.
   Not needed yet because this version has no external API calls.
5. **Reduce nested-loop cost** in the equal-highs/equal-lows detection — it's
   currently O(n²) over the sweep lookback window per bar, which is fine at
   20 bars but wouldn't scale if the lookback grew.

## Honest Limitation

This tool flags confluence. It does not predict outcomes and is not a
substitute for the trader's own confirmation and risk management. Treat
every alert as "look now," not "enter now."