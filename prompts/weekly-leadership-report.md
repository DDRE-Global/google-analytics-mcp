# Weekly Leadership Delivery & Usage Report

Copy this prompt into Cursor each week. The agent gathers live data from MCP tools and `gh`, then fills the HTML template.

---

## Config (edit before running)

```yaml
period: last_7_days_ending_yesterday # or: last_complete_mon_sun
output_dir: reports/
template: reports/weekly-leadership-report.template.html
output_pattern: reports/weekly-leadership-report-YYYY-MM-DD.html # week-ending date

google_analytics:
  property_id: "535285702"
  # REQUIRED for all leadership KPIs/charts — never report unfiltered activeUsers as advisors
  host_filter: "noga.global"          # exact host; www.noga.global is a DISTINCT host — do not include
  exclude_paths: ["/moneypenny"]     # public page — adds ~200 one-view "new" users per month; ALWAYS exclude
  funnels_ref: examples/noga-web-funnels.json
  # Feature usage & instrumentation: audit DDRE-Global/web first (see AGENTS.md → "Auditing local repos").
  # Fully instrumented: AML (8 events), Documents (document_sent), Auth (login/logout — login currently broken in GA).
  # Page-level only: monday, search, offer, viewing, lead, referral, keys, appointment, contact, building, property, listing.

jira:
  # Leave empty to discover via getVisibleJiraProjects, then prefer Noga/DDRE projects
  project_keys: ["SCRUM"]

github:
  repos:
    - DDRE-Global/cms
    - DDRE-Global/external-dataset-mcp
    - DDRE-Global/google-analytics-mcp
    - DDRE-Global/mcp
    - DDRE-Global/migration
    - DDRE-Global/web

hostinger:
  # Shared hosting list endpoints often return empty — prefer explicit VPS IDs below.
  virtual_machines:
    - id: 1465785
      hostname: srv1465785.hstgr.cloud
      ip: 72.61.123.84
      role: edge # Docker + Traefik; fronts DDRE app domains
      domains:
        - hub.ddre.ai
        - buyer.ddre.ai
        - pma.ddre.ai
        - searches.ddre.ai
        - sdlt.ddre.ai
        - tenant.ddre.ai
    - id: 1572697
      hostname: srv1572697.hstgr.cloud
      ip: 72.61.145.175
      role: data # MariaDB u819775029_rmdb + scrapers; no public HTTP; SSH tunnel from Azure MCP
  # KPI kpi_apps_online = count of VPS with state=running (expect 2)

azure:
  resource_groups:
    - noga-mcp
    - noga-migration
    - external-dataset-mcp
  also_discover: true # include other RGs matching noga* or *mcp*
```

---

## Your task

Produce a **lightweight, summaritive HTML report for senior leadership** covering delivery and usage for the configured week.

1. Read the template at `reports/weekly-leadership-report.template.html`.
2. Gather data from each source below (skip a source only if the tool/auth is unavailable — note the gap in Watchouts, do not invent numbers).
3. Write a filled copy to `reports/weekly-leadership-report-YYYY-MM-DD.html` (use the week-ending date).
4. Replace every `{{TOKEN}}` and every element with a `data-fill` attribute. Populate `#report-data` JSON so Chart.js charts render.
5. Keep the visual system from the template (do not redesign).

### Audience & tone

- Senior leadership — plain English, short paragraphs, few numbers.
- Prefer week-over-week deltas where both weeks are available.
- Label estimates clearly (`estimated`, `retail pricing`, not billed).
- **Never invent metrics.** If data is missing, leave `--` / “Unavailable” and explain once in Watchouts.
- Hostinger is an **ops proxy** (health, deploys, VPS load). Do **not** claim in-app user journeys from Hostinger.

---

## Period

- Default: `7daysAgo` → `yesterday` (this week), and the prior 7 days for deltas.
- If config says `last_complete_mon_sun`, use the most recent completed Monday–Sunday.
- Set `{{WEEK_RANGE}}`, `{{GENERATED_AT}}` (today’s date), and `{{OVERALL_POSTURE}}` (one sentence: healthy / mixed / attention needed + why).

---

## Gather steps (in order)

### 1. Jira — `plugin-atlassian-atlassian`

Tools: `getAccessibleAtlassianResources`, `getVisibleJiraProjects` (if keys empty), `searchJiraIssuesUsingJql`.

Suggested JQL (adapt project keys):

```
# Closed this week
project in (KEYS) AND statusCategory = Done AND resolved >= -7d ORDER BY resolved DESC

# Still open / in progress
project in (KEYS) AND statusCategory != Done AND updated >= -14d ORDER BY priority DESC, updated DESC

# Blockers / critical
project in (KEYS) AND statusCategory != Done AND (priority = Highest OR priority = High OR flagged = impediment) ORDER BY priority DESC
```

