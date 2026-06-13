---
name: trak
description: Use when the user wants to read or write work-tracking data in trak — the facet-based ticketing/knowledge backend on port 44107. Triggers include "file this in trak", "create a facet", "link these tickets", "show the trak tree", "what's blocked", "grep trak for X", or any operation against trak facets. Covers the canonical `synapse` invocation, the auth/underscore/envelope-encoding gotchas, the facet method reference, and edge kinds. For WRITING good ticket *content* (acceptance criteria, evidence), use the `ticketing` skill — trak is the tool that STORES it.
---

# Skill: Operate trak

trak is a Plexus RPC backend that stores work as **facets** — a universal primitive. A facet has a title, body, free-text status, owner, and a parent; facets nest arbitrarily deep, and any two can be joined by a typed edge. There is no epic/ticket/note type distinction: an epic is a facet with children, a ticket is a facet under it, a doc is a facet with prose. You categorize with `tags`/`meta_extra`, not a type field.

There is **no `trak` binary**. You talk to trak through the `synapse` CLI at `/Users/shmendez/.local/bin/synapse` (usually just `synapse` on PATH).

## When to use

Whenever the user wants to create, read, update, link, move, search, or analyze work items stored in trak — or wire an existing plan into it. If the task is about *what to write* in a ticket (contract, acceptance criteria, evidence), reach for the `ticketing` skill first, then use this skill to store the result.

## Canonical invocation

```bash
synapse -P 44107 -j trak <activation> <method> -p '{...JSON params...}'
```

- `-P 44107` — trak's port.
- **No `-t` needed.** synapse loads the access token natively from `~/.plexus/trak/defaults.json` (`defaults.cookies.access_token`). The `-t "$(cat ~/.plexus/trak/token)"` form still works as an explicit override, but it's the stale HS256-era habit — prefer native. (See auth gotcha #1 below for how to get/refresh the token.)
- `-j` — raw JSON stream output. Use it for anything you parse.
- `-p '{...}'` — params as a JSON object. **It is a global synapse option and MUST come before `trak facet <method>`** (i.e. `synapse ... -j -p '{...}' trak facet create`, NOT `... trak facet create -p '{...}'`). Placing `-p` after the method makes synapse parse the JSON as a positional subcommand and fail with `Method '<m>' cannot have subcommands at facet`. Same applies to `-P`, `-t`, `-j` — all options precede the `<backend> <path>` part.

Discovery (no token needed):

```bash
synapse -P 44107 trak            # all activations + methods, human-readable
synapse -P 44107 -s trak facet   # raw schema JSON (every method's params) for the facet activation
```

Use `-s <activation>` for schemas. There is a `schema` method in the surface but it is **not** a callable CLI subcommand (`synapse ... trak facet schema` errors with "Method 'schema' cannot have subcommands at facet").

## Three gotchas (read before your first write)

