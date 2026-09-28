# Data quality — traffic pollution & the mandatory prod filter

GA4 property: **Noga, `535285702`** (measurement `G-Q8KR9ZPHVS`). Unfiltered traffic is
dominated by non-real users. **Every user-facing KPI must use the full prod filter.**

### 2026-09-28 — The three pollution layers (measured, 28d to Sep 27)

| Host / scope | Users | Sessions | Why |
| --- | --- | --- | --- |
| `localhost` (+ `:3000`) | 35,501 | 36,175 | dev servers + Playwright E2E (`*-e2e-*` slugs visible in URLs) |
| `noga.global` all pages | 294 | 900 | prod, but includes public page |
| ↳ `noga.global` `/moneypenny` | 195 | 199 | **public page** — one-view visitors, all "new", all Direct, Wrexham IP |
| `noga.global` excl. `/moneypenny` | **101** | **705** | ✅ real app usage |
| `noga-1-real-estate-admin-git-dev-ddre.vercel.app` | 10 | 68 | Vercel staging preview |
| `www.noga.global` | 10 | 16 | **distinct host value** — excluded (call out if ever included) |

- Real user base is <100–150 people (DDRE advisors). If a report shows hundreds of "prod
  users", the moneypenny exclusion was dropped; if tens of thousands, the host filter was dropped.
- `/moneypenny` skews more than the count: it made **Wrexham** look like the #1 city (real
  #1 is London), made **Windows** look dominant (real dominant platform is Apple, ~66%),
  and inflated `form_start` users (auto event on the public page).

### 2026-09-28 — Mandatory prod filter (exact JSON for the MCP tool)

```json
{ "and_group": { "expressions": [
  { "filter": { "field_name": "hostName", "string_filter": { "match_type": 1, "value": "noga.global" } } },
  { "not_expression": { "filter": { "field_name": "pagePath", "string_filter": { "match_type": 1, "value": "/moneypenny" } } } }
] } }
```

- Use `hostName` (standard dimension), **not** `pageLocation` — `pageLocation` contains the
  full URL; a contains-filter for `noga.global` matched 0 rows because dev traffic lives on
  localhost.
- Verify a fresh report's URL rows once (top `pageLocation`) — all rows should be
  `https://noga.global/...`.

### 2026-09-28 — Labeling rule

- KPIs must read **Prod active users** (noga.global, excl. /moneypenny) — never
  "Noga advisors" / "active advisors". `activeUsers` = distinct GA clients, not the Strapi
  agent roster (cross-check order of magnitude with Strapi `list_agent` active count).
