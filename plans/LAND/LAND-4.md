---
id: LAND-4
title: "execution-ticket: realizing the DAG as PRs"
status: Complete
type: implementation
blocked_by: [LAND-2]
unlocks: [LAND-5, LAND-7]
confidence: medium
---

# LAND-4: execution-ticket — the DAG is the merge plan

## Problem

`ticketing/execution-ticket.md` describes the DAG as a *planning* structure (what builds exist, how they depend). It says nothing about realizing it in git once the builds are written — so a long session that finishes the DAG locally has no stated way to branch, PR, and land in dependency order.

## Evidence

[LAND-1](LAND-1.md)'s insight: the execution DAG *is* the merge plan. The dependency edges are branch-base edges; landing is a topological walk through the pipeline. Two PR strategies exist and must be *chosen*, not defaulted — the criterion is reviewer cognitive load + dependency coupling. This is where the diagramming skill applies to the *merge* process (the frontier diagram), so it belongs on the execution ticket, which owns the DAG.

## Provides (output contract)

| Output | Shape | Consumed by |
|---|---|---|
| Branch-topology rule | dependency edge ⇒ branch-base edge (build B branches off A's branch) | autonomous-work (LAND-5) |
| PR-strategy fork + criterion | stacked-per-build vs one-execution-PR, chosen by reviewer-load + coupling, recorded on the execution ticket | LAND-5, LAND-7 |
| Frontier-diagram convention | each stacked PR links next-to-merge + embeds the DAG diagram with merged/current/blocked marked; single-PR case links the execution ticket + its DAG | LAND-5, LAND-7 |

## Required behavior

New section **"Realizing the DAG as PRs"** containing:
- The branch topology (the DAG → branch-tree diagram from LAND-1).
- The two strategies with the **PR-strategy fork diagram** and the explicit choosing criterion; the execution ticket records which was chosen and why.
- The per-PR linking + **merge-frontier diagram** convention (merged=green · this PR=current · blocked-behind=gray), with the single-execution-PR fallback ("link the execution ticket only, show its DAG").
- Merge order = topological order of the DAG.

Lead the section with the diagram (Rule 13).

## What must NOT change

The DAG/planning content of execution-ticket.md stays; this is the realization layer added beneath it.

## Acceptance

1. "Realizing the DAG as PRs" exists, diagram-led, with both strategies + the choosing criterion.
2. The frontier-diagram + next-to-merge linking convention is specified, with the single-PR fallback.
3. Branch-topology rule (dependency edge = branch-base edge) stated.
4. Uses LAND-2's boundary vocabulary; no external refs.
