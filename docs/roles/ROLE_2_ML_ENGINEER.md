# ROLE 2: ML Engineer (P2)

**Read first:** `docs/HANDBOOK.md` (§1, §2, §6) → `docs/CONTRACTS.md` (§1, §2, §4.6) → this file.
**Branch prefix:** `feat/ml-*` | **Language:** Python (scikit-learn) | **Fallback input:** `data/sample/sample_fares.csv`

## 1. Mission
Build, compare, evaluate and explain the regression models, and ship them behind one stable Python interface (`ml/inference.py`). You deliver the technical heart of the PS: model comparison, feature engineering, held-out evaluation, factor analysis, and the quantified limits from market change.

## 2. You own
`ml/`, `notebooks/03_*` and later, `requirements/ml.txt`, `reports/figures/ml_*`.
**Do not edit:** `analytics/`, `data/` (request changes from P1), `backend/`, `frontend/`, `docs/CONTRACTS.md` (propose via PR).

## 3. Inputs / outputs
| Direction | Item | Contract |
|-----------|------|----------|
| In ← P1 | `data/processed/fares_clean.csv` (until it exists, use `data/sample/sample_fares.csv`) | CONTRACTS §1 |
| In ← P1 | `data/external/holidays_in.csv`, optional `airline_tier.csv` | CONTRACTS §1 |
| Out → P3 | `ml/inference.py` (`get_predictor()`), `ml/artifacts/` | CONTRACTS §2 |
| Out → P3/P4 | `metrics.json`, `importance.json`, `drift.json` | CONTRACTS §4.6 |

## 4. Design rules (non-negotiable)
1. **Pipeline inside the artifact:** feature engineering + encoders + estimator are one `sklearn.pipeline.Pipeline` saved with joblib. Serving code must not re-implement feature logic.
2. **Target:** `log1p(price_inr)` via `TransformedTargetRegressor`; report metrics in INR.
3. **Split first, fit later:** `random_state=42`; 70/10/20 train/calibration/test. Dedupe already done by P1; verify again with an assert.
4. **Test set is touched once** per final model comparison. Tune only with CV on train.
5. Every model gets a folder in `ml/artifacts/<model_id>/` and an entry in `registry.json`.
6. `model.joblib` < 50 MB. Use `compress=3`, cap `n_estimators`/`max_depth`/`min_samples_leaf` for forests.
7. No leakage: no target-derived feature outside CV; target encoding only inside the pipeline.

## 5. Feature engineering spec (R3, R4)
**Date/time features** from `journey_date`, `dep_time`, `arrival_time`:
- `month`, `day_of_month`, `weekday`, `is_weekend`, `week_of_year`
- cyclical: `sin/cos` of weekday and month
- `days_to_nearest_holiday` and `is_holiday_window` (±3 days) using `holidays_in.csv`
- `dep_hour`, `dep_bucket` (early morning / morning / afternoon / evening / night), `arr_hour`, `is_red_eye`
- if `days_left` is available: `days_left`, log, and bucket
**Duration/stops:** `duration_minutes`, `duration_per_stop`, `is_nonstop`.
**Categorical:** rare-category grouping (`min_frequency`, e.g. < 20 rows → "Other"), `route_key`, airline tier (if file exists), `additional_info` normalization. Compare **one-hot vs ordinal vs target encoding** (target encoding only inside CV pipeline).
**Handling missing:** nullable columns imputed inside the pipeline (median / "Unknown").

## 6. Task plan

### Phase 0 / Week 1: Baseline on the sample · F-05 base
- [ ] `ml/features.py`: `build_preprocessor(config)` returning a `ColumnTransformer`.
- [ ] `ml/train.py` CLI: `python -m ml.train --data data/processed/fares_clean.csv --models all`. Defaults to the sample if the clean file is missing (prints a loud warning).
- [ ] `ml/evaluate.py`: MAE, RMSE, R², MAPE in INR; residual histogram bins; calibration of intervals.
- [ ] Baseline Linear Regression working end-to-end and saved as an artifact.
- [ ] **`ml/inference.py` + `get_predictor()` implementing CONTRACTS §2 with the baseline. Deadline: end of Week 1** (P3 switches from stub to real at M1).

