# Adoption baseline: what prod usage actually looks like

Dated measurements of the Noga admin app in production, plus the trap they exist to prevent. Every
number here is the filtered prod scope (`mem:data-quality`).

### 2026-11-02 — route families, prod, 2026-08-01 to 2026-11-01 (views | peak users)

| Route family | Views | Peak users |
| --- | --- | --- |
| `/document` | 3,376 | 89 |
| `/home` | 2,141 | 144 |
| `/aml` | 1,727 | 76 |
| `/login` | 1,333 | 217 |
| `/monday` | 372 | 60 |
| `/sso` | 234 | 72 |
| `/settings` | 26 | 11 |
| `/lead`, `/leads` | 5 | 2 |
| `/listing` | **1** | 1 |
| `/property` | 1 | 1 |
| `/search` (requirements + Matches tab) | **0** | 0 |

Monthly totals: Aug 2,239 views / 98 users, Sep 3,038 / 110, Oct 3,946 / 130. Real usage is documents,
AML, Monday and auth; the CRM/property/search half of the app is essentially untouched in prod.
Earlier baseline (28d to 2026-09-27): 10 of the 16 feature routes had zero prod page views.

### 2026-11-02 — counts in the product database are not advisor demand

The whole-market (Sprift) feature set looked busy from the CMS: 418 Sprift listings, 318 of them curated
onto requirements, 24 search runs in 21 days, two "active" agents (one named `DDRE Admin`). Read against
the GA route table, that volume is dev/QA traffic: it arrives with localhost and Vercel-preview page
views, not prod ones. Modelling page views from ingestion or curation counts therefore overstates real
demand by orders of magnitude, and anything sized on it (a third-party API quota, a capacity plan) is
cap-bound on a load nobody generates.

**Rule:** to measure how often a screen is used, query the route, never the records a background job
created. Re-check with: prod-filtered `run_report` on `pagePath` (28d and monthly), split by `hostName`
to see how much of the traffic is dev, and aggregate to route families in code.

**Distinguish "no usage" from "no data" before writing either claim.** Three separate causes look
identical in a zero-row report: the feature is not deployed, the route is deployed but nobody uses it, or
the route is used and sends no GA hit (no instrumentation: `mem:instrumentation`). Prod routes answer 403
to unauthenticated requests, so a plain curl proves nothing about deployment state: check the deployment
or ask, then check the instrumentation, then report adoption.
