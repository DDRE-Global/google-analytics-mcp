# MCP server setup & auth

### 2026-09-28 — Server installed in `.venv` (uv), registered in pi

- The repo's `.venv` was **empty** (no pip, no packages). The old Cursor config
  (`~/.cursor/mcp.json`) ran it with system `python3` + `PYTHONPATH` — but no Python env on the
  box had the deps, so that config was broken too.
- Fix: `uv pip install analytics-mcp==0.6.0 --python .venv/bin/python` (installed from PyPI,
  not editable — editable build fails).
- **Pinned `mcp==1.24.0`** — MCP SDK 2.x renamed `inputSchema` → `input_schema` and breaks
  `analytics_mcp/coordinator.py` at import time with
  `AttributeError: 'Tool' object has no attribute 'inputSchema'`.
- pi config entry lives in `~/.config/mcp/mcp.json` (server name `analytics-mcp`):
  command `.venv/bin/python`, args `-m analytics_mcp.server`, cwd = repo root,
  env `GOOGLE_APPLICATION_CREDENTIALS=/home/kevin/.config/gcloud/application_default_credentials.json`,
  `GOOGLE_PROJECT_ID=serious-bearing-499201-q3`.
- After config changes the pi MCP server must be restarted (`/mcp reload` or reconnect cycle) —
  **a running server caches Google credentials at startup**.

### 2026-09-28 — `invalid_grant` on GA calls = stale ADC refresh token

- Symptom: MCP tools list fine (no network needed), but any GA call returns
  `503 Getting metadata from plugin failed ... ('invalid_grant: Bad Request')`.
- Root cause: the desktop-OAuth refresh token in gcloud ADC was revoked (Google revokes stale
  desktop-app tokens; the previous reports worked in July, broke by late September).
- Fix (needs a browser on the box):
  ```shell
  gcloud auth application-default login \
    --scopes https://www.googleapis.com/auth/analytics.readonly,https://www.googleapis.com/auth/cloud-platform \
    --client-id-file=secret.json
  ```
  Note: `--no-launch-browser` is rejected with `--client-id-file` (use `--no-browser`, which
  triggers the awkward remote-bootstrap flow — prefer the default browser flow; the user opens
  the printed URL, redirect goes to `localhost:8085` on this box).
- After re-login, **restart the MCP server** (it had cached the dead credentials; `connect`
  cycle in the pi mcp proxy does the restart).

### 2026-09-28 — `secret.json` is a live OAuth client — never commit/push

- It's committed-locally-untracked now and `.gitignore`d. Repo remote is
  `git@github.com:DDRE-Global/google-analytics-mcp.git` (private org fork of the Google upstream).
