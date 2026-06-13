# ADR _common/0001 · Linear is the source of truth for plans

Date: 2026-06-12 · Status: accepted

## Context

The vendored methodology described a `plans/<EPIC>/` markdown layer with YAML frontmatter as the canonical plan, "projected" into the tracker. No such directory existed in any worked-on repo — in practice every plan was authored and iterated directly in Linear ticket descriptions, so the doctrine guaranteed drift between a doc nobody wrote and the tickets everybody used.

## Decision

Linear is the source of truth. The scope and execution ticket **descriptions are the canonical documents, iterated in place**; dependencies are blocking relations; statuses are the real workflow states (`Triage → coding queue → coding → in code review → qa testing → preview → prod → done`). The `plans/` folder methodology is retired across the suite (PR #1, commit `274d550`). We say **Linear**, not "the tracker."

## Why

A source-of-truth layer that has stopped being written is worse than none — downstream agents condition on stale prose. Trade-off accepted: we lose offline, greppable, git-versioned plan files; we keep one home with live state, relations, attachments, and human ratification built in. Mechanics for operating on it at scale: `skills/ticketing/BULK_OPS.md`.
