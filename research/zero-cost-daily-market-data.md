# US zero-cost daily market-data research (secondary market only)

> **Scope correction (2026-09-28):** This investigation covered US-listed equities only. The product's primary universe is now UK and European exchange listings; US listings are secondary. This document is retained as the US-secondary comparison and is not the provider recommendation for the primary UK/European market. See the open [UK/European provider comparison](https://github.com/kchapl/greentech/issues/21) and [regional exchange-scope decision](https://github.com/kchapl/greentech/issues/28).

**Research ticket:** [Compare US zero-cost daily market-data vendors (secondary market)](https://github.com/kchapl/greentech/issues/10)
**Map:** [greentech: spec for a mirror-and-forecast green-tech investing app](https://github.com/kchapl/greentech/issues/9)  
**Accessed:** 2026-09-28  
**Scope:** One person, US-listed equities as a secondary universe, daily OHLCV, curated 20–50-name watchlist, zero recurring spend, eventual display in a single-user app, no brokerage orders.

## US-secondary finding only

For a US-secondary pilot, **Alpaca Basic** was the strongest zero-spend candidate behind a small server-side data adapter. **This does not recommend Alpaca for the primary UK/European listings.** It is the strongest fit found for the settled zero-cost constraint: its official plan page states historical US stock data since 2016 and 200 historical API calls/minute. Its FAQ says historical SIP data can be queried when the request's `end` is at least 15 minutes old. Its multi-symbol historical-bars endpoint can fetch bars for a symbol list. For end-of-day requests, a single-symbol-at-a-time daily quota is not an obvious blocker.

This is a **candidate pending a thin integration test**, not a guarantee that every desired symbol, price adjustment mode, account configuration, display use, or future API behavior is covered. Confirm the chosen feed and corporate-action adjustment semantics against actual responses before treating data as backtest-grade. Alpaca's free real-time feed is IEX-only (one venue); do not confuse it with consolidated SIP coverage.

The server-side adapter is recommended because Alpaca data calls require both an API key ID and secret in request headers. Neither credential should be shipped in Scala.js/browser assets. The provider documentation reviewed did not establish a CORS guarantee; do not make browser CORS a required assumption. This means the existing server-side-component decision can now be considered without waiting for the vendor survey.

## Comparison

| Provider | Published free allowance / history | Material constraint for greentech | Assessment |
|---|---|---|---|
| **Alpaca Basic** | 200 historical calls/min; US stocks & ETFs; historical timeframe since 2016. Free real-time stock coverage is IEX. FAQ documents SIP historical queries when `end` is at least 15 minutes old. Historical bars endpoint accepts a list of symbols. | API requires key ID + secret headers. The docs distinguish free real-time IEX from delayed historical SIP; verify requested feed and actual daily-bar results with the account before relying on consolidated data. CORS not verified. | **Best US-secondary pilot candidate**, via server-side adapter. Fits batch daily refresh and keeps credentials out of the browser. Zero spend requires accepting non-real-time prices. |
| **Alpha Vantage** | Free stock API: 25 requests/day. Free `TIME_SERIES_DAILY` is limited to latest 100 points (`compact`); full history is premium. `TIME_SERIES_DAILY_ADJUSTED` is explicitly premium. | Latest 100 daily bars is roughly five trading months, insufficient for serious multi-year strategy evaluation. 20–50 symbols individually exceed or consume most of the daily quota. Adjusted daily history is not free. API key is sent as `apikey` parameter, so browser use exposes it. | **Not suitable** for backtest-ready daily history under the zero-spend constraint. |
| **Twelve Data Basic** | Free Basic lists 8 API credits/minute and 800/day; usage weight is per symbol and endpoint. | Official pricing says Basic is for testing/evaluation/development, data may not be displayed to users or used in production. Thus it cannot power the intended displayed app on the free tier. API uses a credential; direct browser use exposes it. CORS not verified. | Useful for a spike, **not the production-data choice** at zero spend. |
| **Financial Modeling Prep Basic** | Free plan: 250 calls/day, EOD historical data, published 5-year range. | Its current terms restrict the free personal-use license to an individual and prohibit integrating data into third-party-accessible tools/apps or displaying it on applications designed for multiple people without written approval. Not a safe assumption for a web app with an internet-accessible URL; usage/display rights need confirmation. API-key handling also favors a server-side adapter. | Potentially attractive for a private personal experiment; **do not select for a deployed app without clarifying display/licensing terms**. |
| **Stooq downloads** | Publishes free daily historical files. | Stooq states the data is solely for personal use and any commercial use is prohibited. No API contract, CORS support, or SLA was established in the docs reviewed. | Possible local/private exploratory source, but weak fit for an app's dependable data adapter. |

## Important distinctions

- **Historical daily bars are not live prices.** This product is explicitly an end-of-day mirror. A delayed or prior-close value must carry its as-of timestamp in the UI; do not label it "live".
- **IEX is one venue; SIP is consolidated US market data.** Free real-time IEX bars may differ materially from all-venue volume. Alpaca's Market Data FAQ describes historical SIP requests whose `end` is at least 15 minutes old; the app should explicitly request/verify the intended feed rather than silently use the default.
- **Free API access is not automatically a production display license.** Twelve Data Basic and FMP Basic have explicit limitations relevant to app display/deployment. Keep the deployment private and personal in scope unless display rights are separately established.
- **A public source repository is not itself public data display**, but a public deployed app is. Still keep all API credentials out of checked-in source and browser bundles.
- **No CORS claim is made here.** Official pages reviewed establish APIs and authentication, but not an explicit browser-CORS guarantee for these candidates. Even with CORS, secrets in browser code are exposed; the server-side adapter is the safer architecture.
- **Provider selection does not settle backtest validity.** Survivorship bias, corporate actions, delistings, historical universe membership, realistic costs, and avoiding look-ahead are covered by the separate backtesting-validity ticket.

## US-secondary pilot acceptance checks

Before adopting Alpaca for any US-secondary symbols, make a minimal, non-production test with a small set of representative watchlist symbols:

1. Request multi-symbol `1Day` bars with an explicit historical feed and a time range ending more than 15 minutes ago; establish whether SIP is available under Basic for the signed-up account.
2. Compare timestamps, missing sessions, OHLCV and volume against an independent trusted reference for several liquid and less-liquid names.
3. Determine which adjustment modes are available and how split/dividend adjustments affect OHLCV; record the chosen semantics and provider version/date.
4. Exercise paging and rate-limit handling for the full 20–50 symbol watchlist.
5. Keep credentials server-side; verify account/data terms permit the intended private single-user display and any eventual hosting model.
6. Persist the provider's as-of timestamp and make stale/missing bars visible rather than silently presenting them as current.

## Primary sources

1. Alpaca, [About Market Data API](https://docs.alpaca.markets/us/docs/about-market-data-api) — Basic plan is free; US equities, IEX real-time coverage; historical data since 2016; 200 historical calls/minute; authentication headers require key ID and secret.
2. Alpaca, [Market Data FAQ](https://docs.alpaca.markets/us/docs/market-data-faq) — explains IEX versus SIP and states historical SIP requests require `end` to be at least 15 minutes old without subscription.
3. Alpaca, [Historical Stock Data](https://docs.alpaca.markets/us/docs/historical-stock-data-1) — feed selection and source descriptions.
4. Alpaca, [Historical bars API reference](https://docs.alpaca.markets/us/reference/stockbars) — multi-symbol historical bars endpoint; response paging behavior.
5. Alpha Vantage, [Customer Support](https://www.alphavantage.co/support/) — 25 API requests/day for free stock API; notes real-time and 15-minute delayed US data are premium.
6. Alpha Vantage, [API Documentation](https://www.alphavantage.co/documentation/) — `TIME_SERIES_DAILY` raw OHLCV, free latest-100 compact versus premium full history; `TIME_SERIES_DAILY_ADJUSTED` premium; API key request parameter.
7. Twelve Data, [Individual Pricing](https://twelvedata.com/pricing) — Basic free tier 8 API credits/minute, 800/day, non-display/testing/development-only use; separate paid tiers for display/EOD data.
8. Twelve Data, [Commercial and personal usage](https://support.twelvedata.com/en/articles/5332349-commercial-and-personal-usage) — individual plans are personal/internal only; restrictions on redistribution and commercial third-party display.
9. Financial Modeling Prep, [Pricing](https://site.financialmodelingprep.com/developer/docs/pricing) — Basic free tier 250 calls/day, EOD history, 5-year range; licensing caveat on data display/redistribution.
10. Financial Modeling Prep, [Terms of Service](https://site.financialmodelingprep.com/developer/docs/terms-of-service) — individual personal-use conditions and restrictions on third-party-accessible applications and display.
11. Stooq, [Free Historical Market Data](https://stooq.com/db/h/) — download access and personal-use-only statement.
