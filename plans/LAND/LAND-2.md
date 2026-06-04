---
id: LAND-2
title: "methodology: extend the spine past builds; name the output states"
status: Complete
type: implementation
blocked_by: []
unlocks: [LAND-3, LAND-4, LAND-5, LAND-6]
confidence: high
---

# LAND-2: methodology — the execution-and-landing half + output states

## Problem

`skills/methodology/SKILL.md` describes the authoring spine (goal → milestone → scope → execution DAG → spikes → builds) and stops. It never says what a build *becomes* after it's written, so downstream skills have no shared vocabulary for "implemented vs landed," and autonomous work has no stated primary object.

## Evidence

The whole epic ([LAND-1](LAND-1.md)) rests on one distinction the methodology must own: the **agent's finish line (implemented / PR open)** vs the **work's finish line (landed, past the pipeline)**. Every other ticket references these terms, so they are defined *here* once. The "output states" are the lifecycle a build moves through; naming them is what lets autonomous work track overlapping changes (LAND-5) and what lets `Complete` be pinned (LAND-3).

## Provides (output contract)

| Output | Shape | Consumed by |
|---|---|---|
| The build lifecycle states | `Ready → coding → implemented(PR) → review → qa → … → landed`, with the **agent boundary** at `implemented` and the **pipeline boundary** at `landed` | LAND-3, LAND-4, LAND-5, LAND-6 |
| "Methodology is the primary object" statement | the object an agent follows; planning *and* execution are walks of it | LAND-5, LAND-6 |

## Required behavior

- The spine section gains an execution-and-landing arc after "builds": `build → implemented → review/QA pipeline → landed`, with the lifecycle diagram from LAND-1 (the two-boundary one).
- A short "output states" subsection names the states and which boundary each side of.
- An explicit statement that the methodology skill is the **primary object** both planning and autonomous execution follow — added near the top, where the skill orients the reader.
- Cross-links to the consuming skills (`../ticketing/`, `../autonomous-work/`) for the details.

## What must NOT change

The authoring spine (goal→…→builds) stays as-is; this is additive. No renaming of existing sub-skill responsibilities.

## Acceptance

1. `methodology/SKILL.md` contains the lifecycle states with the implemented/landed boundaries named, and the two-boundary diagram.
2. The "methodology is the primary object" framing is present and findable from the skill's opening.
3. No external/cross-repo refs introduced (self-contained discipline holds for the eventual cn-cm2 vendoring).
