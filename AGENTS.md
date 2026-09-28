# google-analytics-mcp (local working repo)

Local fork/worktree of Google's `google-analytics-mcp` used to generate **Noga production usage
reports** (GA4 property **Noga, `535285702`**, measurement ID `G-Q8KR9ZPHVS`).

This repo holds the MCP server source plus our report assets:

- `prompts/weekly-leadership-report.md` — the weekly leadership report runbook (config + gather steps + fill checklist)
- `reports/weekly-leadership-report.template.html` — leadership report template (fill all `{{TOKEN}}` / `data-fill` regions)
- `reports/noga-web-prod-report-*.html` — production usage snapshots
- `reports/noga-web-features-report-*.html` — feature & product health deep-dives
- `examples/noga-web-funnels.json` — reference funnels

## Running the MCP server

Installed in `.venv` (uv). It is registered as MCP server `analytics-mcp` in
`~/.config/mcp/mcp.json` (command: `.venv/bin/python -m analytics_mcp.server`, cwd = this repo,
`GOOGLE_APPLICATION_CREDENTIALS` = gcloud ADC, `GOOGLE_PROJECT_ID=serious-bearing-499201-q3`).

- Pin `mcp==1.24.0` in the venv — the repo code breaks with MCP SDK 2.x (`inputSchema` → `input_schema`).
- If GA calls fail with `invalid_grant` (refresh token revoked/stale), re-auth in a browser:
  `gcloud auth application-default login --scopes https://www.googleapis.com/auth/analytics.readonly,https://www.googleapis.com/auth/cloud-platform --client-id-file=secret.json`
  then restart the MCP server (it caches credentials at startup).
- `secret.json` (OAuth client JSON) is committed locally — **do not push this repo** anywhere shared.

## GA query rules (non-negotiable)

1. **Production filter on EVERY query** for user-facing KPIs:
   ```json
   { "and_group": { "expressions": [
     { "filter": { "field_name": "hostName", "string_filter": { "match_type": 1, "value": "noga.global" } } },
     { "not_expression": { "filter": { "field_name": "pagePath", "string_filter": { "match_type": 1, "value": "/moneypenny" } } } }
   ] } }
   ```
   - `hostName` exact — `www.noga.global` is a **distinct host** (excluded); `localhost` is dev/E2E (excluded by host filter).
   - `/moneypenny` is a **public page** — it adds ~200 one-view "new" users/month and skews users, cities, and devices.
2. **Sanity check**: prod active users should be in the *tens* (~100 per 28 days as of Sep 2026).
   Hundreds → moneypenny exclusion dropped; tens of thousands → host filter dropped.
3. Tool args are **snake_case** (protobuf), not the REST camelCase from Google docs.
   Filter expressions need the `{ "filter": {...} }` wrapper inside `and_group`/`or_group`/`not_expression`.
   `order_bys` use `{ "metric": { "metric_name": "..." } }`.
4. Metric/dimension names that fail: `totalEvents` → `eventCount`; `deviceBrand` → `mobileDeviceBranding`;
   `metric: { fieldName }` → `metric: { metric_name }`.
5. Never report unfiltered `activeUsers` as "advisors" — label KPIs **Prod active users**.

## Auditing local repos (do this before any feature report)

Feature reports must be grounded in the actual product code — **audit the local repo first**,
don't guess features or events:

1. **Find the repo**: local checkouts live under `/home/kevin/@noga-1/` (e.g. `web` = DDRE-Global/web,
   `cms` = DDRE-Global/cms). Confirm with the user if unclear.
2. **Read the repo's `AGENTS.md`** (if present). For `web`, it defers to Serena memories — check the
   Serena server is up (`curl -s -o /dev/null -w "%{http_code}" -X POST http://127.0.0.1:9121/mcp` → 400 = up,
   else `npm run serena` in the repo), then read `mem:core` (feature routes map + invariants) and any
   feature-specific memories (`mem:monday-integration`, `mem:audit-trails`, …).
3. **Read the GA instrumentation source**. For `web` it is `src/lib/ga.ts` (all `sendGAEvent` helpers)
   plus `grep -rn "sendGAEvent" src` for ad-hoc calls (auth-context, hooks). This tells you which events
   exist and with what parameters.
4. **Cross-check against GA**: call `get_custom_dimensions_and_metrics` on the property.
   Only **registered** event parameters can be used as dimensions — unregistered params (e.g.
   `templateId`, `login`'s `method`) are dropped by GA. Metrics (`customEvent:step`, `:document_count`,
   `:query_length`) cannot be grouped by.
5. **Verify hosts/URLs**: pull top `pageLocation`/`hostName` rows and confirm the rows you're reporting
   on are real prod URLs before quoting numbers.
6. **Map features → events → data availability**, and call out gaps (events sent in code but absent in GA,
   unregistered params, zero-usage features) in the report's gaps section.

### Instrumentation snapshot (web, audited 2026-09-28 — re-verify per report)

- **Fully instrumented**: AML — `aml_list_viewed`, `aml_create_started`, `aml_create_step_completed`,
  `aml_request_submitted`, `aml_request_updated`, `aml_status_viewed`, `aml_search`, `aml_metric_filter_selected`.
- **Partially**: Documents — `document_sent` (final send carries `templateId`; draft send bare).
  Auth — `login` (method password|jwt, **not arriving in GA**) and `logout`.
- **Page-level only**: `/monday`, `/search`, `/offer`, `/viewing`, `/lead`, `/referral`, `/keys`,
  `/appointment`, `/building`, `/property`, `/listing`, `/contact`, `/profile`, `/settings`.
- Registered custom dimensions: `request_type`, `variant`, `is_draft`, `waiting_on`, `aml_complete`,
  `source`, `metric_id`, `aml_document_id`.

## Report conventions

- Output to `reports/`, named `<subject>-<type>-YYYY-MM-DD.html` (report date).
- **Commit and push** when meaningful changes have been made: new/updated reports, prompt or template
  updates, and AGENTS.md changes. Use clear conventional-commit-style messages (e.g.
  `docs(reports): Noga prod usage snapshot 2026-09-28`, `feat(reports): feature adoption section in template`).
  Never commit credentials (`secret.json`, ADC files) — if one appears in `git status`, stop and fix the ignore first.
- Reuse the dark design system from `reports/weekly-leadership-report.template.html` (Chart.js via CDN,
  `--surface`/`--border` variables, kpi-grid cards). Standalone/self-contained HTML.
- Every report must state its **period, property, and exact filter** in the hero/note.
- Never invent numbers — if a query fails, mark `--` / "Unavailable" and explain in a note.
- Distinguish "no prod usage" from "no data" — check the feature is actually deployed before claiming zero adoption.
