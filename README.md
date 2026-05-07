# Statistics and Financial Data Analysis

**Oxford Mathematical and Computational Finance — Statistics and Financial Data Analysis | December 2024**

End-to-end statistical analysis of option markets and financial time series. The exam covers four areas: implied volatility smile interpolation, spline model selection via cross-validation, time-series modeling (ARIMA + GARCH), and PCA decomposition of the ATM implied volatility term structure.

---

## Project Structure

```
├── exam_solutions.ipynb                  # Main solutions notebook
├── exam_draft.ipynb                      # Draft / working copy
├── consigne.pdf                          # Exam problem statement
├── report.pdf                            # Submitted exam report
├── data/
│   ├── call_prices.csv                   # European call prices at T=0.1 and T=2
│   ├── implied_vols.csv                  # Implied volatilities at T=0.1 and T=2
│   ├── new_quotes.csv                    # Additional quotes at more strikes (for CV)
│   ├── traded_volume.csv                 # Daily traded volume time series
│   └── atm_implied_vol_term_structure.csv # ATM implied vol across 6 maturities (daily)
└── output/                               # Generated figures (see section below)
```

---

## Section 1 — Implied Volatility Smiles & Call Prices

**Questions 1.i – 1.ii**

Jim has fitted piecewise linear models to option market data at maturities T=0.1 and T=2. The first task is to assess these fits visually and statistically.

