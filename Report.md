# Report: Macro-FX Linkages — Forecasting Gold and Crude with USD/INR

## 1. Motivation

India imports the majority of its crude oil and a significant share of its gold demand in US dollars. As a result, the USD/INR exchange rate is not an isolated series — it is mechanically and economically linked to the domestic cost of both commodities: a weaker rupee raises the landed cost of imported oil and gold, independent of any change in their underlying dollar price.

This project tests that linkage empirically using real, publicly available market data. Specifically, it asks:

1. Are USD/INR, gold, and crude oil individually forecastable from their own history?
2. Are the three series statistically related to one another?
3. Does modeling them jointly (rather than independently) improve forecast accuracy?

## 2. Data

| Series | Ticker | Source |
|---|---|---|
| USD/INR exchange rate | `INR=X` | Yahoo Finance (`yfinance`) |
| Gold (futures) | `GC=F` | Yahoo Finance (`yfinance`) |
| Crude oil (futures) | `CL=F` | Yahoo Finance (`yfinance`) |

- **Frequency:** daily
- **Range:** January 2019 – present
- **Cleaning:** raw closing prices were combined into a single table and rows with missing values in any series were dropped. FX and commodity futures markets do not share identical trading calendars (different holidays across US and global markets), so this step removed real, non-overlapping trading days — not synthetic gaps.

### A note on data quality

On **20 April 2020**, WTI crude futures famously settled at **-$37.63/barrel** — the first negative settlement in the contract's history, driven by a COVID-era collapse in demand and a shortage of physical storage capacity. This event is present in the raw data and was deliberately **kept**, rather than removed as an outlier, since it is a real, well-documented market event. It required explicit handling at the log-transform step (Section 3), since the logarithm of a negative price is undefined.

## 3. Preprocessing

**Log transform.** All three series were converted to natural log scale. This is standard practice for financial price series because it stabilizes variance that otherwise grows with the price level (a $50 move means something very different when gold is at $1,500 versus $5,000) and turns multiplicative price changes into additive ones.

Applying `log()` to the negative crude price on 2020-04-20 produces `NaN`, by construction — this is the correct and expected behavior, not an error. It affects exactly one row, only in the crude series, and is dropped downstream wherever the crude series is differenced or modeled.

## 4. Stationarity

Stationarity was tested using the Augmented Dickey-Fuller (ADF) test, first on the log-level series and then on the first-differenced log series.

**Result:** all three series were **non-stationary in log-levels** (ADF p-value well above 0.05) and became **stationary after first-order differencing** (p-value effectively zero, test statistic far below the 1% critical value in all cases).

This is the standard result for daily financial price series: prices themselves follow something close to a random walk (integrated of order 1), while their differences — approximately equal to daily log returns — are stationary. It confirms `d = 1` is the correct differencing order for ARIMA modeling of all three series.

## 5. Autocorrelation Structure (ACF/PACF)

ACF and PACF were computed on the differenced (return) series for each of the three assets, to characterize their short-term self-predictive structure before model fitting.

- **USD/INR:** a single significant spike at lag 1, otherwise close to white noise — indicating weak, short-lived mean reversion in daily returns.
- **Gold:** almost no significant autocorrelation at any lag — consistent with gold behaving close to a random walk at a daily frequency, itself a meaningful finding (an efficiently-priced, highly liquid market).
- **Crude oil:** the most structured of the three, with significant spikes around lags 2–4 — plausibly reflecting crude's closer connection to physical supply/inventory dynamics (e.g. weekly inventory data) relative to FX or gold.

## 6. Univariate Modeling (ARIMA)

Each series was modeled independently using `auto_arima` for order selection (searched on the log-level training series, allowing it to select `d` itself as a cross-check against the manual ADF result above), followed by a `statsmodels` ARIMA fit on the selected order.

**Evaluation:** last 60 trading days held out as a test set; forecast evaluated by RMSE on the log-price scale.

| Series | ARIMA RMSE (log scale) |
|---|---|
| USD/INR | 0.00980 |
| Gold | 0.07112 |
| Crude Oil | 0.24186 |