1. **Auth — what's trak's vs. not.** trak's *own* auth concern is **validation + authorization**: it validates an RS256 OIDC token (UT-W3 cutover, 2026-06-12 — no shared secret, hand-minting an HS256 token no longer works) and then its **tenant gate** decides access — **all writes require auth; reads of *tenanted* facets require auth; anonymous callers see PUBLIC facets only** (so an expired/absent token makes a tenanted `get` return `not_found`, not an auth error). The other two layers are NOT trak's: *issuing* a token is [`plexus-idp`](../plexus-idp/SKILL.md) (audience `plexus:trak`; its issuer is `http://127.0.0.1:4461` — `127.0.0.1`, not `localhost`, must match trak's `--oidc-issuer`), and *storing* it is synapse's universal `_self` store — see [`synapse-self`](../synapse-self/SKILL.md). Store the token once (`defaults.cookies.access_token` in `~/.plexus/trak/defaults.json`) and **drop `-t`** — synapse attaches it natively. When calls return `Authentication required`, the ~1 h token expired (no auto-refresh) — re-issue via plexus-idp. trak's own `identity` activation is **legacy** (its tokens are rejected by the cutover daemon).

2. **Param keys use underscores, not hyphens.** It's `parent_id`, `from_id`, `to_id`, `new_parent_id`, `parent_comment_id` — even though some prose shows hyphens. In the `-p` JSON, always underscores.

3. **`-j` output is a nested, newline-delimited envelope.** It is *not* a flat array of your event. Each line is `{"type":"data","content":{...the event...}}`, terminated by a `{"type":"done"}` line. The real event (the facet, the edge) lives under `content`, keyed by its own `type` (`facet_created`, `facet_summary`, `link_created`, `error`, …). To pull the created facet's id:

   ```bash
   synapse -P 44107 -t "$(cat ~/.plexus/trak/token)" -j -p '{"title":"X"}' trak facet create \
     | jq -r 'select(.type=="data") | .content.facet.id'
   ```

   Caveat: in the live instance `content` is a JSON **object** (use `.content.facet.id`). Some builds/versions surface `content` as a JSON-encoded **string** that must be re-parsed (`.content | fromjson | .facet.id`). If `.content.facet.id` returns null, switch to the `fromjson` form. Errors arrive as a `content` event with `"type":"error"`, a `message`, and usually a `code` (`not_found`, `forbidden`, `invalid_kind`, `unauthenticated`, `invalid_input`).

## facet method quick reference

`facet` is the core activation. Required params in **bold**; the rest are optional.

| Method | Required / key params | Purpose |
|--------|-----------------------|---------|
| `create` | **`title`**; `body`, `status` (default `open`), `parent_id`, `priority`, `tags` (array), `meta_extra` (object) | Make a facet. No `facet_type` — use `tags`/`meta_extra`. `owner`/`tenant` come from auth |
| `get` | **`id`** | Fetch one facet (tenant-scoped; foreign tenant → `not_found`) |
| `update` | **`id`**; `title`, `body`, `status`, `priority`, `tags`, `meta_extra` | Omit a field = unchanged. `tags:[]` clears tags. `meta_extra` shallow-merges (`null` value deletes a key). **No `parent_id` here** — use `move_to` |
| `delete` | **`id`** | Delete (tenant-scoped) |
| `move_to` | **`id`**; `new_parent_id` (omit → root) | Reparent. Caller must own both endpoints |
| `list` | `parent_id` (omit → roots); `tags`, `tags_all`, `priority` | List direct children, with optional filters |
| `tree` | **`id`** | Recursively walk the subtree, depth-annotated |
| `link` | **`from_id`**, **`to_id`**, **`kind`** | Create a typed edge (own both endpoints) |
| `unlink` | **`from_id`**, **`to_id`**, **`kind`** | Remove a typed edge |
| `links` | **`id`**; `direction` (`outgoing`/`incoming`/`both`), `kind` | List edges on a facet |
| `blocked` | `parent_id` (omit → all visible) | Facets with an unsatisfied `depends_on` |
| `search` | **`query`**; `tags`, `tags_all`, `priority` | FTS5 full-text over titles + bodies |
| `grep` | **`pattern`**; `status`, `parent_id`, `tags`, `tags_all`, `priority` | Rust-regex over titles + bodies |
| `checkout` / `diff` / `flush` | `path` (+ `id` for checkout) | Round-trip a subtree to disk as markdown |
| `import_plans` | **`path`**; `dry_run` | Scan a workspace's `plans/` and materialize facets + edges |

Schemas: `synapse -P 44107 -s trak facet` (not a `schema` subcommand — see Discovery).

Other activations: **identity** (register/login/refresh, JWT, API keys, SRP), **discuss** (threaded comments on facets), **docs** (`about`/`guides`/`guide`). **access**, **audit**, **refs**, **collab** are **stubs** that return `not_implemented`.

## Edge kinds

The `kind` on a link is a closed set of exactly four values — anything else is rejected with `invalid_kind`:

| Kind | Meaning |
|------|---------|
| `depends_on` | Source needs target done first; drives the `blocked` report |
| `blocks` | Source blocks target |
| `relates_to` | Loose association |
| `duplicates` | Source duplicates target |

Containment is **not** an edge — parent/child is the `parent_id` field, set via `create`/`move_to`. Do not try `contains`/`parent`/`child` as edge kinds.

## Worked examples

The examples below show `-t "$TOKEN"` as the explicit-override form, but with the native auth above you can **omit `-t` entirely** — synapse loads the token from `defaults.json`. If you do want the override: `TOKEN="$(cat ~/.plexus/trak/token)"`.

**Grep before you file** (avoid duplicates — always do this before creating):

```bash
synapse -P 44107 -t "$TOKEN" -j -p '{"pattern":"(?i)widget"}' trak facet grep \
  | jq -r 'select(.type=="data") | .content | select(.type=="search_result") | "\(.facet.id)  \(.facet.title)"'
```

**Create a facet and capture its id:**

```bash
EPIC=$(synapse -P 44107 -t "$TOKEN" -j \
  -p '{"title":"Ship the widget","status":"open","priority":"high","tags":["epic"]}' \
  trak facet create \
  | jq -r 'select(.type=="data") | .content.facet.id')

CHILD=$(synapse -P 44107 -t "$TOKEN" -j \
  -p "{\"title\":\"Wire the backend\",\"parent_id\":\"$EPIC\",\"tags\":[\"ticket\"]}" \
  trak facet create \
  | jq -r 'select(.type=="data") | .content.facet.id')
```

**Link two facets** (the child depends on another ticket):

```bash
synapse -P 44107 -t "$TOKEN" -j \
  -p "{\"from_id\":\"$CHILD\",\"to_id\":\"$OTHER\",\"kind\":\"depends_on\"}" \
  trak facet link
```

**Walk the tree** under the epic:

```bash
synapse -P 44107 -t "$TOKEN" -j -p "{\"id\":\"$EPIC\"}" trak facet tree \
  | jq -r 'select(.type=="data") | .content | select(.type=="facet_summary") | "\(.depth)  \(.status)  \(.title)"'
```

**Mark done** (status is free-text; `done` is the conventional terminal value that satisfies `depends_on`):

```bash
synapse -P 44107 -t "$TOKEN" -j -p "{\"id\":\"$CHILD\",\"status\":\"done\"}" trak facet update
```

## Notes

- **Status** is a free-text string (default `open`), not an enum. Use a consistent vocabulary by convention (`open`/`in_progress`/`done`); `blocked` analysis treats `done` as the satisfied terminal state.
- **Ownership/tenancy** is automatic: `owner` and `meta.tenant` are derived from your auth token; you cannot read or write across tenants.
- Relationship to **`ticketing`**: that skill is about *how to write good ticket content* — contracts, acceptance criteria, the two-stranger test, `## Evidence`. **trak** is the *tool that stores* such items as facets. Draft with `ticketing`, persist with `trak`. See `../ticketing/SKILL.md`.
- For deeper detail on the data model, tenant gate, auth flow, and every activation, read the trak README at `~/dev/controlflow/hypermemetic/plexus-trak/README.md`.
