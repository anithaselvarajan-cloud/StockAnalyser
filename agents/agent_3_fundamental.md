# Agent 3 — Fundamental & Quality/Moat Analysis

## Role
Research the fundamental (F-series) and Quality & Moat (Q-series) parameters
defined in `framework/framework.json` for every stock in the list, and score
each one. Runs in parallel with Agents 2, 4, and 5, after Agent 1 completes.

This agent's output is the sole source for three of the four red-flag
triggers Agent 6 evaluates (RF1←F5, RF2←F4, RF3←Q2) — get those three
parameters right and cite real evidence, since a downstream override decision
rests on them.

## Input
`framework/framework.json` (parameter list, rubrics, stock list, research
mode, red-flag definitions).

## Steps
1. Read `framework.json`. Confirm research mode (internet search by default,
   or the specified market/data API).
2. For each stock, research each **fundamental** parameter (e.g. P/E vs
   sector, revenue growth, earnings growth, debt-to-equity, free cash flow
   trend over 3+ years — whichever 3–5 the framework specifies, always
   including F4 and F5).
3. For each stock, research each **Quality & Moat** parameter (competitive
   moat durability, 5-year ROIC trend/consistency, capital allocation track
   record, revenue/earnings durability through a downturn, governance quality
   baseline — whichever 3–5 the framework specifies, always including Q2).
   These are more qualitative than the F-series; ground each in a specific,
   sourced fact rather than a general impression (e.g. cite the actual 5-year
   ROIC figures for Q2, not "returns look strong").
4. Record the **actual value** found, apply the rubric from the framework to
   assign a **score 0–10**, and note the **source** (URL, provider name,
   filing period, and access date) used for that specific value.
5. If a value cannot be found or verified, mark it `null` and note why — do
   not guess or fabricate a number, and do not paper over a genuinely
   uncertain Quality & Moat judgment with false confidence.
6. **Research RF4 (governance/accounting concerns) directly** — restatements,
   auditor changes, SEC actions or investigations, aggressive revenue
   recognition, heavy share-count dilution, related-party transactions,
   dual-class structures concentrating control. Report `triggered`,
   `status` (`"confirmed"` or `"uncertain"` if nothing found but coverage is
   thin), evidence, and source — this feeds Agent 6's override directly and
   is informed by, but not limited to, Q5.
7. Write `framework/fundamental_output.json`:

```json
{
  "category": "fundamental_quality_moat",
  "generated_at": "<ISO timestamp>",
  "stocks": {
    "NVDA": {
      "fundamental_parameters": {
        "F1 - P/E ratio vs sector": {"value": "...", "score": 5, "source": "..."},
        "F4 - Debt-to-equity ratio": {"value": "0.17", "score": 10, "source": "..."},
        "F5 - Free cash flow trend (3+ yrs)": {"value": "rising: $X -> $Y -> $Z", "score": 8, "source": "..."}
      },
      "quality_moat_parameters": {
        "Q2 - 5-year ROIC trend/consistency": {"value": "61.9% -> ... -> 26.2%, declining", "score": 4, "source": "..."}
      },
      "fundamental_category_score": 7.0,
      "quality_moat_category_score": 6.0,
      "red_flags_evidence": {
        "RF1": {"triggered": false, "status": "confirmed", "evidence": "FCF rising per F5 above", "source": "..."},
        "RF2": {"triggered": false, "status": "confirmed", "evidence": "D/E 0.17, well under 2.0 threshold", "source": "..."},
        "RF3": {"triggered": true, "status": "confirmed", "evidence": "ROIC declined monotonically per Q2 above", "source": "..."},
        "RF4": {"triggered": false, "status": "confirmed", "evidence": "No restatements, auditor changes, or SEC actions found", "source": "..."}
      },
      "notes": ""
    }
  }
}
```

## Rules
- Every value must be a real, currently researched figure (most recent
  quarterly/TTM/5-year data as applicable) — no placeholders.
- Every parameter and every red-flag evidence entry must cite a source.
- `fundamental_category_score` = average of the F-series scores;
  `quality_moat_category_score` = average of the Q-series scores (exclude
  nulls from each average, but flag stocks with 2+ nulls in either series for
  the NO ACTION rule).
- **RF1, RF2, and RF3 must be internally consistent with F5, F4, and Q2
  respectively** — never report a flag that contradicts the parameter value
  you just researched. If Q2 (or F5) is `null`/unassessable, report the
  corresponding flag as `triggered: false` with `status: "uncertain"`, never
  as a confirmed pass.
