# Replace Yahoo Finance with SEC EDGAR + Marketstack — Design

**Date:** 2026-10-02
**Branch:** `05-yfinance-replacement`
**Status:** Design approved in brainstorming; awaiting written-spec review.

## 1. Goal and success criteria

Yahoo Finance data may not be used for commercial display, and Intrinsica is a public,
commercial product (live at intrinsica.io). Replace it with a licensed, in-budget stack:

- **Fundamentals:** SEC EDGAR XBRL (`data.sec.gov` `companyfacts` + `submissions`) — public domain, free.
- **Prices:** Marketstack (Basic $9.99/mo to start; Professional $49.99/mo if volume or real-time needs it).
- **Risk-free rate:** a fixed config value — no third external dependency.

**Success = defensible results, not parity with Agent Stock.** Intrinsica may diverge from
Agent Stock (which stays on Yahoo) wherever EDGAR is arguably more correct. Signals with no
source in the new stack are dropped, and the canary breaks this causes are fixed afterwards
with statement-only logic. Every number must trace to a filed or traded fact.

Budget: about $50/month. No licensed full-fundamentals vendor with public-display rights fits
it (Tiingo $50 is internal-use only; EODHD commercial starts at $399; Marketstack Business with
fundamentals is $149.99; FMP display needs a separate licence). EDGAR + Marketstack is the
in-budget licensed route.

## 2. Decisions

| # | Decision |
|---|---|
| D1 | **Approach A, an adapter behind a flag:** new backends output the *same* shapes the engines read today (Yahoo-keyed `info` dict + `{"years","rows"}` statements with Yahoo row labels). Engines are unchanged in Phase 1. |
| D2 | `DATA_PROVIDER=yahoo\|sec` selects the backend. Production stays `yahoo` until Phase 3. |
| D3 | **Lost signals are dropped, not synthesised:** forward EPS/PE, PEG, analyst targets/count/recommendation, insider %, business summary. Rules and metrics needing them don't fire (existing None-paths). R-R axes redistribute weight across the metrics that remain. Earnings yield uses trailing P/E. |
| D4 | **Risk-free rate fixed at `RISK_FREE_RATE = 0.043`** (the existing `DEFAULT_RISK_FREE` fallback), reviewed quarterly or when the real 10Y drifts >50 bp. It applies under *both* providers, so the `^TNX` fetch is removed and doesn't contaminate the comparison. |
| D5 | **Non-USD filers are not supported** in this project (e.g. TSM reports in TWD; Marketstack has no FX). They return `unsupported_currency` and the UI shows "data not available", never a wrong number. |
| D6 | Fixed **4-year annual statement window**, the same length as Yahoo's, so trend/CAGR logic is unchanged. |
| D7 | Agent Stock is not modified. |
| D8 | Gate: no Marketstack code until Marketstack confirms public-display rights **in writing**. |

### 2.1 Where the risk-free rate is used (justifies D4)

`rf` enters only `cost_of_equity = rf + β × ERP(5%)` in `screener/metrics.py:wacc`. WACC then feeds:

- **Quality** Section II: ROIC−WACC spread and the goodwill-floor crossover (`screener/scoring.py`).
- **Moat:** the WACC hurdle (`moat/scoring.py`).
- **Fair Value:** `blended_discount_rate` (30% blend toward WACC, clamped 8.5–13%) and the
  `quality_margin_of_safety` durability ramp (`valuation/models.py`).
- **Owner Earnings Yield:** display only.
- **Risk/Reward:** not used.

A 25 bp error in `rf` moves the spread by about 0.25 pp and the FV discount rate by about 7 bp
(<1% on FV). That's immaterial.

## 3. Architecture

```
engines (valuation / screener / moat / risk_reward / landing)   ← unchanged in Phase 1
        ▼  same shapes as today
services/provider.py            facade; the only data import point for engines
   ├── yahoo backend            today's services/yahoo.py + services/statements.py, moved, unchanged
   └── sec backend
        ├── edgar/client.py     ticker→CIK map, companyfacts + submissions fetch, throttle, cache
        ├── edgar/concepts.py   XBRL tag → Yahoo row label (ordered fallback chains, as data)
        ├── edgar/derive.py     EBIT, EBITDA, Total/Net Debt, Invested Capital, TTM, Q4 = FY − 9M
        ├── edgar/sector.py     SIC → Yahoo-style sector/industry; payment-network override list
        ├── marketstack.py      EOD history, latest quote (batched), splits, dividends, SPY
        └── adapter.py          assembles the Yahoo-shaped info dict + statement dicts
```

