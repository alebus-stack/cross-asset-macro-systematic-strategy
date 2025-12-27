# Cross-Asset Macro Systematic Strategy

This project implements a cross-asset **systematic macro** trading strategy in Python using three core signals:

- **Carry**
- **Momentum**
- **Volatility**

Signals are cleaned (including **winsorization**) and normalized, then combined into a single score that drives portfolio positioning. The notebook also includes a **Monte Carlo search** to find signal weights that maximize cumulative portfolio performance.


---

## Model Overview

### Pipeline

1. **Load & clean data**
   - Prices / returns (and additional fields required for carry)
   - Handle missing values
   - Align all assets to a common calendar

2. **Compute signals**
   - Carry signal (asset-class dependent)
   - Momentum signal (return-based)
   - Volatility signal (realized volatility estimate)

3. **Outlier handling**
   - **Winsorization** applied to reduce the impact of extreme values
   - Z-score normalization applied so signals are comparable across assets

4. **Combine signals**
   - Weighted linear combination:

      S_i = 𝐰ᵀ · 𝐙_i
      
      where:
      
      - 𝐙_i : vector of standardized signals  
      - 𝐰   : vector of signal weights​​

5. **Portfolio construction**
   - Convert combined scores into positions (e.g., long higher-score assets, short lower-score assets)
   - Portfolio returns computed from positions and subsequent asset returns

6. **Monte Carlo weight optimization**
   - Randomly sample many weight triplets \((w_c, w_m, w_v)\)
   - Re-run portfolio construction and compute cumulative return
   - Keep the weights that maximize the objective

---

## Signals

- FX: carry_raw = (base_rate − quote_rate) / fx_vol  
- Rates: rolldown_carry = (near contract − far contract) / duration  
- Commodities: rollyield = (near contract − far contract) / near contract
- Equities: carry = Dividend yield from SPX vs SPXTR

### 2) Momentum

Momentum is implemented as a weighted blend of return momentum measured over
multiple horizons to balance responsiveness (shorter windows) and stability
(longer windows).

The strategy uses three lookback windows (in trading days):

- 3M  ≈ 63 days
- 6M  ≈ 126 days
- 12M ≈ 252 days

Weights are assigned with more emphasis on recent momentum:

- 3M weight: 0.5
- 6M weight: 0.3
- 12M weight: 0.2



### 3) Volatility
- Realized volatility estimate (e.g., rolling standard deviation of returns).
- Used as a signal component and/or to penalize very volatile assets depending on weighting.

---

## Winsorization (Outlier Control)

Before normalization and combination, signals are winsorized to reduce sensitivity to extreme observations.

Example (conceptual):
- Cap values below the lower percentile (e.g., 1%) and above the upper percentile (e.g., 99%)
- This improves stability of z-scores and reduces single-point distortion in ranks

---

## Portfolio Construction

The combined signal produces a score per asset. Positions are derived from that score.

A typical approach used here:
- **Long** assets with the highest scores
- **Short** assets with the lowest scores
- Weights are normalized to control total exposure


## Monte Carlo Weight Search

Goal: find \((w_c, w_m, w_v)\) that maximize cumulative return.

Process:
1. Sample many candidate weight triplets
2. Run strategy for each set
3. Record cumulative return
4. Select best-performing weights

This is a simple and robust way to explore weight space without assuming convexity or smoothness.

---

## Performance Evaluation

Strategy performance is evaluated using daily net portfolio returns
(`port_returns_net`). In addition to cumulative performance, the notebook
computes a set of standard risk/return metrics and monthly return statistics.

### Monthly Returns

Monthly returns are computed by summing daily net returns within each calendar
month:

monthly = port_returns_net.resample('M').sum()

From this monthly series, the best and worst months are reported.

### Metrics Reported

The following metrics are computed:

- Total Return (%)
- Annualized Return (%)
- Annualized Volatility (%)
- Sharpe Ratio
- Maximum Drawdown (%)
- Calmar Ratio
- Win Rate (%)
- Profit Factor
- Best Month (%)
- Worst Month (%)

All metrics are derived from the daily net return series and the monthly
aggregated series shown above.

## Disclaimer

Research project only. Not investment advice.
