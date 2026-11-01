# Core: source map of this repo

Local fork of Google's `google-analytics-mcp` (`origin` = the private `DDRE-Global` fork; upstream is
`googleanalytics/google-analytics-mcp`). Two things live here, and they change on different schedules:
the **MCP server** (tracks upstream) and **Noga's GA reporting assets** (ours only).

## The subject

GA4 property **Noga**, id `535285702`, measurement ID `G-Q8KR9ZPHVS`, in account **Tech DDRE**
(`accounts/393183521`; verified 2026-11-02). The measured app is the DDRE admin frontend (`web`); its
prod host is `noga.global`, and `www.noga.global` is a separate host value.
Property ids are reference data: re-resolve with `get_account_summaries` if anything about the target
looks wrong.

## Package map (`analytics_mcp/`)

| Path | Owns |
| --- | --- |
| `server.py` | stdio entrypoint, argparse (`--allow-write` since upstream v0.8.0), tool registration |
| `coordinator.py` | turns the tool objects into MCP tools; the layer that broke on every MCP SDK rename |
| `tools/client.py` | builds the GA Data / Admin clients from ADC (+ `GOOGLE_PROJECT_ID` as quota project) |
| `tools/reporting/` | read tools: `run_report`, `run_realtime_report`, `run_funnel_report`, `run_conversions_report` |
| `tools/admin/` | account/property metadata tools (`get_account_summaries`, `get_property_details`, annotations) |
| `prompts/` | report runbooks, prose-only (the weekly leadership report gather steps) |
| `reports/` | generated HTML deliverables + the dark-design template; **not** a source of record |
| `examples/` | reference funnel definitions (`noga-web-funnels.json`) |

## Non-negotiables

1. **Every** user-facing KPI query uses the mandatory prod filter (`hostName` exact `noga.global`,
   excluding `/moneypenny`): the exact expression, the pollution layers it removes and the sanity bands
   are in `mem:data-quality`. Unfiltered numbers here are always wrong, not merely slightly off.
2. **No credential is ever committed or pushed**: `secret.json` (a live OAuth client) is gitignored and
   untracked; if it appears in `git status`, stop and fix the ignore first. Details: `mem:tool-setup`.
3. Numbers are never invented: a failed query is reported as unavailable, with the reason
   (`mem:report-conventions`).
4. Zero page views means **either** no usage **or** no data: confirm the feature is deployed before
   claiming non-adoption, and check the route's instrumentation before claiming a feature is unmeasured
   (`mem:adoption-baseline`, `mem:instrumentation`).

## Which memory to read

Per-job entry point: `mem:route-map`. Machine plumbing (venv, harness registration, Google auth):
`mem:tool-setup`.