### Phase 1: Model Arena (Weeks 2-3) · F-03, F-05, F-15
- [ ] Models: Linear, Ridge, Lasso or ElasticNet, Decision Tree, Random Forest, HistGradientBoosting, XGBoost or LightGBM (optional), (stretch) Stacking.
- [ ] `RandomizedSearchCV` (5-fold, scoring = neg MAE on log scale or INR), seeded; save best params in `metrics.json`.
- [ ] Leaderboard: all models, same split, columns per CONTRACTS §3.7. Write `ml/artifacts/leaderboard.json`.
- [ ] **Ablation (F-15):** (a) raw features only, (b) + date features, (c) + categorical engineering, (d) + holiday/tier features. Same model & split. Store in `metrics.json.ablation`.
- [ ] Encoder comparison table (one-hot vs ordinal vs target).
- [ ] Pick `default_model_id` by CV MAE (not by test!), document in `docs/DECISIONS.md`.

### Phase 2: Evaluation & explanation (Weeks 3-4) · F-06, F-11, F-16
- [ ] Residual analysis: residual vs predicted, Q-Q or histogram, error by airline / route / price band / month → `metrics.json.segments`. Report weak spots honestly.
- [ ] Global importance: permutation importance (on test, n_repeats ≥ 5) → `importance.json` with human-readable labels ("Number of stops", not `total_stops`).
- [ ] PDP/ICE for top 3-4 features (figures + JSON for charts).
- [ ] Local explanation (F-11): SHAP (TreeExplainer for tree models) or a permutation/ablation fallback. Return top 5 `{feature, label, impact_inr}`. Make `explain=False` the fast default; keep explanation < 1.5 s.
- [ ] Wording: "associated with", never "causes".

### Phase 3: Uncertainty & drift (Weeks 3-5) · F-08, F-10
- [ ] **Conformal intervals (F-08):** fit on train, compute absolute residuals (log space) on calibration set, use the ⌈(n+1)(1−α)⌉/n quantile (α = 0.2) → interval = prediction × exp(±q). Report **empirical coverage and mean width on test** in `metrics.json.interval`.
- [ ] **Temporal drift (F-10, R7):** with `journey_date` available: train on earliest ~70% of dates, test on the latest 30% (e.g. March-mid May vs late May-June). Compare against random split. Also per-month MAE (leave-one-month-out if feasible). Output `ml/artifacts/drift.json` (CONTRACTS §4.6 drift shape).
- [ ] Check interval coverage **under drift** (time split): expect it to drop; report it. That's the honest result we want.
- [ ] If the secondary dataset exists: train on 2019, evaluate on 2022 (very different markets/schemas, use common features only) as a "cross-collection drift" stress test. Clearly label as exploratory.
- [ ] `known_values()` in `inference.py`: lists for UI dropdowns and for backend's unseen-value warnings.

### Phase 4: Wow / stretch (Week 5) · F-27
- [ ] Stacking ensemble; monotonic constraints on `days_left` if secondary data used (stretch); calibration of the intervals by segment.
- [ ] Model card per artifact (`model_card.md`): data, metrics, limits, intended use.

### Phase 5: Report (Week 5-6)
- [ ] Chapters: *Methodology* (features, splits, models), *Results* (leaderboard, ablation, residuals), *Interpretation* (importance), *Drift & Limitations* (temporal experiment).

## 7. Acceptance criteria
- `python -m ml.train --models all` reproduces the same metrics (seed 42) from a clean clone.
- `get_predictor().predict({...})` works for the contract example and never crashes on unseen categories.
- `metrics.json` validates against CONTRACTS §4.6; test metrics are on held-out rows only.
- A leakage test exists (`ml/tests/test_no_leakage.py`): encoders fit only on train; dedupe assertion; test indices disjoint.
- Empirical coverage reported (target 0.80 ± 0.05 on random split).
- Every artifact < 50 MB.

## 8. Pitfalls
- R² of 0.9+ on a random split with duplicates = leakage, not success.
- Don't choose the "best" model on test data.
- Don't commit a 600 MB forest. Check `size_mb` in training logs.
- Date features on a dataset with only 4 months of journeys: month has few categories; keep it but note the limited seasonality in the report.
- Log-target: remember to back-transform before MAPE/MAE.

## 9. Antigravity prompts (copy-paste)
- "Read `.agent/rules/project-rules.md`, `docs/CONTRACTS.md` §2 and §4.6, and `docs/roles/ROLE_2_ML_ENGINEER.md`. Implement `ml/features.py` per section 5. Do not touch other directories."
- "Add split-conformal prediction intervals to `ml/train.py` per ROLE_2 §6 Phase 3 and write a test that checks empirical coverage on the test split."

## 10. Backup / pairing
Pair with P1 on features and drift interpretation; pair with P3 when swapping stub → real predictor at M1.
