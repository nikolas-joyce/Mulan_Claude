# Mulan 3 — Alpha Swarm Notebook: Master Plan

## Vision
A clean, modular Google Colab notebook that:
1. Defines **8 independent alpha models** (4 long-side, 4 short-side) spanning distinct strategy styles
2. Runs each alpha across a diversified universe of ETFs
3. Benchmarks against passive ETFs **and** hedge-fund-style ETFs (DBMF, MNA, QAI)
4. Builds a **Swarm Portfolio** that agentically includes/excludes alphas based on:
   - Rolling Sharpe vs benchmark
   - Correlation to active peers
   - Market regime (VIX + trend filter)

---

## Universe

### Core Asset ETFs (signal universe)
| Ticker | Asset Class |
|--------|-------------|
| SPY | US Large Cap Equity |
| QQQ | US Tech / Growth |
| IWM | US Small Cap |
| TLT | Long-Duration Treasuries |
| HYG | High Yield Credit |
| GLD | Gold |
| USO | Oil |
| UUP | USD Index |
| EEM | Emerging Markets |
| VNQ | Real Estate |

### Benchmarks
| Ticker | Type |
|--------|------|
| SPY | Passive equity |
| TLT | Passive bonds |
| DBMF | Managed futures (hedge fund style) |
| MNA | Merger arbitrage |
| QAI | Multi-strategy hedge fund replication |

---

## The 8 Alpha Models

### Long-Side Alphas (4)

| # | Name | Style | Entry Logic | Exit Logic |
|---|------|-------|-------------|------------|
| L1 | **Trend_Long** | Trend Following | Price > EMA(200) AND EMA(50) > EMA(200) (golden cross zone) | Price crosses below EMA(50) |
| L2 | **Breakout_Long** | Breakout / Momentum | Close > N-day high + ATR buffer (vectorised, no loop) | Trailing stop: peak × (1 − trail_pct) |
| L3 | **MomScore_Long** | Cross-sectional Momentum | Rank tickers by 12-1 month return; go long top quartile | Monthly rebalance; exit bottom quartile |
| L4 | **MeanRev_Long** | Mean Reversion | RSI(14) < 30 AND price > 200d SMA (oversold in uptrend) | RSI > 55 OR trailing stop |

### Short-Side Alphas (4)

| # | Name | Style | Entry Logic | Exit Logic |
|---|------|-------|-------------|------------|
| S1 | **TrendFilter_Short** | Trend Following | Price < EMA(200) AND EMA(50) < EMA(200) (death cross zone) | Price crosses above EMA(50) |
| S2 | **Breakdown_Short** | Breakout / Momentum | Close < N-day low − ATR buffer | Trailing stop: trough × (1 + trail_pct) |
| S3 | **MomScore_Short** | Cross-sectional Momentum | Rank tickers by 12-1 month return; short bottom quartile | Monthly rebalance; exit top quartile |
| S4 | **MeanRev_Short** | Mean Reversion | RSI(14) > 70 AND price < 200d SMA (overbought in downtrend) | RSI < 45 OR trailing stop |

**Key design principles:**
- All signals are **fully vectorised** (no Python for-loops)
- Every signal uses `.shift(1)` before touching returns (no look-ahead)
- Long and short use **genuinely different logic** (not mirror images)
- Styles deliberately mix trend-following with mean-reversion for low correlation

---

## Notebook Cell Structure

```
Cell 0  │ !pip installs + imports
Cell 1  │ CONFIG — single dict for all parameters + universe
Cell 2  │ Data fetch — one yf.download() call, derive all OHLCV + Close
Cell 3  │ Helper functions — ATR, RSI, EMA, ranking (all vectorised)
Cell 4  │ Long alphas (L1–L4) — signal functions, returns 0/1 Series
Cell 5  │ Short alphas (S1–S4) — signal functions, returns 0/-1 Series
Cell 6  │ Alpha runner — loops tickers × alphas, stores signal matrix
Cell 7  │ Performance engine — Sharpe, Calmar, max drawdown, hit rate
Cell 8  │ Benchmark loader — SPY, TLT, DBMF, MNA, QAI equity curves
Cell 9  │ Regime detector — VIX regime (low/mid/high), trend regime
Cell 10 │ Swarm engine — rolling Sharpe filter + correlation pruning + regime gate
Cell 11 │ Portfolio construction — equal weight surviving alphas, daily rebal
Cell 12 │ Results dashboard — summary table + equity curves vs benchmarks
Cell 13 │ Correlation heatmap — alpha × alpha return correlations
```

