# Airline Fuel Cross-Hedging & Derivatives Backtest

Airlines consume jet fuel continuously, yet long-dated jet kerosene derivatives are relatively illiquid and carry wide bid-ask spreads. In practice, airline fuel desks cross-hedge using liquid proxy contracts,typically WTI crude or ULSD heating oil futures,and structured options. 

This repository models the two core problems of an airline hedging program:
1. **Cross-hedging basis risk:** Jet fuel prices track crude/distillates over time, but refinery bottlenecks cause the jet crack spread to diverge unpredictably.
2. **Payoff asymmetry:** Linear futures eliminate upside price spikes but force the airline to forfeit windfall savings (and post variation margin) when oil prices fall.

The script simulates a 5-year commodity market with jump shocks and mean-reverting crack spreads, estimates dynamic hedge ratios, prices zero-cost option collars via Black-76, and evaluates four hedging strategies out-of-sample over 48 monthly rebalancing periods.

---

## Market & Strategy Mechanics

### 1. Synthetic Market (`MarketDataGenerator`)
Rather than relying on geometric Brownian motion, daily spot and futures prices are generated using two coupled processes:
* **Proxy Futures ($F_t$):** Simulated via a Merton jump-diffusion process to capture sudden geopolitical or supply-driven price spikes.
* **Jet Crack Spread ($C_t$):** Modeled as a mean-reverting Ornstein-Uhlenbeck process partially correlated with supply jumps, producing realistic spot-futures cointegration and short-term basis risk ($S_t = \beta F_t + C_t$).

### 2. Hedge Ratio Estimation (`HedgeRatioModels`)
* **Static OLS:** Regresses spot log-returns on futures log-returns over a 1-year warm-up window to avoid spurious correlation on non-stationary price levels.
* **Dynamic EWMA ($\lambda = 0.94$):** Updates conditional variance and covariance recursively using strictly causal ($t-1$) returns, scaling the return beta by the spot-to-futures price ratio to obtain the physical gallon hedge ratio:

$$h_t^* = \frac{\text{Cov}_{t-1}(\Delta \ln S, \Delta \ln F)}{\text{Var}_{t-1}(\Delta \ln F)} \cdot \frac{S_t}{F_t}$$

### 3. Option Pricing & Zero-Cost Collar (`Black76Engine`)
* Implements analytical Black-76 pricing and Greeks ($\Delta, \Gamma, \mathcal{V}$) for European options on commodity futures.
* At each monthly roll, the airline buys a 3% out-of-the-money call ($K_{\text{call}} = 1.03 F_0$) to cap upside fuel costs and solves for the short put strike ($K_{\text{put}}$) using Brent's root-finding method (`scipy.optimize.brentq`) such that the net upfront option premium is zero.

---

## Out-of-Sample Backtest Results

The backtest assumes a constant monthly consumption of **10 million gallons**, rolled every 21 trading days across a 4-year out-of-sample window (Years 2–5). Budget shock is defined as the realized monthly fuel bill (net of hedge P&L) minus the month-start spot forecast ($Q \cdot S_0$).

| Strategy | Mean Bill ($M) | Shock Std ($M) | Variance Reduction | 95% VaR Shock ($M) | 95% CVaR Shock ($M) | Worst Spike ($M) | Best Savings ($M) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Unhedged** | 24.53 | 2.11 | , | 3.42 | 4.06 | 4.84 | -4.87 |
| **Naive 1:1 Futures** | 24.28 | 0.55 | 93.2% | 0.99 | 1.13 | 1.30 | -1.03 |
| **Dynamic MVHR** | 24.33 | 0.44 | 95.7% | 0.60 | 0.85 | 1.27 | -0.89 |
| **Zero-Cost Collar** | 24.39 | 0.65 | 90.6% | 0.96 | 1.19 | 1.38 | -1.58 |

### Key Takeaways
* **Naive 1:1 over-hedges basis risk:** Because the structural pass-through between proxy futures and jet fuel is below 1.0 ($\hat{\beta}_{\text{OLS}} \approx 0.75$), hedging 100% of physical volume introduces unnecessary tracking error. Dynamic EWMA sizing reduces monthly budget shock volatility by **95.7%** and cuts 95% CVaR tail spikes from **$4.06M to $0.85M**.
* **Linear swaps vs. collars:** Dynamic MVHR achieves the lowest overall variance, while the Zero-Cost Collar caps extreme upward spikes at **$1.38M** (vs. $4.84M unhedged) while retaining nearly double the downside savings (**-$1.58M** vs. -$0.89M) during sharp sell-offs.


---

### Requirements
* Python 3.9+
* `numpy`, `pandas`, `scipy`, `statsmodels`, `matplotlib`

### Run
```bash
pip install numpy pandas scipy statsmodels matplotlib
python airline_fuel_hedging.py
