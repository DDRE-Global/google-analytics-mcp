# GA instrumentation of the app (`web`)

What the measured product actually sends to this GA property. The app repo is `web`; its own memory
library owns product rules and the route map. **This is a dated audit, not a live view: re-verify before
quoting it** against `src/lib/ga.ts` (the `sendGAEvent` helpers) plus a repo-wide search for `sendGAEvent`
(ad-hoc calls in contexts and hooks), then against the registered parameters in the property.

### 2026-09-28 — events arriving in GA

| Area | Events |
| --- | --- |
| AML | `aml_list_viewed`, `aml_create_started` (`source`), `aml_create_step_completed` (`step`, `request_type`, `variant`), `aml_request_submitted` (`request_type`, `variant`, `is_draft`, `document_count`), `aml_request_updated`, `aml_status_viewed`, `aml_search` (`query_length`), `aml_metric_filter_selected` (`metric_id`) |
| Documents | `document_sent` — final send carries `templateId`, draft send sends no params |
| Auth | `logout`; `login` (`method`) is sent in code but **does not arrive** (fired before gtag initialises) |

### 2026-09-28 — registered in the property

- Groupable **custom dimensions**: `customEvent:request_type`, `:variant`, `:is_draft`, `:waiting_on`,
  `:aml_complete`, `:source`, `:metric_id`, `:aml_document_id`.
- **Custom metrics only** (not groupable): `customEvent:step`, `:document_count`, `:query_length`.
- Sent but unregistered, therefore dropped: `document_sent.templateId`, `login.method`.

Everything else is page-view data only: `/monday`, `/search`, `/offer`, `/viewing`, `/lead`, `/referral`,
`/keys`, `/appointment`, `/building`, `/property`, `/listing`, `/contact`, `/profile`, `/settings`.

### Known gaps, as logged (unresolved as of 2026-11-02 unless the app repo says otherwise)

1. `login` never lands → fire after gtag init or re-send on the first authenticated render; register
   `method`. Without it, auth funnels and "how often do people re-enter the magic link" are unanswerable.
2. `step` should be registered as a dimension too, for per-step AML drop-off (see the silent
   `metric_filter` failure in `mem:ga-query-gotchas`).
3. `templateId` unregistered → document sends cannot be split by template; a `kind: draft|final`
   parameter is also missing.
4. `is_draft` always false → the draft-save path likely sends nothing.

### Reading rules

- A zero on a page-view-only feature means **page views are all you get**: no create/status events exist,
  so "no events" is not evidence of "no usage". Route views come from `mem:adoption-baseline` and a fresh
  report.
- Alarm metric worth re-checking each report: AML completion. In the Sep window 1 request reached
  `aml_completed` against 22 submitting users, and `waiting_on` topped out at `compliance_team` (14
  users).
- `/login` at ~4.3 views per user is the re-entry signal to watch while `login` events stay missing.
