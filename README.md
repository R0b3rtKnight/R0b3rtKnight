# R0b3rtKnight

I'm building **TradingLNN**, a research and alert system for intraday trading of US stocks. The code is in a private
repository. Below is a summary of the project and its current results.

## TradingLNN

- **Goal:** BUY / SELL / EXIT alerts for intraday US-stock trades that are placed by hand.
- **Origin:** a per-stock *liquid neural network* (CfC, closed-form continuous-time) model that retrained at startup and
  sent its alerts to Telegram.
- **Now:** a validation pipeline. A trading idea has to pass a pre-registered walk-forward test and a single look at a
  holdout period, with realistic per-trade costs, before it may send alerts.
- **Status (28 Sep 2026):** no strategy has passed. Live alerts are switched off in code until one does.

### Latest results: 20-stock intraday study (28 Sep 2026)

Setup:

- 1-minute consolidated (SIP) bars for 20 volatile US stocks.
- Design window Feb 2022 – Jun 2024. Parameters and exits are chosen on earlier half-years only, then tested on 2023 H1,
  2023 H2 and 2024 H1 (walk-forward).
- Point-in-time eligibility: a stock-day counts only if, on the previous day, the stock's unadjusted price was ≥ $10 and
  its 20-day dollar volume was ≥ $100M.
- Each trade pays its own modelled bid-ask spread at its entry and exit minute, plus a 0.05% broker allowance (typically
  0.12–0.25% per round trip).
- Pass rule: direction hit rate ≥ 55% with the 95% CI lower bound above 50%; net > 0 in at least 2 of the 3 test halves;
  day-clustered t ≥ 2 and net > 0 for both a 10 s and a 0 s fill; positive average day and max drawdown under 50%
  (simulated at 5× leverage, one position at a time).

| Idea | Trades | Net / trade | t | Direction hit rate | Verdict |
|---|---:|---:|---:|---:|---|
| Opening-range breakout (15/30 min) on moderately active stocks, low-cost entries | 463 | +0.015% | 0.15 | 59.9% | FAIL |
| Same, with symmetric ±0.5×ATR exits | 463 | +0.098% | 0.80 | 59.9% | FAIL |
| Trend-day persistence (09:35 filter, opening-drive direction) | 876 | −0.138% | −1.92 | 54.5% | FAIL |
| Sector flow (breakout confirmed by ≥ 2 stocks in the same cluster) | 4,341 | −0.116% | −1.81 | 54.3% | FAIL |
| Sector flow, with symmetric exits | 4,341 | −0.179% | −2.63 | 54.3% | FAIL |
| Lead-lag catch-up to SMH / QQQ / BTC | 594 | −0.241% | −3.67 | 54.5% | FAIL |
| Gap-and-go / gap-fill from premarket bars | 385 | −0.408% | −3.68 | 49.0% | FAIL |
| Cost-aware expected-value model over all breakouts | 0 | – | – | – | INCONCLUSIVE: found no trade with positive expected value |

Net / trade is after costs, with a 10 s fill, unlevered, pooled over the three test halves. Direction hit rate is the
share of trades, among those that moved ±0.5×ATR, whose first such move went the trade's way.

**Holdout (Jul 2024 – Jun 2025), one look at the breakout with symmetric exits:** 509 trades, net −0.199% per trade,
direction hit rate 49.3%, both half-years negative. **FAIL.** The look was taken before the bug fixes described below.
After the fixes the candidate no longer qualified for a holdout test, so there was no second look.

The filter in the first two rows came from an earlier analysis of the same half-years, and the symmetric exits were
added after seeing the first walk-forward. Those two walk-forward rows are therefore partly in-sample; only the holdout
counts.

### What the results show

- Direction from price patterns is close to a coin flip on these stocks: 49–55% for most ideas. The best pocket reached
  60% in walk-forward and 49% on the holdout.
- Intraday prices mean-revert over 5–60 minutes (median variance ratios 0.79–0.95), so breakout entries trade against the
  price process.
- Costs are larger than the edges. A round trip costs 0.12–0.25%; the gross edges found were 0.05–0.2%. The median
  quoted spread is 0.37% at 09:31 and 0.09% at 15:30.
- Models predict how large a move will be (AUC up to 0.79), not which way it goes (0.48–0.51).
- The stock list was chosen with hindsight: only 23–40% of 2022–24 stock-days met its own liquidity rule.

### Earlier versions

| Version | Model | Result |
|---|---|---|
| M0 | Per-stock CfC + MLP confirmer, 8 stocks | 4-fold walk-forward (146 days) showed +5.58% and +3.76% per day (two seeds, 5× leverage). An audit found the model read the bar it traded on (fill-bar look-ahead). Leak-fixed, on clean data: −0.32% and −0.06% per day. No edge. |
| v1 | Breakouts + LightGBM filter, 20 stocks | −4.19% and −4.59% per day in walk-forward (two seeds, 0.25% friction, 5× leverage). Retired. |

### How the testing works

- Every experiment is pre-registered in a hash-chained trial log before any outcome is computed (34 trials so far). Each
  idea gets at most 3 configurations.
- One holdout look per finalist, enforced in code; a second look needs a recorded reason.
- Point-in-time data only: as-of joins, unadjusted prices for costs and filters, SHA-256-pinned data snapshots.
- Cost model fitted on NBBO quote samples: spread by minute of the day, price, dollar volume and ATR, with a one-tick
  floor.
- 95 unit tests, including exact-equality checks of the event replay against the reference simulator (also with 45% of
  minute bars missing), a causality perturbation test, and purged cross-fitting checked on pure noise.
- Leak audits by a leak-auditor agent and an independent code review. The last round found 2 critical, 5 major and about
  12 minor bugs. All were fixed with tests. The conclusion did not change; the best candidate's t-statistic fell from
  2.18 to 0.80.

### Stack

Python · PyTorch · ncps (CfC) · LightGBM · scikit-learn · pandas · NumPy · Alpaca market data (SIP history, IEX live) ·
Telegram Bot API · Claude Code sub-agents for reviews and leak audits

### Next

On this stock list, price patterns show no edge after costs. The candidate next step is to take direction from
information instead of price: a point-in-time pool of "stocks in play", earnings timestamps and news. Anything that
passes must then clear the same walk-forward test, one holdout look, a leak audit, a check on the live IEX data feed
and at least 20 sessions of forward shadow trading before alerts are switched back on.
