# Bulk Operations on Linear Tickets

Operational mechanics for touching many Linear issues at once. Separated from `SKILL.md` because these are mechanics, not the skill itself.

## Querying tickets

Use the `linear-server` MCP tools; filter server-side, never page through everything client-side:

- `list_issues` with `project` / `parentId` / `state` / `assignee` / `query` — e.g. all children of an execution epic: `list_issues(parentId: "CMD-XXXX")`; all ready work: `list_issues(project: "...", state: "coding queue")`.
- `get_issue` for the full description + relations + attachments (list output truncates descriptions).
- Large result sets overflow the context — push the sweep into a subagent that returns the summary table (id · state · title), not raw JSON.

## Bulk state flips

Each flip is one `save_issue(id, state)` call. For N tickets:

- **A handful** — issue the calls directly, in parallel.
- **Many, or with per-ticket edits** — fan out to subagents, each with an explicit list of ids and the exact target state name (use the real workflow state names — `coding queue`, `in code review`, `qa testing` — not paraphrases).
- Bulk *ratification* (`Triage` → `coding queue`) is the human's call: do it only with explicit permission, and report exactly which ids moved.

## Supersession sweeps

When a re-plan cancels or absorbs many tickets, every canceled ticket gets — in the same pass:

1. `save_issue(id, state: "canceled")`
2. a `save_comment` naming the survivor, the deciding evidence, and (if code was built) the archive tag + removal commit
3. the parent execution ticket's DAG updated to the collapsed shape

Never flip states without the pointer comments — a bare `canceled` reads as abandoned work.

## Description edits at scale

`save_issue(id, description)` **replaces the whole description** — there is no patch operation. Always `get_issue` first, edit the fetched text, and resend it in full; never reconstruct a description from memory. For sweeps (e.g. a rename across every ticket in a milestone), give each subagent the fetch-edit-resend loop and the exact find/replace contract.

## Rule

Reaching for bulk operations is itself a signal the plan is mid-rework. A dozen cancellations should land as **one coherent restructure with one explanation** — the rationale on the execution ticket and pointer comments on each child — not N silent state changes.
