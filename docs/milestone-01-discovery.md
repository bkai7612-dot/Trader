# Milestone 1: Environment and Integration Discovery

**Project:** PAIDROCKET, autonomous crypto paper-trading system
**Date:** 2026-10-10
**Mode:** PAPER only. Nothing here places, signs or funds any real order.
**Status:** Revision 2. Revision 1 was reviewed by Agent B (FAIL with conditions: 14 defects). This revision answers each defect; see `docs/review-log.md`. It is awaiting re-review.

Every claim below is labelled with how it was established:

- **[VERIFIED-LOCAL]**: checked by running a command in this environment.
- **[DOCS-SEARCH]**: taken from a web search summary of official or third-party documentation. The pages themselves could not be opened from this environment (see §1.3). These must be re-checked against the live API before Milestone 3 relies on them.
- **[PLATFORM-DOCS]**: taken from this platform's own documentation or tool descriptions, not tested by a command.
- **[UNVERIFIED]**: an assumption or estimate. Treat it as a placeholder.

---

## 1. Environment inspection

### 1.1 Runtime [VERIFIED-LOCAL]

| Item | Finding |
|---|---|
| OS | Ubuntu 24.04.5 LTS, Linux x86_64, 4 vCPU, 15 GiB RAM |
| Python | 3.13.16 with stdlib `sqlite3` (SQLite 3.45.1) |
| Preinstalled Python packages | numpy 2.5.3, pandas 3.0.5, pydantic 2.13.5, requests 2.34.2, starlette 1.6.0, uvicorn 0.53.0, Jinja2 3.1.6, sse-starlette 3.4.11. The installed HTTP package is **`httpx2` 2.13.0**: `import httpx` fails and `import httpx2` works. |
| Not installed (needed later) | pytest, hypothesis, FastAPI. These will be installed from PyPI and pinned in a lock file at Milestone 3. The dashboard can run on the preinstalled Starlette, so FastAPI is optional. |
| Other toolchains | Node 22.22.0, Go 1.24.7, Rust 1.97.0 |
| Package registries | PyPI and npm reachable (HTTP 200) |
| `sqlite3` CLI | Not installed. The Python module is enough. |
| Repository | `bkai7612-dot/Trader`, empty at the start of this milestone |

### 1.2 Execution model [PLATFORM-DOCS]

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
| `api.jup.ag`, `api.rugcheck.xyz` | Jupiter, RugCheck | 403 blocked (re-tested by Agent B) |
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

All figures are [DOCS-SEARCH] unless marked otherwise. Fee schedules changed in both July 2026 (Kraken) and September 2026 (Coinbase), and the sources disagree. Every fee below must be confirmed from the account's own fee page or the API before it is used for anything except conservative simulation.

