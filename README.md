# 🛡️ Global Portfolio Engine — Binance Edition (GPE v45.0)

Global Portfolio Engine (GPE) is a fully client-side quantitative portfolio backtesting engine for crypto assets.  
It uses real daily close price data from the Binance API to simulate portfolio allocation strategies, perform full hyper-parameter grid search, and analyze risk-adjusted performance with advanced visualizations.

🌐 Live Demo  
https://waranyutrkm.github.io/global-portfolio-engine-binance/

---

## 1. Project Objective

GPE is designed to study portfolio-level behavior rather than short-term trade execution.

The core research question is:

“How does a portfolio allocation strategy behave over time under realistic market conditions, fees, and rebalancing rules?”

This project focuses on:
- Portfolio allocation logic
- Risk-adjusted performance
- Parameter sensitivity (Lookback × Rebalance)
- Drawdown-aware analysis

This is not a trading bot and does not generate buy/sell signals for live trading.

---

## 2. High-Level System Flow

Binance API (Daily Close Prices)  
→ Master Timeline Alignment  
→ Daily Return & Volatility Calculation  
→ Momentum Signal Generation  
→ Strategy-Based Weight Allocation  
→ Execution Engine (Rebalance / DCA + Fees)  
→ Daily Portfolio NAV Curve  
→ Performance Metrics & Visualizations  

---

## 3. Data Ingestion

- Data source: Binance REST API
- Symbol format: {COIN}USDT
- Timeframe: 1 day
- Price used: Close price
- Maximum lookback: 1000 days

If API fetching fails, the engine falls back to internally generated mock data to prevent UI failure.  
Mock data is for demonstration only and not intended for analysis.

---

## 4. Master Timeline Alignment

Each asset may have missing dates due to listing differences.

To ensure consistency:
- All unique dates across assets are merged into a master timeline
- Missing prices are forward-filled using the last available close
- All assets end up with equal-length price arrays

This allows accurate day-by-day portfolio valuation.

---

## 5. Backtest Window Selection

The user defines Start Date and End Date.

The engine:
- Locates the corresponding indices in the master timeline
- Slices all price series accordingly
- Enforces a minimum data length requirement to avoid unstable statistics

---

## 6. Daily Return Calculation

For each asset k:

r_t = (P_t / P_{t-1}) − 1

Daily returns are used for:
- Volatility estimation
- Correlation analysis
- Sharpe and Sortino calculations
- Monte Carlo drift estimation

---

## 7. Momentum Signal Generation

Momentum is computed using simple lookback-based returns:

signal_k(t) = (P_k(t) / P_k(t − LB)) − 1

Where:
- LB is the lookback period in days (e.g. 30, 60, 90)

This signal ranks assets by relative strength.

---

## 8. Volatility Estimation

Volatility is estimated from daily returns:

σ_daily = standard deviation of daily returns  
σ_annual = σ_daily × √365

If volatility is zero or undefined, a small fallback value is used to avoid division-by-zero errors.

---

## 9. Portfolio Allocation Strategies

Each strategy produces target weights that sum to 100%, unless no asset qualifies.

### 9.1 Equal Weight (1/N)

w_k = 1 / N

All assets receive equal capital allocation.

---

### 9.2 Rank-Based Allocation

Assets are ranked by momentum.

Score_i = N − rank_i + 1  
SumScore = N(N + 1) / 2  
w_i = Score_i / SumScore

Higher-ranked assets receive larger weights.

---

### 9.3 Top 3 Equal

Only the top 3 ranked assets are selected.

w_top = 1 / 3

---

### 9.4 Top 3 Ranked

Only the top 3 assets are selected and weighted by rank:

SumScore = 3(3 + 1) / 2 = 6  
w1 = 3/6, w2 = 2/6, w3 = 1/6

---

### 9.5 Top 50 Percent Selection

Only the upper half of ranked assets is selected.

K = ceil(N / 2)  
w = 1 / K

---

### 9.6 Absolute Momentum (Trend Filter)

Only assets with positive momentum are eligible.

If signal_k > 0 → included  
w = 1 / (number of eligible assets)

Assets with negative momentum receive zero allocation.

---

### 9.7 Dual Momentum

Assets must satisfy:
1. Positive momentum (absolute filter)
2. Top performance ranking (relative filter)

Top 3 eligible assets are selected and equally weighted.

---

### 9.8 Inverse Volatility (Risk Parity Style)

raw_k = 1 / volatility_k  
w_k = raw_k / sum(raw_k)

Lower-volatility assets receive higher weights.

---

### 9.9 Adaptive Aggressive Allocation (AAA)

