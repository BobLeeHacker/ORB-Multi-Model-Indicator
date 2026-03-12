# Confluence Score System Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-model confluence scoring (0–100), a BEST-signal badge, tier stars, agreement summary, consensus filter, and targeted model filter improvements to `ORB_Multi_Model_Indicator.pine`.

**Architecture:** Two-pass approach — Pass 1 runs all models and accumulates agreement counters at global scope; Pass 2 computes scores for all fired models using the final counters. Dashboard reads scores for SCORE column and BEST badge. Consensus filter gates `plotshape`/`alertcondition` output using pre-computed agreement scalars.

**Tech Stack:** Pine Script v6, TradingView editor (validation by paste-and-compile, no unit test runner)

**Spec:** `docs/superpowers/specs/2026-03-12-confluence-score-system-design.md`

---

## Chunk 1: Infrastructure — New Globals, fast_rsi14, f_modelScore

### Task 1: Add `fast_rsi14` request.security call

**Files:**
- Modify: `ORB_Multi_Model_Indicator.pine:354–366` (fast TF security block)

- [ ] **Step 1: Locate the fast TF security block**

  Find the comment `// Fast Reference TF data (M1 Crabel + Combined mode)` at line ~354.
  The block ends with `fast_time` at line 365 and `_fastEdge` at line 367.

- [ ] **Step 2: Add `fast_rsi14` after `fast_rvol` (line 362)**

  Insert after line 362:
  ```pine
  fast_rsi14     = request.security(syminfo.tickerid, i_refTF_fast, ta.rsi(close, 14),              lookahead=barmerge.lookahead_off)
  ```
  Note: `ref_rsi14` uses the hardcoded literal `14` (confirmed in the file). Use the same literal here — do NOT use `i_rsiLen` as it does not exist.

- [ ] **Step 3: Also update the comment above the fast block**

  Change:
  ```pine
  // RSI omitted — only M4 uses RSI and it stays on primary TF.
  ```
  To:
  ```pine
  // RSI included for f_modelScore (M1 uses fast TF for scoring).
  ```

- [ ] **Step 4: Paste into TradingView — verify no compilation error**

---

### Task 2: Add agreement counter globals and `_lastScore` array

**Files:**
- Modify: `ORB_Multi_Model_Indicator.pine:282–288` (after `_dashSignal` declaration)

- [ ] **Step 1: Add three new declarations after line 288 (`_mLosses` array)**

  Insert:
  ```pine
  // Per-bar agreement counters — reset every bar, incremented at model call sites
  var int _agreeCountBull = 0
  var int _agreeCountBear = 0

  // Per-model confluence score (0–100) — same 11-slot indexing as _lastEntry
  var int[] _lastScore = array.new<int>(11, 0)
  ```

- [ ] **Step 2: Reset `_lastScore` in the `_newDay` block**

  Find the `_newDay` block that resets `_dashSignal` (line ~436):
  ```pine
  for _ri = 0 to 10
      array.set(_dashSignal, _ri, 0)
  ```
  Add immediately after it:
  ```pine
      array.set(_lastScore, _ri, 0)
  ```
  (Same indentation, inside the same `for` loop body — Pine v6 allows multiple statements per loop iteration.)

- [ ] **Step 3: Reset agreement counters in the signal-flag reset block**

  Find line ~474:
  ```pine
  // Reset all signal flags each bar
  sig_m1_bull   := false
  ```
  Add two lines immediately before this comment:
  ```pine
  // Reset per-bar agreement counters
  _agreeCountBull := 0
  _agreeCountBear := 0
  ```

- [ ] **Step 4: Paste into TradingView — verify no compilation error**

---

### Task 3: Add `f_modelScore` pure function

**Files:**
- Modify: `ORB_Multi_Model_Indicator.pine:662–665` (before model functions section)

- [ ] **Step 1: Locate SECTION 12 header at line ~662**

  The section starts:
  ```pine
  // ============================================================
  // SECTION 12 — MODEL FUNCTIONS (stateless: M1, M3, M4, M7)
  ```

- [ ] **Step 2: Insert `f_modelScore` immediately before the section header**

  ```pine
  // ── Per-Model Confluence Score ────────────────────────────────
  // Pure function — reads params + global htfEma20/htfEma50. Returns 0–100.
  // _rc=close, _rvol=rel-volume, _rvwap=vwap, _rsi=RSI14 — all from model's own TF.
  f_modelScore(isBull, _rc, _rvol, _rvwap, _rsi) =>
      vwapPts  = (isBull ? _rc > _rvwap : _rc < _rvwap) ? 25 : 0
      volPts   = _rvol >= 2.0 ? 25 : _rvol >= 1.5 ? 18 : _rvol >= 1.2 ? 10 : 0
      htfPts   = isBull ? (htfEma20 > htfEma50 ? 20 : 0) : (htfEma20 < htfEma50 ? 20 : 0)
      rsiPts   = isBull ? (_rsi >= 45 and _rsi <= 70 ? 15 : 0)
                        : (_rsi >= 30 and _rsi <= 55 ? 15 : 0)
      agreePts = isBull ? math.max(math.min(_agreeCountBull - 1, 3), 0) * 10
                        : math.max(math.min(_agreeCountBear - 1, 3), 0) * 10
      math.min(vwapPts + volPts + htfPts + rsiPts + agreePts, 100)
  ```

  **Note on `agreePts`:** Subtracts 1 because the calling model is already counted in `_agreeCountBull/Bear`. `math.max(..., 0)` floors at 0 to prevent negative values when the counter is 0. Since we call this in the post-pass after all models have run, the counts are final. With 0 extra agreeing models → 0 pts. With 1 extra → 10. With 3+ extra → 30 (capped).

