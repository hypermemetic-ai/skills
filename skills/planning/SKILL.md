# Skill: Plan an Epic

Break a large goal into a dependency DAG of tickets with explicit inputs, outputs, risks, and parallel execution paths. The plan is a program — tickets are functions, the DAG is the call graph, errors are contract violations between tickets.

## When to use

When the user describes a multi-step goal that needs to be broken into implementable units. When a new feature, security hardening pass, or infrastructure change spans multiple files, multiple concerns, or multiple days.

## Inputs

| Input | Description | Example |
|-------|-------------|---------|
| Goal | What the user wants to achieve | "Replace hardcoded goals with USCIS wizard tree data" |
| Constraints | What must not break | "The packages.goals API shape must be backward compatible" |
| Domain context | External systems, APIs, legal/regulatory requirements | USCIS REST API, SOC2 controls, Keycloak realm structure |
| Existing work | What's already built that this builds on | Form reference graph, TenantScope, ValidUser |

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

### 4. Define inputs and outputs at every edge

If ticket A unlocks ticket B, then A's acceptance criteria must pin exactly what B will read. This is the contract. If A's output shape changes, B's implementation breaks. Make the contract explicit — use strong types (domain newtypes, not bare strings) in the contract language.

### 5. Surface risks

For every ticket that touches an external system, an unstable API, or an unproven assumption, add a `## Risks` section. Each risk maps to:
- A **spike** — a binary investigation (see the Spikes skill)
- A **fallback** — a degraded-but-acceptable contract revision
- A **replanning trigger** — "if this fails, downstream tickets need rewriting"

### 6. Check for design decisions

If any ticket requires a choice between multiple approaches, it's not ready for implementation. It either needs:
- A spike ticket that resolves the choice
- A decision pinned in the ticket text (the planner makes the call)

### 7. Write tickets in dependency order

Start with the roots (no blocked_by), then write each ticket in topological order so you can reference upstream outputs in downstream inputs.

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
## Dependency DAG
## Phase Breakdown
## Tickets (table: id, summary, status)
## Out of scope
```

## Principles

- **Parallel by default.** If two tickets CAN run simultaneously, the DAG MUST allow it.
- **One decision per ticket.** Ambiguity in one ticket cascades to every downstream ticket.
- **Contracts at every edge.** Ticket A's output is ticket B's input. Both must name the same shape.
- **Risks are first-class.** A risk section on a ticket is like error handling in code. Without it, the plan works until it doesn't.
- **Pending until approved.** All tickets start at `status: Pending`. Only the user promotes to `Ready`.

## Spikes — Binary Investigations

When a risk materializes or a ticket has an open design question, write a spike instead of guessing.

**What a spike is:** A micro-experiment that answers one specific question. Not a prototype. A spike runs, produces an observable result, and the result decides the approach.

**Spike workflow:**
1. Identify the unknowns. Each unknown becomes a spike program (S-01, S-02, ...).
2. Spikes are ordered: S-01 tries the ideal approach. S-02 tries the fallback if S-01 fails. S-03 tries a further concession.
3. Run spikes in order. Stop at the first one that works.
4. The passing spike's approach becomes the implementation ticket.
5. If ALL spikes fail: the risk is a confirmed constraint. Document it. Replan downstream tickets. This is normal — the plan discovered a real limitation.
6. Spikes live in `<crate>/spike/<epic>/` as standalone programs.

**Spike ticket format:**

```yaml
---
id: EPIC-S01
title: "Spike: Can we extract combo box options from pdf_oxide?"
status: Pending
type: spike
blocked_by: []
unlocks: [EPIC-N]
---
```

```markdown
## Question
Does pdf_oxide 0.3.24's FormField expose the /Opt array for Choice fields?

## Setup
Load the fixture PDF. Call FormExtractor::extract_fields. Check if any
field has options data.

## Pass condition
At least one field has a non-empty options list.

## Fail → next
S-02: Parse the /Opt array manually from the PDF dictionary via pdf_oxide's
low-level document API.

## Fail → fallback
If neither spike works: accept options = [] for pdf_oxide extractor.
Document as a known limitation. Only pdfium provides options.
```

**Key rule:** A spike that requires a judgment call to decide pass/fail is not a spike. The pass condition must be binary.

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

- Ticket format and status values: `skills/ticketing/SKILL.md`
- Strong typing in contracts: `skills/strong-typing/SKILL.md`
- Methodology: `~/CLAUDE.md` → "Methodology" section
- Example epics: FormVeritas `plans/IDN/`, `plans/STF/`, `plans/CHILD/`
