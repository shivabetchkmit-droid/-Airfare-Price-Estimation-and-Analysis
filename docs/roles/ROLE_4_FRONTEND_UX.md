# ROLE 4: Frontend & UX Lead (P4)

**Read first:** `docs/HANDBOOK.md` → `docs/CONTRACTS.md` (§3, §4, §5) → this file.
**Branch prefix:** `feat/ui-*` | **Language:** JavaScript (ES modules), HTML, CSS, Chart.js | **Fallback:** `frontend/mocks/*.json`

## 1. Mission
Build the web application people see and the demo that wins the room: a clear prediction experience, fare-pattern dashboards, and an honest presentation of uncertainty and limits. Also own the demo script, slides and the UI/UX report chapters.

## 2. You own
`frontend/`, `reports/figures/ui_*`, `docs/DEMO_SCRIPT.md`, slides/demo video, `scripts/run_ui.sh`.
**Do not edit:** `ml/`, `backend/`, `analytics/`, `docs/CONTRACTS.md` (propose via PR).

## 3. Inputs / outputs
| Direction | Item | Contract |
|-----------|------|----------|
| In ← P3 | REST API at `http://127.0.0.1:8000/api/v1` | CONTRACTS §3 (**mocks until stub API is live**) |
| Out | Static web app (no build step) | |
| Out → report | Screenshots, UX decisions, demo script | |

## 4. Architecture
```
frontend/
  index.html                # Predict page (home)
  pages/ arena.html insights.html explain.html compare.html limitations.html about.html
  css/  base.css theme.css components.css
  js/
    api.js        # ALL fetch calls live here (single place), mock mode switch
    config.js     # API_BASE, MOCK flag
    state.js      # tiny pub/sub store
    ui.js         # DOM helpers, toasts, loading/error/empty states
    charts.js     # Chart.js wrappers
    pages/ predict.js arena.js insights.js explain.js compare.js limitations.js
    utils/ format.js (INR formatting en-IN), validate.js, dates.js
  mocks/ predict.json models.json insights_airlines.json ... (copy CONTRACTS examples)
  assets/
```
Rules: no inline scripts; no framework required; Chart.js via CDN with a pinned version; **every Fetch goes through `api.js`** (so mock mode and error handling live in one place).

## 5. Task plan

### Phase 0 / Week 1: UI shell on mocks (no dependency) · F-01, F-23 base
- [ ] Shared layout: header nav, footer, theme variables, responsive grid, system-font stack.
- [ ] `api.js` with `getOptions()`, `predict()`, `getModels()`, `getInsights(section)`, etc. Real `fetch` + `?mock=1` mode reading `mocks/*.json`.
- [ ] Predict form (F-01): airline, source, destination, date picker, departure time, duration (optional, with "auto-fill typical" hint), stops, additional info, model dropdown. Client-side validation messages matching CONTRACTS rules; source ≠ destination.
- [ ] Result card from `mocks/predict.json`: big INR price (`en-IN` formatting), **interval bar**, deal badge, **staleness banner** with 4 colored levels, warnings list, "Why this price?" bars (F-11).
- [ ] Loading skeleton, error toast, empty state for every page.
- [ ] Send P3 a screenshot on day 3 and agree any contract nits in a PR (not chat).

### Phase 1: Core pages (Weeks 2-3) · F-03, F-04, F-06, F-07
- [ ] **Insights** (F-04): airline box/bar chart (median + IQR), stops, month, weekday, time-of-day, route × month heatmap (canvas or CSS grid), `findings` as callouts.
- [ ] **Model Arena** (F-03): sortable leaderboard table (MAE, RMSE, R², MAPE, CV, time, size), bar chart toggle by metric, "default" badge, click row → metrics detail (residual histogram, segment error tables, ablation chart, interval coverage).
- [ ] **Explain** (F-06): global importance bar chart (readable labels), notes: "associated with, not caused by".
- [ ] **Limitations page (F-07)**: use HANDBOOK §13 wording + dataset window from `/insights/dataset` + the **drift chart** (random vs time split, per-month MAE) from `/insights/drift`.
- [ ] Switch from mock to the real API at M1 (end Week 2); keep mock mode for offline demos.

