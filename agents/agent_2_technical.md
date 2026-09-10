# Agent 2 — Technical Analysis

## Role
Research the technical parameters defined in `framework/framework.json` for
every stock in the list, and score each one. Runs in parallel with Agents 3
and 4, after Agent 1 completes.

## Input
`framework/framework.json` (parameter list, rubrics, stock list, research
mode).

## Steps
1. Read `framework.json`. Confirm research mode (internet search by default,
   or the specified market/data API).
2. For each stock, research each technical parameter (e.g. price vs 50/200-day
   moving average, RSI, relative performance vs index, volume trend,
   volatility/beta — whichever 3–5 the framework specifies).
3. Record the **actual value** found (e.g. "RSI: 58", "Price vs 200D MA:
   +12%"), apply the rubric from the framework to assign a **score 0–10**,
   and note the **source** (URL, provider name, and access date) used for
   that specific value.
4. If a value cannot be found or verified, mark it `null` and note why —
   do not guess or fabricate a number.
5. Write `framework/technical_output.json`:

```json
{
  "category": "technical",
  "generated_at": "<ISO timestamp>",
  "stocks": {
    "NVDA": {
      "parameters": {
        "Price vs 50-day MA": {"value": "+8.2%", "score": 7, "source": "https://... (Yahoo Finance, accessed 2026-09-06)"},
        "RSI (14-day)": {"value": 61, "score": 5, "source": "..."}
      },
      "category_score": 6.0,
      "notes": ""
    }
  }
}
```

## Rules
- Every value must be a real, currently researched figure — no placeholders.
- Every parameter must cite a source; if using an API, cite the API/provider
  and endpoint instead of a URL.
- Category score = average of that stock's parameter scores (exclude nulls
  from the average, but flag stocks with 2+ nulls for the NO ACTION rule).
