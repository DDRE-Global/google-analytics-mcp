# GA query gotchas

Construction traps for the Analytics Data / Admin APIs, whether called through the MCP tools (args are
**snake_case protobuf**) or the REST API (same structure, **camelCase**). The mandatory prod filter and
why it exists: `mem:data-quality`.

## Filter expressions

- Always wrap: `{ "filter": { "field_name", "string_filter" } }`, nested in `and_group` / `or_group` /
  `not_expression`. Each `and_group` expression is itself a full FilterExpression, so a bare
  `{ "field_name": ... }` inside it fails with `Unknown field for FilterExpression: field_name`.
- `case_sensitive` belongs **inside** `string_filter` / `in_list_filter`, not at the filter level.

### 2026-11-02 — `match_type` is numeric; the enum names are unreliable (corrects "2 = contains")

Measured against the v1beta endpoint with a `pagePath` filter on `/document` (28d):

| Value | Meaning | Rows hit |
| --- | --- | --- |
| `1` | exact | 1 (`/document`) |
| `2` | **starts with** | 148 (`/document`, `/document/send`, `/document/<id>`) |
| `3` | ends with | 1 |
| `4` | contains | 149 (catches `/x/document/y` that `2` misses) |
| `5` | full regexp | as pattern |
| `6` | partial regexp | as pattern |

Send the **numbers**. `CONTAINS`, `FULL_REGEXP` and `PARTIAL_REGEXP` are accepted as names, while
`EQUALS`, `STARTS_WITH` and `MATCH_TYPE_EXACT` all 400, and `REGEXP` has never existed:
`Invalid value at ... StringFilter.MatchType`. Consequence for the house filter: the `/moneypenny`
exclusion is a *starts-with* negation, so it covers `/moneypenny/*` too.

## Dates, rows, totals

- `dateRanges` accepts only `YYYY-MM-DD`, `NdaysAgo`, `yesterday`, `today`. `3monthsAgo` 400s, so use
  explicit dates for monthly comparisons.
- `rowCount` is the number of rows the query **matched**, not the number returned: `limit: 3` reported
  `rowCount: 179`. Count your own rows.
- With `dimensions: []` you get a single aggregate row and no `totals` key. Ask for
  `metricAggregations: ["TOTAL"]` to get a `totals` entry; alongside a dimension it arrives as a row with
  `RESERVED_TOTAL` in the dimension slot.
- `pagePath` is high cardinality (hundreds of rows in a month): aggregate in code by first path segment
  rather than reading a truncated row list.

## Names

| Tried | Correct |
| --- | --- |
| `totalEvents` | `eventCount` |
| `deviceBrand` | `mobileDeviceBranding` |
| `order_bys: [{ metric: { field_name } }]` | `{ "metric": { "metric_name": "activeUsers" }, "desc": true }` |
| `customEvent:step` as a dimension | it is a custom **metric**: `averageCustomEvent:step` (see `mem:instrumentation`) |

## Aggregation traps

- A `metric_filter` on a custom metric is **silently ineffective**: filtering
  `customEvent:step == N` on `aml_create_step_completed` returned the same unfiltered totals for every
  N. Use `averageCustomEvent:step` until `step` is registered as a dimension.
- Only **registered** event parameters are groupable as dimensions; unregistered params are dropped by GA
  at ingest and can neither be grouped nor filtered (which params: `mem:instrumentation`).
- `dimensions: []` is legal (aggregate-only), `limit` accepts up to 250000.
