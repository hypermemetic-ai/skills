# Recipes — "how we do X here"

A recipe is a **path**: a forward procedure with the gotchas baked in, each rule traced to the real wrong-turn that earned it. (Its point-decision sibling is the [ADR](../adr/README.md); the practice that produces both is [`skills/recipe`](../../skills/recipe/SKILL.md).)

## Which recipes live HERE vs in the worked-on repo

| The task is… | Recipe home |
|---|---|
| **project-bound** — touches one codebase's patterns (add a graph node, write a migration, a gRPC handler) | the worked-on repo's skills/docs area (e.g. `app-cm/.claude/skills/`) |
| **agent-bound / method-bound** — running the method itself, independent of any one codebase (orienting on a PR stack, operating the tracker at scale, reconciling tickets with landed code) | **this folder** (`per ADR _common/0004`) |

Distillation runs over *strictly agent-bound tasks* the same way it runs over execution graphs: when an agent task took non-obvious turns — wrong assumptions, dead ends, a mechanic discovered the hard way — that path distills here so the next session skips it.

## Organization

```
docs/recipes/
  README.md          ← this file
  _common/           ← any agent/person running the method
    <slug>.md
  sshmendez/         ← personal-workflow recipes
  <github-username>/ ← create your folder on your first recipe
```

Same scoping rules as [ADRs](../adr/README.md): `_common/` is for everyone; a personal recipe may tighten but not contradict a `_common` one. Slugs, not numbers — recipes are looked up by task, cite as `per recipe _common/<slug>`.

## Format (from the recipe skill)

```markdown
# How we <do X>

<one-line: when this recipe applies>

## Steps
1. <forward step> …

## Gotchas (each earned)
- <rule> — *earned by:* <the wrong-turn that taught it> (<link>)

## Divergence notes
*(append-only — when a later run diverges, record what and why)*
```

A recipe is **earned, never invented** — every rule traces to a real failure. Generic best-practices don't belong here.
