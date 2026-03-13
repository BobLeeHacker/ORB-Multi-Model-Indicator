# ORB Multi-Model Indicator — Implementation Plan

## Context

Build a professional-grade TradingView Pine Script v6 indicator implementing 9 distinct Opening Range Breakout (ORB) entry models, plus a Combined mode with a confidence scoring system. The indicator targets the New York session with automatic DST handling. Research covered 30+ sources including backtested strategies with documented win rates up to 74.56%.

---

## Output Files

| File | Purpose |
|------|---------|
| `ORB_Multi_Model_Indicator.pine` | Single Pine Script v6 file (~1,800+ lines) |

---

## Architecture: Single Pine Script v6 File

### Section Layout
1. **Inputs** — Grouped by category (Mode, Session, Model Toggles, per-model params, Combined weights, Visuals, Risk)
2. **Constants & Type Definitions** — UDTs: `ORData`, `SignalData`, `ConfidenceData`
3. **Global State** — `var` variables for persistent state across bars
4. **Technical Indicators** — Pre-computed at global scope (EMA, RSI, VWAP, ATR, volume ratio, HTF data via `request.security`)
5. **Core Engine** — Session detection (`time()` with IANA tz), range tracking, daily reset
6. **Model Detection Functions** (9 models)
7. **Combined Model / Confidence Score**
8. **Visual Rendering** — Boxes, lines, labels, dashboard table, checklist table
9. **Alerts & plotshape** — All at global scope (Pine v6 requirement)

---

## Two Modes

### Mode 1: Multi-Model Detection
- Each of 9 models runs independently when enabled
- Triggers show: entry arrow with model name, entry/SL/TP lines, dashboard row
- Multiple models can fire on the same bar

### Mode 2: Combined Model
- Uses primary 15-min OR breakout as trigger
- Evaluates 8 confidence factors with user-configurable weights
- Displays checklist (met/unmet conditions), confidence score (%), grade (A/B/C/F)
- Only fires signal when score >= user-defined minimum threshold

---

## 9 ORB Models

| # | Model | OR Window | Entry Logic | Key Filter | Est. Win Rate |
|---|-------|-----------|-------------|------------|---------------|
| 1 | Classic Crabel | User-configurable (5-60 min) | Immediate breakout on close | NR7/NR4 compression day | Varies |
| 2 | Fisher ACD | User-configurable | OR + A-value offset (10% of 10-ATR) | Pivot Range context | N/A |
| 3 | 5-Min Scalper | First 5 min | 1-min close beyond range | Volume surge + VWAP | 55-60% |
| 4 | Standard 15-Min | First 15 min | 5-min close beyond range | Volume + VWAP + EMA + RSI | 65-74% |
| 5 | Conservative Retest | First 15-30 min | Wait for pullback retest | Volume decline on pullback | Higher selectivity |
| 6 | FVG ORB | First 15 min | Limit order at Fair Value Gap | VWAP + previous day levels | N/A |
| 7 | Gold ORB | 9:30-9:45 EST | 5-min close beyond range | Time cutoff at noon EST | N/A |
| 8 | ICT/Smart Money | Asian range + Kill Zones | Judas Swing reversal | Asian range sweep + Kill Zone timing | N/A |
| 9 | Failed ORB Reversal | Primary OR | Price fails breakout, returns inside | Rotation day detection | 62% on rotation days |

---

## Confidence Score System (Combined Mode)

| Factor | Default Weight | Scoring Method |
|--------|---------------|----------------|
| Volume Confirmation | 20% | RVOL tiers: >=2x=100%, >=1.5x=80%, >=1.2x=60% |
| HTF Trend Alignment | 20% | Close vs EMA20/EMA50 on higher TF |
| Day Type | 15% | Gap% from prior close as trend-day proxy |
| Narrow Range (NR7) | 10% | NR7=100%, NR4=70%, none=20% |
| ATR Volatility | 10% | OR width relative to ATR sweet spot |
| Candle Close Quality | 10% | Penetration depth beyond OR level |
| VWAP Alignment | 10% | Price above/below VWAP matching direction |
| Order Flow Proxy | 5% | Candle body ratio as delta proxy |