- [ ] **Step 3: Paste into TradingView — verify no compilation error**

- [ ] **Step 4: Commit Chunk 1**

  ```bash
  git add "ORB_Multi_Model_Indicator.pine"
  git commit -m "feat(score): add f_modelScore, agreement counters, _lastScore array, fast_rsi14"
  ```

---

## Chunk 2: Model Improvements + Agreement Increments

### Task 4: Improve M1 — add optional RVOL filter

**Files:**
- Modify: `ORB_Multi_Model_Indicator.pine:43–45` (M1 inputs block)
- Modify: `ORB_Multi_Model_Indicator.pine:669–696` (`f_model1` function)

- [ ] **Step 1: Add `i_m1_volMult` input after `i_m1_rratio` (line ~45)**

  Find:
  ```pine
  i_m1_rratio = input.float(2.0,  "M1: Risk/Reward ratio",           group="Model 1 — Crabel", minval=1.0, step=0.5)
  ```
  Insert after it:
  ```pine
  i_m1_volMult = input.float(1.2,  "M1: Min RVOL (0 = off)",          group="Model 1 — Crabel", minval=0.0, step=0.1,
       tooltip="Minimum relative volume for M1 signal. Set to 0 to disable the volume filter.")
  ```

- [ ] **Step 2: Update `f_model1` signature to accept `_rvol` parameter**

  Change line ~669:
  ```pine
  f_model1(_firedBull, _firedBear, _rc, _rc1, _ratr, _edge) =>
  ```
  To:
  ```pine
  f_model1(_firedBull, _firedBear, _rc, _rc1, _ratr, _edge, _rvol) =>
  ```

- [ ] **Step 3: Add `volOk` check inside `f_model1`**

  Find line ~674 inside `f_model1`:
  ```pine
            nrPass = i_m1_nr7 ? (isNR7 or isNR4) : true
            if nrPass
  ```
  Change to:
  ```pine
            nrPass = i_m1_nr7 ? (isNR7 or isNR4) : true
            volOk  = i_m1_volMult <= 0 or _rvol >= i_m1_volMult
            if nrPass and volOk
  ```

- [ ] **Step 4: Update the M1 call site (line ~789) to pass `fast_rvol`**

  Change:
  ```pine
  [b, s] = f_model1(_m1FiredBull, _m1FiredBear, fast_close, fast_close1, fast_atr14, _fastEdge)
  ```
  To:
  ```pine
  [b, s] = f_model1(_m1FiredBull, _m1FiredBear, fast_close, fast_close1, fast_atr14, _fastEdge, fast_rvol)
  ```

- [ ] **Step 5: Paste into TradingView — verify no compilation error**

---

### Task 5: Improve M7 — add VWAP + RVOL filters

**Files:**
- Modify: `ORB_Multi_Model_Indicator.pine:64–67` (M7 inputs block)
- Modify: `ORB_Multi_Model_Indicator.pine:758–780` (`f_model7` function)

- [ ] **Step 1: Add `i_m7_volMult` input after `i_m7_rratio` (line ~67)**

  Find:
  ```pine
  i_m7_rratio     = input.float(2.0, "M7: Risk/Reward ratio", group="Model 7 — Gold ORB", minval=1.0, step=0.5)
  ```
  Insert after it:
  ```pine
  i_m7_volMult    = input.float(1.2, "M7: Min RVOL (0 = off)", group="Model 7 — Gold ORB", minval=0.0, step=0.1,
       tooltip="Minimum relative volume for M7 Gold ORB signal. Set to 0 to disable.")
  ```

- [ ] **Step 2: Update `f_model7` signature to accept `_rvol` and `_rvwap`**

  Change line ~758:
  ```pine
  f_model7(_firedBull, _firedBear, _rc, _rc1, _ratr, _edge) =>
  ```
  To:
  ```pine
  f_model7(_firedBull, _firedBear, _rc, _rc1, _ratr, _edge, _rvol, _rvwap) =>
  ```

