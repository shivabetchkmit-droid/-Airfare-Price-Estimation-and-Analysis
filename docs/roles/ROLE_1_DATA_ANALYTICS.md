# ROLE 1: Data & Analytics Lead (P1)

**Read first:** `docs/HANDBOOK.md` → `docs/CONTRACTS.md` (§1, §4.7) → this file.
**Branch prefix:** `feat/data-*` | **Language:** Python (pandas) | **You need nothing from anyone to start.**

## 1. Mission
Own the truth about the data. Deliver clean, documented, leakage-safe data plus the fare-pattern analysis (the "analyze factors associated with higher fares" and "collection dates limit predictions" parts of the PS). Every other role consumes your outputs.

## 2. You own (only edit these)
`data/`, `analytics/`, `notebooks/01_*`, `notebooks/02_*`, `reports/data_quality_report.md`, `reports/figures/eda_*`, `requirements/data.txt`, `scripts/make_sample.py`.

**Do not edit:** `ml/`, `backend/`, `frontend/`, `docs/CONTRACTS.md` (propose via PR).

## 3. Inputs / outputs
| Direction | Item | Contract |
|-----------|------|----------|
| In | Kaggle datasets (primary + optional secondary) | HANDBOOK §5.1 |
| Out → P2 | `data/processed/fares_clean.csv`, `data/sample/sample_fares.csv`, `data/external/holidays_in.csv`, `data/DATASET_CARD.md` | CONTRACTS §1 |
| Out → P3/P4 | `analytics/outputs/insights.json` (+ `insights.sample.json`) | CONTRACTS §4.7 |
| Out → report | EDA figures, data quality report, limitations text for data | |

## 4. Task plan (tick as you go)

### Phase 0 / Day 1 (blocks nobody if late, but do it first)
- [ ] Download primary dataset; put in `data/raw/` (git-ignored). Record source URL, license, download date.
- [ ] **Verify actual column names** against HANDBOOK §5.1; if different, open a PR to update CONTRACTS §1.
- [ ] Commit `data/sample/sample_fares.csv` (500-1000 rows, stratified by airline, schema exactly per CONTRACTS §1; clean enough to train on). **Deadline: within 24h of kickoff.** P2 depends on it as the fallback.
- [ ] Commit `analytics/outputs/insights.sample.json` with hand-made realistic values in the §4.7 shape (P3/P4 mock from it).

### Phase 1: Cleaning pipeline (Week 1) · F-04 base
- [ ] `analytics/clean.py` (one command: `python -m analytics.clean`): raw → `data/processed/fares_clean.csv`.
- [ ] Parse `Date_of_Journey` → ISO; `Duration` ("2h 50m") → minutes; `Total_Stops` text → int; times → `HH:MM`.
- [ ] Normalize city names (e.g. spelling variants, "New Delhi" → "Delhi"), keep `data/external/city_map.csv`.
- [ ] Handle missing rows (log count), **drop exact duplicates before anything else** (log count).
- [ ] Flag/handle outliers (extreme prices, rare airlines with ≤ 3 rows). **Don't silently delete;** add to the data quality report with the reason and count. Keep decision in `docs/DECISIONS.md`.
- [ ] Assert schema with a small validation function (`analytics/validate.py`): types, required columns, price > 0, unique `record_id`.
- [ ] `data/DATASET_CARD.md`: source, window (`journey_date_min/max`), rows before/after, cleaning log, known biases (e.g. only some airlines/routes, one season).
- [ ] `data/external/holidays_in.csv` for the data window (public/major festival dates; cite source in the dataset card).
- [ ] Optional secondary dataset: map into the same schema with `source_dataset="secondary_2022"` in a **separate file** `fares_clean_secondary.csv` (don't merge blindly; prices and eras differ).

### Phase 2: Fare pattern analysis (Weeks 2-3) · F-04, F-12, F-21
- [ ] `analytics/insights.py` (`python -m analytics.insights`) → `analytics/outputs/insights.json`, matching §4.7 exactly.
- [ ] Analyses (each with a chart in `reports/figures/eda_*.png` and 1-2 plain sentences in `findings`):
  - price distribution (raw + log), outliers
  - price by airline (median, IQR), by stops, by route (top 15), by month, by weekday, by departure time bucket, by duration band
  - route × month heatmap
  - correlation of duration/stops/price (Spearman)
  - price by `additional_info` (booking-related)
  - **booking window** (`days_left` vs price) from secondary dataset → `booking_window` section (label "cross-sectional, not the same flight tracked over time")
- [ ] Per-route percentiles (`p33`, `p67`, median) for the **Deal Score** (F-12). P3 reads these.
- [ ] **Collection-date analysis (R7):** counts of records per journey month, coverage gaps, what months/seasons are missing, which holidays fall in window. Output `dataset` section and a chart.
- [ ] Write findings in **associational language** ("is associated with", never "causes").

### Phase 3: Wow items (Weeks 4-5) · optional
- [ ] Price volatility per route (IQR/median) → "predictability" label per route (F-16 support).
- [ ] Airline tier clustering (budget / full-service) from price + service attributes, with a short justification; share mapping file `data/external/airline_tier.csv` with P2 via PR.
- [ ] Notebook `02_eda_story.ipynb`: polished narrative (clear outputs, figures saved).
- [ ] Peak-day insights: festival proximity vs price (use holidays file).

### Phase 4: Report (Week 5-6)
- [ ] Chapters: *Dataset & Collection Window*, *Data Cleaning*, *Exploratory Findings*, *Limitations of Data*. 1-2 pages each.

## 5. Acceptance criteria
- `python -m analytics.clean && python -m analytics.insights` runs from a clean clone in < 2 minutes (after raw data placed).
- Output validates against CONTRACTS §1 and §4.7 (`pytest analytics/tests`).
- Dataset card states the collection window in a sentence that is quoted by the app's Limitations page.
- No raw data in git. Sample ≤ 1000 rows.

## 6. Pitfalls
- Duplicates in both train and test inflate scores: dedupe first.
- Don't "fix" prices (no imputing target). If price is missing, drop the row and log it.
- Don't merge the 2019 and 2022 datasets for training unless P2 and you agree and document it: prices and market differ (that's the drift story).
- Keep currency INR; don't convert.

## 7. Antigravity prompts (copy-paste)
- "Read `.agent/rules/project-rules.md`, `docs/CONTRACTS.md` §1 and `docs/roles/ROLE_1_DATA_ANALYTICS.md`. Implement `analytics/clean.py` for the dataset in `data/raw/`. First print the actual columns and ask me to confirm the mapping before writing code."
- "Generate `analytics/outputs/insights.json` per CONTRACTS §4.7 and a pytest that validates its keys and types."

## 8. Backup / pairing
You and P2 pair on feature ideas (holidays, tiers). If P3 is overloaded you may help with the Postman collection (read-only on `backend/`; commit only under `postman/` via PR).