### Phase 2: Decision-support features (Weeks 3-4) · F-09, F-13, F-14, F-17, F-18
- [ ] Staleness banner finalised: level colors + icon + text (not color alone), tooltip explaining each level.
- [ ] **Cheapest-date finder (F-13):** line chart of predicted price ±7 days with interval band, highlight cheapest day; disclaimer "estimates from historical patterns".
- [ ] **Model switcher (F-14):** change `model_id`, re-run, show side-by-side delta.
- [ ] **Compare tool (F-17):** same route, multiple airlines → grouped bars with intervals; route comparison (A→B vs A→C).
- [ ] **Market-shift slider (F-18):** −20%…+40% multiplier applied **client-side** to price and interval; label: "What-if assumption, not a forecast."
- [ ] Copy-to-clipboard + shareable URL (encode inputs in the hash/query).

### Phase 3: Wow (Weeks 4-5) · F-19, F-22, F-23, F-29
- [ ] **Batch CSV (F-19):** drag-and-drop, preview, results table, download CSV; show per-row errors.
- [ ] **History (F-22):** last 10 predictions in `localStorage`, compare any two.
- [ ] **A11y & polish (F-23):** labels/ARIA, keyboard flow, focus rings, contrast ≥ 4.5:1, dark mode, `prefers-reduced-motion`, Lighthouse a11y ≥ 90 and performance ≥ 85.
- [ ] Export result card as image/PDF (F-29) (optional).
- [ ] Guided tour on first load (3 tooltips) and an "About / How it works" page with the architecture diagram.

### Phase 4: Demo and report (Weeks 5-6)
- [ ] `docs/DEMO_SCRIPT.md`: a 5-minute scripted walkthrough that hits R1-R7 in order (inputs to type, what to point at, fallback if API down = mock mode).
- [ ] Slides (≤ 12): problem, data window, pipeline, Arena, factors, uncertainty, drift/limitations, demo, future work.
- [ ] Report chapters: *UI/UX Design*, *Demo*, *Conclusion & Future Work*; assemble `reports/final_report.md` with all chapters (each owner writes theirs).

## 6. UX rules (they make us look senior)
1. Never show a price without its **interval** and **staleness** message.
2. Currency is always `₹` with `en-IN` grouping (e.g. ₹5,230).
3. Plain-English labels, not column names.
4. Every chart has a title, axis labels, units and a one-line takeaway.
5. Every async view has loading, empty and error states.
6. Disclaimers are visible, not hidden: a persistent footer + the banner.
7. Works at 360 px width (phone) and 1440 px (laptop).

## 7. Acceptance criteria
- Entire app usable offline in mock mode and online with a real API without code changes (toggle only).
- All fetch calls only in `api.js`; no console errors on any page.
- Each requirement R1-R7 is demonstrable in under 5 minutes using the demo script.
- Lighthouse: a11y ≥ 90, best practices ≥ 90.
- Layout verified at 360 px, 768 px and 1440 px.

## 8. Pitfalls
- Don't hardcode `http://localhost:8000` in many files: `config.js` only.
- Don't compute model logic in JS. The market slider is the only client-side math and it's labeled.
- `fetch` doesn't throw on HTTP 4xx/5xx: check `response.ok` and parse the standard error shape.
- CORS errors usually mean the API isn't running or the origin isn't allowed: tell P3, don't hack around it.
- Serve with a static server (`python -m http.server 5500` in `frontend/`), not `file://` (module imports and fetch need http).

## 9. Antigravity prompts (copy-paste)
- "Read `.agent/rules/project-rules.md`, `docs/CONTRACTS.md` §3-5 and `docs/roles/ROLE_4_FRONTEND_UX.md`. Build `frontend/js/api.js` with a mock mode that reads `frontend/mocks/`. Only edit `frontend/`."
- "Build the staleness banner component with four levels per CONTRACTS §3.2 and ROLE_3 §5 thresholds; text and icon must not rely on color alone."

## 10. Backup / pairing
Pair with P3 for the first live fetch and CORS; pair with P2 to word the explanations and limitations clearly. You are also the **integration owner for the demo** (M3 rehearsal).
