# GA Data API / MCP tool query gotchas

All via the `analytics-mcp` MCP server (`run_report` etc.). Tool args are **protobuf snake_case**,
NOT the REST camelCase from Google's docs.

### 2026-09-28 — Field-name traps (each caused a 400)

| Tried | Correct | Context |
| --- | --- | --- |
| `totalEvents` | `eventCount` | metric for event volume |
| `deviceBrand` | `mobileDeviceBranding` | device dimension |
| `order_bys: [{metric: {fieldName}}]` | `{ "metric": { "metric_name": "activeUsers" }, "desc": true }` | OrderBy shape |
| `dimension_filter: {field_name, string_filter}` (bare) | `{ "filter": { field_name, string_filter } }` | always wrap in `filter`/`not_expression`/`and_group`/`or_group` |
| `Filter.case_sensitive` at top level | put `case_sensitive` inside `string_filter`/`in_list_filter` | — |
| `customEvent:step` as dimension | it's a custom **metric** (`averageCustomEvent:step`) — not groupable | see instrumentation.md |

- `and_group`/`or_group` expressions are each a **full FilterExpression**:
  `{ "and_group": { "expressions": [ { "filter": {...} }, { "not_expression": { "filter": {...} } } ] } }`
  (dropping the inner `filter` wrapper → `Unknown field for FilterExpression: field_name`).
- Match types: `1` = exact, `2` = contains.

### 2026-09-28 — metric_filter on custom metrics is silently ineffective

- Filtering `customEvent:step == N` on `aml_create_step_completed` returned the **unfiltered**
  totals for every N (229 events / 31 users for step 1..4 alike). Don't trust per-step
  breakdowns via metric_filter — use `averageCustomEvent:step` instead until `step` is
  registered as a dimension.

### 2026-09-28 — Running many queries: use mcpScript, not tool calls one-by-one

- `mcpScript` tool path format is the **prefixed name**: `tools.call("analytics-mcp_run_report", args)`
  (NOT `mcp__analytics-mcp.run_report` — that gives `tool_not_found`).
- Result shape: `{ ok, data }` where `data` is the MCP content list; the GA JSON is
  `JSON.parse(data[0].text)` (or `v.data` string). Errors come back **inside** `data[0].text`
  as `{"error": "..."}`, not as `v.error` — unwrap before checking.
- The engine is an **older JS runtime**: avoid optional chaining / modern syntax
  (saw `Unexpected token ')'` on code node 22 parses fine). Keep scripts conservative.
- Large outputs get truncated at ~50KB and saved to `/tmp/pi-mcp-output-*/output-*.txt`;
  that file is a JSON-encoded MCP result object (`.content[0].text` holds the payload) — parse it,
  don't eyeball it.
- `dimensions: []` is allowed (aggregate-only report). `limit` ≤ 250000; use 28 for date rows.
- Custom dimensions/metrics in args use the `customEvent:` prefix, e.g. `customEvent:request_type`.
