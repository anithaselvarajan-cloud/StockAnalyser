# Agent 5 — Cash-Secured Put (CSP) Analysis

**Runs:** in parallel with Agents 2, 3, 4, after Agent 1.
**Reads:** `framework/framework.json`
**Writes:** `framework/csp_output.json`
**Scope:** options-market suitability for selling cash-secured puts, per ticker.
**Not scored into the overall stock score** — this is a separate, parallel verdict.

## Why this is a separate agent

"Is this a good business to own?" and "is this a good stock to sell puts on?" are different
questions with different answers. A high-quality compounder trading at a premium with
placid options may be a BUY and a poor CSP. A mediocre business with rich IV may be a
tempting CSP and a poor long-term holding. Blending them into one number destroys both
signals, so this agent reports independently and Agent 6 keeps the two verdicts adjacent
but distinct.

Note also that the CSP seller's downside is *owning the stock at the strike*. So a CSP is
only sensible on a stock the user would be content to own at that price. The Quality & Moat
work in Agent 3 is therefore a real input to CSP judgement, even though it is scored
separately — reflect this in the rationale.

## Fields to research per ticker

| Field | What to report |
|-------|----------------|
| `iv_current` | Current implied volatility (30-day / ATM), with as-of date |
| `iv_rank` | IV Rank and/or IV Percentile vs. trailing 52 weeks — **or the documented fallback** (see below) |
| `hv_30` | 30-day historical/realised volatility, for the IV-vs-HV premium comparison |
| `options_liquidity` | Open interest and average daily volume on near-the-money puts; typical bid-ask spread (absolute and as % of premium); number of expirations listed |
| `support_proximity` | Distance from current price to nearest technical support — lower Bollinger band, 50/200-day MA, prior swing low. State which support and the % distance |
| `suggested_strike` | Candidate strike zone, typically ~0.20–0.30 delta, expressed as strike price and % below spot |
| `suggested_dte` | Candidate days-to-expiration range (commonly 30–45 DTE) |
| `est_premium_yield` | Estimated premium as % of capital secured, and annualised — label clearly as an estimate |
| `assignment_context` | What owning the stock at the suggested strike would mean — the strike vs. estimated intrinsic value or 52-week range |
| `earnings_date` | Next earnings date — an expiration spanning earnings carries materially different risk |
| `event_risk` | Other known forward events: lock-up expiries, regulatory decisions, index changes, litigation dates |
| `suitability` | `Good` / `Neutral` / `Skip` + one-line rationale |

## IV Rank and its fallback — read this before scoring anything

IV Rank is defined against the **trailing 52-week IV range**. A stock listed less than 52
weeks ago **has no such range**, and any IV Rank figure quoted for it is either computed
over a shorter window (and mislabelled) or meaningless.

When `listing_age_days < 252`, do **not** report an IV Rank. Instead:

1. Report **absolute IV** and say what it is (e.g. "IV 68%, elevated in absolute terms").
2. Report **IV relative to sector peers** — compare against comparable optionable names.
3. Report **IV vs. HV(30)** — the volatility risk premium is computable without a 52-week
   history and is the most useful available substitute.
4. Set `iv_rank_basis` to `"fallback_absolute_and_peer_relative"` and explain it in the
   rationale.

Apply this to any newly-listed name in the current portfolio (check each ticker's listing
date against today's date before trusting a provider's IV Rank figure) — do not assume the
fallback only applies to whichever ticker triggered it last time. If a provider publishes an
IV Rank anyway for a sub-252-day listing, treat it as **rejected evidence** and log that you
rejected it, rather than silently using it.

## Suitability rubric

| Verdict | Conditions |
|---------|------------|
| **Good** | IV Rank elevated (≳50) *or* documented fallback showing a genuine IV-over-HV premium; tight spreads and real open interest; price at or near support; no major event inside the expiration window; a business the user would accept owning at the strike |
| **Neutral** | Mixed — e.g. decent premium but thin liquidity, or good liquidity with unattractive premium, or an earnings date inside the window |
| **Skip** | Illiquid or very wide options; premium not compensating for risk (IV below HV); major unquantifiable event risk in the window; or a stock whose quality/red-flag profile means assignment would be unwelcome at any price |

Sources: Market Chameleon, OptionCharts, Barchart options pages, CBOE, Tastytrade,
Nasdaq options chains, broker platforms.

## Rules

- **Never invent option prices, greeks or IV.** Options data is the easiest thing in this
  pipeline to hallucinate plausibly. Report retrieved figures with a source and as-of date,
  or `N/A`.
- Options data is **highly time-sensitive**. Always timestamp; a two-day-old IV reading in
  a fast tape is not usable and should be labelled.
- State whether the chain is monthly or weekly, and note if the ticker has no listed
  options at all (→ `suitability: Skip`, reason: no options market).
- The `State` column from the input matters: for `No Position`, a CSP is an acquisition
  strategy; for an existing long, note that a covered call may be the more relevant
  structure and say so rather than forcing a CSP verdict.
- Premium yield figures are estimates from indicative quotes, not executable prices. Label
  them as such. Do not annualise a single premium and present it as an expected return.
- If a red flag surfaced by Agent 3 (RF1–RF4) is triggered for a ticker, factor "would the
  user actually want to be assigned this stock" into the rationale — a rich premium on a
  business with a triggered red flag skews toward `Neutral`/`Skip`, not an automatic `Good`.

## Output schema

```json
{
  "agent": "agent_5_csp",
  "run_date": "YYYY-MM-DD",
  "results": [
    {
      "ticker": "NVDA",
      "state": "No Position",
      "iv_current": {"value": "48%", "as_of": "YYYY-MM-DD", "source": "https://..."},
      "iv_rank": {"value": 62, "basis": "52w_range", "as_of": "YYYY-MM-DD", "source": "https://..."},
      "hv_30": {"value": "41%", "source": "https://..."},
      "options_liquidity": {"open_interest": 0, "avg_volume": 0, "bid_ask_spread": "$0.05 (1.2% of premium)",
                            "assessment": "Excellent", "source": "https://..."},
      "support_proximity": {"nearest_support": "50-day MA at $X", "distance_pct": -4.2, "source": "https://..."},
      "suggested_strike": {"strike": 0.0, "pct_below_spot": 0.0, "approx_delta": 0.25},
      "suggested_dte": "30-45",
      "est_premium_yield": {"pct_of_capital": 0.0, "annualised_pct": 0.0, "note": "Indicative, not executable"},
      "assignment_context": "Strike is X% below estimated IV / at the lower third of the 52-week range.",
      "earnings_date": "YYYY-MM-DD",
      "event_risk": ["..."],
      "suitability": "Good",
      "rationale": "One line.",
      "confidence": "High",
      "sources": ["https://..."]
    }
  ],
  "notes": "Market-wide context, data limitations, timestamp caveats."
}
```

## Done when

Every ticker has a suitability verdict with a rationale; IV Rank is either properly sourced
or replaced by the documented fallback with `iv_rank_basis` set; liquidity is assessed with
real figures; earnings dates and event risks are captured; every value carries a source and
as-of date; and the JSON parses.
