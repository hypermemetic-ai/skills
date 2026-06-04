---
id: LAND-3
title: "ticketing: Complete = implemented + PR-open, not shipped"
status: Complete
type: implementation
blocked_by: [LAND-2]
unlocks: [LAND-7]
confidence: high
---

# LAND-3: ticketing — redefine `Complete` for a pipeline environment

## Problem

`ticketing/SKILL.md`'s integration gate defines `Complete` as "workspace builds and tests green, committed." In a review/QA pipeline that is *not* shipped — but the skill's own text ("downstream tickets condition on Complete meaning the workspace is in a shippable state") invites reading `Complete` as done. That conflation is the bug LAND-1 names.

## Evidence

Per [LAND-2](LAND-2.md), the agent's finish line is **implemented (PR open)** and the work's finish line is **landed** (past the pipeline). `Complete` must map to the agent boundary, not the work boundary, or every consumer over-reads it as shipped. This is the same calibration the `Superseded`/supersession section already respects — status must mean what it says.

## Provides (output contract)

| Output | Shape | Consumed by |
|---|---|---|
| `Complete` redefinition | "implemented: workspace green + committed to the build's branch + **PR open** (handed to the pipeline)" — explicitly *not* merged/shipped | execution-ticket authors (LAND-4), autonomous-work (LAND-5), cn-cm2 vendoring (LAND-7) |

## Required behavior

- Integration-gate section: keep the green-build requirement, but state the gate produces an **open PR**, and that landing (merge/review/QA) is the pipeline's, tracked by the live tracker's states — not by `Complete`.
- Status table (`Complete` row): reword to "implemented + PR open; pipeline closes it."
- A one-line pointer to `execution-ticket.md` (LAND-4) for how the PR is shaped within the DAG.

## What must NOT change

The two-stranger test, the acceptance-criteria rules, and the "green before Complete" requirement all stand. Rule 17 (link the PR directly) is reinforced, not replaced.

## Acceptance

1. `Complete` in both the status table and the integration-gate section reads as implemented/PR-open, explicitly not shipped.
2. The pipeline (review/QA/…) is named as the closer, tracked by tracker states, not by `Complete`.
3. Consistent with LAND-2's boundary vocabulary (same terms).
