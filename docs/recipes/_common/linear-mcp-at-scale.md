# How we operate Linear through MCP at scale

When a session reads or writes more than a handful of Linear issues — sweeps, audits, description edits. (State-flip and supersession mechanics: [`skills/ticketing/BULK_OPS.md`](../../../skills/ticketing/BULK_OPS.md).)

## Steps

1. Filter server-side (`project` / `parentId` / `state` / `query` on `list_issues`); never page everything and filter client-side.
2. Expect big results to overflow: the tool saves them to a file and returns the path — `grep` it for targeted lookups, python-slice it for full reads, or hand it to a subagent so the raw JSON stays out of the lead context.
3. `list_issues` truncates descriptions — any ticket you'll reason about gets a `get_issue`.
4. To edit a description: `get_issue` → edit the fetched text → `save_issue` with the **full** new text. There is no patch operation.
5. Use the workspace's exact state names (`in code review`, `qa testing`, `coding queue`) — paraphrases fail or mis-route.

## The chip palette (what renders rich, and how to produce it via API)

| Reference | Write it as | Renders as | Carries status? |
|---|---|---|---|
| issue | plain `CMD-XXXX` in prose | inline issue mention (state icon + title) | yes — workflow state, live |
| issue, load-bearing | its URL standalone on its own line | larger embed card | yes |
| person (handoff) | `@displayName` | person chip + notification | — |
| PR / commit | attachment on the issue (`links` param / GitHub integration) — never only a prose mention | attachment chip | yes — draft/open/merged |
| project | project URL | project chip | yes — progress/health |

URLs in prose (bare or markdown-linked) render as plain links and never chip; identifiers inside code spans, code blocks, or mermaid don't convert. Default: inline identifiers everywhere, standalone-line embeds only for the one or two load-bearing pointers (scope ↔ execution), attachments for every PR↔ticket edge.

## Gotchas (each earned)

- `save_issue(description)` replaces the whole description — *earned by:* every scope/execution update this repo's methodology rewrite was based on required fetch-edit-resend; reconstructing from memory would have silently dropped sections.
- Linear's serializer mangles multi-word code spans — `` `(sub, conn) → (user_uuid, tenant)` `` comes back with split backticks — *earned by:* the M2a/M3a scope updates, which needed a visual pass after saving; keep code spans to single tokens where the rendering matters.
- A 50-issue project listing overflows the context window in one call — *earned by:* a project sweep whose result landed as a 75KB file; the `grep`-the-saved-file move recovered it without re-querying.
- Relations are append-only via `blockedBy`/`blocks` params — removing an edge needs the explicit `removeBlockedBy`/`removeBlocks`, not a re-save — *earned by:* the tool schema's own semantics; re-saving without them silently keeps stale edges.
- **Status-bearing issue chips come from plain `CMD-XXXX` identifiers in prose** — Linear auto-converts bare identifiers into true issue mentions (icon + state + title); URLs, bare or markdown-linked, are stored as plain links and never chip. Identifiers inside code spans/blocks/mermaid don't convert — *earned by:* a link-style sweep that discovered the conversion by accident (`CMD-360`, `CMD-420` chipped; every URL stayed a link).
- **`save_issue` can silently no-op** — the response echoes the *old* body with an unchanged `updatedAt` and no error. **Verify `updatedAt` advanced after every description save.** One retry usually lands; if the same entity no-ops repeatedly (write-throttled) while sibling writes succeed, stop hammering and finish later or in the UI — *earned by:* one save that took on retry and three consecutive no-ops on a single execution ticket minutes later.

## Divergence notes
*(append-only — when a later run diverges, record what and why)*