- [ ] **Step 3: Add `volOk` + VWAP check inside `f_model7`**

  Find inside `f_model7` (line ~763):
  ```pine
          slDist = _ratr * i_atrMult
          if _rc > _or15H and _rc1 <= _or15H and not _firedBull
  ```
  Change to:
  ```pine
          slDist = _ratr * i_atrMult
          volOk  = i_m7_volMult <= 0 or _rvol >= i_m7_volMult
          if _rc > _or15H and _rc1 <= _or15H and not _firedBull and volOk and _rc > _rvwap
  ```
  And the bear side (line ~772):
  ```pine
          if _rc < _or15L and _rc1 >= _or15L and not _firedBear
  ```
  To:
  ```pine
          if _rc < _or15L and _rc1 >= _or15L and not _firedBear and volOk and _rc < _rvwap
  ```

- [ ] **Step 4: Update the M7 call site (line ~816) to pass `ref_rvol` and `ref_vwap`**

  Change:
  ```pine
  [b, s] = f_model7(_m7FiredBull, _m7FiredBear, ref_close, ref_close1, ref_atr14, _refEdge)
  ```
  To:
  ```pine
  [b, s] = f_model7(_m7FiredBull, _m7FiredBear, ref_close, ref_close1, ref_atr14, _refEdge, ref_rvol, ref_vwap)
  ```

- [ ] **Step 5: Paste into TradingView — verify no compilation error**

---

### Task 6: Improve M9 — add VWAP + RVOL filters inline

**Files:**
- Modify: `ORB_Multi_Model_Indicator.pine:893–921` (M9 Phase 3 blocks)

- [ ] **Step 1: Replace the bull-failed → bear-entry block**

  Find lines ~893–903 (the full block including state reset):
  ```pine
          if _m9InsideCount >= i_m9_failBars and not pastCutoff
              sig_m9_bear    := true
              entryP          = ref_close
              slP             = _or15H + slDist * 0.5
              tpP             = entryP - (slP - entryP) * i_m9_rratio
              f_drawSLTP(entryP, slP, tpP, i_colBear, i_colSL, i_colTP)
              f_drawLabel("M9 FailORB", entryP, slP, tpP, false, 4.6)
              f_registerTrade(9, entryP, slP, tpP, false)
              _m9BullBreak   := false
              _m9InsideCount := 0
  ```
  Replace the entire block with:
  ```pine
          if _m9InsideCount >= i_m9_failBars and not pastCutoff
              // Always reset state when threshold reached — prevents stuck state on subsequent bars
              _m9BullBreak   := false
              _m9InsideCount := 0
              // Only fire if VWAP and RVOL conditions are met
              if ref_close < ref_vwap and ref_rvol >= 1.2
                  sig_m9_bear    := true
                  entryP          = ref_close
                  slP             = _or15H + slDist * 0.5
                  tpP             = entryP - (slP - entryP) * i_m9_rratio
                  f_drawSLTP(entryP, slP, tpP, i_colBear, i_colSL, i_colTP)
                  f_drawLabel("M9 FailORB", entryP, slP, tpP, false, 4.6)
                  f_registerTrade(9, entryP, slP, tpP, false)
  ```

  **Key change:** The state reset (`_m9BullBreak := false`, `_m9InsideCount := 0`) now runs unconditionally when the bar threshold is met, regardless of VWAP/RVOL. This prevents the state machine from getting stuck re-evaluating the same setup on every subsequent bar if VWAP/RVOL happen to block the fire.

- [ ] **Step 2: Replace the bear-failed → bull-entry block**

  Find lines ~911–921 (the full block including state reset):
  ```pine
          if _m9InsideCount >= i_m9_failBars and not pastCutoff
              sig_m9_bull    := true
              entryP          = ref_close
              slP             = _or15L - slDist * 0.5
              tpP             = entryP + (entryP - slP) * i_m9_rratio
              f_drawSLTP(entryP, slP, tpP, i_colBull, i_colSL, i_colTP)
              f_drawLabel("M9 FailORB", entryP, slP, tpP, true, 4.6)
              f_registerTrade(9, entryP, slP, tpP, true)
              _m9BearBreak   := false
              _m9InsideCount := 0
  ```
  Replace with:
  ```pine
          if _m9InsideCount >= i_m9_failBars and not pastCutoff
              // Always reset state — prevents stuck state if VWAP/RVOL block the fire
              _m9BearBreak   := false
              _m9InsideCount := 0
              if ref_close > ref_vwap and ref_rvol >= 1.2
                  sig_m9_bull    := true
                  entryP          = ref_close
                  slP             = _or15L - slDist * 0.5
                  tpP             = entryP + (entryP - slP) * i_m9_rratio
                  f_drawSLTP(entryP, slP, tpP, i_colBull, i_colSL, i_colTP)
                  f_drawLabel("M9 FailORB", entryP, slP, tpP, true, 4.6)
                  f_registerTrade(9, entryP, slP, tpP, true)
  ```

- [ ] **Step 3: Paste into TradingView — verify no compilation error**

---

### Task 7: Fix M10 retest tolerance — ATR-scaled

