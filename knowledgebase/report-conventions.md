# Report conventions

### 2026-09-28 — Current conventions (as of the 2026-09-28 reports)

- Output dir: `reports/`, naming `<subject>-<type>-YYYY-MM-DD.html` (report date).
  - `noga-web-prod-report-*.html` — production usage snapshot (KPIs, daily, pages, AML, channels/devices/cities, events)
  - `noga-web-features-report-*.html` — feature & product health deep-dive (per-feature sections + instrumentation gaps)
  - `weekly-leadership-report-*.html` — filled from `reports/weekly-leadership-report.template.html`
- Standalone self-contained HTML, dark design system shared with the leadership template
  (Chart.js via jsdelivr CDN, CSS variables `--surface`/`--border`/`--accent`, kpi-grid cards,
  `.note`/`.alert` callouts, `.pill ok|warn|bad|off` status badges).
- Hero must state: period, property (535285702), and the **exact filter** used.
- Every KPI table/chart derives from the mandatory prod filter (data-quality.md).
- Never invent numbers — failed query → `--` / "Unavailable" + one-line note.
- Distinguish "no prod usage" from "no data": confirm the feature is deployed before claiming
  zero adoption (e.g. `/listing` is marked WIP in the feature map).
- Commit & push meaningful report work (see AGENTS.md §Report conventions); never commit
  `secret.json` or ADC files.

### 2026-09-28 — Template/prompt state

- Leadership template has the **Feature Adoption & Product Health** section (added 2026-09-28)
  with data-fill regions: `feature_health_summary`, `feature_adoption_rows`, `aml_health_rows`,
  `docs_health`, `auth_health`.
- `prompts/weekly-leadership-report.md` §4b defines the weekly feature-health gather steps and
  the instrumentation snapshot (keep in sync with knowledgebase/instrumentation.md when it changes).
