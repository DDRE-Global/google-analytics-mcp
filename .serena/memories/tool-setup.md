# Tool setup: venv, harness registration, Google auth

Machine plumbing only. The GA query rules live in `mem:ga-query-gotchas`, the filter in
`mem:data-quality`.

## Running the server

- Python env: `.venv` at the repo root (created with `uv`). The harness starts it as
  `.venv/bin/python -m analytics_mcp.server` with cwd = repo root.
- **`-m` runs the repo's `analytics_mcp/` package, not the copy in `site-packages`** (cwd wins on
  `sys.path`), so the checkout's commit must be compatible with the installed dependencies even though
  the package is also pip-installed there.
- Read-only by default. Write tools (`create_property_annotation`, and the other admin mutators upstream
  added in v0.8.0) need `--allow-write` **and** the `analytics.edit` scope; this setup authenticates with
  `analytics.readonly`, so they refuse either way. Keep the harness entry write-free.
- Verified 2026-11-02 against the running harness server (v0.8.0 code, no flag): nine read tools are
  exposed, and `create_property_annotation` is **not registered at all** without `--allow-write`
  (`coordinator.create_server` only registers it inside `if allow_write:`). A missing write tool is that
  gate working, not a broken install.

### 2026-11-02 — Dependency direction flipped with upstream v0.8.0 (RESOLVED: the old `mcp==1.24.0` pin)

- Upstream v0.8.0 renamed its use of the SDK from `inputSchema` to `input_schema` and raised
  `pyproject.toml` to `mcp>=2.3.0,<3`, `google-adk>=2.11.0`, `google-analytics-data==0.23.3`,
  `google-analytics-admin==0.30.2`, `google-auth~=2.62`.
- With the old pin the server now dies at import: `AttributeError: 'Tool' object has no attribute
  'input_schema'`. The mirror-image failure (`'Tool' object has no attribute 'inputSchema'`) is what the
  old pin existed for. Symptom of a mismatch either way: **the tool list still works from the harness
  cache while a freshly started server crashes**, so a crash shows up on the next reconnect, not on pull.
- Fix after any pull that touches `pyproject.toml`: `uv pip install --python .venv/bin/python -r
  pyproject.toml`, then verify with `./.venv/bin/python -m analytics_mcp.server --help` (imports
  everything and exits without serving).
- Noise to expect on every start, harmless: google library warnings about Python 3.10 being unsupported
  and `[EXPERIMENTAL]` google-adk feature flags.

## Harness registration

- Pi reads `~/.pi/agent/mcp.json` (verified 2026-11-02). `~/.config/mcp/mcp.json` also exists with an
  `analytics-mcp` entry and is **not** what pi loads: check both before concluding a config edit was
  ignored. Server name `analytics-mcp`, env `GOOGLE_APPLICATION_CREDENTIALS` (the gcloud ADC file under
  gcloud's config dir) and `GOOGLE_PROJECT_ID` (quota project only, it does not select the GA property).

## Calling the tools from a script

- Harness tool names are normalised from the server name: the entry `analytics-mcp` is reached as
  `mcp__analytics_mcp__<tool>` from a codemode-style script (verified 2026-11-02). Older harnesses used a
  `<server>_<tool>` spelling, so resolve the name from the tool list rather than assuming it.
- Errors come back **inside** the text content as `{"error": "..."}` with the call itself reported as
  successful: unwrap the payload before concluding anything from an empty-looking result.
- Large reports get truncated by the harness and spilled to a temp file that holds the whole MCP result
  object; parse it, do not eyeball the truncated head.

## Google authentication

Credentials are gcloud **ADC** (`type: authorized_user`, a desktop OAuth client). The client JSON is
`secret.json` at the repo root: **live, gitignored, untracked (verified: never committed); never commit
or push it**, and never print its `client_id` / `client_secret` / `refresh_token` into a chat or a log.

### 2026-09-28 and 2026-11-02 — `invalid_grant` recurs roughly monthly; treat it as routine

- Symptom: the tool list loads fine (no network needed), every GA call answers
  `503 Getting metadata from plugin failed ... ('invalid_grant: Bad Request')`.
- Root cause: Google revokes stale desktop-app refresh tokens. Both incidents were ~1 month after the
  previous login, with no config change.
- Re-auth (needs a browser on the box; WSL2 hands off to the Windows browser via `wslview`):
  ```shell
  gcloud auth application-default login \
    --scopes https://www.googleapis.com/auth/analytics.readonly,https://www.googleapis.com/auth/cloud-platform \
    --client-id-file=secret.json
  ```
  The callback listener binds `127.0.0.1:8085`. `--no-launch-browser` is rejected together with
  `--client-id-file`; `--no-browser` triggers the awkward remote-bootstrap flow, so prefer the default.
- **After re-login the MCP server must be restarted**: it caches Google credentials at startup, so the
  harness keeps answering `invalid_grant` from the old process even though the ADC file is healthy.
- Diagnose before assuming (and without leaking the secret): POST the ADC's refresh-token grant to
  `https://oauth2.googleapis.com/token` from a script that prints only the status and error code.
  `400 invalid_grant` = re-login; a token success plus a failing harness call = the cached-credentials
  case above. A fast end-to-end check that bypasses the harness entirely: in the venv,
  `google.auth.default(scopes=[...analytics.readonly])` → refresh → one call to
  `analyticsadmin.googleapis.com/v1beta/accountSummaries`. That separates broken auth from broken
  plumbing, and it also proves the ADC can see the property (`properties/535285702`).

## Cross-repo reads (auditing `web` for a feature report)

The DDRE repos are Serena projects on a machine-global server: the MCP endpoint on port 9121 plus a
ProjectServer on 24225 that serves the symbol-level tools. Confirm both answer before a symbolic read of
`web` or `cms`; a `read_file` that works while `find_symbol` refuses means 24225 is down, not that the repo
is unindexed. The active project is shared server state, so query siblings rather than switching it, and
re-assert this project afterwards. Startup, trust and the heartbeat auth header belong to those repos'
own `AGENTS.md`; the audit order is `mem:local-repos`.

