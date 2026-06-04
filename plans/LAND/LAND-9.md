---
id: LAND-9
title: "execution vs build ticket: structure, traversal, concurrency, completion gate"
status: Complete
type: implementation
blocked_by: []
unlocks: [LAND-7]
confidence: high
---

# LAND-9: pin the execution/build structure and how it's walked

## Problem

`execution-ticket.md` says the build/execution boundary is "a test, not a label," but leaves the *structure and traversal* underspecified: may an execution ticket hold work directly? is a build really a leaf? how do concurrent subagents avoid colliding? where is the working branch recorded? what closes an execution ticket? Without these pinned, subagent execution and concurrency are improvised.

## Evidence

Surfaced building this epic — LAND is an execution ticket whose builds are leaves, run (potentially) by concurrent subagents, each on its own branch, converging on a completion gate. The model only works if these rules are explicit.

## Provides (output contract)

| Output | Shape | Consumed by |
|---|---|---|
| Execution/build structural contract | container-vs-leaf rule, traversal, in-progress marking, branch-on-build, completion-gate task | autonomous-work concurrency; cn-cm2 vendoring (LAND-7) |

## Required behavior

Edit `execution-ticket.md`:

- **An execution ticket encapsulates a deliverable.** If the deliverable is small enough it *is* a single **build ticket** (a leaf). If it's bigger, it decomposes into **build tickets and/or further execution tickets**.
- **An execution ticket that contains build (or execution) sub-tickets holds no work itself** — it's a pure container/encapsulator. Work lives only in leaves.
- **A build ticket is a leaf — no subtasks, by definition.** (If a "build" grows subtasks, it was an execution ticket in disguise → promote it; existing rule.)
- **Traversal:** when a build completes, move *up* the tree and take the **next unstarted build**. The DAG is walked leaf-by-leaf.
- **Concurrency:** **mark a build in-progress the moment work starts**, so multiple subagents can run concurrently without both claiming the same leaf. (How concurrent results integrate is solved per-case for now — not prescribed here.)
- **Branch-on-build:** the build ticket **records the branch** the work is happening on (so the leaf, its diff, and its PR are all reachable from the ticket — pairs with Rule 17 / LAND-4 topology).
- **Completion gate:** every execution ticket has a **final task that gates its completion** — the convergence step all its leaves feed; the execution ticket isn't done until that gate passes. Add it to Required sections + the Skeleton.

## Acceptance

1. `execution-ticket.md` states container-vs-leaf (execution-with-children holds no work; build is a leaf), the up-the-tree traversal, in-progress-on-start for concurrency, branch-on-build, and the mandatory completion-gate task.
2. The Skeleton + Required sections include the completion gate and the build's branch field.
3. Consistent with LAND-4 (PR topology) and Rule 16/17; no external refs.