The USD/INR forecast was close to flat over the 60-day horizon — consistent with the near-white-noise ACF/PACF result above, and with the well-documented difficulty of forecasting exchange rates from their own past values alone (the "random walk hypothesis" for FX).

## 7. Multivariate Modeling (VAR)

To test whether the three series carry information about one another — and whether modeling them jointly improves forecasts — a Vector Autoregression (VAR) model was fit on the three differenced (return) series together.

**Lag order selection:** `select_order` was run up to 15 lags. AIC and FPE favored lag 7; BIC and HQIC favored lag 1. All criteria differed only marginally across lag lengths, consistent with the weak autocorrelation already observed. **Lag 7** (one trading week) was selected for its more interpretable horizon and to allow a richer impulse response analysis; the closeness of the alternative criteria is noted here for transparency.

### Dependency between series

Rather than a formal Granger causality test, pairwise **correlation of the differenced (return) series** was used to characterize same-day co-movement between the three series (see `corr_matrix` in the notebook). This is a same-day association measure, not a test of lagged predictive dependence — a design choice made to keep the dependency analysis within the project's intended scope.

### Forecast comparison: VAR vs. ARIMA

VAR's differenced-scale forecasts were converted back to log-price levels (cumulative sum from the last known training value) for a fair, like-for-like comparison against the univariate ARIMA RMSE above.

| Series | ARIMA RMSE | VAR RMSE | Change |
|---|---|---|---|
| USD/INR | 0.00980 | 0.00799 | ~19% lower |
| Gold | 0.07112 | 0.05094 | ~28% lower |
| Crude Oil | 0.24186 | 0.23030 | ~5% lower |

VAR outperformed univariate ARIMA on all three series. The improvement was largest for USD/INR and gold, and smallest for crude oil — consistent with crude having the strongest own-lag structure of the three (Section 5), leaving comparatively less room for cross-series information to add value.

### Impulse response functions

Impulse response functions (15-period horizon) were generated to visualize how a one-time shock to one series propagates through forecasts of the other two over subsequent days. See `figures/var_irf.png` and the notebook for the full plots.

### Rolling correlation

A 60-day rolling correlation of USD/INR returns against gold and crude returns was computed to check whether the relationship between series is stable over time or regime-dependent. See `figures/rolling_correlation.png`.

## 8. Model Diagnostics

Residual diagnostics (correlogram, histogram vs. normal density, Q-Q plot) were reviewed for the ARIMA models. Residuals showed no remaining significant autocorrelation (correlogram within confidence bands), but exhibited heavier tails than a normal distribution — i.e., more extreme days than a Gaussian model would predict. This is a well-known and expected feature of financial return series ("fat tails") rather than a sign of model misspecification.

## 9. Limitations

- **Daily returns for FX/commodities are close to white noise** at short lags, which limits forecastability from own-history alone — most visible in the USD/INR ARIMA forecast, which is close to flat over the test horizon.
- **Correlation, not causality:** the dependency analysis in this project uses same-day correlation of returns, not a formal lagged-dependence test (e.g. Granger causality). The relationships described here should be read as associative, not causal.
- **Futures vs. spot prices:** gold and crude are modeled using futures contract prices (`GC=F`, `CL=F`) rather than physical spot/benchmark prices (e.g. LBMA gold, Brent spot) — futures prices can diverge from spot due to contract roll effects and storage/carry costs.
- **Fixed 60-day holdout:** results are based on a single train/test split rather than rolling-window backtesting, so reported RMSE reflects performance over one specific recent period rather than an average across market regimes.

## 10. Conclusion

USD/INR, gold, and crude oil are each individually close to unpredictable from their own history at a daily frequency — but jointly modeling them via VAR meaningfully improved forecast accuracy for USD/INR and gold, and modestly improved it for crude oil. This supports the project's original hypothesis: exchange rate movements carry information relevant to forecasting USD-denominated commodity prices, even though none of the three series is easily forecastable in isolation.
