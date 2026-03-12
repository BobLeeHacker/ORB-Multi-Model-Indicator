# Confluence Score System & Workflow Improvement
**Date:** 2026-03-12
**Scope:** ORB_Multi_Model_Indicator.pine (Pine Script v6)
**Status:** Approved (v2 — post spec-review fixes)

---

## Problem Statement

When multiple ORB models fire simultaneously, the trader has no guidance on which signal to take. Some models have high win rates but low trade frequency; others fire often but with poor selectivity. There is no per-signal quality ranking and no way to filter by agreement count.

---

## Solution Overview

Three coordinated changes:

1. **Per-model real-time confluence score** — each model computes a 0–100 score after all models have run (two-pass approach)
2. **Dashboard upgrades** — tier stars, SCORE column, BEST badge, agreement summary, consensus filter
3. **Targeted model improvements** — add missing filters to M7, M1, M9, M10

---

## Section 1: Per-Model Confluence Score

### Agreement Counters (scalars, global scope)

Two `var int` scalars track per-bar agreement:

```pine
var int _agreeCountBull = 0
var int _agreeCountBear = 0
```

**Reset location:** Lines 474–490 (the signal-flag reset block, runs every bar):
```pine
_agreeCountBull := 0
_agreeCountBear := 0
```

**Increment location:** At the call site in global scope, immediately after each model's `[bull, bear]` return — NOT inside any function. Example:

```pine
[_b, _s] = f_model3(...)
if _b
    sig_m3_bull    := true
    _m3FiredBull   := true
    _agreeCountBull += 1   // ← global scope, valid
if _s
    sig_m3_bear    := true
    _m3FiredBear   := true
    _agreeCountBear += 1
```

For inline models (M6, M9, M10), the increment happens immediately after `f_registerTrade` is called, still at global scope.

### Scoring Function: `f_modelScore(isBull, _rc, _rvol, _rvwap, _rsi)`

A pure function — reads parameters only, writes nothing. Returns int 0–100.

```pine
f_modelScore(isBull, _rc, _rvol, _rvwap, _rsi) =>
    vwapPts  = (isBull ? _rc > _rvwap : _rc < _rvwap) ? 25 : 0
    volPts   = _rvol >= 2.0 ? 25 : _rvol >= 1.5 ? 18 : _rvol >= 1.2 ? 10 : 0
    htfPts   = isBull ? (htfEma20 > htfEma50 ? 20 : 0) : (htfEma20 < htfEma50 ? 20 : 0)
    rsiPts   = isBull ? (_rsi >= 45 and _rsi <= 70 ? 15 : 0)
                      : (_rsi >= 30 and _rsi <= 55 ? 15 : 0)
    agreePts = isBull ? math.min(_agreeCountBull - 1, 3) * 10
                      : math.min(_agreeCountBear - 1, 3) * 10
    // agreePts: -1 because this model itself is already counted; 3 extra models max = +30
    math.min(vwapPts + volPts + htfPts + rsiPts + agreePts, 100)
```

**TF-awareness:** `f_modelScore` accepts `_rc`, `_rvol`, `_rvwap`, `_rsi` as explicit parameters.
- M1, Combined: pass `fast_close`, `fast_rvol`, `fast_vwap`, `fast_rsi` (fast TF)
- M3, M4, M6, M7, M9: pass `ref_close`, `ref_rvol`, `ref_vwap`, `ref_rsi14` (primary TF)
- M10: pass `fast_close`, `fast_rvol`, `fast_vwap`, `ref_rsi14` (mixed — Phase 3 uses fast price, RSI from primary)

### Score Storage

New array (size 11, same indexing as `_lastEntry`):

```pine
var int[] _lastScore = array.new<int>(11, 0)
```

Reset in `_newDay` block alongside other `_lastEntry` resets:
```pine
for _ri = 0 to 10
    array.set(_lastScore, _ri, 0)
```

### Two-Pass Execution Order (critical)