**Files:**
- Modify: `ORB_Multi_Model_Indicator.pine:76–78` (M10 input)
- Modify: `ORB_Multi_Model_Indicator.pine:983–985` (bull Phase 2)
- Modify: `ORB_Multi_Model_Indicator.pine:1031–1033` (bear Phase 2)

- [ ] **Step 1: Update `i_bdm_retestTol` input (line ~76)**

  Change:
  ```pine
  i_bdm_retestTol = input.float(0.05, "BDM: Retest tolerance (% of price)", group="Model 10 — BDM ORB",
       minval=0.01, maxval=1.0, step=0.01,
       tooltip="How close 1-min close must come to OR level to count as a retest.\n0.05% = $2.50 on a $5,000 asset. Adjust upward for low-priced instruments.")
  ```
  To:
  ```pine
  i_bdm_retestTol = input.float(0.15, "BDM: Retest tolerance (× ATR)", group="Model 10 — BDM ORB",
       minval=0.01, maxval=2.0, step=0.01,
       tooltip="How close 1-min close must pull back to the breakout level to qualify as a retest, expressed as a multiple of ATR14. 0.15 = within 15% of one ATR. Increase for wider tolerance on volatile instruments like Gold.")
  ```

- [ ] **Step 2: Fix bull Phase 2 retest calculation (line ~984)**

  Change:
  ```pine
              retestDist = math.abs(fast_close - _bdmBullBreakLevel) / _bdmBullBreakLevel * 100.0
              if retestDist <= i_bdm_retestTol
  ```
  To:
  ```pine
              retestDist = math.abs(fast_close - _bdmBullBreakLevel)
              if retestDist <= fast_atr14 * i_bdm_retestTol
  ```

- [ ] **Step 3: Fix bear Phase 2 retest calculation (line ~1031)**

  Change:
  ```pine
              retestDistB = math.abs(fast_close - _bdmBearBreakLevel) / _bdmBearBreakLevel * 100.0
              if retestDistB <= i_bdm_retestTol
  ```
  To:
  ```pine
              retestDistB = math.abs(fast_close - _bdmBearBreakLevel)
              if retestDistB <= fast_atr14 * i_bdm_retestTol
  ```

- [ ] **Step 4: Paste into TradingView — verify no compilation error**

---

### Task 8: Add agreement counter increments at all model call sites

**Files:**
- Modify: `ORB_Multi_Model_Indicator.pine:786–822` (SECTION 13 — stateless model call sites)
- Modify: `ORB_Multi_Model_Indicator.pine:828–866` (SECTION 15 — M6 inline)
- Modify: `ORB_Multi_Model_Indicator.pine:868–921` (SECTION 17 — M9 inline)
- Modify: `ORB_Multi_Model_Indicator.pine:924–1052` (SECTION 17A-M10 — M10 inline)

- [ ] **Step 1: Add increments after M1 call site (inside `if i_m1_on` block, line ~791)**

  Change:
  ```pine
      sig_m1_bull := b
      sig_m1_bear := s
      if b
          _m1FiredBull := true
      if s
          _m1FiredBear := true
  ```
  To:
  ```pine
      sig_m1_bull := b
      sig_m1_bear := s
      if b
          _m1FiredBull    := true
          _agreeCountBull += 1
      if s
          _m1FiredBear    := true
          _agreeCountBear += 1
  ```

- [ ] **Step 2: Add increments after M3 call site (~line 799)**

  Change:
  ```pine
      sig_m3_bull := b
      sig_m3_bear := s
      if b
          _m3FiredBull := true
      if s
          _m3FiredBear := true
  ```
  To:
  ```pine
      sig_m3_bull := b
      sig_m3_bear := s
      if b
          _m3FiredBull    := true
          _agreeCountBull += 1
      if s
          _m3FiredBear    := true
          _agreeCountBear += 1
  ```

- [ ] **Step 3: Add increments after M4 call site (~line 808)**

  Same pattern as M3 — add `_agreeCountBull += 1` inside `if b` and `_agreeCountBear += 1` inside `if s`.

- [ ] **Step 4: Add increments after M7 call site (~line 817)**

  Same pattern.

- [ ] **Step 5: Add increments after M6 `f_registerTrade` calls (SECTION 15, ~lines 846 and 862)**

  Find the bull M6 `f_registerTrade` call:
  ```pine
                  f_registerTrade(6, entryP, slP, tpP, true)
  ```
  Add immediately after:
  ```pine
                  _agreeCountBull += 1
  ```

  Find the bear M6 `f_registerTrade` call:
  ```pine
                  f_registerTrade(6, entryP, slP, tpP, false)
  ```
  Add immediately after:
  ```pine
                  _agreeCountBear += 1
  ```

- [ ] **Step 6: Add increments after M9 `f_registerTrade` calls (SECTION 17, ~lines 901 and 919)**

  After bear M9 `f_registerTrade(9, entryP, slP, tpP, false)` (line ~901):
  ```pine
              _agreeCountBear += 1
  ```

  After bull M9 `f_registerTrade(9, entryP, slP, tpP, true)` (line ~919):
  ```pine
              _agreeCountBull += 1
  ```