| Criterion | Coinbase Advanced Trade | Kraken (Pro / Spot REST) | Binance |
|---|---|---|---|
| UK account eligibility | Serves UK retail [UNVERIFIED: no source retrieved. Confirm FCA cryptoasset register entry in Milestone 2.] | Serves UK retail. UK entity Payward Ltd is reported on the FCA cryptoasset register, and Payward Services Ltd holds an FCA EMI licence (Kraken support page, Feb 2025). | **New UK users not onboarded since 2023.** An FCA application is reported for the regime opening 2027. Current status unconfirmed. **Excluded.** |
| Spot trading | Yes | Yes | n/a |
| GBP pairs | BTC-GBP, ETH-GBP (to confirm via `/products`) | BTC/GBP (XXBTZGBP), ETH/GBP (to confirm via `AssetPairs`) | n/a |
| Historical candles | `GET /api/v3/brokerage/market/products/{id}/candles`, public, **max 350 per request** (boundary quirk reported: 349 returned, error at 351). Paginate by `start`/`end`. | `GET /0/public/OHLC`, public, **only the latest 720 candles**. Older history comes from Kraken's downloadable bulk OHLCVT files. | n/a |
| Real-time data | REST plus WebSocket | REST plus WebSocket v2 | n/a |
| Sandbox | Static mock sandbox (`api-sandbox.coinbase.com`), accounts and orders only. **No value for paper Phase 1.** Marginal value at the live stage. | No spot sandbox found | n/a |
| API key permissions | CDP API keys (JWT-signed), scoped view / trade / transfer. A trade-only key is possible. | Granular key permissions. Withdraw can be left off. | n/a |
| Entry-tier fees (maker / taker) | ~0.40% / 0.60% (sources disagree, Sept 2026 change not verified) | 0.40% / 0.80% after 9 July 2026 (multiple reviews agree; base row not seen on the official page) | n/a |
| Minimum order | `base_min_size` / `quote_min_size` per product. **Value for BTC-GBP unknown.** | BTC 0.0001, ETH 0.01, **GBP 5** (support article, Feb 2026) | n/a |
| Spreads (GBP books) | **Not measured.** No reliable source found. Measured in Milestone 2 (median and 90th percentile over ≥ 7 days). | **Not measured.** Same plan. | n/a |
| Liquidity | GBP books expected to be thinner than USD/USDC [UNVERIFIED]. Proxy to collect: reported 24h GBP volume and top-of-book depth. | Same [UNVERIFIED] | n/a |
| Rate limits | Published in official docs but not retrievable from here. Fetch task in Milestone 2. Client throttles conservatively until then. | Counter/decay limits published but not retrieved. Same plan. | n/a |
| Operational cost | Public data free. Trading fees only. | Public data free. Trading fees only. | n/a |
| Exchange risks | Counterparty/insolvency (funds held by exchange), frequent fee/API changes (Sept 2026), regulatory change under the 2027 UK regime, account restrictions | Counterparty/insolvency, recent fee increase for small accounts (July 2026), regulatory change, API deprecation | Not available for new UK users |

### 2.1 Feasibility under the mandated risk rules at £50

This is arithmetic only.

- **Maximum single-position exposure = 10% of equity = £5.00 at £50.**
- **The binding constraint is the GBP order-value minimum, not the BTC quantity minimum.** Kraken's GBP minimum is £5 (support article). At exactly £50 equity, the cap equals the minimum. Once equity falls below £50, **or** whenever the 1%-risk sizing gives a position below the cap (any stop wider than 10%), **every Kraken GBP order is below the minimum and must be rejected, whatever BTC's price.** In practice, a £50 Kraken account could stop trading after its first loss.
- The 0.0001 BTC quantity minimum adds a second constraint whenever BTC/GBP > £50,000. I could not verify the current price. Agent B's search pointed to around £48k, which I also can't verify. Either way, the GBP minimum alone is enough to make Kraken unworkable at this capital.
- **Coinbase could have the same problem.** If its BTC-GBP `quote_min_size` is near £5, Coinbase is unworkable too. **Milestone 2 has a blocking check:** fetch the live minimums. If no candidate exchange allows orders well below the cap (target ≤ £1 minimum order value), report to the user that the mandated risk limits and £50 capital are incompatible on that venue, rather than loosening the limits.
- The risk engine rejects any order below the exchange minimum. The simulator applies the same minimums.

### 2.2 Cost drag

- **Fees:** a round trip at taker rates costs ~1.2% (Coinbase 0.60%) to ~1.6% (Kraken 0.80%), plus spread and slippage. Maker entries roughly halve the fee, but some orders won't fill.
- **Spreads** are unmeasured. At £5 position sizes they may cost as much as the fees. Milestone 2 measures them.
- Network fees apply only to Phase 2 Solana swaps.

### 2.3 The £1,000 target, plainly

With a 10% cap per position, a trade that gains +10% net adds about 1% to equity. Reaching 20× by compounding 1% gains needs **~302 consecutive winning full-size trades and no losses** (1.01³⁰¹ = 19.99, so 302 are needed). Running three positions at once (the most the 3% aggregate-risk cap allows at full 1% risk each) cuts the calendar to ~102 rounds, but that is still **~306 winning trades**. Any realistic win rate makes it longer. The risk rules make the target **very unlikely in any reasonable timeframe**. That trade-off is intended: the rules prioritise survival. The system won't loosen them to chase the target, and the dashboard won't show projected dates.