Summarise: count closed, notable shipped items (max 5 titles), open blockers (max 5). Fill delivery insights + `jira_rows` (highlight closed + blockers, not every ticket).

### 2. GitHub — `gh` CLI

For each configured repo (or discover if empty):

```bash
gh pr list --repo ORG/REPO --state merged --search "merged:>=YYYY-MM-DD" --limit 30
gh release list --repo ORG/REPO --limit 5
gh issue list --repo ORG/REPO --label "bug" --state open --limit 10
```

Summarise: PRs merged count, notable releases, critical open bugs. Fill `github_rows` and KPI `kpi_prs_merged`.

### 3. Strapi — `user-strapi-mcp`

Lightweight CMS pulse only (counts + notable changes, not record dumps):

- `list_import-job` — recent / failed jobs this week
- `list_noga-app` — apps registered / changes
- `list_audit-trail` or `list_agent` if useful for volume signals

Fill `strapi_summary` and `strapi_rows` (3–6 signal rows max).

### 4. Google Analytics (Noga production) — `user-analytics-mcp`

Property: **`535285702`**. Reference funnels in `examples/noga-web-funnels.json`.

**Critical — do not use raw `activeUsers` as “advisors”.**

- `activeUsers` = distinct browser clients (`client_id`), **not** Strapi agent headcount.
- Dev/E2E currently writes to this property (`G-Q8KR9ZPHVS` on localhost / Playwright). Unfiltered weeks are dominated by localhost (often &gt;90%).
- **All leadership KPIs, charts, tables, and funnels MUST filter**  
  `dimension_filter`: `hostName` exact `noga.global`  
  **AND** `not_expression`: `pagePath` exact `/moneypenny` (public page).  
  Do **not** include `www.noga.global` — it is a distinct host value (~10 users/month); call it out if you ever decide to include it.
- Label the KPI **Prod active users** (noga.global, excl. /moneypenny), never “Noga advisors” / “active advisors” unless comparing to Strapi roster separately.
- Sanity check: prod active users should be in the **tens** (≈100/28d as of Sep 2026). If you get hundreds+, you have dropped the moneypenny exclusion; if tens of thousands, you have dropped the host filter.
- Optionally note pollution once: unfiltered total vs localhost vs prod (Watchouts or GA summary) — do not put unfiltered totals in the KPI tile.
- Cross-check: Strapi `list_agent` (active) and confirmed users ≈ prod activeUsers order of magnitude; if prod ≫ roster, investigate; if unfiltered ≫ prod, cite localhost/Playwright.

Run at least (**every report with hostName = noga.global**):

| Report           | Dimensions              | Metrics                                                                    | Notes                                   |
| ---------------- | ----------------------- | -------------------------------------------------------------------------- | --------------------------------------- |
| Headline         | (none)                  | `activeUsers`, `sessions`, `engagedSessions`, `engagementRate`, `newUsers` | This week + prior week                  |
| Host breakdown   | `hostName`              | `activeUsers`, `sessions`                                                  | Sanity: localhost vs noga.global        |
| Daily events     | `date`                  | `eventCount`                                                               | Chart `gaDailyEvents`                   |
| Daily engagement | `date`                  | `sessions`, `activeUsers`                                                  | Chart `gaEngagement`                    |
| Top pages        | `pagePath`              | `screenPageViews`, `activeUsers`                                           | Top 8                                   |
| Key events       | `eventName`             | `eventCount`, `activeUsers`                                                | Filter or post-filter to product events |
| Docs / AML       | event filters on prod   | active users per step                                                      | Prefer filtered event report over unfiltered funnel API |

Product events of interest (instrumented in DDRE-Global/web `src/lib/ga.ts`):  
`aml_list_viewed`, `aml_create_started`, `aml_create_step_completed`, `aml_request_submitted`, `aml_request_updated`, `aml_status_viewed`, `aml_search`, `document_sent`, `login` (⚠ not arriving in GA — see features report), `logout`.  
Registered custom dimensions (groupable): `customEvent:request_type`, `:variant`, `:is_draft`, `:waiting_on`, `:aml_complete`, `:source`, `:metric_id`, `:aml_document_id`.  
Registered custom metrics (NOT groupable as dimensions): `customEvent:step`, `:document_count`, `:query_length`.  
Known gaps: `login` missing, `templateId`/`step` unregistered as dimensions — re-verify with `get_custom_dimensions_and_metrics` before building reports.

Fill KPIs, funnels (`funnel_document`, `funnel_aml`), tables, `{{GA_SUMMARY}}`, and **exactly 2–3** data-backed insights (`ga_insight_1`…`3`). Prefer Chart.js visuals over long tables.

### 4b. Feature health (weekly, lightweight)

Fill the **Feature Adoption & Product Health** section of the template:

