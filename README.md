# Multi-Factor Equity Alpha Research Engine

A quant-research system that scores, ranks, and backtests equities across six classic
factor premia — Value, Momentum, Quality, Size, Low Volatility, and Profitability —
to test whether cross-sectional factor exposure produces persistent risk-adjusted
outperformance.

---

## 1. Project Overview

This project builds an end-to-end **factor investing research pipeline**: it pulls
price and fundamental data for a universe of equities, engineers six standardized
factor signals, combines them into a composite alpha score, forms quantile-ranked
portfolios (long top quintile / short bottom quintile), and backtests those
portfolios against a benchmark to measure whether the factors deliver statistically
and economically meaningful risk-adjusted returns.

It mirrors the workflow used by quantitative equity teams at asset managers,
hedge funds, and smart-beta ETF providers — from raw data ingestion through
signal construction, portfolio formation, backtesting, and performance
attribution — packaged as a reproducible, single-notebook research engine.

## 2. Real-World Finance Use Case

- **Smart-beta / factor ETF design**: firms like AQR, Dimensional, and BlackRock
  build products directly on Value/Momentum/Quality/Size/Low-Vol premia. This
  engine reproduces the research step that precedes product construction.
- **Quant equity long/short strategies**: hedge funds rank a universe monthly on
  composite factor scores and go long the top decile / short the bottom decile.
- **Fundamental analyst screening**: sell-side and buy-side analysts use factor
  scores (cheap valuation + strong momentum + high quality) to build screened
  watchlists rather than researching a whole universe by hand.
- **Risk model validation**: risk teams check whether "known" factors still carry
  a premium in the current regime, and whether factor crowding has eroded it.
- **Academic replication**: replicates the spirit of Fama-French / Carhart /
  Asness-Frazzini-Pedersen research (SMB, HML, UMD, QMJ, BAB) using
  a self-contained, freely available data pipeline.

## 3. System Architecture

```
                    ┌───────────────────────────┐
                    │   1. Data Collection       │
                    │  (prices + fundamentals)   │
                    └─────────────┬──────────────┘
                                  ▼
                    ┌───────────────────────────┐
                    │ 2. Cleaning & Validation   │
                    │  (missing data, outliers)  │
                    └─────────────┬──────────────┘
                                  ▼
                    ┌───────────────────────────┐
                    │ 3. Factor Engineering      │
                    │ Value/Momentum/Quality/    │
                    │ Size/LowVol/Profitability  │
                    └─────────────┬──────────────┘
                                  ▼
                    ┌───────────────────────────┐
                    │ 4. Cross-sectional Scoring │
                    │   (z-scores, winsorize,    │
                    │    composite weighting)    │
                    └─────────────┬──────────────┘
                                  ▼
                    ┌───────────────────────────┐
                    │ 5. Portfolio Construction  │
                    │ (quintile sort, long-short,│
                    │   monthly rebalance)       │
                    └─────────────┬──────────────┘
                                  ▼
                    ┌───────────────────────────┐
                    │ 6. Backtest Engine         │
                    │ (returns, turnover, costs) │
                    └─────────────┬──────────────┘
                                  ▼
                    ┌───────────────────────────┐
                    │ 7. Performance Analytics   │
                    │ (Sharpe, alpha, IC, DD)    │
                    └─────────────┬──────────────┘
                                  ▼
                    ┌───────────────────────────┐
                    │ 8. Visualization Dashboard │
                    └───────────────────────────┘
```

Each stage is a pure function operating on pandas DataFrames, so the pipeline can
run as-is in Google Colab, or be lifted into an Airflow DAG / cron job later.

## 4. Required APIs and Data Sources

| Source | Purpose | Notes |
|---|---|---|
| `yfinance` (Yahoo Finance) | Daily adjusted close prices, market cap, trailing PE, price/book, ROE, gross margins | Free, no API key, used as primary data source |
| Wikipedia S&P 500 constituent table (via `pandas.read_html`) | Default equity universe | Free, no key |
| `yfinance` benchmark ticker (`^GSPC` / `SPY`) | Benchmark returns for alpha/beta calc | Free |
| (Optional upgrade) Financial Modeling Prep / Tiingo / Polygon.io / Sharadar | Higher-quality point-in-time fundamentals | Requires API key, avoids survivorship & look-ahead bias |

The engine is written so `DATA_SOURCE` fundamentals can be swapped for a paid
provider by re-implementing `fetch_fundamentals()` — the rest of the pipeline is
data-source agnostic.

## 5. Required Python Libraries

```
yfinance          # market data
pandas            # data wrangling
numpy             # numerics
scipy             # z-scores, stats tests
matplotlib        # charts
seaborn           # styled charts / heatmaps
statsmodels       # regression (alpha/beta, Newey-West t-stats)
tqdm              # progress bars for data pulls
tabulate          # clean printed summary tables
```

