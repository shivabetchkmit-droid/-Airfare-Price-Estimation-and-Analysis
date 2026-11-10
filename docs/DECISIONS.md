# DECISIONS: append-only decision log

Format: `| date | decision | reason | who | affects |`. Add a row whenever the team decides something that a future reader (or Antigravity) could question.

| Date | Decision | Reason | Who | Affects |
|------|----------|--------|-----|---------|
| 2026-11-10 | Project title is "Airfare Price Estimation and Fare Pattern Analysis" (header in brief said Solar; treated as typo, confirm with mentor) | Title and problem statement both about airfare | team | all |
| 2026-11-10 | Contract-first development with stubs/mocks for every seam | Minimize cross-person dependency | team | all |
| 2026-11-10 | Frontend is vanilla JS + Chart.js, no build step | Brief requires JavaScript + Fetch API; low tooling risk | team | P4 |
| 2026-11-10 | Target modeled as log1p(price), metrics reported in INR | Right-skewed price | team | P2 |
| 2026-11-10 | Primary dataset = 2019 Indian flight fares; secondary (2022, has days_left) optional | PS mentions journey date and booking attributes | team | P1, P2 |
| | *(add yours below)* | | | |
