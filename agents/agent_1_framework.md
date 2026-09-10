# Agent 1 — Framework

## Role
Read the input stock file and produce the shared analysis framework that
Agents 2–6 will use. You run first, alone — nothing else depends on you
running in parallel.

## Input
`input/*.xlsx` or `input/*.csv` — expects at minimum a `Ticker` column
(case-insensitive), optionally `Company` and `State`.

## Steps
1. Load the input file. Extract the list of stocks: ticker, company name
   (look it up if blank), and any `State`/notes column, carried through as-is.
2. Confirm the research mode: default is internet search. If the user has
   opted into a market/data API, record the provider name and confirm the
   required API details have been supplied (do not proceed without them).
3. Finalize the parameter set for each category (3–5 parameters each unless
   noted), starting from the defaults in `CLAUDE.md` and adjusting only if a
   default parameter is not obtainable for these particular tickers:
   - **Technical** (Agent 2)
   - **Fundamental**, F-series (Agent 3) — F4 (D/E) and F5 (FCF trend) are
     mandatory since they feed RF2 and RF1
   - **Quality & Moat**, Q-series (Agent 3) — Q2 (5-year ROIC trend) is
     mandatory since it feeds RF3
   - **Sentiment** (Agent 4)
4. Define the 0–10 scoring rubric for each Technical/Fundamental/Quality &
   Moat/Sentiment parameter — be specific enough that two different
   researchers would score the same raw value the same way.
5. Define the CSP field list and IV Rank fallback rule per
   `agents/agent_5_csp.md` — copy its field table and fallback threshold
   (`listing_age_trading_days < 252`) into `framework.json` verbatim so Agent
   5 has a single source of truth.
6. Define the four red-flag triggers (RF1–RF4) exactly as specified in
   `CLAUDE.md`'s Red-Flag Override section, including which parameter each
   one derives from (RF1←F5, RF2←F4, RF3←Q2, RF4←research/Q5) and the
   default D/E threshold (> 2.0, sector-adjustable).
7. Set category weights — **default Technical 20% / Fundamental 25% /
   Sentiment 15% / Quality & Moat 40%** (configurable) — and decision
   thresholds (default: ≥7 BUY, 4–6.9 HOLD, <4 SELL; NO ACTION if a stock has
   missing/insufficient data in 2 or more parameters in any single category).
8. Write `framework/framework.json` with this structure:

```json
{
  "stocks": [
    {"ticker": "NVDA", "company": "Nvidia Corp", "state": "No Position"}
  ],
  "research_mode": "internet_search",
  "api_details": null,
  "parameters": {
    "technical": [ {"name": "...", "rubric": "..."} ],
    "fundamental": [
      {"id": "F1", "name": "P/E ratio vs sector average", "rubric": "..."},
      {"id": "F4", "name": "Debt-to-equity ratio", "rubric": "...", "feeds_flag": "RF2"},
      {"id": "F5", "name": "Free cash flow trend (3+ yrs)", "rubric": "...", "feeds_flag": "RF1"}
    ],
    "quality_moat": [
      {"id": "Q2", "name": "5-year ROIC trend/consistency", "rubric": "...", "feeds_flag": "RF3"}
    ],
    "sentiment": [ {"name": "...", "rubric": "..."} ]
  },
  "csp_fields": ["iv_current", "iv_rank", "hv_30", "options_liquidity",
    "support_proximity", "suggested_strike", "suggested_dte",
    "est_premium_yield", "assignment_context", "earnings_date",
    "event_risk", "suitability"],
  "iv_rank_fallback": {
    "trigger": "listing_age_trading_days < 252",
    "basis": "fallback_absolute_and_peer_relative",
    "report_instead": ["absolute IV", "IV vs sector peers", "IV vs HV(30)"]
  },
  "red_flags": {
    "RF1": {"trigger": "Negative or deteriorating free cash flow", "derived_from": "F5"},
    "RF2": {"trigger": "Debt-to-equity above threshold", "derived_from": "F4", "default_threshold": 2.0, "sector_adjustable": true},
    "RF3": {"trigger": "Declining multi-year ROIC trend", "derived_from": "Q2"},
    "RF4": {"trigger": "Governance/accounting concerns", "derived_from": "research (informed by Q5)"}
  },
  "override_rule": "one_way_ratchet: BUY->HOLD if any RF true; HOLD/SELL/NO ACTION unchanged; never upgrades",
  "weights": {"technical": 0.20, "fundamental": 0.25, "sentiment": 0.15, "quality_moat": 0.40},
  "decision_thresholds": {"buy": 7, "hold_min": 4, "sell_below": 4},
  "no_action_rule": "2 or more parameters missing/unreliable in any single category"
}
```

## Output
`framework/framework.json` — the single source of truth Agents 2, 3, 4, 5,
and 6 all read from. Validate before finishing: every stock has a ticker,
every scored category has 3–5 parameters with rubrics, F4/F5/Q2 are present
and tagged with their `feeds_flag`, the CSP field list and IV Rank fallback
are copied in, and the four category weights sum to 1.0.
