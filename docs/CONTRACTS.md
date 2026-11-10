# CONTRACTS: the shared interfaces (change only by PR, all 4 approve)

These contracts are what let four people work in parallel. **Build to the contract, not to a teammate's code.** If a contract must change: open a PR touching this file, tag all 3 teammates, merge only after 4 approvals, then bump `CONTRACT_VERSION`.

`CONTRACT_VERSION: 1.0.0`

---

## 1. Seam S1: Clean dataset (P1 → P2)

File: `data/processed/fares_clean.csv` (and 500-1000 row `data/sample/sample_fares.csv`, same columns). UTF-8, header row, ISO dates.

| Column | Type | Required | Rules / example |
|--------|------|----------|-----------------|
| `record_id` | int | yes | unique, 0..n |
| `airline` | str | yes | cleaned name e.g. `IndiGo` |
| `source` | str | yes | normalized city e.g. `Bengaluru` (fix spelling variants; keep a mapping file `data/external/city_map.csv`) |
| `destination` | str | yes | normalized city; `New Delhi` → `Delhi` (document) |
| `route_key` | str | yes | `source->destination` e.g. `Delhi->Cochin` (P1 may also keep raw route text in `route_raw`) |
| `journey_date` | date `YYYY-MM-DD` | nullable | null if dataset has no date |
| `dep_time` | `HH:MM` 24h | yes | |
| `arrival_time` | `HH:MM` 24h | nullable | |
| `duration_minutes` | int | yes | parsed from "2h 50m" → 170 |
| `total_stops` | int | yes | 0, 1, 2, 3, 4 |
| `additional_info` | str | nullable | booking-related text (meal/baggage etc.), normalized case |
| `travel_class` | str | nullable | `Economy`/`Business`/... (secondary dataset) |
| `days_left` | int | nullable | booking window (secondary dataset) |
| `source_dataset` | str | yes | `primary_2019` / `secondary_2022` |
| `price_inr` | float | yes | > 0, INR |

Also produced by P1 in `data/DATASET_CARD.md`: collection window `journey_date_min`, `journey_date_max`, download date, license/source URL, row counts before/after cleaning.

Required companion file: `data/external/holidays_in.csv` with columns `date,name,type` (national/major festival dates in the data window) for P2's holiday-proximity features.

---

## 2. Seam S2: Python inference interface (P2 → P3)

File: `ml/inference.py`. P3 imports **only** these names.

```python
from typing import Protocol, TypedDict, Optional

class PredictRequest(TypedDict, total=False):
    airline: str
    source: str
    destination: str
    journey_date: str            # YYYY-MM-DD
    dep_time: str                # HH:MM
    arrival_time: Optional[str]
    duration_minutes: Optional[int]
    total_stops: int
    additional_info: Optional[str]
    travel_class: Optional[str]
    days_left: Optional[int]

class Predictor(Protocol):
    def predict(self, req: PredictRequest, model_id: Optional[str] = None,
                explain: bool = False) -> dict: ...
        # returns: {"price": float, "low": float, "high": float, "confidence": 0.8,
        #           "model_id": str, "top_factors": [{"feature": str, "label": str, "impact_inr": float}]}
    def list_models(self) -> list[dict]: ...      # leaderboard rows (see §4.6)
    def model_info(self, model_id: str) -> dict: ...  # metrics.json content + trained_on window
    def known_values(self) -> dict: ...           # {"airlines": [...], "sources": [...], "destinations": [...],
                                                   #  "stops": [...], "additional_info": [...], "date_min": "...", "date_max": "..."}

def get_predictor() -> Predictor: ...             # loads registry + default model once (cached)
```

- Must never raise on unseen category values; return a prediction and let the backend add a warning. P2 exposes `known_values()` so the backend can detect unseen values.
- `price`, `low`, `high` are in INR, already back-transformed from log space.
- Backend fallback: `backend/app/stub_predictor.py` implements the same interface with deterministic fake numbers. Switch via env `PREDICTOR=stub|real`.

Artifact layout (P2 owns):
```
ml/artifacts/
  registry.json                  # list of models + default_model_id
  <model_id>/model.joblib        # sklearn Pipeline (preprocess + estimator) + conformal quantile
  <model_id>/metrics.json        # contract §4.6
  <model_id>/importance.json     # global importance
```

---

## 3. Seam S3: REST API (P3 → P4)