- [ ] **Step 7: Add increments after M10 `f_registerTrade` calls (SECTION 17A-M10, ~lines 997 and 1044)**

  After bull M10 `f_registerTrade(11, entryP10, slP10, tpP10, true)` (line ~997):
  ```pine
                  _agreeCountBull += 1
  ```

  After bear M10 `f_registerTrade(11, entryP10b, slP10b, tpP10b, false)` (line ~1044):
  ```pine
                  _agreeCountBear += 1
  ```

- [ ] **Step 8: Paste into TradingView — verify no compilation error**

- [ ] **Step 9: Commit Chunk 2**

  ```bash
  git add "ORB_Multi_Model_Indicator.pine"
  git commit -m "feat(models): M1 RVOL filter, M7 VWAP+RVOL, M9 VWAP+RVOL, M10 ATR retest, agreement counters"
  ```

---

## Chunk 3: Scoring Post-Pass Block

### Task 9: Add the two-pass scoring block after all models

**Files:**
- Modify: `ORB_Multi_Model_Indicator.pine` — insert new section after M10 block (~line 1053)

- [ ] **Step 1: Locate the end of the M10 block**

  Find the end of SECTION 17A-M10, which is the last inline model. It ends around line ~1052 (after the bear bounce entry block). Look for the next section comment:
  ```pine
  // ============================================================
  // SECTION 17B — TRADE STATUS RESOLUTION
  ```

- [ ] **Step 2: Insert the scoring post-pass block immediately before SECTION 17B**

  ```pine
  // ============================================================
  // SECTION 17A-SCORE — CONFLUENCE SCORE POST-PASS
  // Runs after ALL models so _agreeCountBull/Bear are fully accumulated.
  // Recomputes score every bar for any model that has fired today (_dashSignal != 0).
  // TF routing: M1/Combined → fast TF params; M3/M4/M6/M7/M9 → primary TF params; M10 → mixed.
  // ============================================================

  // mIdxs: model IDs (1,3,4,6,7,9,11) — must match dashboard mIdxs exactly
  var int[] _scoreModelIdxs = array.from(1, 3, 4, 6, 7, 9, 11)

  // TF selector per model: true = use fast TF data, false = use primary TF data
  // Index maps to mIdxs: M1=fast, M3=primary, M4=primary, M6=primary, M7=primary, M9=primary, M10=fast(price)+primary(rsi)
  var bool[] _scoreFast     = array.from(true, false, false, false, false, false, true)

  for _si = 0 to 6
      _sModelId = array.get(_scoreModelIdxs, _si)
      _sArrIdx  = _sModelId - 1
      _sDashSig = array.get(_dashSignal, _sArrIdx)
      if _sDashSig != 0
          _sIsBull = _sDashSig == 1
          _sUseFast = array.get(_scoreFast, _si)
          _sRc   = _sUseFast ? fast_close  : ref_close
          _sRvol = _sUseFast ? fast_rvol   : ref_rvol
          _sVwap = _sUseFast ? fast_vwap   : ref_vwap
          // RSI: M10 uses primary TF RSI even though price is fast TF
          _sRsi  = (_sModelId == 11) ? ref_rsi14 : (_sUseFast ? fast_rsi14 : ref_rsi14)
          _score = f_modelScore(_sIsBull, _sRc, _sRvol, _sVwap, _sRsi)
          array.set(_lastScore, _sArrIdx, _score)
  ```

  **Note on `fast_vwap`:** `fast_vwap` is already declared at line 361 in the fast TF security block — no new security call needed.

  **Critical note:** `_scoreModelIdxs` and `_scoreFast` are declared `var` but their values never change — this is intentional. Using `var` prevents Pine from re-allocating the array every bar, which would be wasteful. The values are constant; only the loop logic changes based on `_dashSignal`.

- [ ] **Step 3: Paste into TradingView — verify no compilation error**

- [ ] **Step 4: Verify scores appear on chart**

  Enable M3 and M4 in settings, load a XAUUSD 5m chart with NY session visible. After 9:45 EST, if M3 or M4 fired, `_lastScore` should contain non-zero values. You can verify by temporarily adding a label: `label.new(bar_index, high, str.tostring(array.get(_lastScore, 2)))` — slot 2 = M3.

- [ ] **Step 5: Remove any debug label added in Step 4**

- [ ] **Step 6: Commit Chunk 3**

  ```bash
  git add "ORB_Multi_Model_Indicator.pine"
  git commit -m "feat(score): add scoring post-pass block — all fired models scored after full agreement count"
  ```

---

## Chunk 4: Dashboard + Consensus Filter

### Task 10: Add `i_minAgree` consensus filter input

**Files:**
- Modify: `ORB_Multi_Model_Indicator.pine:31–41` (Model Toggles inputs group)

