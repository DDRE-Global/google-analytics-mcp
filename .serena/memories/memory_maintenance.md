# Memory maintenance

The memories under `.serena/memories/` are this repo's **only** agent knowledge store. There is no
`docs/` and no `knowledgebase/` tree, and there must not be one: a second store has its own drift
schedule and nobody notices which copy went stale. If a fact about the data, the auth, the traps or the
conventions is not a memory, it does not exist for the next session.

## What goes where

| Fact type | Home |
| --- | --- |
| Repo identity, package map, hard rules | `mem:core` |
| Which memory to read for which job | `mem:route-map` |
| Endpoints, hosts, ports, venv, harness config paths, key material, tool argument spellings | `mem:tool-setup` (test: delete it and you lose plumbing, not knowledge) |
| Data truths (pollution, the prod filter, sanity bands) | `mem:data-quality` |
| Query construction traps | `mem:ga-query-gotchas` |
| Measured usage, with the date and the re-check query | `mem:adoption-baseline` |
| What the app instruments | `mem:instrumentation` |
| Output format and commit rules | `mem:report-conventions` |
| Sibling repos, audit order | `mem:local-repos` |

One fact, one home: every other mention is a pointer naming the memory and the heading, never a
restatement. Bodies stay tool-neutral; the client-specific glue goes to `mem:tool-setup`. Rules and fast-
changing reference data are split (`mem:core` rules versus the dated tables in `mem:adoption-baseline` and
`mem:instrumentation`).

## Adding an entry

Log it when you hit an error that cost investigation, found a data trap, changed a convention, or learned
how the data behaves. Format: `### YYYY-MM-DD — <topic>` inside the matching memory, with symptom (exact
error text), root cause, fix, and a re-check command where one exists. Dated claims carry the date and the
command that re-verifies them; a stale date is a feature, because it forces the re-check. Mark superseded
entries `RESOLVED <date>` with the correction instead of deleting them, when the old belief is still
tempting (example: the `match_type` correction in `mem:ga-query-gotchas`).

Commit memory changes together with the work that produced them. Never cite a directory of generated
output, a screenshot or a dated artefact as the source of record: cite a committed file, a command, or the
query that reproduces it.

## Checks

- `serena memories check` after any rename, split or delete; renaming goes through the rename tool so
  `mem:` references are rewritten, never by hand.
- Walk the `mem:` graph from `mem:core` and confirm every reference resolves, no memory is orphaned, and
  every cited path exists.
- Grep the library for personal absolute paths (`/home/<user>/`, `~/`, drive letters) before committing.