### 2.4 Phase 2 integration candidates (compared now, built at Milestone 7)

| Component | Candidate | Role | Status |
|---|---|---|---|
| Swap routing and quotes | Jupiter Swap API (`api.jup.ag`, free API key from developers.jup.ag. v1 → v2 migration in progress, `lite-api` status unclear) | Quote-only simulation of route, `outAmount`, price impact. **No transaction signing in paper mode.** | [DOCS-SEARCH], host blocked here |
| Market data and discovery | DexScreener public API (reported ~300 req/min for pairs, 60/min for profiles) | Pairs, liquidity, volume, pair age | [DOCS-SEARCH, third-party], blocked |
| Token risk screen | RugCheck report API (reported keyless) | Risk flags, holder concentration, LP status | [DOCS-SEARCH, third-party], blocked. One input, not proof of safety. |
| On-chain truth | Solana RPC `getAccountInfo` on the mint | Mint and freeze authority, Token-2022 extensions (transfer hooks, fees), read directly | Provider not chosen yet |

Rule for Phase 2: when providers disagree, **trust on-chain data over third-party scores**. If critical data is missing, **reject the token**.

---

## 3. Recommendation: initial centralised exchange (provisional)

**Provisional choice: Coinbase Advanced Trade for market data and paper simulation**, with Kraken behind the same adapter interface. This is **not a final venue decision.** The evidence is mostly secondary and partly conflicting.

What supports it:

1. **Strongest reason (documented):** paginated public historical candles through the same API used for live data, which keeps ingestion and live code paths the same. Kraken's bulk files can also cover backtests, so this is a convenience, not a decisive advantage.
2. **Probably lower taker fee** (~0.60% vs 0.80%), but the Coinbase figure is unverified for the UK after Sept 2026.
3. **Kraken's £5 GBP minimum makes it unworkable at £50** (§2.1). Coinbase's minimum is unknown, so this argues *against Kraken* without proving Coinbase works.

The sandbox is **not** counted as a reason: it is a static mock and adds nothing to paper trading.

**Switch criteria, defined now and checked in Milestone 2 using live public data:**

- If Coinbase's BTC-GBP or ETH-GBP minimum order value is ≥ £2.50 (half the cap), it fails for that pair.
- Compare the **all-in round-trip cost** (2 × taker fee + median measured spread) per pair. If Kraken's is lower by ≥ 0.2 percentage points **and** its minimums pass, switch to Kraken.
- If neither venue passes the minimum check, stop and report the incompatibility to the user. The limits stay as specified.
- If GBP books' 90th-percentile spread is > 0.5%, evaluate USDC pairs with an explicit GBP→USDC conversion cost.

---

## 4. Proposed architecture

Python 3.13, single process, asyncio, SQLite (WAL mode). Chosen because the core stack is installed (test tooling still needs adding, §1.1), it needs no external services, and one process makes the "single authoritative execution service" requirement easy to enforce.

