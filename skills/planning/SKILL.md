---
name: planning
description: Use when the user asks to plan an epic, break a goal into a DAG of tickets, or organize multi-step/multi-file work. Produces an epic directory under `plans/<EPIC>/` with an overview document and individual tickets, treats spikes as evidence sources whose results update the confidence prior on downstream tickets, and maximizes parallel fan-out across the dependency graph.
---

# Skill: Plan an Epic

Break a large goal into a dependency DAG of tickets with explicit inputs, outputs, risks, and parallel execution paths. The plan is a program — tickets are functions, the DAG is the call graph, errors are contract violations between tickets, and **spikes are the evidence-gathering steps that resolve uncertainty before contracts are frozen**.

## When to use

When the user describes a multi-step goal that needs to be broken into implementable units. When a new feature, security hardening pass, or infrastructure change spans multiple files, multiple concerns, or multiple days.

## Inputs

| Input | Description | Example |
|-------|-------------|---------|
| Goal | What the user wants to achieve | "Thread caller identity through Plexus RPC" |
| Constraints | What must not break | "Existing unauthenticated activations must keep working" |
| Domain context | External systems, APIs, regulatory requirements | jsonrpsee 0.26 connection state, cookie validation |
| Existing work | What's already built that this builds on | Plexus dispatch, DynamicHub registration |
| Risk tolerance | How much uncertainty the user accepts before requiring spikes | "Spike anything that depends on jsonrpsee internals" |

## Output

An epic directory under `plans/<EPIC>/` with:
- `<EPIC>-1.md` — epic overview (status: Epic)
- `<EPIC>-N.md` — individual tickets (status: Pending)

## Process

### 1. Identify the end state

What does the system look like when this epic is done? Not the steps to get there — the destination. Write this as the epic's `## Goal`.

### 2. Work backwards from the end state

What's the last thing that needs to happen? That's the leaf ticket. What does it depend on? Those are its `blocked_by` tickets. Keep working backwards until you reach tickets with no dependencies — those are the roots.

### 3. Maximize fan-out

At every node in the DAG, ask: "can any of these blocked tickets run in parallel?" If two tickets touch different files and have no data dependency, they can. The DAG should be wide, not deep. Serial chains are bottlenecks.

### 3.5. Identify shared vocabulary

Before fan-out, list every public type each ticket will need to name. For each type, assign **exactly one owner ticket** — the one that *introduces* it; all others *import* it. If a type is needed by multiple tickets but no upstream ticket owns it, hoist its introduction into a new foundation ticket that runs first.

Sibling tickets that share vocabulary collide at merge time even when their files don't overlap — Rust's orphan rules and `pub use` make the crate namespace a write-shared resource. The ticketing skill's `introduces:` / `imports:` frontmatter fields (Rule 12) are the mechanical check; this step is where the planner populates them.

Wave 1 of phase B (May 2026) skipped this step. `MethodPath`, `HeaderName`, `CookieName` were independently invented by three sibling tickets. The merge cost was a six-step hand-resolution; the planning cost would have been one extra ticket landed first.

### 4. Define inputs and outputs at every edge

If ticket A unlocks ticket B, then A's acceptance criteria must pin exactly what B will read. **A's `## Evidence` section must record *why* the contract has that shape**, so that if A's contract is questioned later, the conversation starts from recorded reasoning rather than restarting cold. Use strong types (domain newtypes, not bare strings) in the contract language.

### 5. Set initial `confidence` per ticket

For each ticket, set the planner's prior on whether the contract survives contact with reality:

- `high` — every dependency is typed and tested; the shape is constrained by upstream contracts.
- `medium` — default; the shape is reasonable but unverified.
- `low` — significant unknowns; spike before promoting to Ready.

A `low`-confidence ticket **must** have a spike in its `blocked_by`. Promoting a `low` ticket to Ready without a spike is a planning error.

### 6. Surface risks → spikes

For every ticket that touches an external system, an unstable API, or an unproven assumption, add a `## Risks` section. Each risk maps to:
- A **spike** — a binary investigation that produces structured evidence (see Spikes section)
- A **fallback** — a degraded-but-acceptable contract revision
- A **replanning trigger** — "if this fails, downstream tickets need rewriting"

### 7. Check for design decisions