```
Pass 1 — Run all models (every bar):
  1a. Reset _agreeCountBull = 0, _agreeCountBear = 0
  1b. Run M1 → M10 model blocks
      → each model: fires signal, calls f_registerTrade, increments agree counts at global scope

Pass 2 — Score all fired models (after all models complete):
  2a. For each model slot i = 0 to 6:
      arrIdx = array.get(mIdxs, i) - 1
      dSig   = array.get(_dashSignal, arrIdx)
      if dSig == 1  // bull fired today
          score = f_modelScore(true,  <fast or ref close>, <rvol>, <vwap>, <rsi>)
          array.set(_lastScore, arrIdx, score)
      else if dSig == -1  // bear fired today
          score = f_modelScore(false, <fast or ref close>, <rvol>, <vwap>, <rsi>)
          array.set(_lastScore, arrIdx, score)

  2b. Render plotshapes (read _agreeCountBull/Bear for consensus gate)
  2c. Render dashboard (read _lastScore for SCORE column + BEST badge)
```

**Why two-pass?** All models run first, so `_agreeCountBull/Bear` are fully accumulated before any score is computed. Every model gets the same complete agreement count — no ordering bias.

**Score update frequency:** Scores are recomputed every bar (not just at fire time), so they reflect current market conditions even for signals that fired earlier in the day. This means a signal that fired at 9:35 will show a different score at 10:00 if VWAP or volume has changed. This is intentional — it tells you how valid the setup still is now.

---

## Section 2: Dashboard Changes

### 2A: Tier Stars in MODEL Column

Model name strings prefixed with tier, applied in `mNames` array at render time:

| Tier | Models | Prefix |
|---|---|---|
| ★★★ | M3, M4 | `"★★★ "` |
| ★★ | M1, M10 | `"★★ "` |
| ★ | M6, M7, M9 | `"★ "` |

```pine
mNames = array.from("★★ 1.Crabel","★★★ 3.5min","★★★ 4.15min","★ 6.FVG","★ 7.Gold","★ 9.FailORB","★★ 10.BDM")
```

### 2B: SCORE Column (repurposes column 9, currently "ENTRY TF")

- Header cell (column 9, row 0): `"SCORE"` instead of `"ENTRY TF"`
- Remove the `mTFs` array construction (currently lines 1416–1416) — it becomes unused
- Per model row: read `array.get(_lastScore, _arrIdx)` instead of `array.get(mTFs, i)`
- Display: `str.tostring(score)` or `"-"` if score == 0

Color coding:
```pine
scoreVal = array.get(_lastScore, _arrIdx)
scoreClr = scoreVal >= 70 ? color.new(#00E676,0) :
           scoreVal >= 50 ? color.new(#FF9800,0) :
           scoreVal >  0  ? color.new(#FF5252,0) : C_TXT_DIM
scoreStr = scoreVal > 0 ? str.tostring(scoreVal) : "-"
```

### 2C: BEST Badge

After the model loop, find the highest-scoring row and re-render all 10 columns of that row with a gold background. Must re-render all columns (not just MODEL + SIGNAL) to achieve a full gold row:

```pine
_bestScore = 0
_bestRow   = -1
for i = 0 to 6
    s = array.get(_lastScore, array.get(mIdxs, i) - 1)
    if s > _bestScore
        _bestScore := s
        _bestRow   := i + 1

if _bestRow > 0 and not na(_dashTable)
    GOLD_BG = color.new(#FFD600, 70)
    // Re-render all 10 columns of the best row with gold background
    table.cell(_dashTable, 0, _bestRow, array.get(mNames, _bestRow-1) + " ◀", text_color=color.new(#FFD600,0), bgcolor=GOLD_BG, text_size=size.tiny)
    table.cell(_dashTable, 1, _bestRow, sigStr,   text_color=sigClr, bgcolor=GOLD_BG, text_size=size.tiny)
    // ... repeat for columns 2–9 (reading from _lastEntry/SL/TP arrays for that row's arrIdx)
```

Note: The MODEL name and SIGNAL text must be read from the same loop-local variables computed in the prior model loop, or recomputed for `_bestRow - 1`. Simplest: run the badge block inside the model loop with a deferred render, or store signal strings in a temp array during the loop.

