# Volatility Band Width
**K. Mohamed Harith — Erdős Institute, Quant Finance Bootcamp (Fall 2026)**

In this project, we build 95% intervals for the next-day return of the S&P 500 (which we call *volatility bands*) using four volatility models: constant volatility GBM, rolling-window GBM, the Exponentially Weighted Moving Average (EWMA) and GARCH(1,1). We then test whether those bands can be trusted.

The models are calibrated on *S&P 500 daily data* from 2005–2017 and tested out of sample on 2018–2026. Both periods contain a major crisis: 2008 in the calibration period, COVID-19 (2020) in the test period.

**Main Observation:** getting the miss rate right is not the same as getting risk right. The constant-volatility GBM model has miss rate almost exactly 5% of days over the entire test period, yet it fails in every seasonal periods (like volatile or calm years) and its misses cluster during crises. By contrast, the models that update volatility daily from the previous day’s return keep their misses close to 5% and spread out the misses over time.  

## What's here

The whole project is in one notebook, [volatility_band_width.ipynb](volatility_band_width.ipynb), which renders directly in the browser (equations, models, figures and results tables included). A rendered PDF of the notebook, the executive summary and the presentation slides are in [docs/](docs/).

## Overview

**Constant-volatility GBM.** Geometric Brownian Motion assumes that volatility is *constant*. Under GBM, daily log returns are normally distributed with standard deviation $\sigma\sqrt{\tau}$, where $\tau$ is the length of one interval in years ($\tau = 1/252$ for daily data). So the standard deviation $s$ of historical daily log returns estimates the daily volatility, and the annual volatility is $\sigma = s/\sqrt{\tau} = s\sqrt{252}$. Estimated on 2005–2017, this gives one daily volatility (1.19%) and therefore one fixed 95% band, $\pm 1.96\,\sigma_{\text{daily}} \approx \pm 2.3\%$, for every day of the test period. Because real volatility changes over time, this band is too wide in calm markets and far too narrow in crises.

**Rolling-window GBM.** The simplest fix is to re-estimate volatility every day from only the most recent $m$ daily returns, giving each of them equal weight. The band now widens in turbulent periods and narrows in calm ones. But equal weights create two problems: the band reacts slowly at the start of a shock, because the shock is diluted among many calm days, and it then stays wide for $m$ days afterwards, dropping suddenly when the shock leaves the window (the "ghost effect"). The window length $m = 160$ trading days was chosen on the calibration period only, as the window with the lowest miss rate.

**Exponentially Weighted Moving Average (EWMA).** Instead of equal weights that are cut off after $m$ days, EWMA gives each past squared return a weight that decays geometrically: the return $i$ days ago receives weight $(1-\lambda)\lambda^{i-1}$. This reduces to a simple recursion,
$$\sigma_t^2 = \lambda\ \sigma_{t-1}^2 + (1-\lambda)\ r_{t-1}^2,$$
so recent shocks dominate and old ones fade smoothly, with no cliff. We use the RiskMetrics (J.P. Morgan/Reuters, 1996; see also Hull, OFOD) value $\lambda = 0.94$, a convention chosen to work well across many assets, which is fixed rather than estimated.

**GARCH(1,1).** GARCH generalises EWMA in two ways:
$$\sigma_t^2 = \omega + \alpha\ r_{t-1}^2 + \beta\ \sigma_{t-1}^2 .$$
First, the reaction to yesterday's shock ($\alpha$) and the persistence of yesterday's volatility ($\beta$) are separate parameters, estimated from the data by maximum likelihood (here with the `arch` package, on 2005–2017). Second, the constant $\omega$ adds a pull towards a long-run variance $V_L = \omega/(1-\alpha-\beta)$: after a crisis, volatility is expected to drift back towards its long-run level (*mean reversion*). EWMA is the special case $\omega = 0$, $\alpha + \beta = 1$.

**Testing the bands.** For each model and each test day, we record a *miss* when the actual return falls outside the band, giving one hit sequence per model. A reliable 95% band should be missed on about 5% of days, at random times. Two standard backtests check this:
- **Kupiec's test (1995):** is the number of misses consistent with 5%? This checks whether the band has the right width on average: too many misses means it is too narrow (risk understated), too few means it is too wide (risk overstated).
- **Christoffersen's test (1998):** Are misses independent, or does a miss today make a miss tomorrow more likely? This checks for clustering of the misses.

We run both on the full test period and on sub-periods (a crisis year and a calm year).

## Key results (test period 2018–2026, 2,223 days)

| Model | Miss rate | Kupiec (right count?) | Christoffersen (independent?) |
|---|---|---|---|
| Constant GBM | 4.9% | passes (p = 0.83) | **fails** (p < 0.001) |
| Rolling GBM (m = 160) | 7.0% | **fails** (p < 0.001) | **fails** (p < 0.001) |
| EWMA (λ = 0.94) | 5.8% | passes (p = 0.11) | passes, borderline (p = 0.09) |
| GARCH(1,1) | 5.2% | passes (p = 0.64) | passes (p = 0.69) |



In 2020, constant GBM is missed on 15.0% of days (rejected: band too narrow); in 2023 on 0.0% of days (rejected: band too wide). After a miss, its next day is a miss 24.8% of the time, against 3.9% after a day without one.

## Running it

```bash
pip install -r requirements.txt
jupyter notebook volatility_band_width.ipynb
```

The S&P 500 data is downloaded once from Yahoo Finance (`yfinance`) and saved to `data/gspc.csv`; later runs load it from that file, so the notebook then runs fully offline and always on the same data. A fixed random seed makes the simulation checks reproducible.

## Files

| File / folder | Description |
|---|---|
| `volatility_band_width.ipynb` | Main notebook: data, models, backtest, hypothesis tests, results and conclusions. |
| `README.md` | This file. |
| `requirements.txt` | Python packages needed to run the notebook. |
| `data/gspc.csv` | S&P 500 (^GSPC) daily prices from Yahoo Finance, 2005 to the download date. |
| `docs/` | Rendered PDF of the notebook, executive summary (PDF) and presentation slides (PDF). |
| `figures/` | Figures exported from the notebook, used in the slides. |

## References

- Hull, J. C. *Options, Futures, and Other Derivatives*: estimating volatility; EWMA and GARCH; back-testing VaR.
- Engle, R. (2001). GARCH 101: The use of ARCH/GARCH models in applied econometrics. *Journal of Economic Perspectives*, 15(4).
- Kupiec, P. (1995). Techniques for verifying the accuracy of risk measurement models. *Journal of Derivatives*, 3(2).
- Christoffersen, P. (1998). Evaluating interval forecasts. *International Economic Review*, 39(4).
- J.P. Morgan/Reuters (1996). RiskMetrics: Technical Document, 4th edition. New York: Morgan Guaranty Trust Company.
