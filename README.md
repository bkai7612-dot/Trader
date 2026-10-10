# PAIDROCKET

Autonomous crypto and meme-coin **paper-trading** system. It starts with £50 of simulated capital and an aspirational £1,000 target. That target is not a promised outcome.

**Mode: PAPER.** No real orders, transactions, deposits or withdrawals can occur. Live trading isn't implemented. It would need a separate, explicitly approved stage (see Milestone 10).

## Status

| Milestone | Status |
|---|---|
| Launch guide | See [`docs/LAUNCH-GUIDE.md`](docs/LAUNCH-GUIDE.md) |
| 1. Environment and documentation discovery | Report done. See [`docs/milestone-01-discovery.md`](docs/milestone-01-discovery.md) and [`docs/review-log.md`](docs/review-log.md) |
| 2. Exchange selection (Coinbase Advanced Trade recommended) | Blocked: exchange API hosts aren't reachable from the dev environment yet |
| 3–10 | Not started |

## Key caveats

- The mandated risk limits (1% risk per trade, 10% max position) make £50 → £1,000 very unlikely in any reasonable timeframe. Progress will be reported honestly, with no projected completion dates.
- A stop-loss can't guarantee an exit price during gaps, crashes or in illiquid markets.
- Backtest and paper results are not evidence of future profit.