If any ticket requires a choice between multiple approaches, it's not ready for implementation. It either needs:
- A spike ticket that resolves the choice (the spike's evidence picks the approach)
- A decision pinned in the ticket text with the rationale recorded in `## Evidence`

### 8. Write tickets in dependency order

Start with the roots (no `blocked_by`), then write each ticket in topological order so you can reference upstream outputs in downstream inputs.

## Epic overview format

```yaml
---
id: EPIC-1
title: "Epic Name — Overview"
status: Epic
type: epic
blocked_by: []
unlocks: []
---
```

Body:

```markdown
## Goal
## Dependency DAG       # ASCII or mermaid
## Phase Breakdown
## Tickets              # table: id, summary, status, confidence
## Risks → Spikes       # which spike addresses which risk
## Out of scope
```

## Principles

- **Parallel by default.** If two tickets CAN run simultaneously, the DAG MUST allow it.
- **One decision per ticket.** Ambiguity in one ticket cascades to every downstream ticket.
- **Contracts at every edge.** Ticket A's output is ticket B's input. Both must name the same shape.
- **Evidence travels with the contract.** A's `## Evidence` is a sufficient statistic for B; B doesn't need to re-derive A's reasoning.
- **Vocabulary is a shared resource.** Two parallel tickets must not introduce the same public type. Either one owns it and the other imports, or a foundation ticket lands first. See Step 3.5.
- **Risks are first-class.** A risk section on a ticket is like error handling in code. Without it, the plan works until it doesn't.
- **Confidence calibrates the plan.** Track which `low`-confidence tickets needed contract revision after their spike, and which didn't. Use that history to set future priors.
- **Pending until approved.** All tickets start at `status: Pending`. Only the user promotes to `Ready`.

## Spikes — evidence-gathering, not binary gates

A spike is a micro-experiment that gathers evidence about an unknown. Its output **updates the confidence prior** on the unlocked ticket; it does not just flip a pass/fail bit.

### Spike workflow

1. **Identify the unknowns.** Each unknown becomes a spike (S-01, S-02, ...).
2. **Order spikes by ambition.** S-01 tries the ideal approach. S-02 tries the fallback. S-03 tries a further concession. Run in order; stop at the first one whose evidence is sufficient to lock the contract.
3. **Spikes produce structured evidence.** A spike's deliverable is not "yes" or "no" — it's a paragraph in the unlocked ticket's `## Evidence` section: what was tried, what was observed, what shape the data took, and what that implies for the contract.
4. **Aggregate evidence across multiple spikes when they bear on the same contract.** If S-01 partially succeeds (proves field X is available but field Y is not), S-02 doesn't start from zero — it inherits S-01's evidence and asks the next narrower question. The downstream ticket's `## Evidence` section is the *combined* belief, not the last spike's report.
5. **Update the confidence prior.** A passing spike usually moves the unlocked ticket from `low` → `medium` or `high`. A spike that uncovers a constraint may move it from `medium` → `low` (or trigger replanning).
6. **If all spikes fail:** the risk is a confirmed constraint. Document it in the unlocked ticket's `## Evidence`, and replan downstream tickets. This is normal — the plan discovered a real limitation.
7. **Spike code lives in `<crate>/spike/<epic>/`** as standalone programs.

### Spike ticket format

```yaml
---
id: EPIC-S01
title: "Spike: Does jsonrpsee 0.26 expose cookie headers in on_request?"
status: Pending
type: spike
blocked_by: []
unlocks: [EPIC-N]
confidence: n/a
---
```

```markdown
## Question
Does jsonrpsee 0.26's `on_request` callback receive the WS-upgrade request
with cookie headers intact, or are they consumed before reaching us?

## Setup
Spin up a minimal jsonrpsee server with an `on_request` hook. Send a WS
upgrade with `Cookie: session=abc`. Log what the hook receives.

## Pass condition
The hook observes `Cookie: session=abc` in the request headers.

## Evidence to record (regardless of pass/fail)
- What the request struct exposes (full headers? trimmed? mutated?)
- Whether `extensions()` is writable from the hook (needed for AUTH-7's design)
- Any version-specific behavior worth pinning in Cargo.toml

## Fail → next
S-02: Use a tower middleware layer above jsonrpsee to capture cookies before
the upgrade, then thread via Extensions.

## Fail → fallback
If neither spike works: route auth through a separate HTTP endpoint that
issues a one-time WS connect token. Document as a known limitation;
contract for AUTH-2 changes from "cookie at upgrade" to "token at upgrade".
```

**Key rule:** A spike that requires a judgment call to decide pass/fail is not a spike. The pass condition must be binary. But the *evidence* the spike records is structured prose, not a bit — that evidence is what downstream tickets condition on.

## Calibration — closing the loop

At the end of an epic (when the last ticket reaches `Complete`), spend five minutes on calibration:

- Which `low`-confidence tickets needed contract revision after their spike? (Confidence was correctly low.)
- Which `low`-confidence tickets had their spike pass first try and didn't need revision? (Confidence was *too* low — we over-hedged.)
- Which `high`-confidence tickets needed mid-flight contract revision? (Confidence was *too* high — we under-hedged.)
- What was the most expensive surprise? Did anything in the upstream `## Evidence` sections hint at it?
- Did any sibling tickets share vocabulary that wasn't centralized? If yes, the planner under-hoisted at Step 3.5.

Record the takeaway in a one-line note at the bottom of the epic overview, e.g.:

> **Calibration (2026-04-25):** AUTH-3's `medium` was right; AUTH-7's `high` should have been `low` — we missed that jsonrpsee Extensions don't survive across the upgrade boundary. Future tickets touching jsonrpsee internals: start at `low` until proven otherwise.

This is the "Platt scaling" of the planning process. You are systematically biased in some direction; explicit calibration shrinks the bias over epics.

## Architecture doc naming convention

Architecture documents in `docs/architecture/` use reverse-chronological naming so newest documents appear first in alphabetical sorting.

**Formula:** `(u64::MAX - nanotime)_title.md`

```python
import time
nanotime = int(time.time() * 1_000_000_000)
filename = (2**64 - 1) - nanotime
print(f'{filename}_your-title.md')
```

Example: `16681577588676290559_type-system.md`

## Pointers

- Ticket format, status values, evidence section: `../ticketing/SKILL.md`
- Strong typing in contracts: `../strong-typing/SKILL.md`
- Methodology: `~/CLAUDE.md` → "Methodology" section
- Real epic for reference: `plans/AUTH/` in the hypermemetic root