**Simplest implementation:** At the end of the model loop, store `sigStr`, `dirStr`, `entStr`, `slStr`, `tpStr`, `wrStr`, `scoreStr` in 7 temporary string arrays (size 7), then render them — first pass normal background, then re-render `_bestRow` with gold.

### 2D: Agreement Summary in RVOL Row (row 8)

Two new cells in the existing RVOL row:

```pine
// Column 4 (was blank):
bullAgreeStr = _agreeCountBull > 0 ? str.tostring(_agreeCountBull) + " BULL ↑" : "-"
table.cell(_dashTable, 4, 8, bullAgreeStr,
    text_color = _agreeCountBull > 0 ? color.new(#00E676,0) : C_TXT_DIM,
    bgcolor=C_NONE, text_size=size.tiny)

// Column 5 (was blank):
bearAgreeStr = _agreeCountBear > 0 ? str.tostring(_agreeCountBear) + " BEAR ↓" : "-"
table.cell(_dashTable, 5, 8, bearAgreeStr,
    text_color = _agreeCountBear > 0 ? color.new(#FF5252,0) : C_TXT_DIM,
    bgcolor=C_NONE, text_size=size.tiny)
```

### 2E: Consensus Filter Input

```pine
i_minAgree = input.int(1, "Min Models to Show Signal", minval=1, maxval=4,
    tooltip="Signal plotshape and alert only shown if at least this many models agree on direction. Set to 2 for consensus mode. Dashboard score always updates regardless.",
    group="Model Toggles")
```

**Applied to `plotshape` calls** — each `sig_mX_bull` condition wrapped:
```pine
// Before:
plotshape(i_mode=="Multi-Model" and sig_m3_bull, ...)
// After:
plotshape(i_mode=="Multi-Model" and sig_m3_bull and _agreeCountBull >= i_minAgree, ...)
```

**Applied to `alertcondition` calls** — same pattern:
```pine
// Individual model:
alertcondition(sig_m3_bull and _agreeCountBull >= i_minAgree, ...)
// Catch-all "ANY ORB LONG":
alertcondition((sig_m1_bull or sig_m3_bull or ...) and _agreeCountBull >= i_minAgree,
    title="ANY ORB — LONG", ...)
// Catch-all "ANY ORB SHORT":
alertcondition((sig_m1_bear or sig_m3_bear or ...) and _agreeCountBear >= i_minAgree,
    title="ANY ORB — SHORT", ...)
```

The catch-all conditions gate on the same `_agreeCountBull/Bear >= i_minAgree` threshold — consistent with individual model behavior.

Does **not** suppress `f_registerTrade` — dashboard score, entries table, and win/loss counters always update regardless of `i_minAgree`.

---

## Section 3: Individual Model Improvements

### M7 — Gold ORB

**Current:** Pure breakout, no filters beyond time gate.
**Add to `f_model7` function:**

```pine
i_m7_volMult = input.float(1.2, "M7 Min RVOL", minval=0.0, step=0.1,
    tooltip="Minimum relative volume for M7 Gold ORB signal. Set to 0 to disable.",
    group="Model 7 — Gold ORB")
```

Inside `f_model7(_firedBull, _firedBear, _rc, _rc1, _ratr, _edge, _rvol, _rvwap)`:
```pine
volOk = i_m7_volMult <= 0 or _rvol >= i_m7_volMult
// Bull: existing conditions and volOk and _rc > _rvwap
// Bear: existing conditions and volOk and _rc < _rvwap
```

Function signature gains `_rvol` and `_rvwap` parameters (pass `ref_rvol`, `ref_vwap` at call site).

### M1 — Classic Crabel

**Add new input:**

```pine
i_m1_volMult = input.float(1.2, "M1 Min RVOL (0 = off)", minval=0.0, step=0.1,
    tooltip="Minimum relative volume for M1 Crabel signal. Set to 0 to disable volume filter.",
    group="Model 1 — Crabel")
```

Inside `f_model1`, alongside existing NR7/NR4 check:
```pine
volOk = i_m1_volMult <= 0 or _rvol >= i_m1_volMult
// Bull: existing conditions and volOk
// Bear: existing conditions and volOk
```

