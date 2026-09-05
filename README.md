# Pairs Trading Strategy — Statistical Arbitrage Backtest

Implementation of the classic **distance/cointegration-based pairs trading** approach
(Gatev, Goetzmann & Rouwenhorst, 2006), applied to five candidate equity pairs across
different sectors, with an out-of-sample backtest over two separate trading periods.

## What this does

1. **Data** — nine years of daily adjusted close prices and volume for 10 tickers
   (5 candidate pairs), pulled via `yfinance`:
   - Utilities: `DUK` – `SO`
   - Construction Materials: `VMC` – `MLM`
   - Mega-cap Tech: `AAPL` – `MSFT`
   - EDA Software: `CDNS` – `SNPS`
   - Healthcare: `JNJ` – `UNH`

2. **Data quality checks** — missing prices and single-day moves >15% (flagging
   potential unadjusted splits/spin-offs vs. genuine market events like the March 2020
   crash).

3. **Formation / trading split** — parameters are estimated only on a 5-year formation
   window (2017–2021); the strategy is then tested out-of-sample on two independent
   trading periods (2022–2023 and 2024–2025), so pair selection can never "see" the
   data it's later graded on.

4. **Pair selection — five criteria, not cointegration alone**:
   - Liquidity (minimum average daily dollar volume across both legs)
   - Return correlation
   - Mean absolute normalized spread
   - Mean squared deviation (MSD) of the spread
   - Engle-Granger cointegration test p-value (worse of the two directions)

   Pairs are ranked on each criterion and selected by lowest **equal-weighted rank sum**.
   A robustness check re-runs the ranking with the COVID window (Feb–May 2020) excluded,
   to confirm the selection isn't an artifact of that period.

5. **Backtest engine** — for each selected pair, per trading period:
   - Re-normalize both legs to 1 at the start of that period
   - Build the spread, standardize to a z-score using the *formation-period* mean/std
   - Enter when `|z| > 2`, exit at `z = 0` (mean-reversion signal)
   - Track cumulative P&L per $1 committed

6. **Transaction costs** — flat 5 bps one-way per leg (Krauss, 2017; Do & Faff, 2010),
   applied per entry/exit, to get a net-of-cost P&L estimate.

7. **Market-neutrality check** — regress each pair's daily P&L on the S&P 500's daily
   return; a market-neutral strategy should show a beta close to zero.

## Tech stack

`Python` · `pandas` · `numpy` · `statsmodels` (OLS, Engle-Granger cointegration, ADF) ·
`yfinance` · `matplotlib`

## Results

**Data**: 2,261 daily observations per ticker (2017-01-03 to 2025-12-30), no missing
prices. A handful of >15% single-day moves were flagged and checked, mostly
recognizable events (March 2020 COVID crash, earnings). One flagged move stands out —
`SNPS` on 2025-09-10 (-35.8%) — large enough that it's worth a specific look before
treating it as a normal earnings move rather than a data artifact.

**Pair selection** (5-year formation period, 2017–2021): correlation ranged from 0.52
(`JNJ`-`UNH`) to 0.996 (`CDNS`-`SNPS`); Engle-Granger cointegration p-values (worse
direction) ranged from 0.0017 (`CDNS`-`SNPS`, strongest) to 0.33 (`DUK`-`SO`, weakest).
By equal-weighted rank sum, the three selected pairs were:

| Pair | Rank sum | Hedge ratio (β) | Half-life |
|---|---|---|---|
| Constr. Materials: VMC–MLM | 11 | 0.398 | 295.4 days |
| Utilities: DUK–SO | 14 | 1.142 | 97.1 days |
| Mega-cap Tech: AAPL–MSFT | 15 (tiebreak: lower MSD over CDNS-SNPS) | 0.531 | 132.7 days |

Selection was unchanged when the COVID window (Feb–May 2020) was excluded from
formation — the ranking isn't an artifact of that period.

**Backtest (entry at \|z\| > 2, formation-period mean/std)**:

| Trading period | Trades | Gross P&L (per $1, 3 pairs) |
|---|---|---|
| TP1 (2022–2023) | 1 (DUK–SO only; still open at period end) | $0.0831 |
| TP2 (2024–2025) | 0 | $0.0000 |

**The headline finding is that the strategy almost never traded out-of-sample** — only
one entry across two 2-year periods and three pairs. Given half-lives of 100–300 days
estimated in formation, a ±2σ entry threshold calibrated on formation-period spread
volatility is simply too conservative to trigger often within a 2-year OOS window —
this is a genuine limitation of the design (and the entry threshold / formation window
length would be the first things to revisit), not a bug. Transaction costs on the one
realized trade were negligible ($0.0010, ~1.2% of gross P&L). The market-neutrality
regression is only meaningful for that one trade (DUK–SO, TP1): alpha ≈ 1.66 bp/day,
beta ≈ 0.000, but neither is statistically significant (t ≈ 1.00 and 0.02) — consistent
with, but not strong evidence for, market neutrality given the tiny sample.

## Notes

This was built as a coursework project (ICM612); methodology choices (rank-sum
selection, flat 5 bps cost assumption, z-score thresholds) follow the assignment
brief and are stated explicitly in the notebook rather than tuned for best-looking
results.
