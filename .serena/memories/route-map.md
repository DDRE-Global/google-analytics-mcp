# Route map: which memory, in what order

| Job | Read |
| --- | --- |
| Run the server / it fails to start / GA calls return `invalid_grant` | `mem:tool-setup` (venv floors, then the Google re-auth recipe, then the restart rule) |
| Write any GA query | `mem:ga-query-gotchas` (filter shapes, field names, aggregation traps) |
| Report a user KPI (users, sessions, adoption) | `mem:data-quality` first: the mandatory prod filter is not optional and the sanity band tells you whether you applied it |
| Say anything about a feature being used / not used | `mem:adoption-baseline` for the measured route reality, then `mem:instrumentation` before claiming a feature is unmeasured |
| Explain what events the app sends | `mem:instrumentation` (audited snapshot; re-verify against the `web` repo before quoting) |
| Write or refresh an HTML report | `mem:report-conventions` (naming, design system, hero must state the filter) |
| Ground a report in the app's code | `mem:local-repos` (audit order for the sibling DDRE repos) |
| Add knowledge you just earned | `mem:memory_maintenance` |

Repo map and the non-negotiables: `mem:core`.
