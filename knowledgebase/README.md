# Knowledgebase

Living log of gotchas, decisions, and learned knowledge from working with this repo and the
Noga GA4 reporting setup. **Read the relevant file before the task it covers; add new learnings
here when you hit something non-obvious** (see AGENTS.md §Knowledgebase).

Keep entries dated, concrete, and reproducible (commands, exact error text, exact filter JSON).
If an entry goes stale (e.g. a gap gets fixed in the web app), update it rather than deleting —
leave a `~~strikethrough~~` note or a "RESOLVED <date>" line.

## Index

| File | Covers |
| --- | --- |
| [mcp-setup.md](mcp-setup.md) | Installing/running the MCP server, pi config, auth failures & re-login flow |
| [ga-query-gotchas.md](ga-query-gotchas.md) | Data API / MCP tool argument gotchas, filter expression shapes, field-name renames |
| [data-quality.md](data-quality.md) | Traffic pollution (localhost, /moneypenny, www), mandatory prod filter, sanity checks |
| [instrumentation.md](instrumentation.md) | GA events in DDRE-Global/web, registered dims/metrics, known gaps & fixes |
| [local-repos.md](local-repos.md) | What the local DDRE repos contain, feature map, audit workflow findings |
| [report-conventions.md](report-conventions.md) | Report formats, templates, naming, commit/push rules |

## Maintenance rules

1. **Log it when:** you hit an error that needed investigation, discovered a data-quality trap,
   changed a convention, or learned how the product works. One-line entries are fine.
2. **Format:** `### <date> — <topic>` + what happened, root cause, fix/workaround.
3. **Date-stamp** entries (YYYY-MM-DD) so staleness is obvious.
4. **Cross-reference** AGENTS.md for the rules, this folder for the why/details.
5. Commit & push knowledgebase changes together with the work that produced them.
