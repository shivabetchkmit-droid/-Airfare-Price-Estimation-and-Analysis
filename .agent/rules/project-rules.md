# FareLens: Rules for the AI coding agent (Antigravity)

You are helping a team of 4 build **"Airfare Price Estimation and Fare Pattern Analysis"** (codename FareLens): a Python/FastAPI + scikit-learn backend and a vanilla JavaScript frontend. These rules apply to **every** task in this workspace.

## 1. Read order (do this before writing code, and whenever you have a doubt)
1. `docs/HANDBOOK.md`: goals, requirement matrix (R1-R7), feature IDs (F-01…F-30), architecture, ownership.
2. `docs/CONTRACTS.md`: data schema, Python inference interface, REST API, JSON shapes. **This is the source of truth for interfaces.**
3. `docs/roles/ROLE_<n>_*.md` for the role of the person who is prompting you (ask which role if not stated).
4. `docs/DECISIONS.md`: past decisions; don't contradict them silently.

## 2. Resolving doubts
- If the answer is in the docs above, follow it and mention which section you used.
- If the docs are silent or ambiguous: **stop and ask the user one concise question** with your recommended default. Do not guess.
- **Never invent**: dataset column names, API fields, metric values, file paths, library versions, or results. If data isn't available yet, say so and use the documented fallback (sample CSV, `StubPredictor`, `frontend/mocks/`, `insights.sample.json`).
- If a docs rule conflicts with the user's request, point out the conflict and ask which wins; if the contract must change, propose a PR to `docs/CONTRACTS.md` instead of editing code around it.

## 3. Ownership boundaries (very important)
| Role | May edit | Must not edit |
|------|----------|---------------|
| P1 Data & Analytics | `data/`, `analytics/`, `notebooks/01_*`, `notebooks/02_*`, `requirements/data.txt` | other roles' directories |
| P2 ML | `ml/`, `notebooks/03_*+`, `requirements/ml.txt` | other roles' directories |
| P3 Backend | `backend/`, `postman/`, `.github/`, `requirements/api.txt` | other roles' directories |
| P4 Frontend | `frontend/`, `docs/DEMO_SCRIPT.md` | other roles' directories |
Shared: `docs/CONTRACTS.md` (all 4 approve), `docs/DECISIONS.md` (append only), `reports/` (each writes own chapter), `README.md`.
If a task needs a change in another role's area, **don't make it**; write a short request note for that person (what, why, file) and continue with a stub/mock.

## 4. Non-negotiable engineering rules
**Contracts**
- Never rename or remove JSON keys, endpoints, CSV columns, or Python interface names defined in `docs/CONTRACTS.md`.
- Keys `snake_case`; dates `YYYY-MM-DD`; times `HH:MM` 24h; currency `INR`.

**Data & ML (leakage and honesty)**
- Deduplicate **before** splitting. Split with seed `42` (70 train / 10 calibration / 20 test). Fit encoders/scalers on train only. Never tune on the test set.
- The preprocessing + estimator live in one scikit-learn `Pipeline`; serving code must not re-implement feature logic.
- Model target is `log1p(price_inr)`; report metrics (MAE, RMSE, R², MAPE) in INR.
- Each `model.joblib` must be under 50 MB.
- Raw data is never committed. Only `data/sample/` (≤ 1000 rows) and small processed files.

**Language about results**
- Say "estimate", "is associated with", "in this dataset". Never "the price is", "causes", "guarantees".
- Every user-visible prediction needs the interval, the staleness message, and the disclaimer (see HANDBOOK §13). Don't remove them to "simplify" the UI.

**Backend (FastAPI)**
- Pydantic v2 models mirror CONTRACTS §3. All errors use the standard error shape (CONTRACTS §3.0). No stack traces to clients.
- Load the predictor once (lifespan), not per request. `PREDICTOR=stub|real` env switch.

**Frontend (JavaScript)**
- Vanilla ES modules + Chart.js; no build step. **All `fetch` calls only in `frontend/js/api.js`**; base URL only in `config.js`.
- Mock mode (`?mock=1`) must keep working. Always handle loading / empty / error states. Check `response.ok`.
- Format money with `en-IN` grouping and `₹`. Accessible markup (labels, focus, contrast); never rely on color alone.

## 5. Code quality
- Python 3.11, type hints, docstrings on public functions, `ruff`-clean, `pytest` for new logic. JS: `const`/`let`, no globals, small modules, JSDoc on exported functions.
- Small, reviewable changes; one purpose per change. Reference the feature ID (e.g., `F-08`) in commit/PR text.
- Commit style: `feat(ml): add conformal intervals (F-08)`. Never push to `main`; use `feat/<role>-<topic>` branches.
- No secrets, tokens, or `.env` in git. No large files.
- Don't add new dependencies without saying why and adding them to the role's `requirements/*.txt`.

## 6. How to work on a task
1. Restate the task in one line and name the role, feature ID(s) and contract sections involved.
2. List the files you'll touch (must be inside the role's allowed directories).
3. Implement the smallest working slice; run the relevant tests/commands; show the commands and output.
4. Update docs if behavior changed (role file checkbox, `DECISIONS.md`, README quickstart).
5. End with: what changed, how to run/verify, open questions, and any request notes for other roles.

## 7. Standard commands (keep README in sync if they change)
```
# setup
python -m venv .venv && source .venv/bin/activate    # Windows: .venv\Scripts\activate
pip install -r requirements.txt
# data (P1)
python -m analytics.clean && python -m analytics.insights
# ml (P2)
python -m ml.train --data data/processed/fares_clean.csv --models all
# api (P3)
PREDICTOR=stub uvicorn backend.app.main:app --reload --port 8000     # Windows PowerShell: $env:PREDICTOR="stub"
pytest backend/tests
# ui (P4)
cd frontend && python -m http.server 5500      # open http://127.0.0.1:5500/?mock=1
```

## 8. Things to refuse or flag
- Scraping airline/booking sites in violation of their terms; presenting scraped or live prices as model output.
- Fabricating metrics, screenshots, or "results" that were not produced by running the code.
- Committing raw datasets, credentials, or oversize model files.
- Changes outside the prompting user's role directories without explicit approval.

## 9. Project facts you may rely on
- Primary data: Indian flight fare dataset (2019 journeys, ~10.7k rows) with Airline, Date_of_Journey, Source, Destination, Route, Dep_Time, Arrival_Time, Duration, Total_Stops, Additional_Info, Price. Optional secondary dataset with `days_left`. **Column names must be verified against the real files before use.**
- Collection-window limits are a core theme (requirement R7): temporal drift, staleness levels, limitations page.
- Staleness levels: `fresh`, `aging` (≤ 60 days outside window), `stale` (61-365), `out_of_range` (> 365 or unseen values).
- Mentor: Arti Budhiraja. Final deliverables listed in HANDBOOK §17.