Base URL (local): `http://127.0.0.1:8000/api/v1`. JSON only. CORS allows `http://localhost:*` and `http://127.0.0.1:*`.

### 3.0 Error shape (all errors)
```json
{ "error": { "code": "VALIDATION_ERROR", "message": "journey_date must be YYYY-MM-DD", "field": "journey_date" } }
```
Codes: `VALIDATION_ERROR` (422), `MODEL_NOT_FOUND` (404), `INTERNAL_ERROR` (500), `PREDICTOR_UNAVAILABLE` (503).

### 3.1 `GET /health`
```json
{ "status": "ok", "predictor": "real", "contract_version": "1.0.0" }
```

### 3.2 `POST /predict`
Request:
```json
{
  "airline": "IndiGo", "source": "Delhi", "destination": "Cochin",
  "journey_date": "2019-05-21", "dep_time": "09:15", "arrival_time": null,
  "duration_minutes": 195, "total_stops": 0, "additional_info": "No info",
  "model_id": "random_forest_v1", "explain": true
}
```
Optional fields: `arrival_time`, `duration_minutes` (backend fills with route median from insights if missing and adds a warning), `additional_info`, `model_id` (default model if omitted), `explain` (default false).

Response `200`:
```json
{
  "currency": "INR",
  "predicted_price": 5230.0,
  "interval": { "low": 4310.0, "high": 6480.0, "confidence": 0.8 },
  "deal": { "label": "Typical", "route_percentile": 48, "route_median": 5100.0 },
  "staleness": {
    "level": "stale",
    "message": "Training data covers journeys 2019-03-01 to 2019-06-27. Your date is outside this window; treat as a rough pattern-based estimate.",
    "training_window": { "start": "2019-03-01", "end": "2019-06-27" },
    "days_outside_window": 2000
  },
  "model": { "id": "random_forest_v1", "name": "Random Forest", "version": "1.0.0" },
  "top_factors": [
    { "feature": "total_stops", "label": "Number of stops", "impact_inr": 1250.0 }
  ],
  "warnings": ["duration_minutes estimated from route median"]
}
```
`staleness.level` ∈ `fresh` | `aging` | `stale` | `out_of_range` (thresholds in `backend/app/staleness.py`; documented in ROLE_3).
`deal.label` ∈ `Cheap` | `Typical` | `Expensive` (≤ 33rd / 34-66 / ≥ 67th route percentile) or `Unknown` if route unseen.

### 3.3 `POST /predict/whatif-dates`
Request: same as `/predict` minus `journey_date`, plus `"center_date": "2019-05-21", "days_before": 7, "days_after": 7`.
Response:
```json
{ "currency": "INR", "points": [ { "date": "2019-05-14", "predicted_price": 5100.0, "low": 4200.0, "high": 6300.0 } ], "cheapest": { "date": "2019-05-15", "predicted_price": 4980.0 }, "staleness": { "...": "same as 3.2" } }
```

### 3.4 `POST /predict/batch`
`multipart/form-data` with `file` (CSV with the §1 request columns). Max 5,000 rows. Response: `{ "rows": [ {"row": 0, "predicted_price": 5230.0, "low": 4310.0, "high": 6480.0, "warnings": []} ], "errors": [ {"row": 7, "message": "..."} ] }` and `?format=csv` returns a CSV download.

### 3.5 `POST /compare`
Request: `{ "source": "...", "destination": "...", "journey_date": "...", "dep_time": "...", "total_stops": 1, "airlines": ["IndiGo","Air India"] }` → `{ "items": [ { "airline": "IndiGo", "predicted_price": 5100.0, "low": 4200.0, "high": 6200.0 } ] }`.

### 3.6 `GET /meta/options`
```json
{ "airlines": ["..."], "sources": ["..."], "destinations": ["..."], "stops": [0,1,2,3],
  "additional_info": ["No info", "In-flight meal not included"],
  "date_range": { "min": "2019-03-01", "max": "2019-06-27" },
  "default_model_id": "random_forest_v1", "contract_version": "1.0.0" }
```

### 3.7 `GET /models` and `GET /models/{model_id}`
`/models` → array of leaderboard rows:
```json
[ { "id": "random_forest_v1", "name": "Random Forest", "mae": 1050.2, "rmse": 1800.4, "r2": 0.84, "mape": 11.9,
    "cv_mae_mean": 1110.0, "cv_mae_std": 40.2, "train_seconds": 12.3, "size_mb": 18.4, "is_default": true } ]
```
`/models/{id}` → full `metrics.json` (§4.6) incl. residual summary, coverage, ablation, segment errors.

