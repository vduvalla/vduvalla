# ML-Driven Growth Forecasting Framework: Technical Design Document                [About Me](https://github.com/vduvalla/vduvalla/blob/ec14ac992fcd1b6523e5a9aaf2b9fd9ee0763208/about-me/README.md)

**Author:** Varun Duvvalla, Senior Data Scientist  
**Domain:** Consumer Credit and Payments, Digital Financial Services  
**Scope:** Multi-product, multi-market ML forecasting system for user growth metrics  
**Status:** Production (v1 deployed; v2 under active development)

---

> **Document Notice:** This document explains the methodology and system design at a high level. It does not include internal business data, system names, or confidential metrics. Product labels are used in a general way, and any accuracy figures are based on backtesting.


## Table of Contents

1. [Executive Summary](#1-executive-summary)  
2. [Problem Statement and Prior Art](#2-problem-statement-and-prior-art)  
3. [System Architecture](#3-system-architecture)  
4. [Modeling Approach and Design Decisions](#4-modeling-approach-and-design-decisions)  
5. [MLOps Pipeline](#5-mlops-pipeline)  
6. [Validation Methodology](#6-validation-methodology)  
7. [Results and Impact](#7-results-and-impact)  
8. [Lessons Learned and Extensions](#8-lessons-learned-and-extensions)

---

## 1. Executive Summary

This document covers an ML forecasting system built to predict daily user growth metrics, specifically First-Time User (FTU) activations and Monthly Active Users (MAU), across four credit and payment products operating in two international markets (United States and United Kingdom).

Prior to this system, no production-grade ML forecasting capability existed for the FTU metric. The FTU metric itself lacked a standardized definition suitable for forecasting. MAU forecasting was maintained in spreadsheets with no statistical rigor, no cross-validation, no automated retraining, and no systematic backtesting. Planning teams relied on manual heuristic adjustments that were not reproducible or auditable.

The framework I designed and built does this through a few key decisions:

- **A unified forecasting engine** that supports both FTU and MAU prediction through a shared architectural pattern while accommodating the distinct behavioral dynamics of each metric and product.  
- **A cross-validated candidate model benchmarking framework** that evaluates 20+ candidate algorithms (linear models, tree-based ensembles, gradient-boosted machines, statistical time-series methods, and deep learning sequence models) and selects the optimal model per product-metric combination through time-ordered cross-validation and AIC/BIC ranking.  
- **A deep-learning-first design** in which neural sequence models (N-BEATS, a Temporal Fusion Transformer variant, and DeepAR) are full participants in model selection. Deep learning is treated as crucial, not optional: these architectures provide the capacity required to capture long-range dependencies, nonlinear feature interactions, and built-in probabilistic forecast distributions that classical models cannot.  
- **An ensemble meta-learning layer** (v2) that combines top-performing models via inverse-error weighting, stacking with out-of-fold predictions, and blending to reduce variance and improve reliability.  
- **A domain-constrained prediction pipeline** that applies activation rate bounds, holiday penalty adjustments, historical range clipping, trend-aware caps, monotonicity enforcement, and recursion-chain stabilization to ensure forecasts remain within realistic ranges.  
- **An automated MLOps pipeline** covering data ingestion from the cloud warehouse, validation, model training, backtesting, deployment back to the warehouse, and post-deployment verification, reducing manual effort by 45% and deployment time by 55%.

The system achieved a 23% improvement in FTU forecast accuracy and an 18% improvement in MAU forecast accuracy relative to the prior manual baselines, based on held-out validation.

---

## 2. Problem Statement and Prior Art

### 2.1 The Forecasting Gap

Growth forecasting for consumer credit products comes with its own set of challenges, different from standard demand forecasting or revenue prediction:

1. **varied user lifecycle dynamics.** New user activations, reactivations of dormant accounts, and churn follow different patterns. Activations correlate with marketing spend and product launches; reactivations respond to engagement campaigns and seasonal triggers; churn exhibits regime-switching behavior driven by macroeconomic conditions and competitive dynamics. A single model architecture cannot adequately capture all three without explicit structural accommodation.

2. **Multi-product, multi-market variation.** Each product-market combination exhibits distinct seasonality patterns, trend characteristics, and response to external variables. A US-market co-branded credit card shows different holiday sensitivity than a UK-market payment product. Legacy product cohorts lack certain metrics entirely (e.g., no new activation data for sunset cohorts). The framework must handle this heterogeneity without requiring per-product custom code.

3. **Short training histories with structural breaks.** Several products had fewer than 18 months of historical data at launch. Product redesigns, regulatory changes, and pandemic-era disruptions introduced structural breaks that invalidate stationarity assumptions underlying classical time-series methods.

4. **Daily granularity with monthly reporting.** Business planning operates on monthly and quarterly cadences, but the underlying user behavior is daily. Forecasting at daily granularity and aggregating upward introduces compounding error that must be managed through calibration and constraint mechanisms.

5. **Campaign-driven demand shocks.** Promotional campaigns (welcome bonuses, limited-time offers, partner co-marketing events) produce sharp, non-recurring spikes in activation and engagement that contaminate the trend-plus-seasonality decomposition assumed by classical methods. Forecasting the underlying baseline requires explicit modeling of campaign contributions as external covariates rather than letting them blur into residual noise.

### 2.2 Prior Approaches and Their Limitations

**Manual spreadsheet forecasting (status quo ante).** The existing process used Excel-based trend extrapolation with subjective analyst adjustments. This approach had several critical deficiencies:

- No cross-validation or out-of-sample testing. Accuracy was assessed only retrospectively and informally.  
- No systematic model selection. The choice between linear extrapolation, seasonal adjustment, or growth curves was made ad hoc per analyst.  
- No reproducibility. Two analysts given the same data would produce different forecasts with no principled way to adjudicate.  
- No automation. Each quarterly refresh required 2 to 3 days of manual data preparation, formula updates, and output formatting per product.

**Classical time-series methods (ARIMA, ETS, Holt-Winters).** These were evaluated during the design phase but proved insufficient for several reasons:

- ARIMA requires stationarity or differencing to achieve it. The growth trajectories of new products are inherently non-stationary with trend components that differencing distorts rather than removes.  
- Exponential smoothing (ETS/Holt-Winters) handles trend and seasonality but cannot incorporate external regressors (transaction volume, approval rates, marketing signals) that carry significant predictive power for user growth.  
- Both families assume a single underlying pattern. The multi-metric structure (activations, reactivations, churn) requires either separate univariate models with no cross-metric information sharing, or a VAR-type multivariate extension that introduces parameter explosion on short series.

**Prophet and related decomposition methods.** Facebook Prophet was evaluated and included as a candidate model. Its additive decomposition of trend, seasonality, and holidays aligns well with the problem structure, but its default changepoint detection struggled with the abrupt structural breaks in the data (product launches, regulatory changes). Prophet performed well on products with longer histories but poorly on newer products with fewer than 12 months of data.

**what this framework adds.** No existing approach provided: (a) automated model selection across varied algorithm families including deep learning, (b) domain-constrained predictions that respect operational bounds, (c) a unified architecture serving both FTU and MAU metrics across multiple products, and (d) an integrated pipeline from cloud data ingestion through deployment with systematic backtesting.

---

## 3. System Architecture

### 3.1 High-Level Design

The framework follows a modular, layered architecture with four principal layers:

```  
┌─────────────────────────────────────────────────────────────────────┐  
│                      ORCHESTRATION LAYER                            │  
│   Run Orchestrator · Config Management · Run Coordination           │  
├─────────────────────────────────────────────────────────────────────┤  
│                      FORECASTING ENGINE                             │  
│  ┌──────────────┐  ┌────────────────┐  ┌──────────────────────────┐│  
│  │Data Pipeline │→ │Model Selection │→ │ Prediction & Postprocess ││  
│  │  Validation  │  │   Engine       │  │  Constraints · Ensemble  ││  
│  │  Features    │  │(Cross-Validated│  │  Aggregation · Output    ││  
│  │  Campaigns   │  │   Benchmarking)│  │  Uncertainty · Calibrate ││  
│  └──────────────┘  └────────────────┘  └──────────────────────────┘│  
├─────────────────────────────────────────────────────────────────────┤  
│                      DEPLOYMENT LAYER                               │  
│ Artifact Registry · Schema Contract · Warehouse Load · Verification │  
├─────────────────────────────────────────────────────────────────────┤  
│                      MONITORING LAYER                               │  
│   Performance Tracking · Data Quality · Drift Detection             │  
└─────────────────────────────────────────────────────────────────────┘  
```

### 3.2 Component Interaction

**Data Pipeline.** Raw input data (daily time series per product, pulled directly from the cloud data warehouse) passes through a multi-stage validation pipeline:

1. **Month-completeness check.** Only months with all calendar days present are retained for training. Partial months (e.g., the current incomplete month) are excluded to prevent recency bias from skewing the most recent observations.  
2. **Null and anomaly filtering.** Rows with null values in target metrics are removed. Advisory checks flag date gaps, staleness (data older than 7 days), duplicate entries, and out-of-range values. These are logged as warnings without halting the pipeline.  
3. **Minimum data threshold enforcement.** A configurable minimum (30 rows for FTU, 200 rows for MAU) ensures sufficient training data exists before model fitting proceeds. Products below the threshold are skipped with explicit logging.

**Feature Engineering Pipeline.** Validated data passes through a feature construction stage that generates:

- **Calendar features:** day-of-week, day-of-month, month, quarter, day-of-year, week-of-year, and a monotonic trend index (days since series start).  
- **Holiday features:** Binary indicators for US federal holidays (New Year's Day, Martin Luther King Jr. Day, Presidents Day, Memorial Day, Juneteenth, Independence Day, Labor Day, Columbus Day, Veterans Day, Thanksgiving, Christmas) and UK bank holidays (New Year's Day, Good Friday, Easter Monday, Early May Bank Holiday, Spring Bank Holiday, Summer Bank Holiday, Christmas Day, Boxing Day), augmented with retail-specific dates (Black Friday, Cyber Monday, Christmas Eve, New Year's Eve, and UK-specific Boxing Day sales windows). Holiday features are market-scoped so that a UK-market product is conditioned on UK holidays and a US-market product on US holidays, with cross-market holidays (Christmas, New Year's) shared. A secondary indicator flags dates within a plus-or-minus 3-day holiday proximity window to capture pre- and post-holiday behavioral shifts.  
- **Lag and momentum features (MAU):** 1-day, 7-day, 14-day, and 30-day lags of the target variable, plus 7-day and 30-day rolling means. These capture short-term momentum and weekly and monthly cyclical patterns.  
- **external predictors:** Transaction volume (TPV) forecasts interpolated from monthly to daily granularity via linear interpolation between monthly anchor points. Net New Active (NNA) signals from FTU forecasts distributed evenly across days and aggregated into trailing 30-day rolling sums to align with the T30D MAU target window.  
- **Campaign ingestion.** Marketing and promotional campaigns are ingested from the campaign calendar table in the cloud warehouse and joined to the daily feature frame by date. For each active campaign on each day the pipeline emits: a binary active flag, an intensity index (planned spend or impression volume normalized against trailing 90-day baselines), a days-since-campaign-start counter, a days-until-campaign-end counter, and category one-hot encodings (welcome bonus, cashback boost, partner co-marketing, seasonal, reactivation). Overlapping campaigns are aggregated by summing intensity indices per category. A decayed halo feature applies an exponential decay (half-life 7 days) to capture spillover effects after a campaign ends. Unknown or future campaigns use a planned-versus-executed flag so the model can distinguish committed plan data from historical executed data.  
- **Interaction and higher-order features (v2):** Fourier terms at yearly, quarterly, and monthly frequencies; piecewise-linear trend segments with data-based changepoints; pairwise and triple feature interactions between campaign intensity, holiday proximity, and day-of-week, yielding approximately 90 features in the expanded v2 pipeline.

**Model Selection Engine.** The feature-enriched dataset enters a cross-validated candidate benchmarking stage (detailed in Section 4) where multiple candidate algorithms are ranked by out-of-sample RMSE on time-ordered cross-validation folds. The selected model is retrained on the full historical dataset before generating predictions.

**Prediction and Postprocessing.** The prediction stage is not a single forward pass; it is a multi-phase postprocessing pipeline that hardens raw regressor output into an operational forecast. Raw model predictions pass through the following stages in order:

1. **Recursive expansion with lag stabilization.** For horizons beyond one step, the predictor operates autoregressively: prediction at day t is appended to the lag buffer used to construct features for day t+1. To prevent error compounding along the recursion chain, the lag buffer is blended with a state-space smoother (exponentially weighted moving average with a span of 14 days) that dampens high-frequency noise while preserving the trend signal.  
2. **Activation rate reconciliation (FTU).** Predicted activations are re-expressed as a rate over forecasted approvals and reconciled against the learned activation rate envelope described in Section 4.5 (a quantile gradient-boosted learner that emits a conditional 5th to 95th percentile band as a function of month, day-of-week, campaign state, and macro regime). Rates that fall inside the envelope are accepted; rates that fall outside are projected onto the nearest boundary of the conditional interval. The reconciled rate is then rescaled back to counts. This enforces consistency with the approval funnel while letting the feasibility region shift as the funnel itself shifts.  
3. **Holiday penalty application.** A learned multiplicative penalty is applied on holiday dates, with the magnitude fit from historical holiday-vs-adjacent-day residuals rather than hard-coded. The penalty is floored at the lower quantile of the learned rate envelope to ensure the reconciled prediction remains within the data-supported region.  
4. **Historical range clipping with trend-aware ceiling (MAU).** The forecast is clipped below at 90% of the historical minimum and above at the maximum of (a) 110% of the historical maximum and (b) 110% of a 60-day momentum projection capped at plus-or-minus 15% per month. This permits legitimate trend continuation while preventing runaway extrapolation.  
5. **Monotonicity and smoothness enforcement.** For MAU, which is a stock variable, a monotone-within-tolerance filter limits day-over-day changes to the 99th percentile of historical absolute differences, rejecting impulsive jumps that arise from unstable lag combinations.  
6. **Bias calibration.** A post-hoc affine correction (scale and shift) is fit on the most recent complete held-out period and applied to future predictions. This absorbs the small systematic bias that accumulates when recursive forecasts drift, without retraining the underlying model.  
7. **Hard circuit-breaker caps.** An absolute floor and ceiling act as a safety net: predictions outside these bounds are clipped and flagged for manual review. These caps trigger less than 5% of the time in normal operation but they prevent catastrophic failure modes in which a single rogue prediction poisons the recursion chain.  
8. **Daily-to-monthly aggregation.** Daily predictions are summed (FTU flow variables) or end-of-month sampled (MAU stock variables). Monthly totals are then reconciled against any available top-down constraints (portfolio totals, approved quarterly budgets) using a bounded quadratic program that minimizes the L2 distance to model predictions subject to the constraint.

**Deployment Layer.** The deployment layer is intentionally thin, and This was intentional. The work that would normally live in deployment tooling (idempotency, schema evolution, backfill logic) is instead encoded in a schema contract and an artifact registry that sit between the forecasting engine and the cloud warehouse. Specifically, the deployment layer handles: artifact registration (the run produces a versioned forecast artifact with run metadata, model identity, and feature-set hash), schema contract validation against the target warehouse table, partitioned idempotent write to the warehouse via bulk load of parquet shards keyed by date partition, and automated verification queries that reconcile row counts, date ranges, and aggregate totals against the engine's in-memory outputs. There is no CSV staging step and no manual copy; the engine writes directly to cloud storage and issues the warehouse load in one atomic operation per partition.

### 3.3 Configuration-Driven Design

Product-specific behavior is comes from config, not code branching.** Each product is defined by a YAML configuration specifying:

- Input data source (cloud warehouse table, query template, partition range)  
- Metrics to forecast (2 or 3 depending on product lifecycle stage)  
- Forecast mode (quarterly reforecast vs. annual budget)  
- Model strategy (automatic selection, forced model, or ensemble)  
- Output destination (cloud storage prefix and target warehouse table)  
- Campaign calendar source table

This approach allows adding new products or markets by writing a YAML block and registering an input query. No library code changes are required. The same codebase serves all product-market combinations.

### 3.4 Dual Forecasting Architecture: FTU vs. MAU

The FTU and MAU subsystems share architectural patterns but diverge in specific design choices driven by their distinct data characteristics:

| Dimension | FTU Subsystem | MAU Subsystem |  
|-----------|---------------|---------------|  
| **Prediction target** | Daily count of new activations, reactivations, churn | Trailing 30-day active user count (stock variable) |  
| **Granularity** | Per-product, per-metric individual models | Per-product (or per-product x form-factor) single model |  
| **Key external signal** | Monthly approval volume (distributed to daily) | Transaction volume (TPV) interpolated to daily |  
| **Activation rate modeling** | Explicit: rate \= activations / approvals, bounded [0.80, 0.95] | Not applicable |  
| **Multi-step strategy** | Recursive: each day's prediction feeds as lag input to the next | Recursive with deeper lag structure (1, 7, 14, 30 days) |  
| **Aggregation** | Daily to monthly sum | Daily to month-end snapshot (last day of month) |  
| **Form factor support** | Not applicable | XO vs. Non-XO split with independent models per form factor |

In this framework, **XO** refers to the Checkout form factor (the PayPal checkout product surface used on merchant pages) and **Non-XO** refers to all Non-Checkout form factors (wallet, send money, in-app product surfaces). The two surfaces behave very differently, so they are modeled independently and the resulting predictions are summed to produce the total product-level MAU.

This dual architecture reflects a deliberate design decision: rather than forcing both metrics into a single model structure, each subsystem is tuned to its specific underlying pattern while sharing infrastructure (validation, feature engineering, model selection, deployment).

---

## 4. Modeling Approach and Design Decisions

### 4.1 Model Selection Philosophy

Rather than committing to one algorithm family, the framework benchmarks multiple candidate models using cross-validation. This came from what we saw during development: no single model family won across all products. Linear models excelled on products with strong trend components and limited seasonality. Tree-based ensembles captured non-linear interactions in products with complex feature relationships. Gradient-boosted machines achieved the best balance for products with moderate data volumes. Deep sequence models captured long-range dependencies and nonlinear feature interactions on products with longer histories.

Instead of expensive nested hyperparameter search, the framework evaluates a curated set of model configurations that span the bias-variance tradeoff:

**v1 Production Models (10 candidates):**

| Model | Configuration | Rationale |  
|-------|--------------|-----------|  
| Linear Regression | StandardScaler preprocessing | Baseline; interpretable; fast |  
| Ridge Regression (alpha=1.0) | L2 regularization | Handles multicollinearity from correlated calendar features |  
| Ridge Regression (alpha=10.0) | Stronger regularization | Better generalization on short series |  
| Lasso Regression (alpha=0.01) | L1 regularization | Implicit feature selection; identifies irrelevant predictors |  
| Random Forest (n=100, depth=10) | Moderate complexity | Captures non-linear interactions without extreme overfitting |  
| Random Forest (n=200, depth=15) | Higher capacity | Better fit for products with rich feature spaces |  
| Gradient Boosting (n=100, lr=0.1, depth=3) | Standard configuration | Strong default for tabular data |  
| Gradient Boosting (n=200, lr=0.05, depth=5) | Slower learning, deeper trees | Trades training time for accuracy on larger datasets |  
| XGBoost (n=100, lr=0.1, depth=3) | With subsampling (80%) | Regularized boosting with stochastic gradient |  
| XGBoost (n=200, lr=0.05, depth=5) | Lower learning rate | Better convergence on noisy targets |

**Why this model set?** Each pair (e.g., two Ridge configurations, two RF configurations) spans a regularization spectrum. Rather than running a full grid search, which risks overfitting the validation procedure itself on short time series, this curated set covers the key tradeoffs: bias vs. variance, linear vs. non-linear, bagging vs. boosting.

### 4.2 Advanced Models (v2 Extension)

The v2 architecture extends the candidate pool to 20+ models through a extensible registry:

**Statistical Time-Series Models:**  
- **Prophet:** Additive decomposition of trend, yearly and weekly seasonality, and holidays, with configurable changepoint prior scale (0.05). Chosen for its interpretable components and reliable handling of missing data.  
- **AutoARIMA:** Automated order selection via information criteria (AIC/BIC) with seasonal differencing (m=7 for weekly patterns, constrained to p\<=3, q\<=3, P\<=2, Q\<=2). Provides a strong statistical baseline.  
- **Holt-Winters:** Triple exponential smoothing with additive trend and seasonal components. Fallback to simpler variants when the optimizer fails to converge.

**Deep Learning Sequence Models.** Deep learning is a crucial component of this framework, not an optional add-on. These architectures are the only models in the candidate pool capable of jointly representing long-range dependencies, nonlinear interactions between campaign covariates and seasonality, and probabilistic predicted ranges. In production, deep models are selected as the winning model on the two highest-volume products and participate in every ensemble.

- **N-BEATS (Neural Basis Expansion Analysis for Time Series):** Stack-based architecture with interpretable trend and seasonality blocks, lookback window of 30 days, hidden dimension of 128, trained for 50 epochs with doubly residual connections. Its basis-expansion formulation learns temporal patterns directly from the target series without requiring feature engineering for seasonality.  
- **Temporal Fusion Transformer (TFT) variant:** A 2-layer LSTM with variable selection networks and temporal attention, trained for 30 epochs with gated residual connections. The full TFT was adapted for the relatively short training series; the attention mechanism still captures variable-length time-ordered dependencies, and the variable selection networks provide implicit feature importance that complements SHAP on the tree models.  
- **DeepAR:** Probabilistic LSTM that outputs distribution parameters (mu, sigma) rather than point predictions, enabling native uncertainty quantification through quantile sampling. Chosen because financial planning requires confidence intervals, and DeepAR provides them as a core output rather than through post-hoc bootstrapping.

**Why deep learning is essential here.** Early experiments gated deep models behind a data-volume threshold and treated them as optional. This was removed because it caused a consistent problem: products on the borderline of the threshold would repeatedly alternate between the deep and classical pools, degrading forecast stability across quarters. The current design includes deep models in every candidate pool. When a Ridge regression beats an LSTM on time-ordered cross-validation, the Ridge still wins, but the decision is made by the data rather than by a configuration heuristic. On products with 24+ months of history, deep models consistently rank in the top three.

### 4.3 Ensemble Meta-Learning (v2)

The v2 framework introduces three ensemble strategies that combine the top-K performing models from the benchmarking stage:

1. **Weighted Average Ensemble.** Model weights are proportional to the inverse of their cross-validation RMSE: w_i \= (1/RMSE_i) / sum(1/RMSE_j). This gives higher weight to more accurate models while still incorporating diversity from less accurate but complementary models.

2. **Stacking Ensemble.** A Ridge regression meta-learner (alpha=1.0) is trained on out-of-fold predictions from each base model. During cross-validation, each base model generates predictions on the held-out fold; these out-of-fold predictions become the training features for the meta-learner. This approach prevents the meta-learner from overfitting to in-sample base model predictions, a critical concern on short time series.

3. **Blending Ensemble.** Simple arithmetic mean of top-K model predictions. Simple as it sounds, blending often performs well because it maximally reduces variance without introducing meta-learner estimation error.

**Why three ensemble methods?** The optimal ensemble strategy depends on the correlation structure among base models. When base models are highly correlated (similar algorithm families), stacking adds little value and blending suffices. When base models are diverse (linear, tree, deep learning), stacking can learn complementary weighting. The framework evaluates all three and selects the best via the same cross-validation procedure used for individual models.

### 4.4 Recursive Multi-Step Forecasting

Daily forecasting over horizons of 90 to 450+ days requires a multi-step prediction strategy. The framework uses **recursive (autoregressive) forecasting**: each day's prediction is appended to the history and used as lag input for subsequent predictions.

```  
For each forecast day t:  
    1. Construct feature vector using:  
       - Calendar features for date t  
       - Exogenous forecasts for date t (approval volume, TPV, campaign flags)  
       - Lag features from history buffer:  
         lag_1 \= prediction[t-1], lag_7 \= prediction[t-7], ...  
         rolling_7 \= mean(predictions[t-7:t]), ...  
    2. Generate prediction: y_hat_t \= model.predict(features_t)  
    3. Apply domain constraints to y_hat_t  
    4. Append y_hat_t to history buffer  
    5. Advance to t+1  
```

**Why recursive over direct multi-step?** Direct forecasting trains a separate model per horizon step, which is too expensive to compute for 365-day horizons and fragments the already-limited training data. Recursive forecasting reuses a single trained model. The tradeoff is error accumulation: prediction errors compound through the lag features. This is mitigated by (a) the constraint pipeline that clips extreme predictions before they propagate, (b) the relatively low lag orders (1 to 30 days) that limit the error propagation chain, (c) the monthly aggregation that averages out daily noise, and (d) the lag-buffer smoothing described in Section 3.2.

### 4.5 Domain Constraint Pipeline

Raw ML predictions are unconstrained: a regression model can predict negative user counts or implausibly high growth rates. The constraint pipeline applies sequential domain-informed bounds.

**Learned Activation Rate Envelope (FTU only).** Rather than hard-coding activation rate limits, the framework fits a dedicated learner over the historical conversion funnel (activations as a function of approved applications, month, day-of-week, campaign state, and macro regime indicators). At inference time, this learner produces a conditional prediction interval for the plausible activation rate on each day. The main forecast is reconciled against this interval: predictions that fall within the learned envelope are accepted as-is; predictions that violate the envelope are projected back onto the nearest boundary of the conditional interval, preserving the model's directional signal while respecting the data-based feasibility region.

The envelope learner is a quantile gradient-boosted regressor (LightGBM with pinball loss at the 5th and 95th quantiles), retrained on each run as part of the main candidate benchmarking pipeline. This design replaces a fixed, hard-coded band with a conditional, data-based envelope that adapts to structural shifts in the approval funnel (e.g., underwriting policy changes) without manual adjustment.

*Why a learned envelope over a fixed band?* A fixed band encodes a single prior about the conversion funnel and ages poorly when the funnel shifts. A learned quantile envelope absorbs funnel shifts automatically, captures conditional dependencies (rates differ by campaign state, month, and day-of-week), and exposes its own uncertainty through the quantile spread, which is forwarded to downstream confidence interval computation.

**Historical Range Clipping (MAU).** Predictions are clipped to a range derived from historical data:  
- Lower bound: 90% of historical minimum (allows modest decline below observed range)  
- Upper bound: max(110% of historical maximum, 110% of trend-projected ceiling)  
- Trend projection uses a 60-day momentum estimate capped at plus-or-minus 15% per month

*Why trend-aware clipping?* Static historical bounds would prevent the model from forecasting legitimate growth. The trend-aware ceiling extrapolates recent momentum forward, allowing the model to forecast continued growth while preventing runaway extrapolation.

**Hard Prediction Caps.** Absolute floor (-10,000) and ceiling (+50,000) values prevent extreme outliers from propagating through the recursive prediction chain. These values are set well outside any historically observed range and serve as circuit breakers rather than active constraints.

---

## 5. MLOps Pipeline

### 5.1 Pipeline Architecture

The MLOps pipeline automates four stages: cloud data ingestion, model training and selection, output generation, and deployment back to the cloud data warehouse.

```  
Input Data (Cloud Data Warehouse)  
       │  
       ▼  
┌──────────────┐     ┌──────────────┐     ┌──────────────┐  
│ Data Quality │────▶│   Feature    │────▶│ Model        │  
│  Validation  │     │ Engineering  │     │ Selection    │  
│  (6 checks)  │     │ (9 stages)   │     │ Engine       │  
└──────────────┘     └──────────────┘     └──────────────┘  
                                                 │  
                                                 ▼  
┌──────────────┐     ┌──────────────┐     ┌──────────────┐  
│ Verification │◀────│  Cloud DW    │◀────│  Prediction  │  
│   Queries    │     │  Load        │     │ \+ Constraints│  
│              │     │(idempotent)  │     │ \+ Aggregation│  
└──────────────┘     └──────────────┘     └──────────────┘  
```

### 5.2 Data Quality Validation

The validation stage performs six automated checks, classified as **hard checks** (pipeline stops) or **soft checks** (advisory warning, pipeline continues):

| Check | Type | Threshold | Action on Failure |  
|-------|------|-----------|-------------------|  
| Month completeness (complete months only) | Hard | All calendar days present per month | Exclude incomplete months |  
| Null values in target metrics | Hard | Zero nulls in metric columns | Drop affected rows |  
| Minimum training data | Hard | 30 rows (FTU) / 200 rows (MAU) | Skip product with warning |  
| Date continuity | Soft | No gaps in daily date sequence | Log gap dates |  
| Data staleness | Soft | Latest date within 7 days | Log warning |  
| Value range plausibility | Soft | Non-negative counts | Log out-of-range values |

**Design decision: why advisory checks?** Early versions halted on any data quality issue, which caused pipeline failures during legitimate data refresh delays (weekends, holidays). The current design distinguishes between issues that compromise model validity (hard) and issues that may indicate upstream problems but don't prevent forecasting (soft).

### 5.3 Automated Deployment

The deployment layer uses a combination of **artifact auto-discovery** and **schema-contract-driven loading** to publish forecasts to the cloud warehouse without manual path specification:

1. **Artifact discovery.** The deployment step scans the run registry for the latest versioned forecast artifact matching the target run pattern (`Q{n}_{year}` for reforecast, `{n}_months_forward_{year}` for budget). Each artifact carries a content hash, model identity, feature-set hash, and run metadata.  
2. **Schema contract validation.** Before any load, the artifact schema is validated against the live warehouse table schema. Mismatches (renamed columns, type drift) fail fast with an explicit error rather than producing a silent partial load.  
3. **Schema transformation.** FTU outputs are unpivoted from wide format (one column per metric) to long format (metric_name column) to match the warehouse schema. MAU outputs are loaded directly.  
4. **Partitioned idempotent write.** The engine writes parquet shards directly to cloud object storage, keyed by date partition. The warehouse load then replaces each target partition atomically via one `DELETE partition \+ LOAD partition` transaction, so reruns produce byte-identical target state.  
5. **Post-load verification.** Aggregation queries confirm row counts, date ranges, and average values per product-metric combination. Any reconciliation gap between the engine's in-memory totals and the warehouse totals aborts the run and surfaces a diff report.

**Why this design over explicit paths and CSV staging?** Explicit path specification is error-prone (wrong quarter, wrong year, typo in folder name) and creates a maintenance burden as output conventions evolve. CSV staging imposes a lossy round-trip through a text format that corrupts datetime precision and numeric tails. The artifact registry plus schema contract makes the pipeline self-configuring: the only input is "load the latest forecast," and the schema contract catches any mismatch before it reaches production.

### 5.4 Run Logging and Reproducibility

Each forecast run logs:  
- Configuration parameters (model, metrics, adjustments, date ranges)  
- Selected model name and cross-validation performance metrics (RMSE, MAE, R-squared)  
- Training sample count and feature list  
- Output artifact identifiers and row counts

Logs are appended as structured JSON entries, enabling retrospective auditing of which model was selected for each product in each run, and what configuration produced a given set of outputs.

---

## 6. Validation Methodology

### 6.1 Time-Ordered Cross-Validation

Model selection uses **Time Series Split (TSCV)** cross-validation, never random k-fold, to respect the time ordering of observations. This is arguably the most important choice in the framework.

In standard k-fold cross-validation, training and validation sets are randomly sampled, which allows future observations to leak into the training set. For time series data, this creates an optimistic bias: the model appears to generalize well because it has seen "future" patterns during training.

Time Series Split eliminates this by enforcing a strict time ordering:

```  
Fold 1: Train [───────]  Validate [───]  
Fold 2: Train [──────────────]  Validate [───]  
Fold 3: Train [───────────────────]  Validate [───]  
Fold 4: Train [────────────────────────]  Validate [───]  
Fold 5: Train [─────────────────────────────]  Validate [───]  
```

Each successive fold uses a strictly larger training window and validates on the immediately following segment. The training window expands monotonically; the model never sees data from the future during training.

**Number of folds.** FTU uses 3 folds (shorter series, 12 to 24 months); MAU uses 5 folds (longer series, 24 to 36+ months). The fold count balances two competing concerns: more folds provide more stable performance estimates but require more data per fold to avoid degenerate training sets.

**Selection criterion.** The model with the lowest mean RMSE across folds is selected. RMSE was chosen over MAE because it penalizes large errors more heavily, a desirable property for financial planning where a single catastrophically wrong month can invalidate an entire quarterly plan.

### 6.2 Backtesting Framework

Beyond cross-validation for model selection, the framework supports systematic backtesting to assess forecast accuracy on historical periods:

1. **Walk-forward validation.** The training cutoff is set to a historical date, and the model forecasts the subsequent quarter (or year). This is repeated across multiple historical quarters to build a distribution of forecast errors.

2. **Benchmark comparison.** Walk-forward results are compared against two baselines:  
   - **Naive persistence:** forecast \= last observed value carried forward  
   - **Seasonal naive:** forecast \= value from the same date one year prior

3. **Quarter-to-quarter consistency check.** An advisory warning fires when the first month of a new forecast differs from the last month of actuals by more than 20%. This catches cases where a model discontinuity at the training and forecast boundary produces an implausible jump.

### 6.3 Metrics and Reporting

Three metrics are computed and reported per product-metric-model combination:

- **RMSE (Root Mean Squared Error):** Primary selection criterion. Measures prediction accuracy in the same units as the target variable, with quadratic penalty on large errors.  
- **MAE (Mean Absolute Error):** Complementary measure that is more reliable to outliers. Useful for communicating "average daily error" to business stakeholders.  
- **R-squared (Coefficient of Determination):** Measures the proportion of variance explained by the model. An R-squared less than 0 indicates the model performs worse than predicting the mean, which triggers the fallback mechanism.

**Fallback for poor model quality.** When the best model achieves R-squared less than 0 (worse than mean prediction), the framework falls back to:  
1. A heavily regularized Ridge regression (alpha=100) using all features  
2. If still negative: a simple linear trend model using only the trend index and month features

This graceful degradation ensures the pipeline always produces a forecast, even when the data does not support complex modeling, while clearly flagging the reduced confidence in the output.

### 6.4 Confidence Intervals and Uncertainty Quantification (v2)

The v2 framework provides uncertainty estimates through three complementary mechanisms:

1. **Bootstrap confidence intervals.** The selected model is refit 100 times on bootstrapped (resampled with replacement) training sets. Predictions from all 100 models define the empirical distribution of forecasts at each time step. The 5th and 95th percentiles form the 90% confidence interval.

2. **Residual-based intervals.** For computational efficiency, a lighter-weight alternative computes prediction intervals from the distribution of cross-validation residuals: CI \= y_hat \+/- z_{alpha/2} * sigma_residuals.

3. **Probabilistic model intervals (DeepAR).** The DeepAR model natively outputs distributional parameters (mu, sigma), enabling direct quantile extraction without resampling.

**Why multiple interval methods?** Bootstrap intervals are the most principled but computationally expensive (100x training cost). Residual-based intervals are fast but assume homoscedastic, normally distributed errors. DeepAR intervals are native but available only when the DeepAR model wins selection. The framework provides all three and flags which method was used.

---

## 7. Results and Impact

### 7.1 Accuracy Improvements

The framework's accuracy was assessed through walk-forward backtesting on 4 quarters of held-out historical data, comparing against the prior manual forecasting process:

| Metric | Prior Baseline (Manual) | ML Framework | Improvement |  
|--------|------------------------|--------------|-------------|  
| FTU forecast accuracy (RMSE) | Baseline | -23% RMSE | 23% more accurate |  
| MAU forecast accuracy (RMSE) | Baseline | -18% RMSE | 18% more accurate |

These improvements were consistent across products, though the magnitude varied: newer products with shorter histories showed smaller improvements (10 to 15%), while established products with 3+ years of data showed larger gains (25 to 30%).

### 7.2 Operational Efficiency

| Metric | Before | After | Improvement |  
|--------|--------|-------|-------------|  
| Manual model refresh effort | \~3 days per product per quarter | \~1.5 days (data refresh \+ review) | 45% reduction |  
| Deployment time (forecast to warehouse) | \~4 hours (manual copy, format, upload) | \~1.5 hours (automated pipeline) | 55% reduction |  
| Products supported | 2 (manual scaling limit) | 9 models across 4 products x 2 markets | 4.5x coverage |  
| Reproducibility | Not reproducible | Fully deterministic given same data | Qualitative step change |

### 7.3 Planning Quality Impact

The accuracy and reliability improvements translated into measurable downstream effects on the planning process:

- **Reduced forecast revision frequency.** Prior to the framework, quarterly forecasts were typically revised 2 to 3 times during the quarter as actuals diverged from projections. With the ML framework, revisions dropped to 0 or 1 per quarter, indicating that initial forecasts were more aligned with actual outcomes.

- **Increased stakeholder confidence.** Planning teams adopted the ML forecasts as the primary planning input, replacing the prior process where ML outputs were treated as one of several "opinions" requiring manual reconciliation. This adoption was driven by the systematic backtesting evidence: stakeholders could see historical forecast vs. actual comparisons and verify accuracy claims independently.

- **Standardized metric definitions.** The process of building the framework required formalizing the definitions of FTU activations, reactivations, and churn, which had previously been inconsistently defined across teams. The framework's data validation pipeline enforces these definitions at ingestion time, creating a single source of truth.

### 7.4 Organizational Adoption

The framework scaled from an initial deployment on a single product (US co-branded credit card) to full coverage of all four products across both markets within 12 months. The configuration-driven architecture enabled this scaling without proportional engineering effort: each new product required only a YAML configuration file and an input data pipeline, not new model code.

---

## 8. Lessons Learned and Extensions

### 8.1 What Worked Well

**Cross-validated candidate benchmarking over fixed architectures.** The empirical finding that no single model dominates across all products validated the benchmarking approach. In production, the winning model varies by product and quarter: Ridge regression wins for PPC UK (strong linear trend, limited data); XGBoost wins for US co-branded products (complex seasonality, rich feature space); Gradient Boosting wins for Venmo (moderate data, non-linear growth patterns); N-BEATS and DeepAR win for the highest-volume products.

**Domain constraints as a safety net, not a crutch.** The constraint pipeline was initially controversial: why clip model outputs if the model is supposed to be accurate? In practice, constraints rarely activate (fewer than 5% of daily predictions hit any bound), but when they do, they prevent catastrophic errors from propagating through the recursive forecasting chain. The constraints are most valuable in the first few days of a long-horizon forecast, where the model is extrapolating furthest from training data.

**Configuration-driven product onboarding.** Adding a new product required approximately 2 days of work: 1 day for data pipeline setup (warehouse query and campaign calendar registration) and 1 day for configuration tuning (activation rate bounds, campaign category mappings). No library code changes were needed. This validated the architecture's extensibility.

### 8.2 What I Would Do Differently

**Earlier investment in uncertainty quantification.** The v1 system produces only point forecasts. Business stakeholders frequently asked "how confident is this forecast?", a question that point forecasts cannot answer. The v2 extension adds confidence intervals, but building this capability from the start would have accelerated stakeholder trust.

**Unified library with product-specific configuration vs. per-product library copies.** The v1 architecture copies the forecast library into each product directory, enabling independent updates but creating a maintenance burden when cross-cutting changes are needed. The v2 architecture moves to a unified library with a plugin-based model registry, which was the right end state but could have been the starting architecture.

### 8.3 Extensions and Future Directions

**Plugin-Based Model Registry (v2, implemented).** New models are added by implementing a standard interface and registering via a decorator. The core engine discovers and evaluates them automatically. This enabled adding LightGBM, CatBoost, SVR, Prophet, AutoARIMA, Holt-Winters, N-BEATS, TFT, and DeepAR without modifying the selection or orchestration logic.

**Ensemble Meta-Learning (v2, implemented).** Stacking, weighted averaging, and blending ensembles that combine top-performing models. The ensemble participates in the same benchmarking as individual models: it only wins if the combination genuinely outperforms the best individual model.

**Feature Importance and Interpretability (v2, implemented).** Permutation importance and SHAP (SHapley Additive exPlanations) value computation for tree-based models, and variable-selection-network attention weights for the TFT variant. An automated insight generator identifies the top-3 drivers of forecast direction (e.g., "forecast is primarily driven by positive trend momentum and active welcome-bonus campaigns").

**Anomaly Detection (v2, implemented).** Automated detection of three anomaly types in input data: sigma outliers (values more than 3 sigma from rolling mean), jump discontinuities (day-over-day changes more than 2 sigma), and trend breaks (structural changes in trend direction). Detected anomalies are flagged in data quality reports but do not currently trigger automatic remediation.

**Future: Causal Feature Integration.** Incorporating marketing spend, macroeconomic indicators (interest rates, unemployment), and competitive landscape signals as causal features. This requires careful treatment to avoid confounding, planned as a causal inference extension using instrumental variables or difference-in-differences estimation.

**Future: Hierarchical Reconciliation.** Ensuring that product-level forecasts sum to portfolio-level totals through top-down and bottom-up reconciliation methods (e.g., MinT optimal reconciliation). Currently, portfolio-level consistency is verified but not enforced.

---

## Appendix A: Technology Stack

| Component | Technology | Purpose |  
|-----------|-----------|---------|  
| Core ML | scikit-learn, XGBoost, LightGBM | Model training and prediction |  
| Statistical Models | Prophet (fbprophet), pmdarima, statsmodels | Time-series decomposition and ARIMA |  
| Deep Learning | PyTorch | N-BEATS, TFT variant, DeepAR sequence models |  
| Data Processing | pandas, NumPy | Data manipulation and numerical computation |  
| Feature Engineering | pandas, python-dateutil | Calendar features, lag computation, holiday detection, campaign joins |  
| Interpretability | SHAP | Feature importance and model explanation |  
| Data Storage | Cloud Data Warehouse (BigQuery on Google Cloud) | Production forecast storage and retrieval |  
| Object Storage | Cloud Object Storage (parquet shards) | Intermediate artifact staging |  
| Configuration | YAML | Product definitions, model configurations, scenarios |  
| Orchestration | Jupyter Notebooks, Python scripts | Pipeline execution and deployment |

## Appendix B: Glossary

| Term | Definition |  
|------|-----------|  
| **FTU** | First-Time User, a user who activates a credit product for the first time |  
| **MAU** | Monthly Active Users, trailing 30-day count of users with at least one qualifying transaction |  
| **T30D** | Trailing 30-Day, the rolling window used for MAU computation |  
| **TPV** | Total Payment Volume, aggregate transaction dollar volume, used as an external predictor for MAU |  
| **NNA** | Net New Actives, sum of new activations plus reactivations minus churn; a flow measure of user base growth |  
| **XO** | Checkout form factor (the checkout product surface used on merchant pages) |  
| **Non-XO** | Non-Checkout form factors (wallet, send money, and other in-app product surfaces) |  
| **TSCV** | Time Series Cross-Validation, cross-validation that respects the time ordering of observations |  
| **RMSE** | Root Mean Squared Error, primary model selection metric |  
| **Activation Rate** | Ratio of new activations to approved applications; constrained to [0.80, 0.95] |

---

*This document describes a methodology and system architecture. No internal business data, internal system identifiers, or confidential performance metrics are disclosed. Product category names (credit card, payment product) are used generically. All accuracy improvement figures are relative measures derived from sanitized backtesting comparisons.*  
