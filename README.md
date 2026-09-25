
# Macro-FX Linkages: Forecasting Gold and Crude with USD/INR

A time series analysis of the relationship between the USD/INR exchange rate and two USD-denominated commodities (gold, crude oil). Tests whether exchange rate movements carry predictive information about commodity prices, using real, unclean market data. Built for the Applied Time Series Analysis course (NMIMS MPSTME).

## Motivation

India imports the majority of its crude oil and a large share of its gold demand in US dollars, so the rupee-dollar exchange rate is mechanically linked to the domestic cost of both commodities. This project tests that linkage empirically: does USD/INR help forecast gold and crude prices, or are the three series largely independent?

## Data

- **Source:** Yahoo Finance (`yfinance`), daily frequency, Jan 2019 – present
- **Series:** USD/INR (`INR=X`), Gold futures (`GC=F`), Crude oil futures (`CL=F`)
- Real, unaligned trading calendars across FX and commodity markets — genuine missing-date handling required, not a pre-cleaned dataset
- Includes the 2020-04-20 WTI crude event (futures settled at **-$37.63/barrel**), a real, documented market anomaly kept in the dataset and handled explicitly at the log-transform step

## Methodology

| Stage                 | Technique                                                                                          |
| --------------------- | -------------------------------------------------------------------------------------------------- |
| Preprocessing         | Log transform (variance stabilization); negative-price date handled explicitly                     |
| Stationarity          | Augmented Dickey-Fuller test on levels and first differences                                       |
| Univariate modeling   | `auto_arima` order selection, ARIMA fit/forecast/evaluate per series                             |
| Multivariate modeling | VAR (Vector Autoregression) — lag order selection via AIC/FPE, joint modeling of all three series |
| Dependency analysis   | Pairwise correlation matrix of differenced (return) series                                         |
| Diagnostics           | Residual analysis (correlogram, Q-Q, histogram) for both ARIMA and VAR                             |

## Key Results

All three series are non-stationary in log-levels (ADF p > 0.05) and stationary after first differencing (d=1), consistent with standard random-walk behavior in daily financial prices.

**VAR outperformed univariate ARIMA** on out-of-sample RMSE for all three series (log-price scale, 60-day holdout):

| Series    | ARIMA RMSE | VAR RMSE |
| --------- | ---------- | -------- |
| USD/INR   | 0.0140     | 0.0099   |
| Gold      | 0.0785     | 0.0523   |
| Crude Oil | 0.2065     | 0.1972   |

Crude oil showed the strongest own-lag structure of the three, which limited how much cross-series information from INR/gold could add to its forecast. Impulse response functions show how a shock to USD/INR propagates through gold and crude forecasts over the following days.

## Repository Structure

```
├── notebooks/
│   └── time_series_analysis.ipynb   # full pipeline: data → EDA → ARIMA → VAR
├── models/                          # saved fitted models (.pkl)
├── figures/                         # exported charts
└── requirements.txt
```

## Tech Stack

Python · pandas · numpy · statsmodels · pmdarima · yfinance · plotly · scikit-learn

## Setup

```bash
pip install -r requirements.txt
```

Run `notebooks/time_series_analysis.ipynb` top to bottom. Fitted models are saved via `results.save()` for reuse outside the notebook.

## Limitations

- Daily return series for FX/commodities are close to white noise at short lags, limiting forecastability from own-history alone (consistent with the random walk hypothesis for exchange rates).
- Correlation-based dependency analysis (rather than a formal Granger causality test) was used to characterize cross-series relationships, in line with the project's intended scope.

---

Built by [Gauri Deshmukh](https://github.com/Gauri210)