- [ ] **Step 1: Add `i_minAgree` after the `i_m10_on` toggle (line ~41)**

  Find:
  ```pine
  i_m10_on = input.bool(false, "Model 10 — BDM ORB",            group="Model Toggles",
       tooltip="Big Daddy Max's 3-phase ORB: 5-min breakout → 1-min retest → 1-min bounce entry.")
  ```
  Insert after it:
  ```pine
  i_minAgree = input.int(1, "Min Models to Show Signal", minval=1, maxval=4,
       tooltip="Plotshape and alert only shown if at least this many models agree on direction. Set to 2 for consensus mode. Dashboard score always updates regardless.",
       group="Model Toggles")
  ```

- [ ] **Step 2: Paste into TradingView — verify no compilation error**

---

### Task 11: Update dashboard — tier stars, SCORE column, BEST badge

**Files:**
- Modify: `ORB_Multi_Model_Indicator.pine:1408–1448` (dashboard model loop)

- [ ] **Step 1: Update `mNames` with tier stars (line ~1408)**

  Change:
  ```pine
  mNames = array.from("1.Crabel","3.5min","4.15min","6.FVG","7.Gold","9.FailORB","10.BDM")
  ```
  To:
  ```pine
  mNames = array.from("★★ 1.Crabel","★★★ 3.5min","★★★ 4.15min","★ 6.FVG","★ 7.Gold","★ 9.FailORB","★★ 10.BDM")
  ```

- [ ] **Step 2: Change column 9 header from "ENTRY TF" to "SCORE" (line ~1402)**

  Change:
  ```pine
  table.cell(_dashTable, 9, 0, "ENTRY TF",text_color=C_TXT, bgcolor=C_HEAD, text_size=size.small)
  ```
  To:
  ```pine
  table.cell(_dashTable, 9, 0, "SCORE",   text_color=C_TXT, bgcolor=C_HEAD, text_size=size.small)
  ```

- [ ] **Step 3: Remove the `mTFs` array and the `C_TF` color (lines ~1415–1416)**

  Delete these two lines entirely:
  ```pine
  C_TF = color.new(#00BCD4, 0)  // cyan — distinct from signal colours
  mTFs = array.from(i_refTF_fast, i_refTF, i_refTF, i_refTF, i_refTF, i_refTF, i_refTF_fast + "/" + i_refTF)
  ```

- [ ] **Step 4: Add string-storage arrays before the model loop for BEST badge deferred render**

  Insert before `for i = 0 to 6` (line ~1418):
  ```pine
  // Temp storage for BEST badge deferred re-render
  var string[] _rowName  = array.new<string>(7, "")
  var string[] _rowSig   = array.new<string>(7, "")
  var string[] _rowDir   = array.new<string>(7, "")
  var string[] _rowEnt   = array.new<string>(7, "")
  var string[] _rowSl    = array.new<string>(7, "")
  var string[] _rowTp    = array.new<string>(7, "")
  var string[] _rowWr    = array.new<string>(7, "")
  var color[]  _rowSigClr = array.new<color>(7, color.gray)
  var color[]  _rowWrClr  = array.new<color>(7, color.gray)
  var color[]  _rowBG     = array.new<color>(7, color.new(#1A1D26, 10))
  ```

- [ ] **Step 5: Update the model loop body to: (a) compute score display, (b) store strings, (c) render rows**

  Inside the `for i = 0 to 6` loop, **replace** the existing line 1447:
  ```pine
      table.cell(_dashTable, 9, row, array.get(mTFs, i),    text_color=C_TF,     bgcolor=rowBG, text_size=size.tiny)
  ```
  With:
  ```pine
      scoreVal = array.get(_lastScore, _arrIdx)
      scoreClr = scoreVal >= 70 ? color.new(#00E676,0) :
                 scoreVal >= 50 ? color.new(#FF9800,0) :
                 scoreVal >  0  ? color.new(#FF5252,0) : C_TXT_DIM
      scoreStr = scoreVal > 0 ? str.tostring(scoreVal) : "-"
      table.cell(_dashTable, 9, row, scoreStr, text_color=scoreClr, bgcolor=rowBG, text_size=size.tiny)
      // Store for BEST badge deferred render
      array.set(_rowName,   i, array.get(mNames, i))
      array.set(_rowSig,    i, sigStr)
      array.set(_rowDir,    i, dirStr)
      array.set(_rowEnt,    i, entStr)
      array.set(_rowSl,     i, slStr)
      array.set(_rowTp,     i, tpStr)
      array.set(_rowWr,     i, wrStr)
      array.set(_rowSigClr, i, sigClr)
      array.set(_rowWrClr,  i, wrClr)
      array.set(_rowBG,     i, rowBG)
  ```

