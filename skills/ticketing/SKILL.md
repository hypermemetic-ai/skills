---
name: ticketing
description: Use when the user asks to write an implementation ticket, capture acceptance criteria, or turn a feature/bug into a unit of work. Produces a markdown ticket in `plans/<EPIC>/` with YAML frontmatter, status `Pending` (only humans flip to `Ready`), file-boundary-aware concurrency, an `## Evidence` section that downstream tickets condition on, and acceptance criteria that pass the two-stranger test.
---

# Skill: Write a TDD Ticket

A ticket is a sufficient statistic for the next agent in the chain. It carries forward not just **what** to build, but the **evidence** that justifies the contract — so when the contract is questioned later, the rationale is in the file, not in someone's head.

The binding constraint is the **two-stranger test**: two agents who have never spoken — one implementing, one verifying — can independently agree on Done using only the ticket text.

## When to use

When the user asks to ticket work, plan a feature, or write acceptance criteria. Also invoked implicitly when any epic needs to be broken into implementable units.

## Inputs

| Input | Description | Example |
|-------|-------------|---------|
| Problem description | What gap, bug, or capability is missing | "Activations can't see caller identity" |
| Domain context | External APIs, crate APIs, or domain knowledge needed | jsonrpsee 0.26 connection extensions, cookie validation flow |
| Upstream tickets | What this ticket depends on (their `unlocks` includes this id) | AUTH-2 must be Complete |
| Downstream consumers | What tickets will read this ticket's output | AUTH-7 reads `AuthContext` shape |
| Confidence prior | How sure the planner is the contract is achievable | `high` if all dependencies are typed and tested; `low` if a spike is needed first |

## Output

A markdown file at `plans/<EPIC>/<EPIC>-<N>.md` with YAML frontmatter. **Status is always `Pending` on creation** — Claude must never set a ticket to `Ready` on creation. Promotion to `Ready` requires human approval: either the user edits the file themselves, or the user explicitly grants Claude permission to flip the status (e.g., "promote these to Ready", "you have permission to mark X as Ready"). Without explicit permission, Claude leaves status at `Pending`. Implementation must not begin on `Pending` tickets.

## Directory convention

```
plans/                           # Top-level: cross-crate epics
  <EPIC>/
    <EPIC>-1.md                  # Epic overview (type: epic)
    <EPIC>-2.md                  # Individual ticket
    <EPIC>-3.md
<crate>/plans/                   # Per-crate epics that don't span repos
  <EPIC>/
    <EPIC>-1.md
```

`<EPIC>` is a short prefix (e.g., AUTH, IDN, STF). `<N>` is sequential within the epic. The epic overview is always `-1`. Top-level `plans/` is the default; use a per-crate `plans/` directory only when the epic genuinely doesn't escape one crate.

## Frontmatter

```yaml
---
id: EPIC-N
title: "Short description"
status: Pending
type: implementation        # implementation | analysis | spike | epic
blocked_by: []
unlocks: []
confidence: medium          # high | medium | low (optional, default medium)
severity: Medium            # Critical | High | Medium | Low (optional)
---
```

### `confidence` field

The planner's prior on whether the contract as written is achievable as written. Updated by spike outcomes (see `planning` skill). Concrete meaning:

| Value | Meaning | Implication |
|-------|---------|-------------|
| `high` | All dependencies are typed and tested. The shape is constrained by upstream contracts. | Implementation can proceed directly once Ready. |
| `medium` | Default. The shape is reasonable but unverified against the real system. | Implementor should sanity-check assumptions early. |
| `low` | Significant unknowns. The contract may need revision once reality intrudes. | Spike first. Do not flip to Ready until a spike has updated confidence to `medium` or `high`. |

Confidence is *not* time estimation. It's the planner's belief that the *contract* (not the implementation) survives contact with reality.

## Status values

| Value | Meaning | Who sets it |
|-------|---------|-------------|
| `Idea` | Captured but the contract isn't writable yet — open design questions remain | Claude or human |
| `Pending` | Contract written, awaiting human review | Claude (on creation) |
| `Ready` | Approved, no blockers, ready for implementation | Human (directly, or Claude with explicit permission) |
| `Blocked` | Approved but waiting on `blocked_by` tickets | Human or Claude |
| `Complete` | Implemented, integration gate green, committed | Implementor (in the same commit as the code) |
| `Epic` | Overview document, not a unit of work | Claude (on epic creation) |
| `Superseded` | Absorbed by another ticket (see `superseded_by` field) | Claude or human |

`Idea` vs. `Pending`: an `Idea` cannot be promoted to `Ready` without first being rewritten as `Pending` with the open questions resolved. A `Pending` ticket has a complete contract that the human is being asked to ratify.

## Body sections

```markdown
## Problem
One paragraph. The gap, the bug, the missing capability.

## Context
(Optional) External API shapes, domain knowledge, crate API refs.
NOT implementation instructions. This is the terrain, not the route.

## Evidence
The reasoning that justifies this contract. Why this shape and not another?
What spike result, prior ticket, or external constraint pinned the choice?
This is the sufficient statistic downstream tickets and reviewers condition on.

## Required behavior
Input/output tables. Observable behaviors. "When X, then Y."
No code. No SQL. No file paths.

## Risks
(Optional) Places where the output might not be achievable.
Each risk maps to a spike, a fallback, or a replanning trigger.

## What must NOT change
Regression criteria. Existing behaviors that must survive.

## Acceptance criteria
Numbered items. Each checkable by running a command, reading a
response, or counting rows. No criterion requires reading source
code to evaluate.

## Completion
What the implementor delivers: test command + output, status flip.
```