## 6. Folder / File Structure

Even though the engine is built and run in Google Colab, it is organized as if it
were a proper repository so it can be lifted onto GitHub directly:

```
factor-alpha-engine/
│
├── README.md                     <- this document
├── requirements.txt
├── factor_alpha_engine.ipynb     <- main Colab notebook (or .py w/ cell markers)
│
├── src/
│   ├── config.py                 <- universe, dates, factor weights
│   ├── data_collection.py        <- price & fundamentals pull
│   ├── cleaning.py                <- missing data / outlier handling
│   ├── factors.py                 <- factor calculations
│   ├── scoring.py                  <- z-scores + composite alpha score
│   ├── portfolio.py               <- quantile portfolio construction
│   ├── backtest.py                <- backtest engine
│   ├── metrics.py                 <- performance metrics
│   └── visuals.py                 <- charts / dashboard
│
├── data/
│   ├── raw/                       <- cached raw pulls (parquet/csv)
│   └── processed/                 <- factor scores, portfolio weights
│
├── outputs/
│   ├── figures/                   <- exported PNG charts
│   └── reports/                   <- performance tear sheet (PDF/HTML)
│
└── tests/
    └── test_factors.py            <- unit tests for factor math
```

The Colab deliverable below collapses `src/` into clearly labeled cells so it
runs top-to-bottom in one notebook, but keeps the same functional boundaries.

## 7. Step-by-Step Build Guide

1. **Define the universe & config** — pull S&P 500 tickers, set backtest window,
   rebalance frequency, factor weights, and number of quantile buckets.
2. **Collect data** — download adjusted daily prices for the universe + benchmark;
   pull fundamental snapshots (PE, PB, ROE, margins, market cap) per ticker.
3. **Clean data** — drop tickers with insufficient history, forward-fill small
   gaps, winsorize fundamental outliers, align all series to a common calendar.
4. **Engineer factors** — compute Value, Momentum, Quality, Size, Low Volatility,
   and Profitability signals for every ticker at every rebalance date.
5. **Score cross-sectionally** — convert each raw factor to a z-score within the
   universe at each rebalance date, flip sign where "lower is better" (e.g. PE,
   volatility), then blend into one composite score.
6. **Form portfolios** — sort the universe into quintiles by composite score each
   rebalance date; build an equal-weighted long-top / short-bottom portfolio.
7. **Backtest** — roll the portfolio forward, compounding monthly returns, and
   applying a simple transaction-cost drag on rebalance turnover.
8. **Evaluate performance** — compute CAGR, volatility, Sharpe, Sortino, max
   drawdown, alpha/beta vs. the benchmark, and the Information Coefficient (IC)
   of the composite score.
9. **Visualize** — plot cumulative return curves, drawdowns, quintile spread bar
   chart, rolling Sharpe, and a factor correlation heatmap.
10. **Package outputs** — export a performance tear sheet and save factor/score
    tables for reuse.

## 8. Data Collection Pipeline

- Universe pulled from the Wikipedia S&P 500 table, with a configurable cap on
  number of names (useful for fast iteration / API-limit friendliness in Colab).
- Prices: `yfinance.download` in a single batched call for all tickers +
  benchmark, adjusted close only, resampled to month-end for rebalancing and
  kept daily for volatility/momentum lookbacks.
- Fundamentals: per-ticker `yfinance.Ticker(t).info` pull (trailing PE,
  price/book, ROE, gross margins, market cap), wrapped in retry logic with
  exponential backoff and per-ticker exception isolation so one bad ticker
  never kills the run.
- All raw pulls are cached to disk (CSV) so re-running the notebook doesn't
  re-hit the API every time.

## 9. Data Cleaning & Feature Engineering

- Drop tickers with less than a configurable minimum price history.
- Forward-fill short gaps in price series; drop tickers with too much missing
  fundamental data rather than imputing (imputing valuation ratios is
  dangerous — better to exclude).
- Winsorize each factor at the 1st/99th percentile cross-sectionally to blunt
  the influence of data errors or extreme outliers.
- Factor definitions:
  - **Value** — Earnings Yield (1/PE) and Book-to-Market (1/PB), averaged.
  - **Momentum** — 12-month price return, skipping the most recent month
    (12-1 momentum, the standard academic construction).
  - **Quality** — Return on Equity (ROE).
  - **Size** — log market capitalization (small-cap premium: sign flipped so
    smaller = higher score, matching the historical size effect).
  - **Low Volatility** — trailing 12-month realized daily volatility, sign
    flipped so lower vol = higher score.
  - **Profitability** — gross margin.

