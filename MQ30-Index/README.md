# Momentum Quality 30 Index (MQ30)

A rules-based equity index built in Python, inspired by institutional index construction methodologies used at firms like MerQube, S&P, and MSCI.

---

## Overview

The MQ30 selects the top 30 S&P 500 stocks by momentum, filters out high-volatility names, and weights survivors by inverse volatility. The index is rebalanced quarterly using a Laspeyres chain-linked divisor to ensure continuity across turnover events.

| Parameter | Value |
|---|---|
| Universe | S&P 500 |
| Constituents | 30 |
| Momentum signal | 12-month total return (t-21 to t-273) |
| Volatility filter | Exclude top decile by 60-day realised vol |
| Weighting | Inverse volatility |
| Rebalance frequency | Quarterly(last trading day of Mar/Jun/Sep/Dec) |
| Base level | 1000 |
| Base date | First rebalance date in range |

---

## Project Structure

```
M30_index/
│
├── m30_ive_index.ipynb       # Single notebook — all modules, simulation, main execution
│
├── index_levels.csv          # Daily MQ30 index levels (generated on run)
├── rebalance_log.csv         # Divisor and portfolio values at each rebalance (generated on run)
├── constituent_history.csv   # Ticker, weight, momentum, vol at each rebalance (generated on run)
├── quality_report.csv        # Data quality flags with severity ratings (generated on run)
├── mq30_vs_spy.png           # Performance and drawdown chart vs SPY (generated on run)
└── README.md
```

---

## Methodology

### Index Level Formula

```
Index Level_t = SUM(Price_i,t × Notional_Units_i) / Divisor
```

### Initialisation

On the first rebalance date the divisor is set so the index opens at exactly 1000:

```
Divisor_0 = Portfolio_Value_0 / 1000
```

### Divisor Adjustment at Rebalance

At each subsequent quarterly rebalance, the divisor is adjusted to absorb the mechanical effect of turnover, keeping the index level continuous:

```
New Divisor = New Portfolio Value / Pre-rebalance Index Level
```

### Constituent Selection

1. Compute 12-month momentum for every ticker (skipping the last month to avoid short-term reversal)
2. Compute 60-day realised volatility for every ticker
3. Remove the top volatility decile
4. Select the top 30 by momentum from the remaining universe

### Weighting

Inverse-volatility weights, normalised to sum to 1:

```
weight_i = (1/sig_i) / SUM(1/sig_j)
```

---

## Notebook Structure

All code lives in `m30_ive_index.ipynb`. Cells are organised in order:

### Module 1 — Data Acquisition
- `get_sp500_tickers()` — scrapes S&P 500 tickers from Wikipedia; hardcoded fallback if blocked
- `download_prices()` — bulk download of split/dividend-adjusted closes via `yfinance`
- `clean_prices()` — forward-fills gaps ≤ 5 days, drops tickers with > 20% missing data

### Module 2 — Selection & Weighting
- `compute_momentum_simple()` — 12-month total return, skipping last month
- `compute_momentum_clenow()` — regression-based momentum: `((1 + slope)^252) × R²`
- `compute_realized_vol()` — annualised 60-day volatility
- `select_constituents()` — full selection and inverse-vol weighting pipeline

### Module 3 — Index Engine
- `get_rebalance_dates()` — last trading day of Mar/Jun/Sep/Dec
- `compute_portfolio_value()` — `SUM(Price × Notional)`
- `run_index()` — daily index calculation with chain-linked divisor adjustment at each rebalance

### Module 4 — Data Quality
- `check_stale_prices()` — 3+ consecutive unchanged prices → `warning`
- `check_missing_data()` — missing data per ticker → `info`
- `check_return_outliers()` — `|log return| > 15%` → `warning`
- `check_index_anomalies()` — index daily move `> ±5%` → `critical`
- `run_quality_checks()` — runs all four, returns combined report

### Module 5 — Corporate Action Simulation
- `apply_stock_split()` — divides full price series by ratio, doubles notional units
- `verify_index_continuity()` — checks portfolio value is identical before and after adjustment
- `simulate_corp_action()` — orchestrates a 2:1 split simulation and prints PASSED/FAILED

### Simulation Test
- Runs Modules 1–5 on a 40-stock, 3-year subset to verify correctness before the full run

### Main Execution
- Full S&P 500 universe, configurable date range
- Saves all output CSVs, generates performance chart, prints summary statistics

---

## Outputs

| File | Description |
|---|---|
| `index_levels.csv` | Daily index level, base = 1000 |
| `rebalance_log.csv` | Pre/post portfolio values and divisor at each rebalance |
| `constituent_history.csv` | Full constituent snapshot at every rebalance |
| `quality_report.csv` | All data quality flags with severity and dates |
| `mq30_vs_spy.png` | Cumulative performance and drawdown vs SPY |

---

## Setup

### Requirements

```bash
pip install pandas numpy scipy matplotlib yfinance
```

### Running

Open `m30_ive_index.ipynb` in Jupyter and run all cells top to bottom. Configuration is at the top of the main execution cell:

```python
START_DATE = '2018-01-01'
END_DATE   = '2023-12-31'
N_STOCKS   = 30
USE_CLENOW = False      # set True for regression-based momentum
OUTPUT_DIR = '.'
```

---

## Performance (2018–2023)

Results will vary based on the date range and universe. Key metrics reported:

- Annualised return
- Annualised volatility
- Sharpe ratio (risk-free rate = 4%)
- Maximum drawdown
- Calmar ratio
- Total return vs SPY benchmark

---

## Design Notes

- **No look-ahead bias**: all signals use only data available strictly before `as_of`
- **Divisor continuity**: every rebalance is chain-linked so the index level never jumps mechanically
- **Corporate action safety**: splits are handled via full-series price adjustment; portfolio value is algebraically unchanged
- **Quality-first**: four independent data checks run on every execution; flags are written to CSV for audit

---