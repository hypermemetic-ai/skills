# Scope ticket

The canonical *what/why* of a milestone — its **single entry point**. One per milestone. Carries the **contract + evidence, not the work** (the work lives in the [execution ticket](execution-ticket.md)). It states **what is** and **what should be**, and marks the edges — what's **out of scope**, what's **defined elsewhere**, and what's **not in scope but arguably should be** (surfaced as a gap for a human call, not silently dropped). Diagram-led — lead with the diagram that carries the shape (see the [diagramming](../diagramming/SKILL.md) skill).

> **Lifecycle & state** live in [planning](../planning/SKILL.md) → "Projecting the plan into a live tracker": `Done` once it has spawned ≥1 work issue (never before — else the milestone reads complete with no work in it), and **never `Canceled`** (canceled reads as abandoned; this is a living reference). *This doc is the body format.*

## Required sections

- **End state** — what the system looks like when this milestone is done, with the load-bearing diagram(s).
- **Decided shape** — the design decisions, each with its *why* (the evidence that justifies it).
- **Core transition / problem** — what changes from today, and why it matters.
- **Decisions & open questions** — settled items marked `DECIDED`; open ones listed; inferences marked `[confirm]` rather than asserted.
- **Interface contract — the milestone's output contract** (the milestone-level `## Provides`). What this milestone hands the next: the named, strong-typed surface downstream milestones condition on — the *union of its builds' `## Provides`*, distilled to what the next milestone consumes. **This is the highest-stakes section in the ticket** — it's the edge whole milestones hang off, so an unstated or sloppy contract here cascades furthest. Define it crisply even while the internals are still open. (It's the scope ticket's analog of a build's `## Provides` + `## Evidence`.)
- **Dependencies.**
- **`## Execution` pointer** — link to the execution ticket. The DAG does **not** live here.

## Not a scope-ticket section

These crept in from the Shape-Up *pitch* template; they are pitch-level or execution-level and **do not belong in a scope ticket**:

- **Appetite / Rough appetite** → the **pitch** (the bet's budget, once per bet). Sizing of the actual work → estimates on the **build / execution** tickets, where it's concrete.
- **Purpose** ("this is a scoping ticket, not implementation") → boilerplate; drop it.
- **Who's affected** → the **pitch**.
- **Rabbit holes** → the **pitch** (or a build's `## Risks`).
- **Phase sketch / "What we'll ticket"** → the **execution ticket** (the DAG and the work list).
- **No-gos** → fold into **Decided shape** (they're decided constraints), not a standalone section.

The scope ticket carries the **shape and the contract** — what the milestone *is* and what it hands the next one — never how much we'll spend or the step-by-step.

## Skeleton

```markdown
*Done by definition — the canonical view of the system + plan for this milestone; not a progress signal.*

## End state
<the load-bearing diagram + 1–2 lines>

## Decided shape
- <decision> — *why:* <evidence>

## The core transition
<what changes from today, and why it matters>

## Decisions & open questions
1. DECIDED: <settled point>
2. <open question>   [confirm: <inference not yet confirmed>]

## Interface contract (what this milestone hands the next)
| element | definition-state | runtime | consumer |
| -- | -- | -- | -- |

## Dependencies
<blocked-by / blocks>

## Execution
The DAG and work tracking live in the execution ticket → <link>.
```

Worked example: `CMD-1503` (M2a · identity foundation).
