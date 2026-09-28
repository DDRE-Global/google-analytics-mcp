# Local DDRE repos & audit workflow

Local checkouts live under `/home/kevin/@noga-1/` (org: `DDRE-Global` on GitHub).

### 2026-09-28 — Repo → purpose map (as discovered)

| Path | Repo | Notes |
| --- | --- | --- |
| `@noga-1/web` | DDRE-Global/web | Next.js 16 BFF frontend for the Noga admin panel (this report's subject) |
| `@noga-1/cms` | DDRE-Global/cms | Strapi CMS backend (`/api/frontend/*` BFF contract — derive payloads from source, not specs) |
| `@noga-1/google-analytics-mcp` | this repo | GA MCP + report assets |
| `@noga-1/mcp`, `data-pipeline`, `ddre-bridge`, `ddre-mcpb`, `external-dataset`, `migration`, `real-estate-admin*`, `web`, `run-noga.sh` | various | Azure MCP services, scrapers, worktrees |

### 2026-09-28 — DDRE-Global/web feature map (from `AGENTS.md` → Serena `mem:core`)

- Stack: Next.js 16 App Router + React 19, Tailwind 4, Untitled UI; BFF `/api/*` → services → Strapi.
- Auth: magic-link/password → Strapi JWT → `NOGA_AUTH` HttpOnly cookie (BFF pattern).
- Feature routes (16): `/home`, `/document` (PandaDoc), `/monday` (onboarding checklist),
  `/building`, `/property`, `/listing` (WIP), `/contact`, `/aml`, `/search` (requirements + Brief
  Hub), `/offer`, `/lead`, `/referral`, `/keys`, `/viewing`, `/appointment`, `/profile`, `/settings`.
- Public: `/login`, `/sso`, `/moneypenny` (public page — excluded from prod KPIs), 401/403.
- **2026-09-28 prod usage reality check:** only home/document/AML/monday + auth are used in
  prod; 10 routes had zero prod page views in 28 days (see features report + data-quality.md).

### 2026-09-28 — Audit workflow gotchas

- The web repo's `AGENTS.md` defers everything to **Serena memories** — check the Serena server
  is up first: `curl -s -o /dev/null -w "%{http_code}" -X POST http://127.0.0.1:9121/mcp`
  → `400` = up, `000` = down (start with `npm run serena` in the repo, backgrounded).
- Serena tool arg quirk: `read_memory` takes **`memory_name`** (not `name`).
- `initial_instructions` returns the active project + memory list; read `mem:core` for the
  source map/invariants, `mem:monday-integration`, `mem:audit-trails`, `mem:cms-frontend-api`
  for feature specifics.
- "This is NOT the Next.js you know" block in web's AGENTS.md: Next 16 docs live in
  `node_modules/next/dist/docs/` — don't trust training-data Next conventions there.
- GA instrumentation source for web: `src/lib/ga.ts` + `grep -rn "sendGAEvent" src`
  (see instrumentation.md for the audited result).