Logic:
- If assets with positive momentum exist, select them
- Otherwise, fall back to top 3 ranked assets
- Allocate weights using inverse volatility

This combines trend-following and risk parity concepts.

---

## 10. Execution Engine

The engine supports two execution modes.

---

## 10.1 Lump Sum Rebalance Mode

### Initial Allocation

Initial capital after front-end fee:

NetInitial = InitialCapital × (1 − FrontFee%)

Trading fee is applied to each transaction:

NetTrade = TradeAmount × (1 − TradingFee%)

---

### Rebalancing (Every RB Days)

Portfolio value:

Value_k = Shares_k × Price_k  
NAV = Cash + Σ(Value_k)

Target allocation:

Target_k = NAV × w_k  
Turnover = Σ |Target_k − Value_k|

Estimated trading cost:

Cost = Turnover × TradingFee%  
NetNAV = NAV − Cost

New position sizing:

Shares_k = (NetNAV × w_k) / Price_k

Transaction logs record percentage of total portfolio traded.

---

## 10.2 Smart DCA Mode (No-Sell)

This mode injects new capital without selling existing holdings.

Capital injection:

Inflow = DCA_Amount

Target after injection:

Ideal_k = (CurrentPortfolioValue + Inflow) × w_k  
Deficit_k = max(0, Ideal_k − CurrentValue_k)

Allocation of new capital:

Allocation_k = Deficit_k / Σ(Deficit)  
BuyAmount_k = Inflow × Allocation_k

Trading fees apply only to new purchases.

Logs show allocation as percentage of injected capital (100%).

---

## 11. Fee Model

### Front-End Fee (One-Time)

InitialNet = Initial × (1 − FrontFee%)

---

### Trading Fee

Applied on every buy or sell:

NetTrade = Amount × (1 − TradingFee%)

---

### Management Fee (Annual, Deducted Daily)

DailyFee = (ManagementFee% / 100) / 365  
NAV_t = NAV_{t−1} × (1 − DailyFee)

Shares are adjusted proportionally to reflect daily NAV decay.

---

## 12. Performance Metrics

Let NAV_t be the portfolio value series.

### 12.1 CAGR

TotalReturn = NAV_end / NAV_start  
Years = number_of_days / 365  

CAGR = (TotalReturn)^(1 / Years) − 1

---

### 12.2 Volatility

σ_annual = standard deviation(daily returns) × √365

---

### 12.3 Sharpe Ratio

rf_daily = (RiskFreeRate% / 100) / 365  
ExcessReturn = daily_return − rf_daily  

Sharpe = (mean(ExcessReturn) × 365) / σ_annual

---

### 12.4 Sortino Ratio

DownsideDeviation = standard deviation(negative excess returns only) × √365  

Sortino = (mean(ExcessReturn) × 365) / DownsideDeviation

---

### 12.5 Max Drawdown

Peak_t = max(NAV_0 … NAV_t)  
Drawdown_t = (NAV_t − Peak_t) / Peak_t  

MaxDrawdown = min(Drawdown_t)

---

### 12.6 Calmar Ratio

Calmar = CAGR / |MaxDrawdown|

---

### 12.7 Win Rate

WinRate = (Number of days with positive return) / (Total days)

---

## 13. Grid Search and Strategy Matrix

When multiple lookbacks, rebalance periods, and strategies are selected, the engine evaluates:

Strategies × Lookbacks × RebalancePeriods

Each combination is fully simulated and ranked by Sharpe Ratio.

Results are sortable and filterable in the UI.

---

## 14. Visualization Modules

- Equity Curve (Linear and Log Scale)
- Drawdown Curve
- Monthly Return Heatmap (Compounded)
- Monthly Drawdown Heatmap (Worst Monthly Drawdown)
- Correlation Heatmap (Recent returns)
- 3D Hyper-Parameter Surface (Lookback × Rebalance × Metric)
- Time-Warp Surface Animation
- Monte Carlo Simulation (50 paths, GBM-style)

Monte Carlo uses historical drift and volatility for scenario exploration, not prediction.

---

## 15. Limitations

- Uses daily close prices only
- No intraday data or slippage modeling
- Binance API limited to 1000 days
- Results are regime-dependent
- Smart DCA does not rebalance existing holdings
- Not intended for live trading execution

---

## 16. Best Practices

- Do not optimize for Sharpe Ratio alone
- Always examine Max Drawdown and Calmar Ratio
- Prefer smooth parameter surfaces over sharp peaks
- Compare against benchmarks such as BTC or ETH
- Use heatmaps to understand yearly behavior

---

## 17. Disclaimer

This project is for educational and research purposes only.  
It does not constitute financial advice.  
Cryptocurrency markets are highly volatile.  
Use at your own risk.
