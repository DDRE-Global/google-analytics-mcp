# GA instrumentation of DDRE-Global/web

Source of truth: `src/lib/ga.ts` (helpers) + ad-hoc `sendGAEvent` calls (auth-context, hooks).
**Re-audit before each report** — this snapshot is dated.

### 2026-09-28 — Events actually firing (verified against GA)

| Feature | Events (params) | Notes |
| --- | --- | --- |
| AML | `aml_list_viewed` | |
| | `aml_create_started` (`source`) | all "direct" in Sep window |
| | `aml_create_step_completed` (`step`, `request_type`, `variant`) | 229 ev / 31 u, avg step 1.81 |
| | `aml_request_submitted` (`request_type`, `variant`, `is_draft`, `document_count`) | 55 ev / 22 u, 0 drafts |
| | `aml_request_updated` (`request_type`, `aml_document_id`) | 7 / 6 |
| | `aml_status_viewed` (`request_type`, `aml_complete`, `waiting_on`, `aml_document_id`, `source`) | 57 / 18 |
| | `aml_search` (`query_length`) | 13 / 6, avg len 18.6 |
| | `aml_metric_filter_selected` (`metric_id`) | registered, low volume |
| Documents | `document_sent` (final send: `templateId`; draft send: **no params**) | 68 ev / 28 u |
| Auth | `login` (`method`: password \| jwt) | ⚠ **NOT arriving in GA** — sent in `src/contexts/auth-context.tsx` before gtag init (presumed) |
| | `logout` | 8 / 2 |

### 2026-09-28 — Registered in the GA property (groupable vs not)

- **Custom dimensions** (usable in `dimensions`): `customEvent:request_type`, `:variant`,
  `:is_draft`, `:waiting_on`, `:aml_complete`, `:source`, `:metric_id`, `:aml_document_id`
- **Custom metrics only** (NOT groupable as dimensions): `customEvent:step`,
  `customEvent:document_count`, `customEvent:query_length`
- **Sent in code but unregistered** (dropped by GA, can't group): `document_sent`'s
  `templateId`, `login`'s `method`.

### 2026-09-28 — Known gaps & recommended fixes (product/engineering follow-ups)

1. `login` events missing in GA → fire after gtag init or re-send on first authenticated page;
   register `method` as dimension.
2. Register `customEvent:step` as a **dimension** too → per-step AML drop-off charts.
3. Register `customEvent:templateId` (or map to a small enum) → document sends by template/group.
4. Add `kind: draft|final` param to `document_sent` → currently indistinguishable.
5. Zero events on `/search`, `/offer`, `/viewing`, `/lead`, `/monday` tasks, `/keys`, `/referral`,
   `/appointment` → add create/status events before these ship to prod.
6. `is_draft` always false → verify draft-save path fires an event (add `aml_draft_saved`).

### 2026-09-28 — Product numbers worth re-checking weekly

- AML completion is the alarm metric: in the Sep window only **1** request reached
  `aml_completed` (of 22 submitting users); `waiting_on` top = `compliance_team` (14 u).
- `/login` ≈ 4.3 views/user (92 u) — SSO token re-entry hypothesis unconfirmed (login events missing).
