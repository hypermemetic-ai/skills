# ADR _common/0002 · The methodology never cites the work it governs

Date: 2026-06-12 · Status: accepted

## Context

The methodology suite cited a live milestone — its execution epic, its scope ticket, its builds — as its own reference example. That milestone had itself been shaped by these skills, making the reference circular ("does the work follow the method?" becomes unfalsifiable when the method is defined by the work), and the ticket pointers rot as the work is superseded.

## Decision

The methodology core (`methodology`, `planning`, `ticketing` + the scope/execution formats, `orient`) uses **fictional or generic examples only**. `recipe` and `grill` are exempt: citing the real history that earned a rule is precisely their job — as it is for ADRs like this one. The rule is enforced in `AGENTS.md`.

## Why

Grounded real examples teach well — that's the trade-off — but the grounding now lives where decay is handled by design: recipes, grills, and ADRs are *records* with their citations pinned, while the methodology stays a timeless reference that any milestone can be judged against.
