# Data quality: pollution layers and the mandatory prod filter

Unfiltered traffic on this property is dominated by non-real users, so an unfiltered number is not
conservative, it is fiction. Every user-facing KPI uses the filter below. Property identity: `mem:core`.

## Mandatory prod filter

```json
{ "and_group": { "expressions": [
  { "filter": { "field_name": "hostName", "string_filter": { "match_type": 1, "value": "noga.global" } } },
  { "not_expression": { "filter": { "field_name": "pagePath", "string_filter": { "match_type": 2, "value": "/moneypenny" } } } }
] } }
```

- `match_type` `1` = exact, `2` = starts with (`4` = contains; the table is in
  `mem:ga-query-gotchas`), verified 2026-11-02 through both the REST endpoint and these tools' snake_case
  arguments, which return identical rows.
- Use the `hostName` dimension, **not** `pageLocation`: a contains-filter on `pageLocation` for
  `noga.global` matched 0 rows because the dev traffic lives on localhost.
- `www.noga.global` is a **distinct host value** and stays excluded; if a report ever includes it, say so.
- Verify once per report that the rows are real prod URLs (top `pageLocation` should all be
  `https://noga.global/...`).

## The pollution layers (28d to 2026-09-27)

| Scope | Users | Sessions | Why |
| --- | --- | --- | --- |
| `localhost` (+ `:3000`) | 35,501 | 36,175 | dev servers and Playwright E2E; the fixture slugs give it away (`/listing/listing-e2e-1`, `/search/requirement-e2e-1`) |
| `noga.global`, all pages | 294 | 900 | prod, but the public page is included |
| ↳ `/moneypenny` alone | 195 | 199 | **public page**: one-view visitors, all new, all Direct, Wrexham IP |
| ↳ excluding `/moneypenny` | **101** | **705** | the real app |
| `*.vercel.app` previews | 10 | 68 | staging previews, also excluded by the exact-host filter |
| `www.noga.global` | 10 | 16 | separate host value |

`/moneypenny` distorts more than the headline: it made Wrexham the top city (real top city: London),
made Windows look dominant (real dominant platform: Apple, ~66%), and inflated `form_start` users.

### 2026-11-02 — scale of the noise, re-measured (90d)

`localhost` 125,274 views / 103,899 users against prod's 9,771 / 883: **~93% of the property is us**.
Prod monthly, filtered: Aug 2,239 views / 98 users, Sep 3,038 / 110, Oct 3,946 / 130. Nothing on the GA
side filters this automatically (no internal-traffic or hostname filter is configured on the data
stream), so the exclusion is the caller's job every single time, and the GA console UI is misleading to
anyone who does not know that.

## Sanity bands

Real usage is a few dozen to a hundred-plus people per month. Hundreds of prod users → the
`/moneypenny` exclusion leaked. Tens of thousands → the host filter leaked. Cross-check the order of
magnitude against the agent/advisor roster in the CMS before publishing anything that counts people.

## Labelling

Label KPIs **Prod active users** (noga.global, excluding `/moneypenny`). `activeUsers` counts distinct GA
clients, not accounts, so never write it as "advisors" or "Noga advisors": one person on two devices is
two, and an unauthenticated hit on a public route can be one more.
