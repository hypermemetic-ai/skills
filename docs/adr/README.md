# Decision records (ADRs)

One decision per file, about **how we work** — the methodology, the skills, the conventions this repo carries. Decisions about *project code* belong in the worked-on repo's own `docs/adr/`, not here.

## Organization

```
docs/adr/
  README.md          ← this file
  _common/           ← binds EVERYONE who uses these skills
    0001-....md
  sshmendez/         ← binds only sshmendez's own workflow
    0001-....md
  <github-username>/ ← create your folder on your first ADR
```

- **`_common/`** — decisions any user of this repo's skills is expected to follow. Changing one is a team-visible act: supersede with a new ADR, don't edit history.
- **`<github-username>/`** — personal-scope conventions: how that person runs their own workflow, docs they own, agent-collaboration rules that apply only to them. A personal ADR may **tighten** a `_common` rule for its owner; it may never contradict one — if it would, the decision belongs in `_common` (or the `_common` rule needs superseding).

## Numbering & citation

`NNNN-slug.md`, sequential **within each scope folder** (scan for the highest, increment). Cite as `per ADR _common/0001` / `per ADR sshmendez/0001` from tickets, reviews, and `## Evidence` sections.

## What earns an ADR — all three, or skip it

1. **Hard to reverse** — changing your mind later costs something real.
2. **Surprising without context** — a future reader would ask "why on earth is it this way?"
3. **A real trade-off** — genuine alternatives existed and one was picked for reasons.

Unlike the methodology skills (whose examples are fictional — `per ADR _common/0002`), ADRs are **records**: they cite the real work that earned them.

## Template

```markdown
# ADR <scope>/NNNN · <decision in one line>

Date: YYYY-MM-DD · Status: accepted | superseded by <scope>/NNNN

## Context
1–3 sentences: the situation that forced a choice.

## Decision
1–3 sentences: what was decided.

## Why
1–3 sentences: the trade-off, and why this side of it.
```

Optional `Considered options` / `Consequences` only when they earn their lines.
