# ROLE 3: Backend & API Lead (P3)

**Read first:** `docs/HANDBOOK.md` → `docs/CONTRACTS.md` (all sections, esp. §2, §3) → this file.
**Branch prefix:** `feat/api-*` | **Language:** Python (FastAPI, Pydantic v2) | **Fallbacks:** `StubPredictor`, `insights.sample.json`

## 1. Mission
Expose the model and insights through a clean, validated REST API; own API quality (Postman, pytest, OpenAPI docs), staleness/deal logic, and optional CI/deploy. You are the glue, so **build to contracts and stay decoupled**.

## 2. You own
`backend/`, `postman/`, `.github/workflows/`, `requirements/api.txt`, `scripts/run_api.sh`, Dockerfile.
**Do not edit:** `ml/` (import `ml.inference` only), `analytics/`, `frontend/`, `docs/CONTRACTS.md` (propose via PR).

## 3. Inputs / outputs
| Direction | Item | Contract |
|-----------|------|----------|
| In ← P2 | `ml.inference.get_predictor()` | CONTRACTS §2 (**stub until M1**) |
| In ← P1 | `analytics/outputs/insights.json` | CONTRACTS §4.7 (**sample until M1**) |
| Out → P4 | REST API + OpenAPI at `/docs` | CONTRACTS §3 |
| Out → all | Postman collection, pytest suite | |

## 4. Folder layout (you create)
```
backend/
  app/
    main.py            # FastAPI app, CORS, router mount at /api/v1
    config.py          # env: PREDICTOR=stub|real, INSIGHTS_PATH, ARTIFACT_DIR, CORS_ORIGINS
    schemas.py         # Pydantic models mirroring CONTRACTS §3
    deps.py            # get_predictor(), get_insights() (cached)
    stub_predictor.py  # deterministic fake implementing CONTRACTS §2
    staleness.py       # staleness level logic
    deal.py            # deal score using insights route percentiles
    routers/ predict.py meta.py models.py insights.py
    errors.py          # standard error shape
  tests/
  Dockerfile
postman/  FareLens.postman_collection.json  local.postman_environment.json
```

## 5. Task plan

### Phase 0 / Week 1: Walking skeleton on the stub (no dependency) · F-02, F-24
- [ ] `uvicorn backend.app.main:app --reload` starts; `GET /api/v1/health` works; Swagger at `/docs`.
- [ ] Pydantic schemas for every endpoint in CONTRACTS §3; field validators (date format, time `HH:MM`, stops 0-4, non-empty strings).
- [ ] Standard error handler (CONTRACTS §3.0) for validation/unknown errors.
- [ ] `StubPredictor`: deterministic price from a simple formula (stops, duration, airline hash) with plausible interval; same interface as CONTRACTS §2. `PREDICTOR=stub` default.
- [ ] `/meta/options`, `/predict`, `/models`, `/insights/{section}` using stub + `insights.sample.json`.
- [ ] CORS for local frontend origins.
- [ ] Postman collection v1 with example requests + tests (status 200, schema keys present).
- [ ] **Tell P4 on day 3 that the stub API is live** (share base URL).

### Phase 1: Real integration (M1, end Week 2)
- [ ] `PREDICTOR=real` path loads `ml.inference.get_predictor()` once on startup (lifespan event); health shows `"predictor":"real"`.
- [ ] Graceful `503 PREDICTOR_UNAVAILABLE` if artifacts missing (never crash the process).
- [ ] Switch default to real when P2's baseline lands; keep the stub for tests/CI.

### Phase 2: Core logic (Weeks 2-4) · F-09, F-12, F-13, F-14, F-20
- [ ] **Staleness (F-09)** in `staleness.py` using the training window from `model_info()` (`trained_on.journey_date_min/max`):
  | Level | Rule (document & keep in sync with UI) |
  |-------|-----------------------------------------|
  | `fresh` | journey_date inside the window |
  | `aging` | outside the window by ≤ 60 days |
  | `stale` | outside by 61-365 days |
  | `out_of_range` | outside by > 365 days, **or** date missing/invalid for model, **or** an unseen airline/route |
  Message string explains in plain English (CONTRACTS §3.2 example).
- [ ] **Deal score (F-12)** in `deal.py`: compare estimate to route `p33`/`p67`/`median` from insights; `Unknown` if route not found.
- [ ] **Unseen-value warnings (F-20):** use `known_values()` to add `warnings[]` for unknown airline/route/city and for missing `duration_minutes` (fill from route median in insights, else global median).
- [ ] **What-if dates (F-13)** `/predict/whatif-dates`: loop dates ±N (cap N ≤ 30), compute `cheapest`. Reuse one predictor instance; keep response < 1 s for 15 points.
- [ ] **Compare (F-17 support)** `/compare`: loop airlines, cap 6.
- [ ] **Model switch (F-14):** `model_id` param, `404 MODEL_NOT_FOUND` if unknown.
- [ ] `/models`, `/models/{id}` return P2's leaderboard/metrics as-is.
- [ ] Simple in-memory LRU cache for identical predict calls (key = request hash) and a basic rate limit (e.g. 60 req/min/IP) using a small middleware.

### Phase 3: Wow (Weeks 4-5) · F-19, F-25
- [ ] **Batch CSV (F-19)** `/predict/batch` with per-row error list; `?format=csv` download; max 5,000 rows and 2 MB.
- [ ] Structured request logging (method, path, status, latency ms), `X-Request-ID` header.
- [ ] GitHub Actions: lint (`ruff`) + `pytest` using the stub predictor.
- [ ] Postman: environments (local/deployed), collection-level tests, exported JSON committed.

### Phase 4: Stretch · F-26, F-28
- [ ] Dockerfile + `docker compose up` for API; deploy to Render/Railway if time; document URL in README.
- [ ] Live-fare adapter interface (`backend/app/live_fares.py`) with a **mock** implementation. Only wire a real provider if its free-tier terms are verified, and never present live data as model output.

### Phase 5: Report (Weeks 5-6)
- [ ] Chapters: *System Architecture*, *API Design* (endpoint table, error model, validation), *Testing*, *Deployment*.

## 6. Acceptance criteria
- Every endpoint in CONTRACTS §3 implemented and returns the documented shape (a pytest compares keys to the schemas).
- `/docs` (Swagger) is complete and examples match CONTRACTS.
- Invalid input always yields the standard error shape; the server never returns a stack trace.
- Postman collection runs green (Newman or the app) in both stub and real mode.
- `pytest backend/tests` passes with `PREDICTOR=stub` and no network.

## 7. Pitfalls
- Don't implement ML logic in the API; call the predictor.
- Don't change JSON keys without a contract PR: P4 depends on them.
- Don't load the model per request; load once.
- Don't read files via relative paths that break when started from another directory: use `config.py` with `Path(__file__)`-based roots and env overrides.
- Dates: validate range sanity (year 2000-2100) but **allow outside the training window**: staleness handles it.

## 8. Antigravity prompts (copy-paste)
- "Read `.agent/rules/project-rules.md`, `docs/CONTRACTS.md` §2-3 and `docs/roles/ROLE_3_BACKEND_API.md`. Scaffold `backend/app` per section 4 with a `StubPredictor`. Only edit `backend/` and `postman/`."
- "Write pytest tests that validate every `/api/v1` response against the examples in CONTRACTS §3."

## 9. Backup / pairing
Pair with P4 on the first fetch integration and CORS issues; pair with P2 at M1 for the stub → real switch.