| data-fill | Source |
| --- | --- |
| `feature_adoption_rows` | `pagePath` top rows (prod filter) grouped by first route segment vs the 16-route feature map in `DDRE-Global/web` `mem:core`; flag routes with 0 prod usage |
| `aml_health_rows` | `aml_status_viewed` by `customEvent:waiting_on` + `customEvent:aml_complete`; call out completed count |
| `docs_health` | `document_sent` users/events + `/document/send` avg session duration |
| `auth_health` | `/login` + `/sso` users/views; note `login` event availability (0 = tracking still broken) |

Deeper feature reports live in `reports/noga-web-features-report-*.html` — link from Watchouts when a feature needs attention.

### 5. Hostinger VPS fleet — `user-hostinger-api`

**Ops proxy only.** Two known VPS (do not invent a third). Shared-hosting website list is usually empty for this account — **do not** treat that as “apps down”.

| ID | Role | Template | Purpose |
|----|------|----------|---------|
| `1465785` | Edge | Ubuntu 24.04 + Docker + Traefik | Reverse proxy / TLS; DNS for hub, buyer, pma, searches, sdlt, tenant → `72.61.123.84` |
| `1572697` | Data | Ubuntu 24.04 LTS | MariaDB `u819775029_rmdb` (Rightmove / LonRes / Land Registry) + scrapers; Azure external-dataset MCP tunnels here over SSH. **No public HTTP** (timeouts are expected). |

Gather (use **explicit IDs** — `VPS_getVirtualMachinesV1` often returns `[]`):

1. `VPS_getVirtualMachineDetailsV1` for `1465785` and `1572697` — state, plan, template, IPs.
2. `VPS_getMetricsV1` each VM (`date_from` / `date_to` = week) — `cpu_usage`, `ram_usage`, `disk_space`, traffic, `uptime`. Convert RAM/disk to **% of 8 GB / 100 GB**. Daily-average into chart series.
3. Edge only: `VPS_getProjectListV1` (`1465785`) — Traefik + any compose projects. Note Docker Manager does **not** list every app container; Traefik file provider may route hosts not visible in project list. Data VPS returns “OS does not support Docker Manager”.
4. `VPS_getActionsV1` + `VPS_getBackupsV1` — weekly `backup_create` success/fail (platform backups, not app deploys).
5. Optional sanity: `dig +short` configured domains → edge IP.

**Do not report from Hostinger:** advisor journeys, page views, document/AML funnels (those are GA).

Fill: `hostinger_vps_rows`, `hostinger_domains`, `hostinger_load_rows`, `hostinger_docker_*`, `hostinger_deploys` (backups/actions), `{{HOSTINGER_SUMMARY}}`, KPI `kpi_apps_online` (e.g. `2/2`), and `#report-data.hostingerVps`:

```json
"hostingerVps": {
  "labels": ["20 Jul", "..."],
  "edgeCpu": [], "dataCpu": [],
  "edgeMemory": [], "dataMemory": []
}
```

### 6. Azure — `plugin-azure-azure`

1. `subscription_list` / `group_list` — confirm subscription.
2. For each RG in config (+ discovered `noga*` / `*mcp*` if enabled): `group_resource_list`.
3. `monitor` — simple usage signals where available (requests, CPU, availability). Skip quietly if no workspace/metrics.
4. Cost: prefer actual Cost Management via `az` if accessible; otherwise retail `pricing` estimates from SKUs. Always label **estimated** when not billed spend.

Fill `azure_rg_rows`, `azureCost` chart, `{{AZURE_SUMMARY}}`, KPI `kpi_azure_cost`, and the cost disclaimer.

---

## Synthesis rules

1. Write `{{OVERALL_POSTURE}}` last — one sentence after all data is in.
2. Delivery section: both **shipped outcomes** and **health** (blockers / open critical), concise.
3. Watchouts: **max 3** bullets. Only items that need leadership attention (risk, cost spike, usage drop, blocked delivery). Remove unused list items if fewer than 3.
4. Do not dump raw API JSON into the HTML.
5. Use `--` for unknown KPIs; use status pills (`ok` / `warn` / `bad`) where helpful in Hostinger status.

---

## Fill checklist

- [ ] All `{{...}}` tokens replaced or removed
- [ ] All `data-fill` regions updated (including Feature Adoption & Product Health: `feature_health_summary`, `feature_adoption_rows`, `aml_health_rows`, `docs_health`, `auth_health`)
- [ ] `#report-data` JSON valid with chart series (empty arrays OK if no data)
- [ ] Output saved as `reports/weekly-leadership-report-YYYY-MM-DD.html`
- [ ] Open file briefly / sanity-check that Chart.js canvases have data when series are non-empty
- [ ] Reply to the user with the output path and a 3–5 bullet executive teaser (no need to paste the full HTML)

---

## Example kickoff (paste below this line)

```
Run the Weekly Leadership Delivery & Usage Report using prompts/weekly-leadership-report.md.
Use the config block in that file. Fill reports/weekly-leadership-report.template.html and save the filled report under reports/.
```