---

## Swarm / Agentic Portfolio Logic (Cell 10 detail)

The swarm runs a daily **three-gate filter**. An alpha must pass ALL three gates to be included:

### Gate 1 — Performance Gate
```
Rolling 63-day Sharpe (alpha) > 0.0
AND
Rolling 63-day return (alpha) > Rolling 63-day return (SPY) × 0.5
```
*Rationale: exclude alphas that are actively losing money or severely lagging.*

### Gate 2 — Correlation Gate
```
For each pair of active alphas:
    if rolling 63-day return correlation > 0.7:
        keep the one with higher recent Sharpe, exclude the other
```
*Rationale: redundant alphas add weight but not diversification.*

### Gate 3 — Regime Gate
```
VIX Regime:
    LOW  (<15): all 8 alphas eligible
    MID  (15-25): disable S4 (mean-rev short — too whippy)
    HIGH (>25): disable L4, L3 (mean-rev long and CS momentum suffer)

Trend Regime (SPY vs SMA200):
    BULL (SPY > SMA200): boost weight of L1, L2; halve weight of S1, S2
    BEAR (SPY < SMA200): boost weight of S1, S2; halve weight of L1, L2
```

### Portfolio Weights
```python
# After gates, surviving alphas get equal weight
# Regime-boosted alphas get 1.5× weight (normalised to sum=1)
# Maximum single-alpha weight: 25%
# If fewer than 2 alphas pass all gates: go to cash (weight = 0)
```

---

## Key Design Decisions & Rationale

| Decision | Choice | Why |
|----------|--------|-----|
| Signal computation | Fully vectorised pandas | Pure Python loops are 50–100× slower; no numba needed |
| HLC data | Use real OHLCV (not close × 1.001 proxy) | You fetch it anyway; proxies distort ATR/CCI |
| Parameter storage | Single `CONFIG` dict | Eliminates the 5-dict confusion from Mulan 2 |
| Look-ahead prevention | `.shift(1)` on all position series | Consistent, easy to audit |
| Short signals | Independent logic (not mirror of longs) | Different edge, lower correlation |
| Trailing stops | Vectorised via `cummax/cummin` on position windows | No row-by-row loop |
| Transaction costs | 5 bps round-trip on every position change | Realistic for ETFs |
| Benchmarks | Include DBMF/MNA/QAI | Tests whether alphas beat sophisticated alternatives |

---

## What We'll Learn

1. **Do styles complement each other?** → Alpha correlation heatmap + portfolio diversification ratio
2. **Does the swarm beat passive?** → Equity curves vs SPY, TLT
3. **Does the swarm beat hedge-fund ETFs?** → Equity curves vs DBMF, MNA, QAI
4. **Does regime filtering help?** → Swarm with gates vs equal-weight all-alphas
5. **Which alphas are most durable?** → Rolling Sharpe chart per alpha

---

## Build Order (our conversation roadmap)

| Step | What we'll do together |
|------|----------------------|
| **Step 1** | Cell 0–2: installs, CONFIG, data fetch |
| **Step 2** | Cell 3: helper functions (vectorised) |
| **Step 3** | Cell 4–5: all 8 alpha signal functions |
| **Step 4** | Cell 6–7: alpha runner + performance engine |
| **Step 5** | Cell 8–9: benchmark loader + regime detector |
| **Step 6** | Cell 10–11: swarm engine + portfolio construction |
| **Step 7** | Cell 12–13: results dashboard + correlation analysis |

Say **"let's start"** and we'll build Step 1 together.