Function signature gains `_rvol` parameter (pass `fast_rvol` at call site — M1 uses fast TF).

### M9 — Failed ORB Reversal

Applied inline in M9 Phase 3 blocks (around existing `f_registerTrade(9, ...)` calls):

```pine
// Bull reversal (was: just pastCutoff check)
if _m9BullBreak and _m9InsideCount >= i_m9_failBars
    if not pastCutoff and ref_close > ref_vwap and ref_rvol >= 1.2
        // existing entry/SL/TP logic
```

```pine
// Bear reversal
if _m9BearBreak and _m9InsideCount >= i_m9_failBars
    if not pastCutoff and ref_close < ref_vwap and ref_rvol >= 1.2
        // existing entry/SL/TP logic
```

No new inputs — RVOL 1.2 hardcoded for M9 (disabled by default, low sample size, doesn't warrant its own input).

### M10 — BDM ORB Retest Tolerance

**Current calculation in Phase 2 (both bull and bear blocks):**
```pine
retestDist = math.abs(fast_close - _bdmBullBreakLevel) / _bdmBullBreakLevel * 100.0
if retestDist <= i_bdm_retestTol
```

**New calculation — change BOTH sides:**
```pine
retestDist = math.abs(fast_close - _bdmBullBreakLevel)   // raw price distance, no normalization
if retestDist <= fast_atr14 * i_bdm_retestTol             // ATR-scaled threshold
```

Input change:
```pine
// Before:
i_bdm_retestTol = input.float(0.05, "M10 Retest Tolerance (%)", ...)
// After:
i_bdm_retestTol = input.float(0.15, "M10 Retest Tolerance (× ATR)",
    tooltip="How close price must pull back to the breakout level to qualify as a retest, expressed as a multiple of ATR14. 0.15 = within 15% of one ATR. Increase for wider tolerance on volatile instruments like Gold.",
    group="Model 10 — BDM ORB")
```

Same change applies to both bull (Phase 2 bull block) and bear (Phase 2 bear block) retest checks.

---

## Implementation Notes

### Pine v6 Compliance Checklist

| Rule | How this design complies |
|---|---|
| `plotshape()` at global scope | Consensus gate uses pre-computed `_agreeCountBull/Bear` — no function wrapping |
| `alertcondition()` at global scope | Same — reads scalars set before alert block executes |
| Functions cannot write `var` scalar globals | `f_modelScore` is pure (no writes). Agreement increments at call sites, not inside functions |
| `request.security()` at global scope | No new security calls added |
| Array writes from functions allowed | `_lastScore` written via `array.set()` in scoring post-pass at global scope |

### New Global Variables Summary

| Variable | Type | Purpose |
|---|---|---|
| `_agreeCountBull` | `var int` | Tracks bull-direction model agreement per bar |
| `_agreeCountBear` | `var int` | Tracks bear-direction model agreement per bar |
| `_lastScore` | `var int[]` size 11 | Stores per-model confluence score |

### Files Changed

| File | Changes |
|---|---|
| `ORB_Multi_Model_Indicator.pine` | All changes — score function, post-pass scoring block, dashboard, model filter additions, agreement tracking |

No other files modified.

---

## Success Criteria

1. Each active model row in dashboard shows a 0–100 score; `-` if no signal today
2. Score color: green ≥70, orange ≥50, red >0
3. Highest-scoring active signal row highlighted gold with `◀` marker
4. RVOL row shows `N BULL ↑` / `N BEAR ↓` agreement counts in correct colors
5. Setting `i_minAgree = 2` suppresses plotshapes and alerts for lone-model signals; dashboard still updates
6. M7 no longer fires on low-RVOL or wrong-VWAP-side bars
7. M1 has an optional RVOL filter (default 1.2, set to 0 to disable)
8. M9 reversal only fires when price is on correct VWAP side with RVOL ≥ 1.2
9. M10 retest tolerance scales with ATR (both left and right side of comparison changed)
10. No new compilation errors in TradingView Pine Script editor