The facade exposes today's function names and signatures: `fetch_ticker_info`, `fetch_quote`,
`fetch_income_stmt`, `fetch_balance_sheet`, `fetch_cashflow_annual`, `fetch_price_monthly`,
`fetch_price_daily`, `fetch_ticker_cashflow`, `fetch_quarterly_revenue`,
`fetch_ev_ebitda_history`, `extract_financials`, `real_fcf`, `validate_ticker`,
`format_financial_block`. The ~12 engine import sites switch from `services.yahoo` /
`services.statements` to `services.provider`. That's the only change to engine files in Phase 1.
Pure helpers (`real_fcf`, `ev_ebitda_history_ewma`, `_effective_shares`, `statements_predate_split`)
are provider-neutral and stay shared.

## 4. EDGAR mapping and derivation

### 4.1 Fact selection
- Forms: 10-K, 10-Q, 20-F, 40-F (and their amendments). Units: `USD` (`USD/shares` for EPS).
- **Annual** = duration about 365 days (±30) with `fp=FY`. **Quarterly** = duration about 90 days, or
  year-to-date values differenced into discrete quarters.
- **Restatements:** among facts with the same concept, period end and duration, the latest `filed` wins.
- Annual window: the 4 most recent fiscal years, newest first; `years` are fiscal-year-end years.

### 4.2 Row mapping
Each Yahoo row label maps to an ordered chain of XBRL concepts. The first one with a value for
that period wins. Chains live in `concepts.py` as plain data. Rows the engines read today:

Total Revenue, Gross Profit, Operating Income, Net Income, EBIT, EBITDA, Interest Expense,
Diluted EPS, Diluted Average Shares, Tax Rate For Calcs, Operating Cash Flow, Capital
Expenditure, Free Cash Flow, Repurchase Of Capital Stock, Cash Dividends Paid, Stock Based
Compensation, Goodwill, Other Intangible Assets, Goodwill And Other Intangible Assets, Invested
Capital, Tangible Book Value, Net Debt, Total Debt, Cash Cash Equivalents And Short Term
Investments, Ordinary Shares Number, Share Issued.

Example chain: `Total Revenue` → `Revenues` → `RevenueFromContractWithCustomerExcludingAssessedTax`
→ `SalesRevenueNet` → (banks) `InterestAndDividendIncomeOperating + NoninterestIncome`.

### 4.3 Derived lines (matching Yahoo's definitions)

| Row | Definition |
|---|---|
| EBIT | Pre-tax income + interest expense |
| EBITDA | EBIT + depreciation & amortisation |
| Total Debt | LT debt (current + non-current) + short-term borrowings + finance leases |
| Net Debt | Total Debt − cash & equivalents |
| Invested Capital | Total Debt + common equity |
| Tangible Book Value | Equity − goodwill − intangibles |
| Free Cash Flow | Operating cash flow − capex |
| Tax Rate For Calcs | Tax ÷ pre-tax income; `None` outside 0–0.6 (the engine falls back to 21%) |

The Phase 0 spike checks each definition against Yahoo's values. **Where Yahoo's definition
differs, Yahoo's is adopted** (leases are the likely case), so these lines don't create artificial
differences.

### 4.4 Trailing twelve months
- Flow figures = the sum of the last 4 discrete quarters. Q4 = FY − 9M year-to-date. Cash-flow
  10-Q values are year-to-date and are differenced.
- Balance-sheet figures = the latest reported value.
- Annual-only filers: TTM = latest FY, and the result carries `ttm_basis="annual"`.

### 4.5 Shares
`dei:EntityCommonStockSharesOutstanding`, summed across all share classes from the latest filing
cover page. This gives the true multi-class total (BRK-B, GOOGL, MBLY). `_effective_shares`
remains as a safety net.

### 4.6 Sector and industry
- `submissions.sic` maps to Yahoo's sector/industry strings through a table in `sector.py`, so
  `"Financial Services"`, `"Real Estate"`, `CYCLICAL_SECTORS` and `CORE_FINANCIAL_INDUSTRIES`
  keywords keep working in `valuation/classifier.py` and `screener/gics.py`.
- `_is_payment_network` currently reads `long_business_summary`, which is lost. It becomes
  SIC + an explicit ticker override list (V, MA, PYPL, AXP to start; extended when the canary run finds a misclassified payment name).

### 4.7 Non-USD filers
If the primary reporting unit for revenue is not `USD`, the adapter raises a typed
`UnsupportedCurrency` error. The engines treat it like "not found", and the UI shows "data not available".

## 5. Marketstack and the price-based `info` fields

