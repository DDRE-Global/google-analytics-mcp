# Sibling DDRE repos and the audit order

DDRE-Global checkouts sit side by side in one parent directory per machine. Name the repo and resolve
the path locally; never write a personal absolute path into a memory, report or commit.

| Repo | What it is |
| --- | --- |
| `web` | DDRE-Global/web: Next.js 16 App Router + React 19 admin frontend, Tailwind 4 + Untitled UI, BFF `/api/*` → services → Strapi. **The subject of every report here.** |
| `cms` | DDRE-Global/cms: Strapi backend behind `/api/frontend/*`. Derive payloads from that source, never from a static spec (it goes stale). |
| `google-analytics-mcp` | this repo: the GA MCP server plus the report assets |
| `mcp`, `data-pipeline`, `ddre-bridge`, `ddre-mcpb`, `external-dataset`, `migration` | Azure MCP services, scrapers, migration tooling; not reporting inputs |

Each of these repos carries its own agent instructions; `web` and `cms` keep their knowledge in Serena
memories rather than a docs tree, so read their `AGENTS.md` and then their memories before assuming
anything about their routes or events.

## Feature map of `web` (audited 2026-09-28; re-check against the repo)

- Feature routes: `/home`, `/document`, `/monday`, `/building`, `/property`, `/listing` (WIP), `/contact`,
  `/aml`, `/search` (requirements + Brief Hub), `/offer`, `/lead`, `/referral`, `/keys`, `/viewing`,
  `/appointment`, `/profile`, `/settings`.
- Public: `/login`, `/sso`, `/moneypenny` (public page, excluded from prod KPIs: `mem:data-quality`), 401/403.
- Auth: magic link / password → Strapi JWT → `NOGA_AUTH` HttpOnly cookie.
- **Gated in production:** `/building`, `/property`, `/listing`, `/contact`, `/lead`, `/referral`,
  `/appointment`, `/keys`, `/search`, `/offer`, `/client` answer 403 on the prod host unless the deploy is
  a preview, so a zero report row for them is expected rather than informative (`mem:adoption-baseline`).
  The rest of the list (`/home`, `/document`, `/monday`, `/aml`, `/profile`, `/settings`) is live.

## Audit order before any feature report

1. Open the app repo and read its `AGENTS.md`, then the memories it points at (source map, then the
   feature-specific ones). Its route map beats any guess about which features exist.
2. Read the GA instrumentation source in that repo (`src/lib/ga.ts` and every `sendGAEvent` call site) to
   learn which events exist and with which parameters.
3. Cross-check the property: list its custom dimensions and metrics. Only registered parameters are
   groupable (`mem:instrumentation`).
4. Verify the traffic you are about to report is real prod traffic (`hostName` / `pageLocation` rows)
   before quoting a single number (`mem:data-quality`).
5. Map features → events → data availability, and write the gaps into the report rather than inferring
   silence: events sent but absent in GA, parameters unregistered, routes with page views only.
6. If the claim is about adoption, resolve deployed-versus-unused first (`mem:adoption-baseline`).
