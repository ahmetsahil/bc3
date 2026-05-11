# `eug.ipynb` - Target-Only Portfolio Replication Notebook

## Overview
`eug.ipynb` is a target-only portfolio replication notebook built to optimize **out-of-sample tracking error** under realistic implementation constraints.

It compares multiple model families, enforces institutional long/short constraints, applies transaction costs, and validates correctness with explicit checks (no-lookahead, accounting consistency, reporting completeness).

---

## Notebook Structure

The notebook is organized into concise sections:

1. **Imports and Configuration**
2. **Load Data and Build Target**
3. **Helper Functions and Validation Checks**
4. **Elastic Net 5-Fold CV Optimization**
5. **Rolling Elastic Net Backtest**
6. **Kalman Replication Initialized from Elastic Net**
7. **Additional TE-Optimized Models** (Models 1, 2, 5)
8. **Final Comparison and Diagnostics**
9. **Conclusion**

---

## Data Pipeline

### Input
- Excel file at:
  - `D:\QuantProjects\Fintech\BusinessCase3\Dataset3_PortfolioReplicaStrategy.xlsx`
- Sheet:
  - `Copia_statica`

### Core Steps
1. Parse and index dates.
2. Validate schema and data quality:
   - duplicate date checks,
   - missing expected columns,
   - all-NaN column checks.
3. Compute weekly returns.
4. Build the target return series from a weighted mix of index returns.
5. Align target and futures returns on common dates.

---

## Objective and Constraints

## Primary objective
Minimize **validation tracking error** (TE) and then confirm on full out-of-sample TE.

## Institutional L/S constraints
Applied in constrained optimizers:
- Gross exposure: `sum(|w|) <= 2.0`
- Net exposure: `sum(w) in [-0.1, 0.1]`
- Position bounds: `w_i in [-0.5, 0.5]`

## Cost model
- Transaction cost proportional to turnover:
  - `cost_t = TRANSACTION_COST * turnover_t`

## Risk scaling
- Post-solve scaling enforces global gross and VaR controls before PnL realization.

---

## Model Families in the Notebook

## Baseline family A: Rolling Elastic Net
- Hyperparameters (`alpha`, `l1_ratio`) tuned with 5-fold time-series CV.
- Search methods:
  - GridSearchCV,
  - RandomizedSearchCV,
  - BayesSearchCV (when available).
- Converted into a rolling strategy with rebalance schedule and cost-aware returns.

## Baseline family B: Kalman (ENet initialization)
- Initial state vector from tuned Elastic Net weights.
- Recursive state updates with:
  - Joseph covariance update,
  - PSD projection for numerical stability.
- Produces dynamic exposures and cost-aware replication returns.

## Model 1: Constrained Tracking-Error Optimizer (QP-like)
- Uses SLSQP to solve constrained tracking problem each rebalance:
  - `min ||Xw - y||^2 + Î»2||w||^2`
- Includes turnover cap/fallback behavior for failed solves.

## Model 2: Elastic Net + L2 Turnover Penalty
- Uses Elastic Net signal as anchor.
- Solves constrained objective with:
  - fit term,
  - L2 turnover penalty `||w - w_prev||^2`,
  - optional anchor penalty to ENet signal.

## Model 5: PCA Factor Tracking Optimizer
- Builds statistical factors via PCA on futures returns.
- Projects target to factor space and maps factor-optimal exposure back to instrument weights.
- Solves constrained portfolio each rebalance under the same Institutional L/S rules.

---

## Optimization and Selection Logic

The notebook uses a **TE-first selector**:

1. Run model sweeps per family over configured hyperparameter grids.
2. Compute validation TE on the last `VALIDATION_FRACTION` segment.
3. Select best spec per family by:
   1. minimum validation TE,
   2. tie-break by full OOS TE,
   3. tie-break by lower turnover.
4. Compare family winners + baselines in one final table.
5. Pick final winner using the same TE-first ordering.

---

## Validation and Safety Checks

The notebook enforces:

## 1) No-lookahead checks
- Training end timestamp must be strictly before prediction timestamp.

## 2) Accounting checks
- `net == gross - cost` at each period.
- Turnover/cost consistency with rebalance events.

## 3) Constraint compliance checks
- Logs max violation of:
  - gross,
  - net lower/upper bound,
  - per-asset bounds.
- Fails if violations exceed tolerance.

## 4) Reporting completeness checks
- Final comparison table must include required metrics without NaNs:
  - TE, correlation, IR,
  - turnover, gross exposure,
  - VaR/breaches,
  - replica and target drawdowns.

---

## Key Outputs

Expected outputs include:
- Hyperparameter search summaries,
- Candidate model leaderboards by validation TE,
- Constraint-violation diagnostics,
- Final comparison table across retained models,
- Winner selection summary,
- Residual diagnostics (Ljung-Box),
- Gross vs net performance table.

---

## How to Run

1. Open `eug.ipynb`.
2. **Restart kernel**.
3. Run all cells top-to-bottom.

If Bayesian search dependency is missing, the notebook attempts to install `scikit-optimize`.

---

## Notes for Extension

Possible next upgrades:
- Purged/embargo CV for serial dependence control.
- Scenario-dependent transaction cost model (slippage + spread).
- Robust covariance/risk model integration in constrained solves.
- Regime-conditional hyperparameter scheduling.

---

## Sources

- Elastic Net (scikit-learn):  
  https://scikit-learn.org/stable/modules/generated/sklearn.linear_model.ElasticNet.html
- TimeSeriesSplit (scikit-learn):  
  https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.TimeSeriesSplit.html
- BayesSearchCV (scikit-optimize):  
  https://scikit-optimize.github.io/stable/modules/generated/skopt.BayesSearchCV.html
- SLSQP optimizer (SciPy):  
  https://docs.scipy.org/doc/scipy/reference/optimize.minimize-slsqp.html
- PCA (scikit-learn):  
  https://scikit-learn.org/stable/modules/generated/sklearn.decomposition.PCA.html
- Ljung-Box test (statsmodels):  
  https://www.statsmodels.org/dev/generated/statsmodels.stats.diagnostic.acorr_ljungbox.html
- Kalman filter tutorial (Welch & Bishop):  
  https://homepages.inf.ed.ac.uk/rbf/CVonline/LOCAL_COPIES/WELCH/kalman.1.html