- [ ] **Step 6: Add BEST badge block immediately after the `for` loop ends (after line ~1448)**

  ```pine
  // ── BEST badge: highlight highest-scoring row in gold ─────────
  _bestScore2 = 0
  _bestRow2   = -1
  for _bi2 = 0 to 6
      _bs2 = array.get(_lastScore, array.get(mIdxs, _bi2) - 1)
      if _bs2 > _bestScore2
          _bestScore2 := _bs2
          _bestRow2   := _bi2

  if _bestRow2 >= 0 and not na(_dashTable)
      GOLD_BG  = color.new(#FFD600, 70)
      GOLD_TXT = color.new(#FFD600, 0)
      _br      = _bestRow2 + 1  // table row (0 = header, 1–7 = models)
      _bSigClr = array.get(_rowSigClr, _bestRow2)
      _bWrClr  = array.get(_rowWrClr,  _bestRow2)
      _bScoreVal = array.get(_lastScore, array.get(mIdxs, _bestRow2) - 1)
      _bScoreStr = _bScoreVal > 0 ? str.tostring(_bScoreVal) : "-"
      _bScoreClr = _bScoreVal >= 70 ? color.new(#00E676,0) : _bScoreVal >= 50 ? color.new(#FF9800,0) : color.new(#FF5252,0)
      table.cell(_dashTable, 0, _br, array.get(_rowName, _bestRow2) + " ◀", text_color=GOLD_TXT, bgcolor=GOLD_BG, text_size=size.tiny)
      table.cell(_dashTable, 1, _br, array.get(_rowSig,  _bestRow2),         text_color=_bSigClr,  bgcolor=GOLD_BG, text_size=size.tiny)
      table.cell(_dashTable, 2, _br, array.get(_rowDir,  _bestRow2),         text_color=_bSigClr,  bgcolor=GOLD_BG, text_size=size.tiny)
      table.cell(_dashTable, 3, _br, array.get(_rowEnt,  _bestRow2),         text_color=_bSigClr,  bgcolor=GOLD_BG, text_size=size.tiny)
      table.cell(_dashTable, 4, _br, array.get(_rowSl,   _bestRow2),         text_color=color.new(#FF5252,0), bgcolor=GOLD_BG, text_size=size.tiny)
      table.cell(_dashTable, 5, _br, array.get(_rowTp,   _bestRow2),         text_color=color.new(#69F0AE,0), bgcolor=GOLD_BG, text_size=size.tiny)
      table.cell(_dashTable, 6, _br, array.get(_rowWr,   _bestRow2),         text_color=_bWrClr,   bgcolor=GOLD_BG, text_size=size.tiny)
      table.cell(_dashTable, 7, _br, orHStr,                                  text_color=C_TXT_DIM,  bgcolor=GOLD_BG, text_size=size.tiny)
      table.cell(_dashTable, 8, _br, orLStr,                                  text_color=C_TXT_DIM,  bgcolor=GOLD_BG, text_size=size.tiny)
      table.cell(_dashTable, 9, _br, _bScoreStr,                              text_color=_bScoreClr, bgcolor=GOLD_BG, text_size=size.tiny)
  ```

- [ ] **Step 7: Paste into TradingView — verify no compilation error**

- [ ] **Step 8: Verify dashboard visually**

  On a XAUUSD chart with NY session visible, check that:
  - MODEL column shows tier stars (★★ 1.Crabel, ★★★ 3.5min, etc.)
  - Column 9 shows "SCORE" header and numeric scores where signals have fired
  - The highest-scoring row has gold background and `◀` marker

---

### Task 12: Add agreement summary to RVOL row

**Files:**
- Modify: `ORB_Multi_Model_Indicator.pine` — RVOL row section (~line 1503–1518)

- [ ] **Step 1: Locate the RVOL row block**

  Find:
  ```pine
  // ── RVOL display row (row 8) ─────────────────────────────────
  if i_showDashboard and i_mode == "Multi-Model" and not na(_dashTable)
  ```

- [ ] **Step 2: Replace the two blank cells at columns 4 and 5 in row 8**

  Find:
  ```pine
  table.cell(_dashTable, 4, 8, "", text_color=C_TXT_DIM, bgcolor=C_NONE, text_size=size.tiny)
  table.cell(_dashTable, 5, 8, "", text_color=C_TXT_DIM, bgcolor=C_NONE, text_size=size.tiny)
  ```
  Replace with:
  ```pine
  _bullAgStr = _agreeCountBull > 0 ? str.tostring(_agreeCountBull) + " BULL ↑" : "-"
  _bearAgStr = _agreeCountBear > 0 ? str.tostring(_agreeCountBear) + " BEAR ↓" : "-"
  table.cell(_dashTable, 4, 8, _bullAgStr, text_color=_agreeCountBull > 0 ? color.new(#00E676,0) : C_TXT_DIM, bgcolor=C_NONE, text_size=size.tiny)
  table.cell(_dashTable, 5, 8, _bearAgStr, text_color=_agreeCountBear > 0 ? color.new(#FF5252,0) : C_TXT_DIM, bgcolor=C_NONE, text_size=size.tiny)
  ```

- [ ] **Step 3: Paste into TradingView — verify no compilation error**

- [ ] **Step 4: Verify agreement counts display**

  On a day where multiple models fired in the same direction, the RVOL row should show e.g. `2 BULL ↑` in green.