**Piecewise linear model (Jim's specification):**

$$f(k) = b_0 + b_1 k + c_1 (k - K_1/S_0)_+ + c_2 (k - K_2/S_0)_+$$

where $k = K/S_0$ is moneyness and the knots are at $K_1 = 83.33$, $K_2 = 116.67$.

**Key finding:** Jim's model contains a scale error — the knots are expressed in absolute strike space but applied to moneyness-normalized data, causing a systematic misfit. The piecewise linear shape also cannot reproduce the convexity of market call prices, violating the no-arbitrage condition that call prices must be convex in strike.

**Q1.ii — Stationarity tests (ADF):**
- `traded_volume.csv` (Jul 2020–Jun 2024): ADF test determines whether the series is stationary.
- `atm_implied_vol_term_structure.csv` T=0.1 (Nov 2015–Jul 2020): ADF test on the ATM IV level.

---

## Section 2 — Model Assessment & Selection

**Questions 2, 3, 4**

### Q2 — AIC and Adjusted R² for Piecewise Linear Fits

Jim's models are evaluated using:
- **AIC** = $n \log(\text{RSS}/n) + 2k$
- **Adjusted R²** = $1 - (1 - R^2)(n-1)/(n-k)$

Jim's coefficients are compared against OLS refits with the same knots. A grid search over knot positions $(k_1, k_2)$ finds the globally optimal piecewise linear model by minimizing AIC — consistently outperforming Jim's choice of knots.

### Q3 — Cubic Spline (Truncated Power Basis)

To enforce finite positive second derivatives (required by no-arbitrage), the piecewise linear model is replaced by a **truncated power basis (TPB) cubic spline**:

$$f(k) = b_0 + b_1 k + b_2 k^2 + b_3 k^3 + \sum_j c_j (k - \kappa_j)^3_+$$

Jim's cubic has 1 interior knot at $\kappa = 1.039$ (5 parameters, 1 more degree of freedom than the linear model). AIC comparison shows the cubic spline improves fit meaningfully while remaining parsimonious.

### Q4 — Cross-Validation and Optimal Complexity

Using the extended dataset from `new_quotes.csv`:

**Jim's LOO-CV (flawed):** standardizes the full dataset before splitting, causing data leakage — test points inform the scaler, inflating out-of-sample performance.

**Proper 5-fold CV:** scaler and knot positions are fit exclusively on training folds. This gives an honest estimate of generalization error.

**Result:** The 5-fold CV curve identifies **k = 6 interior knots** as optimal. An elbow at k=3 is also examined as a more parsimonious alternative. Full regression diagnostics are run on the k=6 model (residuals, QQ plot, Ljung-Box, Durbin-Watson, VIF).

---

## Section 3 — Time Series Analysis

**Questions 5, 6**

### Q5 — Traded Volume: ARIMA + GARCH

**Data:** `traded_volume.csv`, period 2014-01-10 to 2017-01-10.

**Pipeline:**
1. Stationarity: ADF + KPSS tests — series is non-stationary in levels. Differencing required (ADF and KPSS disagree at the 1% level on the raw series).
2. Seasonality diagnostics (STL decomposition, period=252 trading days).
3. ARIMA auto-selection by AIC over p, q ∈ {0,...,3} — finds best mean model.
4. Residual diagnostics: ACF, PACF, Ljung-Box — evidence of remaining volatility clustering.
5. **GARCH(1,1) with Student-t distribution** fitted via the `arch` library. Captures heteroscedasticity in the residuals.

### Q6 — ATM Implied Volatility: ARIMA + GARCH

**Data:** `atm_implied_vol_term_structure.csv`, column T=0.25, full period.

Same pipeline as Q5 applied to the ATM IV series:
1. Stationarity tests + differencing.
2. ARIMA auto-selection.
3. **GARCH(1,0,2) with Skew-Student distribution** — selected for better tail fit given the asymmetric distribution of IV changes.

---

## Section 4 — PCA on ATM Implied Volatility Term Structure

**Question 9.2**

**Data:** `atm_implied_vol_term_structure.csv` — daily ATM implied volatility across 6 maturities: T = 0.1, 0.25, 0.5, 1, 2, 5 years.

**Method:** PCA applied to the **daily changes** of the term structure (first differences), which removes the non-stationary level and focuses on the dynamics.

**Results — 3 principal components:**

| Component | Interpretation | Variance explained |
|-----------|---------------|-------------------|
| PC1 | **Parallel shift** — all maturities move together with similar loadings | ~80% |
| PC2 | **Slope / rotation** — short-end moves opposite to long-end | ~12% |
| PC3 | **Curvature / hump** — medium maturities move opposite to the wings | ~5% |

This is the classic 3-factor decomposition of interest rate and volatility term structures. Together, 3 PCs explain ~97% of total variance in ATM IV changes.

**Visualizations:**
- Term structure curves at 3 dates (Jan 2016, Dec 2017, Dec 2018) + historical mean.
- Explained variance ratio bar chart.
- For each PC: loading bar chart + effect on the mean term structure (mean ± PC).
- Parallel analysis to validate the number of significant components.

---

## Data Files

| File | Description |
|------|-------------|
| `call_prices.csv` | European call prices vs strike at T=0.1 and T=2 |
| `implied_vols.csv` | Implied volatilities vs strike at T=0.1 and T=2 |
| `new_quotes.csv` | Extended dataset with more strikes at T=0.1 and T=2 — used for CV model selection |
| `traded_volume.csv` | Daily traded volume time series (full history) |
| `atm_implied_vol_term_structure.csv` | Daily ATM implied vol across 6 maturities (0.1, 0.25, 0.5, 1, 2, 5 years) |

---

## Output Figures

| File | Content |
|------|---------|
| `pca_atm_term_structure_curves.png` | ATM vol term structure at 3 dates + historical mean |
| `atm_iv_daily_changes.png` | Daily changes in ATM IV across all maturities |
| `pca_variance_ratio.png` | PCA explained variance ratio per component |
| `pca_pc1_parallel_shift.png` | PC1 loadings + effect on term structure |
| `pca_pc2_slope.png` | PC2 loadings + effect on term structure |
| `pca_pc3_curvature.png` | PC3 loadings + effect on term structure |
| `pca_parallel_analysis.png` | Parallel analysis for PCA component selection |
| `spline_residuals_vs_fitted_k6.png` | Residuals vs fitted for cubic spline (k=6) |
| `spline_qq_residuals_k6.png` | QQ plot of spline residuals |
| `spline_acf_residuals_k6.png` | ACF of spline residuals |
| `spline_ljungbox_residuals_k6.png` | Ljung-Box p-values across lags |
| `spline_convexity_k6.png` | Second derivative of call price curve (convexity check) |
| `spline_predictor_correlation_k6.png` | Correlation matrix of TPB basis functions (multicollinearity) |

---

## Dependencies

```
numpy
pandas
scipy
matplotlib
seaborn
statsmodels
scikit-learn
arch
```

Custom time-series utilities (`stats_libs`) used for stationarity checks, ARIMA auto-selection, seasonality diagnostics, and GARCH selection — available at `/Users/dnn/Projects/stats_libs`.