```
                ┌──────────────── Dashboard (Starlette + Jinja, bound to 127.0.0.1 only) ──────┐
                │  equity, target progress, positions, orders, risk status, freshness, controls │
                └────────────────────────────────────────────▲─────────────────────────────────┘
                                                             │ read-only queries + authenticated control commands
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

**Separation and authority**

- **Strategies are pure.** A strategy receives closed candles up to time *t* and returns a signal object. It has no reference to the broker, ledger or risk config, so it **cannot** place orders or change limits.
- **ExecutionService is the only writer** of orders and the ledger. A process lock file plus a DB `owner` row prevents a second instance from starting.
- **Paper/live separation.** Separate DB files. Each DB stamps its mode in a metadata table that is checked on open. The PaperBroker has no network client. In Phase 1 the LiveBroker isn't implemented, and a test asserts that paper mode can't construct one.

**RiskGate: entries and exits are treated differently**

- **Entries** (anything that increases exposure) must pass every check below. Any failure or uncertainty rejects the entry.
- **Reduce-only exits are always allowed** while the daily-loss latch, drawdown latch, kill switch or stale-data block is active. An exit passes only if it is reduce-only: it sells no more than the held quantity, on the same symbol. Exits are never blocked by risk *budgets*.
- **Stops when data is stale:** in paper mode, a stop is evaluated on the next fresh price after data recovers, and fills at that price (which may be far worse than the stop). The position is flagged `STALE_EXPOSURE` on the dashboard. A position can't be valued while data is stale, so it is shown at its last price with a staleness warning, and no new entries are allowed.
- **No shorting:** sell quantity ≤ held quantity, checked against the ledger.
- **No leverage or borrowing:** buy cost including fees and slippage buffer ≤ free cash. Cash can never go negative (DB CHECK constraint plus code assertion).
- **Recovery-trading ban:**
  - No position size is ever derived from past losses. Sizing inputs are only current equity, stop distance, costs and limits.
  - After a losing exit, there is a **cool-down of one full candle interval** before re-entering the same symbol.
  - No martingale or averaging down: you can't add to a position that is in loss.

**Limits are hard ceilings**

- The spec values (1% / 3% / 10% / 3% daily / 10% drawdown, leverage 0) are **compile-time constants**. Config may set **stricter** values only. A looser value fails start-up validation. Nothing changes limits at runtime.

**Risk metric definitions**

- **Equity (liquidation value)** = cash + Σ(qty × current bid × (1 − taker fee) − estimated slippage). Exit costs are included, so equity never overstates what could be realised.
- **Daily loss** = (start-of-day equity − current equity) / start-of-day equity, **realised plus unrealised**. Trips at ≥ 3%. Start-of-day equity is snapshotted at the daily boundary.
- **Daily boundary:** 00:00 **Europe/London** (the user's local day, which moves with BST). It is stored as a UTC timestamp for each day.
- **Drawdown** = (peak equity − current equity) / peak equity. Peak is the highest liquidation-value equity ever recorded, starting at £50. Trips at ≥ 10%. The latch clears only through a manual `review-resume` command with a written reason, which is logged. Peak is **not** reset on resume.

**Kill switch**

- Two levels:
  - `HALT`: block all new entries and keep managing existing positions' stops and targets.
  - `FLATTEN`: also submit reduce-only market exits for every position.
- Persisted in the DB, so it survives restart, and checked before every order.
- Triggered from the dashboard, a CLI command or a sentinel file. Clearing it needs an explicit command and is logged.
- A separate live kill switch is specified at Milestone 10.

**Idempotency and order lifecycle**

- Every order has a deterministic `client_order_id` from its origin:
  - entries: `(strategy, symbol, candle_close_ts, side)`
  - stop or target exits: `(position_id, exit_reason)`
  - kill-switch exits: `(position_id, "kill", switch_event_id)`
- A UNIQUE constraint means a replay is a no-op.
- **States:** `NEW → (PARTIALLY_FILLED)* → FILLED | CANCELLED | REJECTED | EXPIRED`. Each fill is a separate ledger row keyed by `(client_order_id, fill_seq)`, so partial fills can't double-count.

**Fail closed, restart and reconciliation**

- **Stale data:** no closed candle within 2× the interval, or no ticker within 3× the poll interval. This blocks **entries only**.
- **Uncertain state:** reconciliation mismatch, an order in an unknown state, or a DB error. This blocks entries and raises an alert.
- **On start-up:**
  1. Rebuild positions from fills.
  2. Check cash = initial capital + Σ cash flows.
  3. Check that every open position has its protective stop record.
  4. **Replay candles missed while down** in time order, triggering any stop or target that would have hit, at the worse of the stop level and the first candle open after it was crossed.
  5. Only then allow entries.
- If any check fails, the system stays in `HALT` and says why.

**Dashboard controls**

- Bound to 127.0.0.1 only.
- Control endpoints (pause, resume, halt, flatten, review-resume) need a local token from an env var. Everything else is read-only.
- Every control action is written to an audit table.

### 4.2 Conservative paper fill model

- **Market buy** fills at the **ask** plus slippage. **Market sell** fills at the **bid** minus slippage. Slippage = base bps + a size-dependent term from order-book depth where available, otherwise a fixed conservative value.
- **Limit orders** fill only when the price **trades through** the limit (touching is not enough). Fills are partial when the order is larger than a configured share of displayed depth.
- **Stops** trigger as market orders on the next observed price. On a gap, the fill is at the gapped price, never at the stop level. The dashboard and docs state that a stop doesn't guarantee an exit price.
- Fees use the **taker** rate unless a limit order genuinely rests.
- Orders that break exchange minimums, precision or available cash are **rejected**.

**Backtests** use the same fill and risk code, plus:

- **Candle timestamps** are normalised to bucket **open** time (Coinbase `start`). A candle opening at time *T* with interval Δ is usable only once its close at *T+Δ* has passed. A signal from it executes no earlier than the open of the next candle. The newest, unfinished candle is always dropped. Tests check this, including a test that fails if the signal function sees any data stamped later than its decision time.
- **Spread:** candles carry no bid/ask, so backtests charge a half-spread on each side equal to the **90th-percentile spread measured in Milestone 2**, with a floor of 0.10% per side. Until measurements exist, they use 0.25% per side.
- **Same-candle stop and target:** if one candle's range covers both, assume the **stop hit first**.
- **Gaps:** if a candle opens beyond the stop, fill at the open.

### 4.3 Proposed package layout (Milestone 3 onward)

```
paidrocket/
  config/        settings (pydantic), RiskLimits (hard ceilings), mode
  data/          adapters (coinbase_public, kraken_public), candle_store, freshness
  strategies/    base, sma_trend, rsi_reversion, donchian_breakout (each toggleable)
  risk/          sizing, limits, circuit_breakers, kill_switch
  execution/     service (single authority), order_state, paper_broker, fill_model
  portfolio/     ledger, valuation, reconciliation, recovery
  backtest/      engine, walk_forward, metrics, benchmark
  meme/          (Phase 2) discovery, screening, jupiter_quote_sim
  web/           dashboard