---

### Task 13: Apply consensus filter to plotshape and alertcondition

**Files:**
- Modify: `ORB_Multi_Model_Indicator.pine` — plotshape section (~lines 1750–1820)
- Modify: `ORB_Multi_Model_Indicator.pine` — alertcondition section (~lines 1763–1790)

- [ ] **Step 1: Find all `plotshape` calls for Multi-Model mode**

  Search for `plotshape` in the file. They follow the pattern:
  ```pine
  plotshape(i_mode=="Multi-Model" and sig_mX_bull, ...)
  plotshape(i_mode=="Multi-Model" and sig_mX_bear, ...)
  ```

- [ ] **Step 2: Wrap every Multi-Model bull plotshape condition with the agreement gate**

  For each bull plotshape, change:
  ```pine
  plotshape(i_mode=="Multi-Model" and sig_mX_bull, ...)
  ```
  To:
  ```pine
  plotshape(i_mode=="Multi-Model" and sig_mX_bull and _agreeCountBull >= i_minAgree, ...)
  ```
  Do this for all 7 models (M1, M3, M4, M6, M7, M9, M10 bull signals).

- [ ] **Step 3: Wrap every Multi-Model bear plotshape condition**

  Same pattern but `_agreeCountBear >= i_minAgree`:
  ```pine
  plotshape(i_mode=="Multi-Model" and sig_mX_bear and _agreeCountBear >= i_minAgree, ...)
  ```
  Do this for all 7 models bear signals.

- [ ] **Step 4: Update individual-model `alertcondition` calls**

  Search for `alertcondition(sig_mX_bull` and `alertcondition(sig_mX_bear`.
  For each one, add the agreement gate:
  ```pine
  // Bull alert:
  alertcondition(sig_mX_bull and _agreeCountBull >= i_minAgree, ...)
  // Bear alert:
  alertcondition(sig_mX_bear and _agreeCountBear >= i_minAgree, ...)
  ```

- [ ] **Step 5: Update the catch-all "ANY ORB" alertconditions**

  Find the catch-all alerts (title contains "ANY ORB"). They currently include `sig_comb_bull/bear`.
  The consensus gate applies only to Multi-Model signals (Combined mode has its own confidence scoring and is unaffected by `i_minAgree`). Structure the condition so Combined signals are always passed through, and Multi-Model signals are gated:

  ```pine
  alertcondition(
       sig_comb_bull or
       ((sig_m1_bull or sig_m3_bull or sig_m4_bull or sig_m6_bull or sig_m7_bull or sig_m9_bull or sig_m10_bull) and _agreeCountBull >= i_minAgree),
       title="ANY ORB — LONG", ...)
  alertcondition(
       sig_comb_bear or
       ((sig_m1_bear or sig_m3_bear or sig_m4_bear or sig_m6_bear or sig_m7_bear or sig_m9_bear or sig_m10_bear) and _agreeCountBear >= i_minAgree),
       title="ANY ORB — SHORT", ...)
  ```

  This preserves Combined-mode catch-all behavior while applying the agreement gate only to Multi-Model signals.

- [ ] **Step 6: Paste into TradingView — verify no compilation error**

- [ ] **Step 7: Verify consensus filter behavior**

  Set `i_minAgree = 2` in settings. On a day where only one model fired, the plotshape triangle should NOT appear (even though the dashboard still shows the signal and score). Set `i_minAgree = 1` — triangle reappears.

- [ ] **Step 8: Commit Chunk 4**

  ```bash
  git add "ORB_Multi_Model_Indicator.pine"
  git commit -m "feat(dashboard): tier stars, SCORE col, BEST badge, agreement summary, consensus filter"
  ```

---

## Final Validation

- [ ] **Load XAUUSD or NQ on a 5m chart with NY session visible**
- [ ] **Enable all models (M1–M10) in settings**
- [ ] **Navigate to a day with multiple model signals**
- [ ] **Verify all 10 success criteria from the spec:**
  1. Each active model row shows a 0–100 score; `-` if no signal
  2. Score color: green ≥70, orange ≥50, red >0, dim if `-`
  3. Highest-scoring row has gold background + `◀` marker
  4. RVOL row (row 8) shows `N BULL ↑` / `N BEAR ↓` counts
  5. Setting `i_minAgree = 2` suppresses lone-model plotshapes/alerts
  6. M7 does not fire when RVOL < 1.2 or price is on wrong VWAP side
  7. M1 volume filter can be disabled by setting `i_m1_volMult = 0`
  8. M9 reversal only fires with correct VWAP side + RVOL ≥ 1.2
  9. M10 Phase 2 retest uses ATR-scaled distance (not % of price)
  10. No compilation errors in TradingView Pine Script editor

- [ ] **Final commit**

  ```bash
  git add "ORB_Multi_Model_Indicator.pine"
  git commit -m "feat: confluence score system complete — scores, BEST badge, model filters, consensus gate"
  ```
