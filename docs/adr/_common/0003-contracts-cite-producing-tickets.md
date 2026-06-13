# ADR _common/0003 · Every consumed type cites the ticket that produces it

Date: 2026-06-12 · Status: accepted

## Context

A milestone's frozen type surface declared a new `TenantID int64` while a sibling milestone had already landed a validated `Tenant` newtype in the same Go package — a duplicate owner that would have collided at merge. The consuming ticket named the type but not its producer, so nothing forced the collision into view; it was caught by a manual review, not by the planning mechanics.

## Decision

In every `## Provides`/`## Consumes` table and frozen contract surface, each consumed type cites its producing ticket inline (e.g. `Tenant ← M2a.B5.B7 · CMD-415`). A type with no producer cited is either minted *by this ticket* (then it goes in `## Provides`) or a **missing root** to resolve before ratification. Recorded as the "Types reference their producers" cross-cutting discipline in the methodology.

## Why

The Provides/Consumes diff only catches duplicate owners and missing roots when both sides are declared; the inline producer citation makes the dependency edge readable from one ticket alone and turns "who owns this type" into a lookup instead of an archaeology dig. Cost: a few characters per table row.
