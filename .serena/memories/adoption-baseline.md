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
Earlier baseline (28d to 2026-09-27): 10 of the 16 feature routes had zero prod page views. For why most
of those zeros are structural, see the preview gate section below.

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

### 2026-11-02 — the zeros have a cause: the app's preview gate

Eleven route prefixes are blocked in production by the app repo's `src/proxy.ts`: `/building`,
`/property`, `/listing`, `/contact`, `/lead`, `/referral`, `/appointment`, `/keys`, `/search`, `/offer`,
`/client`. The test is `isPreview()` in `src/lib/is-preview.ts`, which is true whenever
`NEXT_PUBLIC_VERCEL_ENV ?? VERCEL_ENV` is anything other than `production`. So those modules work in local
dev and on preview deploys and answer a `403` rewrite on `noga.global`. A plain `curl` to a gated prod URL
returning `403` is that gate, not a WAF, and it says nothing about whether the route is otherwise
deployed.

Consequences:

- **Zero prod page views on a gated route is a deploy state, not an adoption finding.** Rule the gate out
  before writing any non-adoption claim, and label gated routes as gated rather than unused.
- A gated URL someone still navigates to records a page view on the *requested* path while serving the 403
  content, which is why the prod `/listing` family shows 1 view / 1 user instead of 0.
- Whole-market features (requirement search, Matches, listing record) therefore have **no** prod demand
  signal at all. Measure them on the preview hosts by `hostName` (`*.vercel.app`: 1,213 views / 18 users
  over 90d) or do not measure them, and say which you did. Nothing sized on prod numbers for these routes
  means anything yet, which is exactly how a third-party API quota gets estimated 100x too high.
- Re-check the list by reading `src/lib/is-preview.ts` in the app repo, and re-measure by dropping the host
  filter and splitting page views by `hostName`.

## Distinguishing "no usage" from "no data"

Three causes look identical in a zero-row report, in the order to rule them out: the route is behind the
preview gate above, the route is live and nobody uses it, or the route is live and used but sends no GA hit
(no instrumentation: `mem:instrumentation`). Only the second is an adoption finding, and each has a
different re-check: the gate list in the app repo, a prod-filtered report over a wider window, and the
event audit in `mem:instrumentation`.
