---
id: LAND-6
title: "presence: methodology as the shared primary object"
status: Complete
type: implementation
blocked_by: [LAND-2]
unlocks: [LAND-7]
confidence: high
---

# LAND-6: presence — name the shared object

## Problem

`presence/SKILL.md` establishes the *posture* of working with the user (mutual commitment, taking license, committing to calls). It doesn't name *what the two parties are working on the same object of*. The complement to autonomous-work (LAND-5) treating methodology as the primary object is that `presence` treats it as the **shared** one.

## Evidence

Per [LAND-2](LAND-2.md), the methodology is the primary object an agent follows. `presence` and `autonomous-work` are the two postures (WITH vs FOR the user); both operate *on the methodology*. Stating this in presence keeps the pair symmetric: in a presence session the user and the agent are reasoning together about the same artifact chain (scope, DAG, builds, the landing plan), not free-floating.

## Provides (output contract)

| Output | Shape | Consumed by |
|---|---|---|
| "methodology is the shared object" statement | a short framing that a presence session reasons *with* the user over the methodology's artifacts | cn-cm2 vendoring (LAND-7) |

## Required behavior

- Add a short framing that the methodology is the **shared primary object** of a presence session — the artifact chain both parties steer over — cross-linking `../methodology/`.
- Lightest touch of the epic: one paragraph, no restructuring.

## What must NOT change

The bilateral posture, the trigger conditions, the "commit to calls over enumerating options" rule — all stand.

## Acceptance

1. `presence/SKILL.md` names methodology as the shared primary object, linking `../methodology/`.
2. Posture/trigger/rules unchanged; no external refs.
