---
id: LAND-7
title: "re-vendor LAND-2…6 into cn-cm2-skills + map to CM2 pipeline"
status: Pending
type: implementation
blocked_by: [LAND-2, LAND-3, LAND-4, LAND-5, LAND-6]
unlocks: []
confidence: medium
---

# LAND-7: re-vendor into cn-cm2-skills with the CM2 pipeline mapping

## Problem

The LAND changes land first in the canonical hypermemetic skills. `cn-cm2-skills` holds vendored copies (methodology, ticketing, execution-ticket, autonomous-work, presence) that must be refreshed — and the abstract pipeline (`review → qa → preview → prod`) must be mapped to CM2's concrete Linear states.

## Evidence

`cn-cm2-skills/AGENTS.md` requires self-containment (no external refs, no cross-repo absolute paths) — established when these skills were first vendored. The landing half is where the environment is most concrete: CM2's tracker has the literal pipeline (`coding → in code review → qa testing → … → done`), so the cn-cm2 copy should name those states where hypermemetic stays abstract. This is the only ticket that touches a second repo.

## Provides (output contract)

| Output | Shape | Consumed by |
|---|---|---|
| Refreshed cn-cm2 vendored skills | LAND-2…6 changes ported, self-contained | every CM2 agent session |
| CM2 pipeline mapping | the abstract `implemented → review → qa → … → landed` bound to the live Linear states | CM2 autonomous/presence sessions |

## Required behavior

- Port the LAND-2…6 edits into the cn-cm2 copies, scrubbing any external/cross-repo refs (repoint to `../methodology/SKILL.md` / `../../AGENTS.md` per the existing vendoring convention).
- Where the hypermemetic text says "the pipeline (review/QA/…)", the cn-cm2 copy names CM2's Linear states and notes `Complete` = the build's branch PR'd into the CM2 flow.
- Land via a feature branch + PR (no `main` push), per cn-cm2 `AGENTS.md` git rules.

## What must NOT change

The cn-cm2 self-contained rule; the CM2-authored skills (concept-mapping, manager-daily-review, pitch-status-review).

## Acceptance

1. cn-cm2 copies of the five skills reflect LAND-2…6, self-contained (no external refs — verified by grep).
2. The CM2 Linear pipeline states are named in the landing/Complete text.
3. Change is on a feature branch with a PR; `main` untouched.
