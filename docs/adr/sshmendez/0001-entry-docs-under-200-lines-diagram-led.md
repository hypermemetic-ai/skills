# ADR sshmendez/0001 · Entry docs stay under 200 lines, diagram-led

Date: 2026-06-12 · Status: accepted

## Context

`methodology/SKILL.md` — the doc read on every cold start — had grown to 226 lines of layered prose, with the load-bearing structure (the node kinds, the state machine, the recursion) buried below the on-ramp and duplicated between a scan-table and expanded bullets.

## Decision

Entry-point docs sshmendez owns (the methodology first) stay **under 200 lines** and lead with the structural diagrams: the node-kind map (what each kind does for itself / what it can become), the Linear state machine, and one example tree with a mid-flight walkthrough. Depth lives in the per-kind skills, not the entry doc.

## Why

Reader bandwidth is the bottleneck (per the diagramming skill), and the entry doc pays its cost on every session. Trade-off: less inline depth — accepted because the per-step skills carry it and the entry doc links them.
