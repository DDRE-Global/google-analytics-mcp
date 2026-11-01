# google-analytics-mcp (DDRE working repo)

Local fork of Google's `google-analytics-mcp` (`origin`: the private `DDRE-Global` fork, upstream:
`googleanalytics/google-analytics-mcp`). It holds the MCP server source **and** Noga's GA reporting
assets, used to produce the production usage and feature-health reports.

Before doing anything, read `mem:core` (what this repo is and the two absolute rules) and
`mem:route-map` (which memory to read for the job at hand).

# Serena

Knowledge lives in **Serena memories** (`.serena/memories/`, git-tracked) and nowhere else. There is no
`docs/` and no `knowledgebase/` directory, and one must not be reintroduced: a second store drifts
silently, and a dead path reads exactly like a live one. Rules for writing memories:
`mem:memory_maintenance`.

Serena is a machine-global MCP server shared by every DDRE repo; no harness starts it by itself. At the
start of a session:

1. **Make sure it is up.** `bash scripts/serena-up.sh` in any DDRE clone that has it (`web`, `cms`,
   `sprift-gateway`) probes and starts both processes. From this repo, or if nothing is listening:
   `serena start-mcp-server --project-from-cwd` (MCP on `127.0.0.1:9121`) and, if symbol-level tools are
   wanted, `serena start-project-server` (machine-global, one instance, port `24225`). A bare probe of the
   MCP endpoint answers `400` on a healthy server, and the project server answers `401` without its auth
   header, so a failed probe proves nothing.
2. **Read the manual once per session** by calling that server's `initial_instructions` tool (name
   spelling depends on the harness). It carries the operating modes, the active project, and the memory
   list. Skip it only if it is already in the conversation.
3. **Verify the active project.** It must be `google-analytics-mcp`. The active project is *shared server
   state*, so a session opened after work in `web` finds `web` active: re-assert it with `activate_project`
   (`{"project": "google-analytics-mcp"}`) instead of flipping it back and forth, and read sibling repos
   with `query_project`. This repo has no project server of its own and no worktrees.

Newly registered projects are read once at startup: adding this repo to the registry needs **both**
Serena processes restarted before `query_project` or symbol tools will see it.

If no MCP connection can be established, the same memories are plain markdown under
`.serena/memories/` and can be read from disk. Say so when you fall back, because symbol-level tools
(references, safe renames, diagnostics) are not replaced by reading files.

# Layout

| Path | Contents |
| --- | --- |
| `analytics_mcp/` | server (`server.py`), tool wiring (`coordinator.py`), GA clients and tools (`tools/`) |
| `prompts/` | report runbooks (the weekly leadership report gather steps) |
| `reports/` | generated HTML reports + the dark-design leadership template |
| `examples/` | reference funnel definitions |
| `.serena/memories/` | **the knowledge store** (auth, data quality, query traps, instrumentation, conventions) |

GA4 property **Noga `535285702`**, measurement `G-Q8KR9ZPHVS`, prod host `noga.global`.

# Running the server

`.venv` at the repo root (uv), started as `.venv/bin/python -m analytics_mcp.server`, registered with the
harness as `analytics-mcp`. `python -m` runs **this checkout's** code, so after a pull that touches
`pyproject.toml`, sync the environment (`uv pip install --python .venv/bin/python -r pyproject.toml`)
before restarting it, then verify with `--help`. The dependency floor and the historical pin saga, the
read-only `--allow-write` gate, the harness config location, and the Google re-auth recipe (its refresh
token dies roughly monthly) are all in `mem:tool-setup`.

# Absolute rules

1. **Every** user-facing KPI query carries the mandatory prod filter. The exact expression, what it
   removes and the sanity bands: `mem:data-quality`. An unfiltered number on this property is not
   conservative, it is wrong: ~93% of the raw traffic is dev and E2E.
2. **No credential is ever committed, pushed or printed.** `secret.json` is a live OAuth client; it is
   gitignored and untracked (verified: never in history). So are ADC files and any service-account JSON.
   If one appears in `git status`, fix the ignore before committing anything else.
3. Never invent a number, and never report a page-view-only feature as unused without working through
   `mem:adoption-baseline` and `mem:instrumentation`.
4. Report output goes to `reports/`, self-contained, hero stating period, property and exact filter
   (`mem:report-conventions`).
