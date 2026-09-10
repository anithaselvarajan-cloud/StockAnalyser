# Multi-Agent Stock Analysis Pipeline

## Project Goal
Analyze a portfolio watchlist of stocks (from a CSV/Excel input) across technical,
fundamental (including Quality & Moat), sentiment, and cash-secured-put (CSP)
suitability dimensions using a team of specialized agents, then produce a single
Excel workbook with scored, sourced, decision-ready output (BUY / HOLD / SELL /
NO ACTION per stock, plus a separate CSP verdict) with a Munger-style red-flag
override applied. Designed to be re-run periodically, tracking review date and
prior decision over time.

## Folder Structure
```
Stock/
├── CLAUDE.md                  # this file
├── input/
│   └── v1_portfolio_state_starter_template.xlsx   # source stock list
├── framework/
│   └── framework.json          # output of Agent 1 — shared contract for all agents
│   └── technical_output.json   # output of Agent 2
│   └── fundamental_output.json # output of Agent 3 (incl. Quality & Moat + red-flag source data)
│   └── sentiment_output.json   # output of Agent 4
│   └── csp_output.json         # output of Agent 5
├── agents/
│   ├── agent_1_framework.md
│   ├── agent_2_technical.md
│   ├── agent_3_fundamental.md
│   ├── agent_4_sentiment.md
│   ├── agent_5_csp.md
│   └── agent_6_decision.md
└── output/
    └── stock_analysis_<YYYY-MM-DD>.xlsx   # final workbook, one per run
```

## Workflow
1. **Agent 1 (Framework)** reads `input/*.xlsx` (or `.csv`), extracts the stock
   list (Ticker, Company, State), and writes `framework/framework.json` defining:
   required columns, the parameter set for each category (Technical, Fundamental,
   Quality & Moat, Sentiment), scoring rubrics, category weights, decision
   thresholds, red-flag trigger definitions, and CSP fields.
2. **Agents 2, 3, 4, 5 (Technical / Fundamental+Quality&Moat / Sentiment / CSP)**
   run **in parallel**, each reading `framework/framework.json`, researching
   their scope for every stock via internet search (default) or a market/data
   API (if the user opts in — see Research Mode), and writing their own JSON
   output file with actual values, scores out of 10 (Agents 2–4) or a
   suitability verdict (Agent 5), and sources per stock.
3. **Agent 6 (Decision)** reads all four research outputs plus the framework:
   computes the weighted overall score from Technical, Fundamental, Sentiment,
   and Quality & Moat; evaluates the four red flags (RF1–RF4) against their
   source parameters; applies the one-way red-flag override; keeps the CSP
   verdict adjacent but separate from the stock decision; carries forward
   `Previous Decision` from the last run's output file; stamps `Review Date`;
   and writes the final Excel workbook to `output/`.

## Input File Format
Excel or CSV with at minimum these columns (case-insensitive header match):
- `Ticker` (required) — stock symbol
- `Company` (optional) — company name, looked up if missing
- `State` (optional) — free-text position notes; carried through untouched,
  and read by Agent 5 (CSP: `No Position` → acquisition strategy; an existing
  long → note that a covered call may be more relevant than a CSP)

## Research Mode
- **Default:** internet search. Agents 2, 3, 4, and 5 must list the specific
  sources (URLs or named providers) used for each stock's data.
- **Optional:** a market/data API. If the user opts into API mode, ask for:
  API provider name, API key/credentials, and any endpoint/rate-limit
  constraints, before agents 2–5 run. Record the chosen mode in
  `framework.json` so all research agents behave consistently.

## Analysis Framework

### Technical parameters (Agent 2) — pick 3–5 per framework.json
- Price vs 50-day moving average
- Price vs 200-day moving average
- RSI (14-day)
- Relative performance vs sector/index (e.g. S&P 500) over 3 months
- Volatility / Beta

### Fundamental parameters (Agent 3, F-series) — pick 3–5 per framework.json
- F1 — P/E ratio vs sector average
- F2 — Revenue growth (YoY, most recent quarter or TTM)
- F3 — Earnings growth (YoY)
- F4 — Debt-to-equity ratio *(also feeds RF2)*
- F5 — Free cash flow trend, 3+ years *(also feeds RF1)*

### Quality & Moat parameters (Agent 3, Q-series) — new category, pick 3–5
- Q1 — Competitive moat durability (qualitative, sourced: switching costs,
  network effects, scale, brand, regulatory barriers)
- Q2 — 5-year ROIC trend/consistency *(also feeds RF3)*
- Q3 — Capital allocation track record (buybacks/dividends vs. reinvestment
  discipline, historical M&A quality)