| Yahoo key | How it's built |
|---|---|
| `currentPrice`, `regularMarketPrice` | Latest Marketstack quote |
| `marketCap` | Price × total EDGAR shares |
| `enterpriseValue` | Market cap + Total Debt − cash |
| `trailingPE`, `trailingEps` | Price ÷ TTM diluted EPS; TTM diluted EPS |
| `priceToSalesTrailing12Months` | Market cap ÷ TTM revenue |
| `enterpriseToEbitda`, `enterpriseToRevenue` | EV ÷ TTM EBITDA, EV ÷ TTM revenue |
| `fiftyTwoWeekHigh`, `twoHundredDayAverage` | From daily adjusted closes |
| `beta` | 60 monthly returns vs SPY (Yahoo's method); `BETA_CEILING` still applies |
| `dividendRate`, `payoutRatio` | Sum of the last 4 Marketstack dividends; dividends ÷ net income |
| `grossMargins`, `operatingMargins`, `profitMargins` | TTM ratios as fractions |
| `returnOnEquity`, `returnOnAssets` | TTM net income ÷ equity / assets, as fractions |
| `currentRatio`, `quickRatio` | Latest balance sheet |
| `debtToEquity` | **In percent** (Yahoo's scale) |
| `revenueGrowth`, `earningsGrowth` | **Latest quarter vs the same quarter a year earlier** (Yahoo's basis; guards depend on it) |
| `totalDebt`, `totalCash`, `totalRevenue`, `ebitda`, `freeCashflow`, `operatingCashflow`, `interestExpense`, `effectiveTaxRate`, `bookValue`, `sharesOutstanding` | TTM / latest EDGAR values |
| `symbol`, `shortName`, `longName`, `sector`, `industry` | Ticker, EDGAR entity name, SIC mapping |
| `forwardEps`, `forwardPE`, `pegRatio`, `trailingPegRatio`, `targetMeanPrice`, `targetHighPrice`, `targetLowPrice`, `numberOfAnalystOpinions`, `recommendationKey`, `heldPercentInsiders`, `heldPercentInstitutions`, `longBusinessSummary` | **`None`** (D3) |

- **Price history:** one adjusted daily pull of 6 years per ticker (2 requests at 1,000 rows each).
  The 1-year daily series (R-R) and the 6-year monthly series (screener, EV/EBITDA history) are
  both cut from it. Marketstack splits feed `statements_predate_split`.
- **Request budget:** about 2–3 Marketstack requests per evaluation. The landing fast refresh is one
  batched multi-symbol request (about 2,900/month). Basic's 10,000/month is enough at launch.

## 6. Caching

The landing result cache (`landing/cache.py`: 7-day results, 4-hour price refresh in production)
is unchanged. The data-fetch layer underneath becomes:

| Data | TTL |
|---|---|
| EDGAR `companyfacts`, `submissions` | 7 days |
| Ticker → CIK map (`company_tickers.json`) | 7 days |
| Daily price history, SPY history | Until the next US market close |
| Latest price for the engines | 4 hours |
| `fetch_quote` | Uncached by design; the landing layer refreshes every 4 hours (see the `fetch_quote` docstring and memory note) |

Caches live in process memory, so they're lost when Cloud Run scales an instance down. A cold
fetch costs about 1.5 s. Persistent caching is out of scope.

## 7. Errors and limits

- EDGAR: `SEC_USER_AGENT` header (with a contact email) is required. Throttled to under 10 req/s.
  10 s timeout, 2 retries with backoff on 429/5xx. An unknown ticker or CIK raises the same
  "not found" `ValueError` as Yahoo today.
- Marketstack: `MARKETSTACK_API_KEY` from Secret Manager. Same timeout and retry policy.
- Missing price → the ticker fails (as today). A missing statement row → `None`.
- Performance targets per ticker for data assembly: **≤ 1.5 s cold, ≤ 50 ms warm**. Measured today:
  EDGAR `companyfacts` 0.39–0.54 s for AAPL, NVDA and JPM; Yahoo about 2–3 s across 8 calls.

## 8. Testing and validation

### 8.1 Unit tests (offline, TDD)
- Recorded fixtures under `backend/tests/fixtures/` for AAPL, JPM, TSM, BRK-B, O (EDGAR + Marketstack).
- Tests cover: fallback chains, latest-filed restatements, Q4 = FY − 9M, YTD cash-flow differencing,
  multi-class shares, `UnsupportedCurrency`, SIC → sector, payment-network overrides, and
  `debtToEquity` / growth-basis units.
- **Shape test:** the adapter's output has exactly the keys, row labels and units the engines read.
- **Unchanged-behaviour test:** with `DATA_PROVIDER=yahoo` the existing suite passes untouched.

### 8.2 Phase 0 spike (live, 5 tickers)
Compare EDGAR-derived values with Yahoo: ±2% for figures taken directly from filings, ±5% for
derived lines. Record any definition changes back into §4.3. Measure cold and warm latency.

### 8.3 Canary comparison (live, 36 tickers)
**Tickers:** IREN, NBIS, KLAC, AAPL, JPM, NVDA, SNPS, ANET, TEM, BWXT, KO, NFLX, AVGO, CRWV,
HON, CF, CCJ, AXP, NXT, MBLY, KVYO, GOOGL, PLTR, CDNS, OPFI, STRL, LITE, LYFT, SOFI, VST, WDC,
HOOD, AMD, TSM, BRK-B, O.

`scripts/provider_compare.py` runs three columns in one session, close together in time:

| Column | Source |
|---|---|
| AS | Agent Stock `validate_ticker.py --inputs` (Yahoo), run as a subprocess from its own repo |
| IY | Intrinsica, `DATA_PROVIDER=yahoo` |
| IS | Intrinsica, `DATA_PROVIDER=sec` |

AS vs IY = code differences between the repos (expected to be about zero). **IY vs IS = the
provider effect.** AS vs IS = the user-visible difference.

| Level | Flagged when |
|---|---|
| About 40 input fields | Differs by more than 5% (2% for figures taken directly from filings) |
| Fair Value | FV differs by more than 10%, or the verdict or stock-type tier changes |
| Quality | Overall differs by more than 0.5, or any section by more than 1.0 |
| Moat | Differs by more than 1.0 |
| Risk/Reward | Tier changes |

### 8.4 Triage
1. Flags with a **known structural cause** (a D3 lost signal, annual-only TTM, D5 currency) are
   labelled expected.
2. Every other flag goes through `/validating-agent-stock` on both AS and IS. The verdict is one of:
   **EDGAR more defensible** (accept, record why), **mapping bug** (fix in `concepts.py`/`derive.py`,
   re-run), or **engine rule broken by a lost signal** (Phase 2 fix).
3. Phase 2 fixes reuse existing patterns, carry a worked numeric before→after example, must leave
   the IY column unchanged, and are followed by a full 36-ticker re-run.

### 8.5 Report and exit criteria
One HTML report: three columns per ticker and per field, with flags linked to verdict notes,
published as a private artifact. **Exit criteria:** every flag is expected, accepted with a reason,
or fixed; only non-USD filers are `unsupported_currency`; latency targets are met.

## 9. Phases

| Phase | Scope | Done when |
|---|---|---|
| 0 | Gate 1: Marketstack written display confirmation. Gate 2: 5-ticker spike (§8.2) | Both gates pass; spec updated with findings |
| 1 | Facade + fixed `rf`; EDGAR client, mapping, derivation, sector; Marketstack client; adapter; compare script + first report | Unit tests green; `yahoo` suite unchanged; first report published. Merged to `main` via PR with production still on `yahoo` |
| 2 | Canary fixes, one at a time (§8.4) | Exit criteria in §8.5 |
| 3 | Secrets + env on Cloud Run (`MARKETSTACK_API_KEY`, `SEC_USER_AGENT`, `DATA_PROVIDER=sec`); DEPLOY.md; "Recalculate all" so the Sheets-backed grid serves new results; footer attribution per Marketstack's reply; "—" for lost fields; Owner Earnings label "vs 4.3% risk-free" | Live on `sec` |
| 3b | After 2 clean weeks: delete `yahoo.py`, `statements.py`, `yf_pool.py`, rate-limit tests, the `yfinance` dependency and the flag | yfinance removed |

**Rollback** (only during the 2-week window): set `DATA_PROVIDER=yahoo` and redeploy. This is for
emergencies only, because it restores the unlicensed source.

If Gate 1 fails, stop before any Marketstack work and revisit price providers. The EDGAR half of
this design stands on its own.

## 10. Out of scope
FX / non-USD filers; persistent (restart-surviving) caching; incremental price-bar fetches;
any change to Agent Stock; new UI beyond "—" placeholders and the attribution line;
re-introducing forward or analyst data from another source.

## 11. Risks

| Risk | Mitigation |
|---|---|
| Marketstack terms don't allow public display | Gate 1, before any Marketstack code |
| XBRL tag variety and custom tags leave rows empty | Fallback chains as data; the canary report shows gaps per field |
| Guards tuned on Yahoo values shift | IY vs IS isolates the provider effect; Phase 2 fixes, validated per ticker |
| Bank and insurer statements differ structurally | JPM, AXP, SOFI, OPFI, BRK-B in the canary set; bank-specific chains |
| Marketstack price-data quality (adjustments, gaps) | Spike compares adjusted closes with Yahoo; split guard retained |
| Cold-start latency after scale-to-zero | ≤ 1.5 s cold target, measured in Phase 0 and the canary run |