tests/           unit, property-based (hypothesis), scenario (crash, gaps, stale data, restart, ...)
docs/
```

---

## 5. Data requirements

| Data | Use | Source | Volume |
|---|---|---|---|
| 1h and 1d candles, BTC-GBP and ETH-GBP, ≥ 5 years | Backtests (train / validation / out-of-sample) | Coinbase public candles, paged at **≤ 300 per request** with continuity checks (no gaps or duplicates), cross-checked against Kraken bulk files | ~44k hourly candles per pair, a few MB |
| Best bid/ask (ticker), every 10–60s | Spread measurement, paper fills, freshness | Coinbase public ticker / WebSocket | Small |
| L2 order-book snapshot (top N levels) | Slippage and partial-fill model | Coinbase product book | Small, sampled |
| Product metadata (min sizes, increments, status) | Order validation, §2.1 blocking check | `/products` | Refreshed daily |
| Fee tier | Cost model | Account fee endpoint (needs a key, so live stage only). Until then the config uses 0.80% taker for both venues, the worse figure. | n/a |
| Phase 2 | Meme screening | §2.4 | Small |

**Chronological split, fixed in advance:** the oldest 50% of history for training, the next 20% for validation and the most recent 30% for out-of-sample. With ≥ 5 years of history, that gives ≥ 18 months out-of-sample.

---

## 6. Evidence threshold for live-trading review (pre-registered)

Defined before any backtest runs. Below, OOS means out-of-sample. A strategy configuration is *eligible for live review* only if **all** of these hold.

**Process controls**

- **P1:** At most **5 configurations per strategy family** may be evaluated on OOS, chosen using training and validation data only. All OOS results are reported, including failures.
- **P2:** The OOS window is ≥ 18 months and includes at least one fall of 20% or more in the underlying asset. Each configuration is evaluated on it once.

**Criteria**

1. ≥ **50** completed OOS trades.
2. OOS net profit factor ≥ **1.2** with costs stressed at **1.5×** the modelled fees, spread and slippage. If there are no losing trades, profit factor is treated as undefined and the configuration **fails** (too few trades to judge).
3. OOS max drawdown of the account ≤ **7%**, a margin below the 10% live stop.
4. OOS expectancy per trade > 0 after stressed costs.
5. **Benchmark:** OOS net return > the return of buy-and-hold **scaled to the strategy's average exposure** (B&H return × average fraction of equity invested). Plain buy-and-hold is reported alongside but isn't a pass/fail test.
6. Net profitable in at least 2 of 3 equal chronological OOS sub-windows.
7. **Forward paper test:** ≥ 90 days on live data with ≥ 20 trades, **and**:
   - forward net profit factor ≥ **1.0** (absolute floor), **and**
   - forward profit factor ≥ the 5th percentile of a bootstrap distribution of OOS profit factor. The bootstrap uses 10,000 resamples, each drawing *n_forward* trades with replacement from the OOS trade list, with seed fixed at 20261010. Resamples with no losses are dropped.

Meeting this threshold makes a strategy eligible for **review**. It is not proof of future profit. Expect most or all configurations to fail. That outcome will be reported as it is.

---

## 7. Initial cost estimates

| Item | Cost | Basis |
|---|---|---|
| Market-data APIs (Coinbase / Kraken public, DexScreener, RugCheck) | £0 | Public endpoints [DOCS-SEARCH] |
| Jupiter API (Phase 2, quotes only) | £0 on free tier, needs a free API key | [DOCS-SEARCH]. Confirm the tier and rate limit. |
| Solana RPC (Phase 2) | £0 on a provider's free tier | [UNVERIFIED]. Choose a provider. |
| Host for 24/7 paper trading | £0 on the user's PC, or a small VPS at typically a few pounds a month | [UNVERIFIED]. Check provider pricing. |
| Paper trading | £0. No funds involved. | n/a |
| Dev tooling (pytest, hypothesis) | £0, open source | PyPI reachable |
| Future live trading costs | ~1.2–1.6% per round trip at taker rates plus spread (CEX). Solana base fee plus priority fee per swap (Phase 2). | §2.2 |

---

## 8. Key limitations and risks

1. **No live data from this environment** until the hosts in §1.3 are allowed.
2. **No continuous runtime here.** Autonomous paper trading needs a user-controlled host.
3. **Fees and minimums are unverified.** The sources conflict and changed in 2026. The simulator will use conservative defaults (taker 0.80%) until the live API values are recorded.
4. **£50 with a 10% position cap is at or below exchange minimums.** Kraken's £5 GBP minimum equals the cap, so it is unworkable after any loss. Coinbase is unverified. This may make the specified limits incompatible with £50 on the chosen venue (§2.1).
5. **The £1,000 target is mathematically remote** under the mandated limits (§2.3).
6. **Stops are not guaranteed.** Gaps and illiquidity can cause losses far larger than the planned 1%.
7. **Backtests are not forecasts.** The threshold in §6 reduces overfitting risk but cannot remove it.
8. **Phase 2 (meme coins) carries much higher risk.** Security scores and audits are not proof of safety. Most candidates are expected to be rejected.

---

## 9. Development plan

| # | Milestone | Done when | Needs from user |
|---|---|---|---|
| 1 | Environment and docs discovery | This report plus independent review | n/a |
| 2 | Exchange selection | Live minimums, spreads (≥ 7 days), rate limits and UK registration recorded. §3 switch criteria applied. **Blocking check from §2.1** passed or the incompatibility reported. | **Allow network hosts** (§1.3), or run the fetch script locally |
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