## 10. Core Models / Algorithms

- **Cross-sectional z-scoring**: `(x - mean) / std` computed *within each
  rebalance date* (not across time), which is the correct way to compare
  stocks against their peers at a point in time.
- **Composite alpha score**: configurable weighted sum of the six factor
  z-scores (equal-weighted by default).
- **Quantile portfolio construction**: `pandas.qcut` into 5 buckets per
  rebalance date; long the top bucket, short the bottom bucket, equal-weighted
  within each leg.
- **Backtest engine**: monthly rebalancing loop that (a) forms the portfolio
  from that month's scores, (b) holds it for one month, (c) compounds realized
  returns, (d) charges a basis-point transaction cost proportional to
  turnover.
- **Regression-based alpha/beta**: OLS of portfolio excess returns on benchmark
  excess returns (CAPM-style single-factor regression) using `statsmodels`,
  with Newey-West (HAC) standard errors for the alpha t-stat.
- **Information Coefficient (IC)**: monthly Spearman rank correlation between
  the composite score and next-month forward return, averaged and t-tested
  (this is the standard test of whether a factor *actually* predicts returns).

## 11. Visualizations & Dashboard Components

- Cumulative return line chart: Long-Short portfolio vs. Long-only top
  quintile vs. benchmark.
- Drawdown chart (underwater plot) for the long-short strategy.
- Quintile bar chart: annualized return by quintile bucket (Q1 = cheapest/best
  factor score … Q5 = worst), showing monotonicity.
- Rolling 12-month Sharpe ratio line chart.
- Factor correlation heatmap (are Value and Quality fighting each other?).
- IC time series bar chart with rolling mean overlay.
- Printed performance tear-sheet table (tabulate) summarizing all metrics.

## 12. Performance Metrics

- CAGR (annualized compounded return)
- Annualized volatility
- Sharpe ratio (and Sortino ratio, downside-deviation based)
- Maximum drawdown and drawdown duration
- CAPM alpha (annualized) and beta vs. benchmark, with t-stats
- Hit rate (% of positive months)
- Average monthly Information Coefficient + t-stat
- Turnover (avg. % of portfolio replaced per rebalance)
- Quintile spread (Q1 return − Q5 return), the core "does the factor work" test

## 13. Final Deliverables

- A single runnable Colab notebook / `.py` script producing:
  - Cleaned factor and price datasets
  - A composite factor score table across time
  - A full backtest of the long-short and long-only factor portfolios
  - A performance tear sheet (printed + optionally exported to PDF/HTML)
  - Five to six publication-quality matplotlib/seaborn charts
- A GitHub-ready repository structure (Section 6) with a clear README (this
  document) describing methodology, assumptions, and limitations.
- Cached raw and processed data files so results are reproducible without
  re-hitting external APIs.

## 14. Resume Description

> **Multi-Factor Equity Alpha Research Engine** — Designed and built an
> end-to-end quantitative research pipeline in Python that engineers six
> academic equity factors (Value, Momentum, Quality, Size, Low Volatility,
> Profitability) from Yahoo Finance data, cross-sectionally scores and ranks a
> 100+ stock universe, constructs long-short quintile portfolios, and
> backtests them monthly. Implemented CAPM alpha/beta regression, Information
> Coefficient analysis, and a full risk-adjusted performance tear sheet
> (Sharpe, Sortino, max drawdown, turnover-adjusted returns); delivered
> reproducible results via an automated data-caching and visualization
> dashboard.

## 15. Potential Upgrades

- Swap Yahoo Finance fundamentals for point-in-time data (Sharadar, Compustat,
  Refinitiv) to eliminate look-ahead and survivorship bias — the single
  biggest methodological weakness of a free-data version of this project.
- Add sector/industry neutralization (z-score within sector, not just within
  the whole universe) so the composite score isn't just a sector bet.
- Add a proper multi-factor risk model (Fama-French 5-factor or Barra-style)
  to compute risk-adjusted alpha instead of single-factor CAPM.
- Add machine-learning factor combination (ridge/LASSO/gradient-boosted trees
  predicting forward returns from factor exposures) instead of an
  equal-weighted composite.
- Add transaction-cost and capacity modeling (market impact, slippage) for a
  more realistic net-of-cost backtest.
- Extend the universe internationally and add currency-hedging logic.
- Wrap the pipeline in an Airflow/Prefect DAG for scheduled, production-style
  reruns, and store outputs in a proper database instead of CSV/parquet.
- Build a Streamlit or Dash interactive dashboard on top of the same backend
  functions for live exploration of factor performance.
- Add a walk-forward / expanding-window validation scheme and a factor
  "crowding" monitor (valuation spread of the factor itself over time).
