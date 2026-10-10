# Milestone 1: Environment and Integration Discovery

**Project:** PAIDROCKET, autonomous crypto paper-trading system
**Date:** 2026-10-10
**Mode:** PAPER only. Nothing here places, signs or funds any real order.
**Status:** Report complete. Independent review is in `docs/review-log.md`.

Every claim below is labelled with how it was established:

- **[VERIFIED-LOCAL]**: checked by running a command in this environment.
- **[DOCS-SEARCH]**: taken from a web search summary of official or third-party documentation. The pages themselves could not be opened from this environment (see §1.3). These must be re-checked against the live API before Milestone 3 relies on them.
- **[UNVERIFIED]**: an assumption or estimate. Treat it as a placeholder.

---

## 1. Environment inspection

### 1.1 Runtime [VERIFIED-LOCAL]

| Item | Finding |
|---|---|
| OS | Ubuntu 24.04.5 LTS, Linux x86_64, 4 vCPU, 15 GiB RAM |
| Python | 3.13.16 with stdlib `sqlite3` (SQLite 3.45.1) |
| Preinstalled Python packages | numpy 2.5.3, pandas 3.0.5, pydantic 2.13.5, httpx 2.13.0, requests 2.34.2, starlette 1.6.0, uvicorn 0.53.0, Jinja2 3.1.6, sse-starlette 3.4.11 |
| Other toolchains | Node 22.22.0, Go 1.24.7, Rust 1.97.0 |
| Package registries | PyPI and npm reachable (HTTP 200) |
| `sqlite3` CLI | Not installed. The Python module is enough. |
| Repository | `bkai7612-dot/trader`, empty at the start of this milestone |

### 1.2 Execution model [VERIFIED-LOCAL]

- This is an **ephemeral cloud container**. It is reclaimed after inactivity, so anything not committed and pushed is lost.
- It is **not a 24/7 host**. It cannot run an autonomous trading loop continuously. Scheduled "Routines" exist, but their minimum interval is normally hourly and each firing has to rebuild state from the repository. That is fine for development and batch backtests, not for live position monitoring.
- **Consequence:** the system will be built here, tested here against recorded fixtures, and deployed to a host the user controls (their own PC or a small VPS) for continuous paper trading.

### 1.3 Network access: the main blocker [VERIFIED-LOCAL]

Every market-data and trading host tested was refused by the environment's egress policy with `CONNECT tunnel failed, response 403`:

| Host | Purpose | Result |
|---|---|---|
| `api.kraken.com` | Kraken REST | 403 blocked |
| `api.coinbase.com` | Coinbase Advanced Trade REST | 403 blocked |
| `api.exchange.coinbase.com` | Coinbase Exchange public data | 403 blocked |
| `api.binance.com` | Binance REST | 403 blocked |
| `lite-api.jup.ag` | Jupiter quote API | 403 blocked |
| `api.dexscreener.com` | DEX market data | 403 blocked |
| `pypi.org` | Python packages | 200 OK |

The web-fetch tool could not resolve documentation hosts either (`docs.kraken.com`, `docs.cdp.coinbase.com`, `dev.jup.ag`). Documentation research therefore came from web-search summaries only, hence the [DOCS-SEARCH] labels.

**What this means:**