The `## Evidence` section is the BLF-style sufficient statistic. When a downstream ticket reads "AUTH-2 unlocks me," it should be able to look at AUTH-2's Evidence section and understand *why* the `AuthContext` struct has the fields it does, so that if AUTH-2's contract is questioned, the conversation starts from the recorded reasoning rather than restarting from scratch.

## Rules

1. **One decision per ticket.** Multiple options = not ready = write a spike instead.
2. **Specify behavior, not code.** Use input/output tables.
3. **No implementation in acceptance criteria.** Criteria describe what the system does, not how.
4. **State what does NOT change.** Regression criteria are as important as new behavior.
5. **Scope is one diff.** Multi-repo changes that can't be tested together = split.
6. **Verification is high-level.** "Run tests; they pass." Not "grep for this string."
7. **Fixtures are committed.** If criteria reference a test fixture, it exists in the repo.
8. **Downstream tickets can read the contract.** If ticket B depends on A's output, A's criteria pin the shape AND A's `## Evidence` records why.
9. **Strong types in contracts.** Reference `AuthContext`, not "a struct with user id and roles." See `strong-typing` skill.
10. **Concurrency is file-bounded.** Two tickets that *write* the same file cannot run in parallel — they collide at commit time, regardless of `blocked_by`. When planning, check file boundaries across tickets in addition to logical dependencies. Reading the same file is fine; writing is not.
11. **Split tickets along file boundaries to expose parallelism.** A single ticket modifying multiple independent files is a serial chain disguised as one unit. Split into per-file units so the pieces can be implemented concurrently.

## Integration gate (the rule for `Complete`)

A ticket reaches `Complete` only when the workspace it touches builds and tests green, end-to-end. Specifically:

- Every ticket's acceptance criteria must include "`cargo build` and `cargo test` (or workspace-language equivalent) pass green for the affected workspace."
- A ticket can be `Ready` while pre-existing issues block the gate, but it stays `Ready` — not `Complete` — until the gate passes.
- If pre-existing issues prevent the gate from ever passing as-is, write a dependency ticket that fixes the blocker and add it to `blocked_by`.

**Why this is a rule and not a guideline.** Downstream tickets condition on "Complete" meaning "the workspace is in a shippable state." That assumption cascades into later epics' promotion decisions. A `Complete` ticket whose workspace is red is a contract violation that infects every consumer.

**Pre-existing blockers.** If ticket A's own work is done and committed but a separate pre-existing issue prevents the workspace from building, A stays `Ready`. File a ticket B that fixes the blocker, add B to A's `blocked_by`. When B lands, retry A's acceptance criteria — if green, flip A to `Complete`. A and B can ship in a single commit run if the implementor batches.

**Retrofit policy.** If a past ticket was flipped to `Complete` without the gate passing, flip it back to `Ready`, add the missing `blocked_by`, and carry it through cleanly. History gets a small correction; future tickets avoid the same error.

## Meta-acceptance criteria (check before writing to disk)

- [ ] Two-stranger test passes
- [ ] No open design decisions
- [ ] `## Evidence` section present and load-bearing
- [ ] Acceptance criteria are observable
- [ ] Fixtures are committed (or described precisely enough to create)
- [ ] Regression section is complete
- [ ] Downstream tickets can read both the contract AND the evidence
- [ ] File-write boundaries are disjoint from any sibling ticket marked concurrent in the DAG
- [ ] `confidence` field set; if `low`, a spike ticket exists in `blocked_by`

## Examples

**Good acceptance criteria:**

| Caller | Operation | Expected result |
|---|---|---|
| Unauthenticated client | `chat.list` | Auth error |
| User A | `chat.get` with User B's room id | "Not found" error |
| Admin | `chat.list` | Returns all rooms |

**Bad acceptance criteria:**

> `cargo test --test isolation::cannot_see_other` passes

(Pins test name, file, framework. The contract should say what the system does, not which test to run.)

**Good `## Evidence` section:**

> `AuthContext` carries `user_id`, `session_id`, `roles`, and `metadata: serde_json::Value` because (1) AUTH-1 epic established that activation methods need to discriminate users for tenant filtering, (2) jsonrpsee 0.26 exposes per-connection state via `Extensions`, which is the cleanest place to attach the context at WS-upgrade time, and (3) `metadata` is intentionally untyped because backends differ in what session data they need to thread through (Keycloak vs. local sessions vs. API keys), and freezing a schema here would force every backend into the same shape.

**Bad `## Evidence` section:**

> We need auth context.

(Restates the problem. Provides no rationale a downstream ticket can condition on.)

**Good `## Context` section:**

> jsonrpsee 0.26 exposes per-connection state via `ConnectionState::extensions()`. The cookie header arrives during the WS upgrade in the `on_request` callback; after that point the connection is upgraded and headers are not re-readable.

(Documents external contract.)

**Bad `## Context` section:**

> Use `tower::ServiceBuilder` to add middleware. Parse cookies with the `cookie` crate. Store in a HashMap.

(Implementation prescription. The ticket should say what to achieve, not how.)

## Pointers

- Planning epics, spikes, evidence aggregation: `../planning/SKILL.md`
- Bulk operations on ticket frontmatter (status flips, retargeting): `BULK_OPS.md` (sibling)
- Strong typing in contracts: `../strong-typing/SKILL.md`
- Methodology: `~/CLAUDE.md` → "Methodology" section
- Real epic for reference: `plans/AUTH/` in the hypermemetic root
