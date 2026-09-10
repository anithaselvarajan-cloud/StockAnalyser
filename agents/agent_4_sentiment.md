# Agent 4 — Sentiment Analysis

## Role
Research the sentiment/news parameters defined in `framework/framework.json`
for every stock in the list, and score each one. Runs in parallel with
Agents 2 and 3, after Agent 1 completes.

## Input
`framework/framework.json` (parameter list, rubrics, stock list, research
mode).

## Steps
1. Read `framework.json`. Confirm research mode (internet search by default,
   or the specified market/data API).
2. For each stock, research each sentiment parameter (e.g. analyst consensus
   and recent rating changes, recent news sentiment (last 30 days), insider
   buying/selling, short interest, social/media attention — whichever 3–5
   the framework specifies).
3. Record the **actual value** found (e.g. "Analyst consensus: Buy (32
   analysts), 3 upgrades in last 30 days", "Short interest: 1.8% of float"),
   apply the rubric from the framework to assign a **score 0–10**, and note
   the **source** (URL, provider name, and access date) used for that
   specific value.
4. If a value cannot be found or verified, mark it `null` and note why —
   do not guess or fabricate a number.
5. Write `framework/sentiment_output.json`:

```json
{
  "category": "sentiment",
  "generated_at": "<ISO timestamp>",
  "stocks": {
    "NVDA": {
      "parameters": {
        "Analyst consensus": {"value": "Buy (32 analysts)", "score": 8, "source": "https://... (accessed 2026-09-06)"},
        "Recent news sentiment (30d)": {"value": "Mostly positive, AI demand coverage", "score": 7, "source": "..."}
      },
      "category_score": 7.5,
      "notes": ""
    }
  }
}
```

## Rules
- Every value must be real, currently researched information — no
  placeholders or generic statements.
- Every parameter must cite a source (article URL, provider, and date).
- Category score = average of that stock's parameter scores (exclude nulls
  from the average, but flag stocks with 2+ nulls for the NO ACTION rule).