1. No API connection has been verified. I am not claiming any endpoint works from here.
2. Real market data cannot be fetched in this environment until the user allows these hosts (cloud environment settings → Network access → Allowed domains; steps at <https://code.claude.com/docs/en/cloud-environments#network-access>). Hosts to allow:
   - Phase 1: `api.coinbase.com`, `api.exchange.coinbase.com`, `api.kraken.com`
   - Phase 2: `api.jup.ag` (and `lite-api.jup.ag` if still served), `api.dexscreener.com`, `api.rugcheck.xyz`, and a Solana RPC host (provider still to be chosen)
3. Until then, development continues with **unit tests on synthetic data and recorded API fixtures**, clearly labelled as such. I will not present synthetic data as market data or synthetic backtests as evidence of profitability.
4. Alternatively, the user can run the data-ingestion command on their own machine, which has normal internet access, and commit the resulting candle files.

---

## 2. Integration comparison (centralised exchanges)

All figures are [DOCS-SEARCH] unless marked otherwise. Fee schedules changed in both July 2026 (Kraken) and September 2026 (Coinbase), and the sources disagree, so every fee below must be confirmed from the account's own fee page or API before it is used for anything except conservative simulation.

| Criterion | Coinbase Advanced Trade | Kraken (Pro / Spot REST) | Binance |
|---|---|---|---|
| UK account eligibility | Available to UK retail. | Available to UK retail. UK entity Payward Ltd is reported on the FCA cryptoasset register, and Payward Services Ltd holds an FCA EMI licence (Kraken support page, Feb 2025). | **New UK users not onboarded since 2023.** An FCA application is reported for the regime opening 2027. Current status unconfirmed. **Excluded.** |
| Spot trading | Yes | Yes | n/a |
| GBP pairs | BTC-GBP, ETH-GBP (to confirm via `/products`) | BTC/GBP (XXBTZGBP), ETH/GBP (to confirm via `AssetPairs`) | n/a |
| Historical candles | `GET /api/v3/brokerage/market/products/{id}/candles`, public, **max 350 candles per request**, paginate by `start`/`end` | `GET /0/public/OHLC`, public, **max 720 most recent candles**, so it can't page back far. Kraken publishes downloadable bulk OHLCVT history files instead. | n/a |
| Real-time data | REST plus WebSocket | REST plus WebSocket v2 | n/a |
| Sandbox | **Static mock sandbox** at `api-sandbox.coinbase.com`, covering accounts and orders only, unauthenticated. Useful for testing request shapes, not fills. | No spot sandbox found | n/a |
| API key permissions | CDP API keys (JWT-signed) with scoped permissions (view / trade / transfer). A trade-only key is possible. | API keys with granular permissions. Withdraw can be left off. | n/a |
| Entry-tier fees (maker / taker) | ~0.40% / 0.60% [DOCS-SEARCH, sources disagree; Sept 2026 change] | **0.40% / 0.80%** after the 9 July 2026 restructure per several reviews. Older sources say 0.25% / 0.40%. The official table excerpt did not show the base tier. | n/a |
| Minimum order | Per-product `base_min_size` / `quote_min_size` from the API. **Value for BTC-GBP not confirmed.** | BTC 0.0001, ETH 0.01, GBP-side 5 GBP (support article, Feb 2026) | n/a |
| Liquidity (GBP books) | Thinner than USD/USDC books [UNVERIFIED] | Thinner than USD/EUR books [UNVERIFIED] | n/a |
| Rate limits | Not found in a reliable source. Design for conservative client-side throttling. | Counter-based limits. Exact values not found. | n/a |
| Operational cost | Public data free. Trading fees only. | Public data free. Trading fees only. | n/a |

### 2.1 Feasibility under the mandated risk rules at £50

These calculations are arithmetic and do not depend on any API.

- **Maximum single-position exposure = 10% of £50 = £5.00.**
- Kraken's BTC minimum is 0.0001 BTC. At a BTC/GBP price of P, that costs 0.0001 × P. This exceeds £5 whenever P > £50,000 (£6 at £60k, £8 at £80k, £10 at £100k). I could not fetch the current price, but if BTC/GBP is above £50,000, **a minimum BTC order on Kraken breaks the 10% exposure cap.** ETH's 0.01 minimum is likely above £5 too, depending on price.
- The risk engine will then **reject the trade**, as the spec requires. With the exposure cap as specified, a £50 Kraken account may be unable to trade BTC at all. This is a real constraint, not a bug.
- Coinbase's per-product minimums are reported to be small, but **this is unconfirmed for BTC-GBP**. It is the single most important fact to verify once the network is open.

### 2.2 Cost drag

- A round trip at taker rates costs ~1.2% (Coinbase at 0.60%) to ~1.6% (Kraken at 0.80%) **plus** the spread and slippage. A strategy needs an average gross gain above that just to break even. Maker (limit) entries roughly halve the fee, but some orders won't fill.
- Network fees don't apply to centralised trading. They apply only to Phase 2 Solana swaps.

### 2.3 The £1,000 target, plainly

With a 10% cap per position, a trade that gains +10% net adds only about 1% to equity. Compounding 1% gains needs about **301 consecutive winning full-size trades and no losses** to go from £50 to £1,000. With three concurrent full positions (limited by the 3% aggregate-risk cap), it still needs about 100. The risk rules make the target **very unlikely in any reasonable timeframe**. That is the intended trade-off: the rules prioritise survival. The system will not loosen them to chase the target, and the dashboard will show progress honestly without projected dates.

---

## 3. Recommendation: initial centralised exchange

**Recommended: Coinbase Advanced Trade**, with Kraken as a secondary public-data source behind the same adapter interface.

Reasons, in order of weight:

1. **Lower entry-tier taker fee** (~0.60% vs ~0.80% reported for Kraken after July 2026). At £5 position sizes, fees dominate.
2. **Paginated historical candles** from a public endpoint, so multi-year hourly history can be pulled through the same API used live. Kraken's REST OHLC returns only the latest 720 candles, and anything older needs its bulk download files.
3. **Small per-product minimums are likely** on Coinbase (to be verified). Kraken's 0.0001 BTC minimum probably conflicts with the 10% cap at £50 (§2.1).
4. **An official, if static, sandbox** for checking order request formats before any future live stage.
5. Both exchanges serve UK retail and allow trade-only keys. Binance is excluded for UK onboarding reasons.

**What could reverse this choice:** if Coinbase's verified BTC-GBP or ETH-GBP minimums, UK fees or GBP spreads turn out worse than Kraken's. The exchange adapter is an interface, so switching is a configuration change, not a rewrite.

**Pairs for Phase 1:** BTC-GBP and ETH-GBP. If GBP books prove too thin (spread check fails), consider BTC-USDC/ETH-USDC with an explicit GBP conversion cost. That decision is deferred until spreads are measured.

---

## 4. Proposed architecture

Python 3.13, single process, asyncio, SQLite (WAL mode). Chosen because the stack is already installed, it needs no external services, and a single process makes the "one authoritative execution service" requirement easy to enforce.

```
                ┌───────────────────────── Dashboard (Starlette/FastAPI + Jinja, local only) ─┐
                │  equity, target progress, positions, orders, risk status, freshness, controls │
                └────────────────────────────────────────────▲─────────────────────────────────┘
                                                             │ read-only queries + control commands
┌──────────────┐  candles/ticker  ┌──────────────┐ signals  ┌┴─────────────────────────────────────┐
│ Data layer   ├─────────────────►│ Strategies   ├─────────►│ ExecutionService  (single instance)   │
│ - Exchange   │  (closed candles │ (pure funcs, │ (intent, │  1. idempotency check (unique key)    │
│   adapters   │   only)          │  no I/O,     │  stop,   │  2. RiskGate  ← cannot be bypassed    │
│ - Candle     │                  │  no broker   │  reason) │  3. sizing (fees, slippage, cash)     │
│   store      │                  │  access)     │          │  4. Broker.submit()                   │
│ - Freshness  │                  └──────────────┘          │  5. Ledger update (atomic txn)        │
│   monitor    │                                            └──────┬───────────────────────────────┘
└──────────────┘                                                   │
                                                  ┌────────────────┴──────────────┐
                                                  │ PaperBroker (default)         │  LiveBroker: not built in
                                                  │ conservative fill model       │  Phase 1. Refuses to construct
                                                  └────────────────┬──────────────┘  unless the live gate passes.
                                                                   │
                                          paper.db (mode='PAPER' stamped; live.db is a separate file)
```

### 4.1 Key design rules

- **Strategies are pure.** A strategy receives closed candles up to time *t* and returns a signal object. It has no reference to the broker, ledger or risk config, so it **cannot** place orders or change limits.
- **RiskGate lives inside ExecutionService.** Every order path, including exits, goes through it. Limits load from a frozen config object at start-up and cannot change at runtime. Lowering a limit needs a restart. Raising one needs a restart *and* a config change recorded in the audit log.
- **Circuit breakers are persisted latches.** The daily-loss latch clears at 00:00 UTC. The drawdown latch only clears through an explicit manual review command that is logged with a reason.
- **Idempotency.** Each signal gets a deterministic key `(strategy, symbol, candle_close_ts, side)` with a UNIQUE constraint in the DB. Replays and duplicates are no-ops.
- **Fail closed.** Stale data (no fresh candle or ticker within 2× the expected interval), failed reconciliation, uncertain order state or DB errors all block **new entries**. Exits still run on the protective-stop policy.
- **Paper/live separation.** These are separate DB files. Each DB stamps its mode in a metadata table that is checked on open. The PaperBroker has no network client. A LiveBroker would need `PAIDROCKET_MODE=LIVE`, a signed approval file and credentials present. In Phase 1 it isn't implemented at all, and a test asserts that paper mode cannot construct it.
- **Ledger.** Double-entry style cash/position records, each stored with the label `PAPER`. Equity = cash + Σ(position qty × current **bid**). Positions are valued at the price they could be sold at, not the mid.
- **Restart recovery.** On start-up, rebuild state from the ledger, reconcile cash and positions against the transaction log, and refuse to trade if they disagree.

### 4.2 Conservative paper fill model

- **Market buy** fills at the **ask** plus slippage. **Market sell** fills at the **bid** minus slippage. Slippage = base bps + a size-dependent term from order-book depth where available, otherwise a fixed conservative value.
- **Limit orders** fill only when the price **trades through** the limit (touching is not enough). Fills are partial when the order is larger than a configured share of displayed depth.
- **Stops** trigger as market orders on the next observed price. On a gap, the fill is at the gapped price, never at the stop level. The dashboard and docs state that a stop does not guarantee an exit price.
- Fees use the **taker** rate unless a limit order genuinely rests. Fees are deducted in quote currency.
- Orders that break exchange minimums, precision or available cash are **rejected**, as the exchange would reject them.
- The backtester uses the *same* fill and risk code. Signals computed on candle *t* close execute no earlier than candle *t+1*, at its open plus modelled spread and slippage.

### 4.3 Proposed package layout (Milestone 3 onward)

```
paidrocket/
  config/        settings (pydantic), frozen RiskLimits, mode
  data/          adapters (coinbase_public, kraken_public), candle_store, freshness
  strategies/    base, sma_trend, rsi_reversion, donchian_breakout (each toggleable)
  risk/          sizing, limits, circuit_breakers
  execution/     service (single authority), paper_broker, fill_model
  portfolio/     ledger, valuation, reconciliation
  backtest/      engine, walk_forward, metrics, benchmark
  meme/          (Phase 2) discovery, screening, jupiter_quote_sim
  web/           dashboard
tests/           unit, property-based, scenario (crash, gaps, stale data, restart, ...)
docs/
```

---

## 5. Data requirements

| Data | Use | Source | Volume |
|---|---|---|---|
| 1h and 1d candles, BTC-GBP and ETH-GBP, ≥4 years | Backtests (train / validation / out-of-sample) | Coinbase public candles (paginated), cross-checked against Kraken bulk files | ~35k hourly candles per pair, a few MB |
| Best bid/ask (ticker), every 10–60s | Spread check, paper fills, freshness | Coinbase public ticker / WebSocket | Small |
| L2 order-book snapshot (top N levels) | Slippage and partial-fill model | Coinbase product book | Small, sampled |
| Product metadata (min sizes, increments, status) | Order validation | `/products` | Refreshed daily |
| Fee tier | Cost model | Account fee endpoint (needs a key, so live stage only). Until then, conservative config values (taker 0.80%) | n/a |
| Phase 2: token pairs, liquidity, volume, holders, authorities, quotes | Meme screening | DexScreener, RugCheck, Solana RPC, Jupiter quote | Small |

**Chronological split, fixed in advance:** the oldest 50% of history for training, the next 25% for validation and the most recent 25% for out-of-sample. The out-of-sample window is evaluated **once** per strategy configuration and is never used to pick parameters.

---

## 6. Evidence threshold for live-trading review (pre-registered)

Defined now, before any backtest is run, so the results can't shape it. A strategy is *eligible for live review* only if **all** of these hold:

1. Out-of-sample period of at least 12 months, containing at least one drawdown of 20% or more in the underlying asset.
2. At least 30 completed out-of-sample trades.
3. Out-of-sample net profit factor ≥ 1.2 **with costs stressed at 1.5×** the modelled fees and slippage.
4. Out-of-sample maximum drawdown ≤ 10% (matching the live drawdown stop).
5. Positive out-of-sample expectancy after costs, **and** a better return-to-max-drawdown ratio than buy-and-hold of the same asset over the same window.
6. Profitable in at least 2 of the 3 chronological sub-windows of the out-of-sample period, so one lucky stretch can't carry it.
7. At least 90 days of forward paper trading on live data with at least 20 trades, with a profit factor no worse than the 5th percentile of a bootstrap of the out-of-sample trades.

Meeting this threshold makes a strategy eligible for **review**. It is not proof of future profit.

---

## 7. Initial cost estimates

| Item | Cost | Basis |
|---|---|---|
| Market-data APIs (Coinbase / Kraken public, DexScreener, RugCheck) | £0 | Public endpoints [DOCS-SEARCH] |
| Jupiter API (Phase 2, quotes only) | £0 on free tier, needs a free API key | [DOCS-SEARCH]. Confirm the tier and rate limit. |
| Solana RPC (Phase 2) | £0 on a provider's free tier | [UNVERIFIED]. Choose a provider. |
| Host for 24/7 paper trading | £0 on the user's PC, or a small VPS at typically a few pounds a month | [UNVERIFIED]. Check provider pricing. |
| Paper trading | £0. No funds involved. | n/a |
| Future live trading costs | ~1.2–1.6% per round trip at taker rates plus spread (CEX). Solana base fee plus priority fee per swap (Phase 2). | §2.2 |

---

## 8. Key limitations and risks

1. **No live data from this environment** until the hosts in §1.3 are allowed.
2. **No continuous runtime here.** Autonomous paper trading needs a user-controlled host.
3. **Fees and minimums are unverified.** The sources conflict and changed in 2026. The simulator will use conservative defaults (taker 0.80%) until the live API values are recorded.
4. **£50 with a 10% position cap is near or below exchange minimums.** Many signals will rightly be rejected.
5. **The £1,000 target is mathematically remote** under the mandated limits (§2.3).
6. **Stops are not guaranteed.** Gaps and illiquidity can cause losses far larger than the planned 1%.
7. **Backtests are not forecasts.** The threshold in §6 reduces overfitting risk but cannot remove it.
8. **Phase 2 (meme coins) carries much higher risk.** Security scores and audits are not proof of safety. Most candidates are expected to be rejected.

---

## 9. Development plan

| # | Milestone | Done when | Needs from user |
|---|---|---|---|
| 1 | Environment and docs discovery | This report plus independent review | n/a |
| 2 | Exchange selection | Coinbase adapter contract written. Fees, minimums and spreads **verified via the live public API** and recorded. | **Allow network hosts** (§1.3), or run the fetch script locally |
| 3 | Data, ledger, backtester | Candle ingestion, ledger, fill model, risk sizing and backtest engine, with unit and property tests passing | n/a |
| 4 | Paper engine | ExecutionService, RiskGate, circuit breakers, restart recovery, kill switch | n/a |
| 5 | Independent tests | Agent B runs the 20-scenario suite (spec §13). Critical defects fixed. | n/a |
| 6 | Dashboard, persistence, notifications | Dashboard as specified, labelled PAPER throughout | n/a |
| 7 | Meme module (paper) | Discovery, screening, Jupiter quote simulation, rejection rules | Allow Phase 2 hosts. Free Jupiter API key (only via env var, later). |
| 8 | Combined tests | Portfolio-level, liquidity and risk scenario tests pass | n/a |
| 9 | Docs | Setup guide, operating manual, config template, test report | n/a |
| 10 | Live-readiness dossier | Evidence against §6, security review, explicit approval checklist. **Live stays disabled.** | Decision only |

Each milestone ends with a distinct reviewer stage (Agent B), recorded in `docs/review-log.md` with defects, severity, evidence, fixes and approval status. A milestone doesn't pass on the developer's own approval.

---

## Sources (via web search; pages not directly reachable from this environment)

- Kraken historical data guide: <https://docs.kraken.com/exchange/guides/general/historical-data>
- Kraken fee tier changes (July 2026): <https://support.kraken.com/articles/cross-platform-fee-tier-changes>
- Kraken trading fees: <https://support.kraken.com/articles/201893638-how-trading-fees-work-on-kraken>
- Kraken minimums: <https://support.kraken.com/articles/205893708-Minimum-order-size-volume-for-trading>
- Kraken UK funding / EMI licence: <https://support.kraken.com/au/articles/changes-to-funding-for-uk-retail-clients-feb-2025>
- Kraken fee reviews (2026): <https://www.datawallet.com/crypto/kraken-fees-explained>, <https://cryptoslate.com/crypto-exchanges/kraken-exchange-review/>, <https://www.dollarscout.net/crypto/reviews/kraken/>
- Coinbase public candles: <https://docs.cdp.coinbase.com/api-reference/advanced-trade-api/rest-api/public/get-public-product-candles>
- Coinbase Advanced Trade sandbox: <https://docs.cdp.coinbase.com/coinbase-app/advanced-trade-apis/sandbox.md>
- Coinbase fee change (Sept 2026): <https://www.securities.io/coinbase-lowers-advanced-trading-fees-with-tiers-starting-at-10-000/>
- Coinbase/Kraken fee comparison: <https://brokerchooser.com/news/coinbase-vs-kraken-fee-structures-and-trading-features-compared--bbeea493>
- Binance UK onboarding: <https://www.theblock.co/post/257721/binance-to-temporarily-stop-accepting-new-uk-users-after-fca-restriction>, <https://en.cryptonomist.ch/2026/08/17/binance-fca-license-uk/>
- Jupiter integration and migration: <https://tessl.io/registry/skills/github/jup-ag/agent-skills/jupiter-swap-migration>
- DexScreener API wrapper notes: <https://github.com/opensvm/dexscreener-mcp-server>
- Solana token safety guide: <https://docs.solanatracker.io/guides/token-safety.md>