- Q4 — Revenue/earnings durability through a downturn (cyclicality assessment)
- Q5 — Governance quality baseline (board independence, insider ownership
  alignment, related-party transaction history) *(also feeds RF4 alongside
  Agent 3's dedicated research, see Red Flags below)*

### Sentiment parameters (Agent 4) — pick 3–5 per framework.json
- Analyst consensus rating and recent rating changes
- Recent news sentiment (last 30 days)
- Insider buying/selling activity
- Short interest level/trend

### CSP suitability (Agent 5) — see `agents/agent_5_csp.md` for the full field
list, IV Rank fallback rule, and suitability rubric. **Not scored into the
overall stock score** — reported as an independent, adjacent verdict
(`Good` / `Neutral` / `Skip`) so it never dilutes or is diluted by the stock
decision.

## Scoring Logic
- Each Technical/Fundamental/Quality & Moat/Sentiment parameter is scored
  **0–10** individually, per a rubric Agent 1 defines in `framework.json`.
- **Category score** = average of that category's parameter scores.
- **Overall score** = weighted average of the four category scores.
  - **Default weights: Technical 20% / Fundamental 25% / Sentiment 15% /
    Quality & Moat 40%** (configurable in `framework.json`).
- **Decision thresholds** (configurable in `framework.json`):
  - Overall score ≥ 7 → **BUY**
  - Overall score 4–6.9 → **HOLD** (or **NO ACTION** if data is incomplete for
    that stock)
  - Overall score < 4 → **SELL**
- Thresholds and weights are parameters, not hardcoded — change them in
  `framework.json` without touching agent logic.

## Red-Flag Override (Munger inversion — "avoid stupidity before seeking brilliance")
Some findings must not be averaged away by a good score elsewhere. Four flags,
each reported as `triggered: true / false / uncertain` **with evidence and a
source**, verified against the parameter it derives from:

| Flag | Trigger | Derived from |
|------|---------|--------------|
| RF1 | Negative or deteriorating free cash flow | F5 |
| RF2 | Debt-to-equity above threshold (default > 2.0, sector-adjusted) | F4 |
| RF3 | Declining multi-year ROIC trend | Q2 |
| RF4 | Governance/accounting concerns — restatements, auditor changes, SEC
  actions, aggressive revenue recognition, heavy dilution | Research (informed
  by Q5) |

**Mechanism — a one-way ratchet, evaluated by Agent 6 after the overall score:**
```
if any flag triggered == true:
    BUY  → HOLD          ← the only transition that fires
    HOLD → HOLD
    SELL → SELL
    NO ACTION → NO ACTION
```
The override can only make a decision *more conservative* — it never upgrades.

Rules:
1. Every triggered flag must be named in the output with its evidence — an
   override with no named trigger is a defect.
2. A flag whose source parameter is `N/A`/unassessable is reported as
   `triggered: false` **with `status: "uncertain"`**, never as a clean pass,
   and does **not** by itself participate in capping a decision — it is
   surfaced to the reader but not treated as either a pass or a fail.
3. RF1 must be consistent with that ticker's F5, RF2 with F4, RF3 with Q2 — a
   contradiction between a flag and its source parameter is a merge bug to be
   resolved before the workbook is written, not shipped.

## Outputs
Final Excel workbook (`output/stock_analysis_<date>.xlsx`) containing:
- **Table 1 — Actual Values**: one row per stock, one column per technical,
  fundamental, Quality & Moat, and sentiment metric, holding the real
  researched value.
- **Table 2 — Parameter Scores**: same layout, each metric expressed as a
  0–10 score, plus category subtotal columns (now four: Technical,
  Fundamental, Sentiment, Quality & Moat).
- **Overall Score** column (weighted across the four categories).
- **Red Flags** columns (RF1–RF4: triggered/false/uncertain + evidence).
- **Decision** column (BUY / HOLD / SELL / NO ACTION), post-override.
- **CSP Suitability** column (Good / Neutral / Skip + rationale) — kept
  visually adjacent to Decision but never blended into it.
- **Sources** column (consolidated list/citations per stock).
- **Review Date** column (date this run was performed).
- **Previous Decision** column (decision from the prior run's output file, for
  tracking changes over time; blank on first run).

## Agent Roles
| Agent | File | Role |
|-------|------|------|
| 1 | `agents/agent_1_framework.md` | Reads input file, defines framework: columns, parameters (incl. Quality & Moat and red-flag definitions), scoring rubric, weights, decision structure. Writes `framework/framework.json`. |
| 2 | `agents/agent_2_technical.md` | Researches technical parameters for every stock. Writes `framework/technical_output.json`. |
| 3 | `agents/agent_3_fundamental.md` | Researches fundamental (F-series) and Quality & Moat (Q-series) parameters for every stock, including the raw evidence RF1–RF3 and RF4 derive from. Writes `framework/fundamental_output.json`. |
| 4 | `agents/agent_4_sentiment.md` | Researches sentiment/news parameters for every stock. Writes `framework/sentiment_output.json`. |
| 5 | `agents/agent_5_csp.md` | Researches cash-secured-put suitability per stock — IV, IV Rank (or fallback), HV, liquidity, support proximity, suggested strike/DTE, earnings/event risk. Writes `framework/csp_output.json`. Independent of the stock score. |
| 6 | `agents/agent_6_decision.md` | Combines all four research outputs, computes the weighted overall score, evaluates and applies the RF1–RF4 red-flag override, keeps the CSP verdict adjacent, applies decision thresholds, writes final Excel to `output/`. |

Agents 2, 3, 4, and 5 have no dependency on each other and should be run in
parallel once Agent 1's framework file exists. Agent 6 requires all four to
have completed.

## Caveats
This pipeline organizes public information into a repeatable framework — it
is not investment advice. It has no knowledge of the user's tax position, risk
tolerance, time horizon, concentration, or liquidity needs. CSP verdicts in
particular describe a real obligation: a CSP seller can be required to buy
shares at the strike, which may be well above the prevailing market price at
assignment. Every run is a point-in-time snapshot and is stale by the time it
is read.
