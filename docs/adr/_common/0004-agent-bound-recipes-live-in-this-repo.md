# ADR _common/0004 · Agent-bound recipes live in this repo; distillation covers agent tasks

Date: 2026-06-12 · Status: accepted

## Context

The recipe skill's Rule 7 said recipes are project-specific and land in the worked-on repo — but a session's hardest-won paths turned out to be **agent-bound**, not codebase-bound: orienting on a PR stack, operating the tracker at scale, reconciling tickets with landed code. Those recur across every project, so no single worked-on repo is their home, and the distill pass as written only ran over execution graphs.

## Decision

Two recipe homes: project-bound recipes stay in the worked-on repo; **agent-bound / method-bound recipes land in this repo's `docs/recipes/`**, scoped `_common/` + `<github-username>/` exactly like ADRs. The distill pass extends to **strictly agent-bound tasks** — a completed agent task whose path was non-obvious distills the same way a closed execution segment does.

## Why

A recipe is only worth writing where the next runner will look; method-bound lessons filed in one project's repo are invisible to every other project. Trade-off: this repo now carries *records* alongside *practices* — accepted, with the boundary kept explicit in both READMEs (`docs/recipes/README.md`, `docs/adr/README.md`).
