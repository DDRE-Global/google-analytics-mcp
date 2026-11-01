# Report conventions

The deliverables of this repo are standalone HTML reports in `reports/`.

## Files

- Name: `<subject>-<type>-YYYY-MM-DD.html`, dated by report date.
  - `noga-web-prod-report-*` — production usage snapshot (KPIs, daily, pages, AML, channels/devices/cities, events)
  - `noga-web-features-report-*` — feature and product-health deep dive (per-feature sections + instrumentation gaps)
  - `weekly-leadership-report-*` — filled from `reports/weekly-leadership-report.template.html`, whose
    `data-fill` regions are `feature_health_summary`, `feature_adoption_rows`, `aml_health_rows`,
    `docs_health`, `auth_health`
- Self-contained: the dark design system of the leadership template, Chart.js from jsdelivr, the
  `--surface` / `--border` / `--accent` variables, kpi-grid cards, `.note` / `.alert` callouts and
  `.pill ok|warn|bad|off` badges.
- The hero states the period, the property, and the **exact filter** used (`mem:data-quality`). Anything
  without the filter in the hero is not a report, it is a rumour.

## Numbers

- Never invent or extrapolate a figure. A failed query is `--` / "Unavailable" plus a one-line reason.
- Every KPI derives from the mandatory prod filter, and the sanity band is checked before publishing
  (`mem:data-quality`).
- Label counts by what they measure: **Prod active users**, not advisors (`mem:data-quality`).
- Before claiming a feature is unused or unmeasured, work through `mem:adoption-baseline` (deployed?),
  `mem:instrumentation` (does it send anything?) and a fresh query.

## Commit

Meaningful report work is committed and pushed with conventional messages
(`docs(reports): Noga prod usage snapshot YYYY-MM-DD`). Reports are committed artefacts but not the
source of record: the memory library carries the knowledge, and a report can always be regenerated
against GA. Never stage credentials (`secret.json`, ADC files, service-account JSON): if one shows up in
`git status`, fix the ignore before committing anything.