### 3.8 `GET /insights/{section}`
Sections served from `analytics/outputs/insights.json` (§4.7): `summary`, `airlines`, `routes`, `stops`, `calendar`, `time_of_day`, `importance`, `drift`, `dataset`. Response is that section's JSON as-is, wrapped: `{ "section": "airlines", "generated_at": "...", "data": { ... } }`.

---

## 4. Other shared schemas

### 4.6 `metrics.json` (P2 → P3 → P4)
```json
{
  "model_id": "random_forest_v1", "name": "Random Forest", "version": "1.0.0", "seed": 42,
  "trained_on": { "rows": 7400, "journey_date_min": "2019-03-01", "journey_date_max": "2019-06-27" },
  "test": { "mae": 1050.2, "rmse": 1800.4, "r2": 0.84, "mape": 11.9 },
  "cv": { "folds": 5, "mae_mean": 1110.0, "mae_std": 40.2 },
  "interval": { "confidence": 0.8, "empirical_coverage": 0.79, "mean_width": 2100.0 },
  "ablation": [ { "setting": "raw features only", "mae": 1500.0 }, { "setting": "+ date features", "mae": 1250.0 }, { "setting": "+ categorical engineering", "mae": 1050.0 } ],
  "segments": { "airline": [ { "name": "IndiGo", "n": 900, "mae": 800.0 } ], "price_band": [ { "name": "<5k", "n": 1200, "mae": 600.0 } ] },
  "residuals": { "bins": [-4000, -3000], "counts": [10, 40] },
  "train_seconds": 12.3, "size_mb": 18.4
}
```

### 4.7 `insights.json` (P1 → P3 → P4)
```json
{
  "generated_at": "2026-11-10T10:00:00Z",
  "dataset": { "rows": 10000, "journey_date_min": "...", "journey_date_max": "...", "source": "...", "currency": "INR" },
  "summary": { "median_price": 8500.0, "mean_price": 9100.0, "n_airlines": 12, "n_routes": 30 },
  "airlines": [ { "airline": "IndiGo", "n": 2000, "median": 5200.0, "p25": 4500.0, "p75": 6400.0 } ],
  "routes": [ { "route_key": "Delhi->Cochin", "n": 500, "median": 9000.0, "p33": 7500.0, "p67": 10500.0, "median_duration_minutes": 480 } ],
  "stops": [ { "total_stops": 0, "n": 3500, "median": 4800.0 } ],
  "calendar": { "by_month": [ { "month": 3, "median": 8000.0, "n": 1000 } ], "by_weekday": [ { "weekday": 0, "median": 8300.0, "n": 1200 } ] },
  "time_of_day": [ { "bucket": "early_morning", "median": 8800.0, "n": 900 } ],
  "heatmap_route_month": { "routes": ["..."], "months": [3,4,5,6], "values": [[8000, 8200, 8600, 9000]] },
  "booking_window": [ { "days_left_bucket": "1-3", "median": 15000.0, "n": 300 } ],
  "findings": [ "Non-stop flights are associated with lower median fares on this dataset." ]
}
```
`booking_window` may be an empty array if the secondary dataset is unavailable. `findings` are P1's plain-English sentences (associational language only).

`drift` section (from P2's experiment, merged into insights by P1's script or served by P3 from `ml/artifacts/drift.json`):
```json
{ "random_split": { "mae": 1050.0, "r2": 0.84 }, "time_split": { "train_until": "2019-05-15", "test_from": "2019-05-16", "mae": 1480.0, "r2": 0.71 }, "by_month": [ { "month": 3, "mae": 980.0 } ], "note": "..." }
```

---

## 5. Mock fixtures (P4)
P4 stores one JSON file per endpoint in `frontend/mocks/` named after the path, e.g. `predict.json`, `models.json`, `insights_airlines.json`, copying the examples above. Mock mode toggle: `?mock=1` query param or `localStorage.mockMode = "1"`.

## 6. Naming and conventions
- JSON keys `snake_case`. Dates ISO `YYYY-MM-DD`, times `HH:MM` 24h. Currency always `INR`.
- Weekday numbering: Monday=0 … Sunday=6. Month 1-12.
- Model IDs: `<slug>_v<major>` e.g. `ridge_v1`, `hist_gb_v1`.