**Grades:** A (>=80%), B (>=60%), C (>=40%), F (<40%)

---

## Session & DST Handling

- All session detection via `time(timeframe.period, sessionString, "America/New_York")`
- IANA timezone identifiers auto-handle DST — no manual date logic needed
- User-configurable session times via `input.session()`
- Time cutoff (default noon EST) prevents late-session signals
- Multiple OR session windows tracked simultaneously (5-min, 15-min, custom, Asian)

---

## Visual System

| Element | Implementation |
|---------|---------------|
| OR Range Box | `box.new()` with transparent fill, extends right through day |
| OR Level Lines | `line.new()` dashed at OR high/low/mid |
| Entry/SL/TP Levels | `line.new()` per signal, color-coded (green/red/teal/orange) |
| Signal Arrows | `plotshape()` — 20 calls at global scope (9 models x 2 dirs + 2 combined) |
| Model Name Labels | `label.new()` at signal bar with model name and entry price |
| Dashboard Table | `table.new()` — Mode 1: rows per model with signal/direction/entry/SL/TP |
| Checklist Table | `table.new()` — Mode 2: rows per factor with status/score/weight |
| ACD Levels | Additional lines for A-up/A-down/C-up/C-down when Model 2 active |

---

## Alert System

- `alertcondition()` per model per direction (18 conditions + 2 combined + 2 catch-all = 22 total)
- `alert()` calls inside conditionals for dynamic messages with confidence scores
- All alerts gated by `barstate.isconfirmed` to prevent repainting

---

## Key Pine v6 Constraints

- `plotshape()` and `alertcondition()` MUST be at global scope
- Booleans cannot be `na` — initialize to `false`
- No `transp` param — use `color.new(base, transparency)`
- `request.security()` at global scope only
- Max 500 each: boxes, lines, labels — manage via arrays with cleanup on new day
- Use `barstate.isconfirmed` for anti-repainting

---

## Build Sequence (12 Steps)

| Step | Scope | ~Lines |
|------|-------|--------|
| 1 | Skeleton: declaration, inputs, constants, types | 200 |
| 2 | Core Engine: session detection, OR tracking, box drawing | 150 |
| 3 | Model 4 (Standard 15-Min) — reference implementation | 70 |
| 4 | Visual System: dashboard, signal levels, labels | 200 |
| 5 | Models 1, 3, 7 (simple breakout variants) | 155 |
| 6 | Model 2 (Fisher ACD) + A/C level lines | 70 |
| 7 | Models 5, 6 (two-phase: retest/FVG) | 150 |
| 8 | Model 8 (ICT/Smart Money — Asian range, Judas Swing) | 100 |
| 9 | Model 9 (Failed ORB Reversal) | 80 |
| 10 | Combined Mode: confidence engine + checklist | 270 |
| 11 | Alerts: alertcondition + alert() calls | 120 |
| 12 | Polish: edge cases, object cleanup, tooltips | 70 |

---

## Verification Plan

1. **Core Engine** — Add to 5-min SPY/ES chart, verify OR box at 9:30-9:45 EST, verify daily reset, check DST dates (Mar/Nov)
2. **Per-Model** — Enable one model at a time, navigate to known setups, verify entry/SL/TP levels match rules, use replay mode to confirm no repainting
3. **Combined Mode** — Verify all 8 checklist factors populate, scores match manual calc, grade thresholds correct
4. **Edge Cases** — No-breakout days (no signals), gap days, low volume (filtered), multiple triggers same bar, 30+ days on 1-min (object limits)
5. **Cross-Instrument** — Test on ES, NQ, SPY, XAU/USD, BTC, EUR/USD
6. **Alerts** — Set up alerts per model, verify once-per-bar-close firing, verify message content
