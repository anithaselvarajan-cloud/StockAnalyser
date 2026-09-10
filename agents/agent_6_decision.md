# Agent 6 — Decision

## Role
Combine all four parallel research outputs (Technical, Fundamental & Quality/Moat,
Sentiment, CSP) plus `framework/framework.json`, compute the weighted overall
score, apply the Munger red-flag override, apply decision thresholds and the
NO ACTION rule, keep the CSP verdict adjacent (never blended), carry forward
`Previous Decision`, and write the final Excel workbook. Runs alone, after
Agents 2, 3, 4, and 5 have all completed.

## Inputs
- `framework/framework.json`
- `framework/technical_output.json`
- `framework/fundamental_output.json` (includes `quality_moat_parameters` and
  `red_flags_evidence`)
- `framework/sentiment_output.json`
- `framework/csp_output.json`
- Previous run's `output/stock_analysis_<prior-date>.xlsx`, if one exists (for
  `Previous Decision`)

## Steps
1. Read `framework.json` for weights, thresholds, the NO ACTION rule, and the
   red-flag/override definitions.
2. For each stock, compute:
   - `technical_category_score` = average of technical parameter scores
     (already computed upstream, or recompute from `technical_output.json`)
   - `fundamental_category_score`, `quality_moat_category_score` — read
     directly from `fundamental_output.json`
   - `sentiment_category_score` = average of sentiment parameter scores
3. Compute `overall_score` as the weighted sum using `framework.json`'s
   weights (default Technical 0.20 / Fundamental 0.25 / Sentiment 0.15 /
   Quality & Moat 0.40). If any single category has 2+ null/unresolved
   parameters, do not silently drop it from the weighted sum — flag the stock
   for the NO ACTION rule instead (see step 6).
4. Apply decision thresholds to `overall_score`:
   - ≥ 7 → **BUY**
   - 4–6.9 → **HOLD**
   - < 4 → **SELL**
5. **Evaluate the red-flag override** using `fundamental_output.json`'s
   `red_flags_evidence` for that stock (RF1, RF2, RF3, RF4):
   - Before applying anything, sanity-check internal consistency: RF1 should
     match the F5 value/direction, RF2 the F4 value vs. the 2.0 (or
     sector-adjusted) threshold, RF3 the Q2 trend. If Agent 3's evidence is
     inconsistent with its own source parameter, treat the flag as
     `status: "uncertain"` rather than trusting a contradictory `triggered`
     value, and note the discrepancy.
   - A flag counts toward the override only when `triggered: true` AND
     `status: "confirmed"`. Flags with `status: "uncertain"` do not by
     themselves trigger the override.
   - If any qualifying flag is triggered: apply the **one-way ratchet** —
     `BUY → HOLD` only. HOLD, SELL, and NO ACTION are never changed by the
     override (it never upgrades, and never touches anything already at or
     below HOLD).
   - Record which flag(s) triggered the downgrade, if any, in a
     `red_flag_override_applied` note.
6. **Apply the NO ACTION rule**: if any single category (technical,
   fundamental, quality_moat, sentiment) has 2 or more null/unresolved
   parameters for a stock, set `Decision = NO ACTION` regardless of the
   computed score — data is too incomplete to act on. This check runs
   independently of, and after, the red-flag override (NO ACTION overrides a
   red-flag-downgraded HOLD too, since the underlying issue is missing data,
   not business quality).
7. Pull the **CSP suitability verdict** (`Good` / `Neutral` / `Skip` +
   rationale) from `csp_output.json` for each stock. This is reported
   **alongside** the Decision column, in its own column(s) — never averaged,
   weighted, or blended into `overall_score` or `Decision` under any
   circumstance.
8. **Carry forward Previous Decision**: if a prior `output/stock_analysis_*.xlsx`
   exists, read its Decision column per ticker into `Previous Decision` for
   this run. Blank if this is the first run for that ticker.
9. Stamp `Review Date` with today's date for every row.
10. Consolidate `Sources` per stock — the union of sources cited across all
    four research outputs for that ticker (dedupe by URL/provider).
11. Write `output/stock_analysis_<YYYY-MM-DD>.xlsx` with:
    - **Table 1 — Actual Values**: one row per stock, one column per
      technical, fundamental, quality & moat, and sentiment metric, holding
      the real researched value. CSP fields (iv_current, iv_rank, hv_30,
      suggested_strike, suggested_dte, est_premium_yield, earnings_date, etc.)
      included here too.
    - **Table 2 — Parameter Scores**: same layout, each metric as a 0–10
      score, plus category subtotal columns for all four categories.
    - **Red Flags** columns: RF1–RF4 (`triggered`/`status`) plus a
      `Red Flag Override Applied` (Y/N + which flag) column.
    - **Overall Score** column (weighted across the four categories, live
      formula).
    - **Decision** column (BUY / HOLD / SELL / NO ACTION, live formula
      incorporating thresholds + red-flag override + NO ACTION rule).
    - **CSP Suitability** column (Good / Neutral / Skip) + **CSP Rationale**.
    - **Sources** column (consolidated per stock).
    - **Review Date** column.
    - **Previous Decision** column.
    - Use LIVE FORMULAS throughout (`AVERAGE`, weighted-sum, nested `IF` for
      thresholds and the override/NO ACTION logic) per the xlsx skill — never
      hardcode a computed value. Validate with `recalc.py` before finishing;
      `total_errors` must be 0. Independently spot-check a few rows' computed
      Overall Score/Decision against a manual calculation to catch
      column-offset or reference errors that `recalc.py` alone won't catch.

## Rules
- The CSP verdict is **never** a scoring input to `overall_score` or
  `Decision`. It is reported adjacent, full stop — even for a stock with a
  triggered red flag, the CSP column still reports its own independent
  verdict (which should reflect the red flag in its own rationale, per
  `agent_5_csp.md`, but the Decision column itself is untouched by CSP).
- The red-flag override can only move BUY → HOLD. It never produces a SELL,
  never touches an already-HOLD/SELL/NO ACTION stock, and never upgrades
  anything.
- Do not treat an `uncertain`-status flag as a clean pass or as a trigger —
  surface the uncertainty in the notes/override column rather than resolving
  it silently in either direction.
- If `fundamental_output.json`, `technical_output.json`, `sentiment_output.json`,
  or `csp_output.json` is missing or unparseable for a stock, do not
  fabricate its data — mark the affected parameters/fields null and let the
  NO ACTION rule govern that stock's Decision.
- Keep `framework.json`'s weights and thresholds as the single source of
  truth — do not hardcode 0.20/0.25/0.15/0.40 or the 7/4 thresholds directly
  in formulas without reading them from the framework (or reproducing them
  exactly as configured there).

## Done when
Every stock has: four category scores, a weighted Overall Score, RF1–RF4
evidence with any override applied, a Decision (BUY/HOLD/SELL/NO ACTION), an
independent CSP Suitability verdict, consolidated Sources, Review Date, and
Previous Decision (where applicable) — all via live formulas, `recalc.py`
clean (0 errors), and spot-checked against manual calculation.
