---
name: diagramming
description: Use whenever you construct information for a human to consume — a plan, ticket, concept doc, review, explanation. Prioritize the highest-density faithful representation: a diagram compresses language into an image. Diagram over prose, draw a system from several lenses (see lenses.md), pick the form that compresses the data best, and avoid the layout gotchas that break rendering. Referenced by planning, ticketing, and concept-mapping.
---

# Skill: Diagramming — high-density representations for human consumption

**Any time you construct information *for a human*, prioritize the highest-density representation that's still faithful.** The reader's bandwidth is the bottleneck. Prose is low-density and serial; a diagram is high-density and parallel — its shape lands in one glance. The author pays the compression once; every reader saves it. Diagramming is the sharpest instance — the goal is broader: reach for the efficient representation by default.

## Best practices

- **Diagram over prose.** Lead with the diagram that carries the shape; reserve prose for what it can't say (rationale, caveats). A wall of prose describing a structure means it hasn't been understood *as a shape* yet — drawing it is the thinking.
- **Load-bearing, not decoration.** A reader should be able to reconstruct the design from the diagram. If it adds nothing over the text, cut it.
- **Many lenses.** A system is understood through several diagrams, each a different angle — never one. The lens set, with a drawn example of each → **[lenses.md](lenses.md)**.
- **Pick the densest faithful form.** Diagrams aren't the only high-density vector: a **table** for comparisons, a **pass/fail metric sheet** for quantitative findings, an **annotated example** over an abstract description. Use what compresses *this* data best.
- **Conventions.** Default **vertical** (`graph TD`); keep the diagram **in the artifact** where it renders (link, don't paste an unrenderable blob); **several small focused** diagrams beat one that says everything; use color / `classDef` to encode a dimension.

## Gotchas — layouts that break the render

- **Don't fan many siblings onto one rank.** `A → {nine children}` is a single row far wider than any column; the renderer scales it down until boxes and labels collide. Split into separate diagrams or pick a different shape. Grouping into a tree does **not** help — in a mermaid flowchart all the leaves still share one rank.
- **Keep subgraph titles short and one-line.** A long title that wraps on a single-rank (short) subgraph overflows *down onto its nodes* (real example: CMD-1769). Shorten the title, or stack the subgraph's nodes (`A ~~~ B ~~~ C`) so the box is tall enough for the title band to clear.
- **Smell test.** If a diagram needs three paragraphs to explain it, it's the wrong diagram; if a section is three paragraphs describing how things connect, it wants a diagram.

## Pointers

- Lens set + drawn gallery: **`lenses.md`** (sibling)
- Used by: `../planning/SKILL.md` · `../ticketing/SKILL.md` · `../methodology/SKILL.md`; concept-mapping (`cn-cm2-skills`)
