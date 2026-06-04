---
id: LAND-5
title: "autonomous-work: methodology as object; PR-stack as terminal deliverable"
status: Complete
type: implementation
blocked_by: [LAND-2, LAND-4]
unlocks: [LAND-7]
confidence: medium
---

# LAND-5: autonomous-work — the deliverable is the PR topology, not a commit pile

## Problem

`autonomous-work/SKILL.md` frames a long block as "work for the user, stub honestly, log issues, leave a SESSION_LOG." It doesn't say *what shape the produced work takes*. After a session that completes a nested DAG, the result today is an unstructured pile of local commits — exactly the "multiple overlapping changes" tracking gap LAND-1 names.

## Evidence

Two inputs pin this. From [LAND-2](LAND-2.md): the methodology is the **primary object** an autonomous session follows, and the agent's finish line is **implemented (PR open)**, never landed — so the session *cannot* terminate at "done," only at "reviewable." From [LAND-4](LAND-4.md): the DAG is the merge plan, so the natural terminal shape of completed nested work is a **DAG-ordered set of PRs** (stacked, or one execution PR), each carrying its merge-frontier diagram. The SESSION_LOG then narrates *where the frontier is*, not just what was done.

## Provides (output contract)

| Output | Shape | Consumed by |
|---|---|---|
| Autonomous terminal-deliverable definition | a DAG-ordered PR stack (or one execution PR) + frontier diagrams + a SESSION_LOG that states the merge frontier | cn-cm2 vendoring (LAND-7); every future autonomous session |

## Required behavior

- Recenter the skill's opening on **methodology as the primary object** + the **output states** (link LAND-2).
- Add: the terminal deliverable of a long session is the **PR topology** per LAND-4 — not raw commits. Pick the PR strategy, branch along the DAG, open PRs in dependency order, link each to the next-to-merge, embed the frontier diagram.
- SESSION_LOG gains a "merge frontier" line: what's open, what's merged, what's next, what's blocked on review/QA.
- Preserve the existing discipline (commit only to new repos, stub honestly, log every issue, match model to task).

## What must NOT change

The companion relationship to `presence`; the "work FOR not WITH" framing; the honest-stub and issue-log rules.

## Acceptance

1. The skill states methodology-as-primary-object + output states up front.
2. The terminal deliverable is defined as the DAG-ordered PR topology (+ frontier diagrams), with the SESSION_LOG carrying the frontier.
3. Existing autonomous-block disciplines retained; no external refs.
