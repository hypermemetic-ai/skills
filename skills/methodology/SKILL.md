---
name: methodology
description: The operating methodology for agentic, multi-session software work — the spine tying the per-step skills together. Linear is the source of truth; the scope and execution ticket descriptions ARE the canonical documents, iterated in place. Read this to orient on the node kinds (scope, execution, spike, build), what each does and can become, the Linear state machine they run, and how a plan looks mid-flight.
---

# Skill: The Methodology

**The human decides; the agent executes; the ticket is the interface.** You open with a **barebones scope** — a few honest sentences of desire — and **grill** it against the durable records (glossary, ADRs) until it serves as a scope. Then one fork decides everything after: *is this small enough for one agent to build in one diff?* Yes → it's a **build**; implement it. No → it's a **DAG of smaller executions**, each re-entering the same fork. **Spikes** answer the unknowns that shape the sub-executions. That recursion is the whole method.

Two invariants every artifact obeys:

- **Linguistic belief state.** Every artifact carries its *value* (the contract, the type, the finding) AND the *evidence* that justifies it, so a contract questioned later starts from recorded reasoning, not cold.
- **Linear is the source of truth.** The scope and execution ticket **descriptions are the canonical documents** — long-form prose, evidence, and mermaid live there, and **we iterate there**: grill rounds edit the scope description; DAG changes land on the execution description in the same unit of work as the child tickets. There is no separate plans directory or frontmatter file layer. Evidence pins reality by link — commit-pinned GitHub permalinks, ticket links. (`cn-cm2-concepts` is an explainer layer, produced only on request — never the plan's home.)

## The node kinds — what each does, what each can become

```mermaid
graph TD
  IDEA["barebones scope<br/>statement of desire"] -->|"grill rounds<br/>iterated in Linear"| SCOPE
  SCOPE["SCOPE<br/>what/why · language ·<br/>interface contract"] -->|"spawns one"| EXEC
  EXEC["EXECUTION (epic)<br/>holds the DAG ·<br/>owns children · no own work"] --> SPIKE["SPIKE<br/>one unknown ·<br/>binary pass condition"]
  EXEC --> BUILD["BUILD (leaf)<br/>one diff · one agent ·<br/>from the ticket alone"]
  EXEC --> SUB["sub-EXECUTION<br/>recurse"]
  SPIKE -->|"evidence defines /<br/>descopes"| BUILD
  BUILD -->|"too big on contact:<br/>PROMOTE"| SUB
  BUILD -->|"superseded:<br/>cancel WITH pointer"| CXL["canceled"]
  SCOPE -->|"model shifts:<br/>superseded by new scope"| SCOPE2["new SCOPE<br/>old stays done, points forward"]
  classDef canon fill:#d9ead3,stroke:#38761d,color:#111;
  classDef ex fill:#cfe2f3,stroke:#0066cc,color:#111;
  classDef f fill:#fff2cc,stroke:#bf9000,color:#111;
  classDef x fill:#f4cccc,stroke:#cc0000,color:#111;
  class SCOPE,SCOPE2 canon;
  class EXEC,SUB,BUILD ex;
  class IDEA,SPIKE f;
  class CXL x;
```

| Kind | Does for itself | Can become | Skill |
|---|---|---|---|
| **Scope** | canonical *what/why*: language, decided shape, edges, **interface contract** the next milestone conditions on | superseded by a new scope (stays `done`, points forward); **never canceled** | [scope-ticket](../ticketing/scope-ticket.md) |
| **Execution** | epic holding the live DAG; owns spikes/builds/sub-executions as children; pure container — work lives only in leaves | collapsed (children absorbed elsewhere, with pointers) | [execution-ticket](../ticketing/execution-ticket.md) |
| **Spike** | resolves one unknown; binary pass condition; output is evidence (often a metric sheet) that sizes, defines, or descopes builds | nothing — it ends in a result; the result reshapes builds | [planning](../planning/SKILL.md) |
| **Build** | one diff a subagent implements from the ticket alone; declares `Provides`/`Consumes`; acceptance = its spike's metric sheet or its own criteria | **promoted** to a sub-execution when too big on contact; **canceled with a pointer** when superseded | [ticketing](../ticketing/SKILL.md) |
| **Plan node** | mints the next tranche of child tickets once the contract they condition on is ratified — planning in dependency order, shown in the DAG | nothing — it ends in tickets (the DAG's ghost nodes made real) | [execution-ticket](../ticketing/execution-ticket.md) |

Promotion is the size test firing, not a failure: "small enough for one agent from the ticket alone" *is* the build/execution boundary, reassessed on contact. Dotted names track the recursion: `m9.b3.b2` = milestone M9 → build B3 (promoted) → its sub-build B2.

## The state machine (Linear states, per kind)

One pipeline, four overlays. Builds run the full SDLC; the other kinds use a prefix of it.

```mermaid
graph TD
  T["Triage / new<br/>= proposed — HUMAN ratifies"] --> CQ["coding queue<br/>= ready"]
  CQ --> C["coding"]
  C --> ICR["in code review<br/>= gate green + PR open<br/>THE AGENT'S FINISH LINE"]
  ICR --> QA["qa testing"] --> PV["preview"] --> PR["prod"] --> D["done<br/>the WORK's finish line"]
  C -.->|"superseded"| X["canceled — only ever<br/>WITH a pointer"]
  classDef a fill:#cfe2f3,stroke:#0066cc,color:#111;
  classDef h fill:#fff2cc,stroke:#bf9000,color:#111;
  classDef d fill:#d9ead3,stroke:#38761d,color:#111;
  classDef x fill:#f4cccc,stroke:#cc0000,color:#111;
  class T,X h; class CQ,C,ICR a; class QA,PV,PR a; class D d;
```

| Kind | Convention |
|---|---|
| **Build** | full pipeline. `Triage` = contract written, awaiting human ratification (only the human moves it on). `in code review` = implemented: tests green, PR linked on the ticket. The pipeline — not the agent — carries it to `done`. |
| **Spike** | `coding` → `done` on a *result* — "a decision is still needed" is a valid result. |
| **Execution** | `coding` while any child is unresolved; `done` only when the completion gate passes (after the distill pass). May sit `in code review` as a deliberate human-approval gate. `canceled` propagates to children. |
| **Scope** | `done` the moment it has ≥1 work issue — done-by-definition, a living reference, **never canceled**. |

## An example tree (fictional milestone)

```
CM2|pitch|<pitch>                                  Linear project
└─ M9 · audit log                                  milestone
   ├─ Scope: M9 · audit log                        done · interface contract: AuditEvent, append-only store
   └─ Execution · M9 · audit log                   coding · holds the DAG
      ├─ S1 · spike: any writers bypassing the event bus?     done · "no — all 3 go through Publish()"
      ├─ B0 · AuditEvent type root                 done · Provides: AuditEvent, ActorRef
      ├─ B1 · writer adoption (3 sites)            in code review · PR #412 · Consumes: B0
      ├─ B2 · retention job                        coding queue · Consumes: B0
      └─ B3 · query API → PROMOTED                 Execution · M9.B3 · coding
         ├─ M9.B3.B1 · cursor pagination           done
         └─ M9.B3.B2 · filter grammar              coding
```

The shape to notice: the **shared contract (B0) is the root** every stream consumes — extracted first so no two siblings re-mint it; the spike ran **before** the builds that conditioned on its answer; B3 was a build until contact showed it contained its own DAG, then it was promoted and its children took the dotted names.

## Planning in progress — the same tree, mid-flight

Every event lands as Linear writes **in the same unit of work** — the DAG never lags the work:

| Moment | What happened | Linear writes, same pass |
|---|---|---|
| Spike lands | S1 finds no bypass writers — planned build B4 ("bypass shim") is unnecessary | S1 → `done` with the evidence in its description; B4 → `canceled` + comment pointing at S1; execution DAG strikes the node; scope's Decided-shape gains the why |
| Review surfaces work | B1's PR review finds the codec reads untyped props | new build B1.1 created under the execution; parent DAG + work list + decision gate updated; B1.1 cites the review as its sizing evidence |
| Build outgrows | B3's contract needs pagination *and* a grammar — two diffs | B3 promoted to `Execution · M9.B3`; sub-builds created as its children; parent DAG keeps B3 as one node |
| Contract questioned | a consumer asks why `ActorRef` has no email | answered from B0's `## Evidence` — recorded reasoning, no re-derivation |
| Exit | all leaves feed the completion gate | distill pass runs (ADRs/recipes for tracks that earn them); execution → `done`; scope's interface contract is what M10 reads |

The human ratifies at the forks (`Triage` → onward, promotions, cancellations, milestone boundaries); the agent sweeps the mechanics and keeps every parent artifact current.

## The loop

1. **Goal in** → barebones scope → **grill** ([grill](../grill/SKILL.md)): glossary + ADRs come in as the rubric; new language gets pinned (a new term is a candidate newtype). Iterate the scope description in Linear until it serves ([scope-ticket](../ticketing/scope-ticket.md) → "When is scoping done").
2. **Spawn the execution ticket**; apply the fork. One diff → a single build. More → grow the DAG: **shared contracts as roots**, spikes for unknowns, builds fanned out wide ([planning](../planning/SKILL.md)).
3. **Unknowns become spikes, not premature builds** — a `low`-confidence build must be blocked by one. Run spikes count-first under a token ceiling; evidence is a metric sheet whose command becomes the build's acceptance gate.
4. **Human ratifies** the builds (`Triage` → `coding queue`); agents implement in parallel where the DAG allows — one worktree per body of work, every worktree tracked ([building](../building/SKILL.md)); a build finishes at **PR-open** (`in code review`), and the pipeline lands it.
5. **Propagate every structural change** (add/remove/promote/re-sequence) onto the execution description in the same pass ([execution-ticket](../ticketing/execution-ticket.md) → "The DAG is live").
6. **Distill before concluding complete**: calibrate confidence priors; run the distill pass over the closed graph — ADRs for decision tracks that pass the earn-it test, recipes for recurring non-obvious paths ([recipe](../recipe/SKILL.md)). The next scope consumes these records instead of re-deriving them — the loop feeds itself.

## Durable records — what survives the loop

Tickets decay with their epic. Three records outlive it, each gated by an earn-it test (the anti-landfill mechanism), all living in the **worked-on repo**:

| Record | Captures | Born at | Earn-it test |
|---|---|---|---|
| **Glossary** (`CONTEXT.md`) | a term — "what do we call this" | scope (grilling) | project-specific concept, not general programming |
| **ADR** (`docs/adr/`) | one decision — "why this way" | execution (distill pass, or live) | hard to reverse ∧ surprising without context ∧ real trade-off |
| **Recipe** | a procedure — "how we do X here" | execution close (distill pass) | recurs ∧ path was non-obvious |

The glossary feeds the type system directly: the canonical term becomes the newtype's name (**grill → glossary → newtype**, [strong-typing](../strong-typing/SKILL.md)).

## The skills, by what they govern

- [planning](../planning/SKILL.md) — goal → DAG; the scope/execution split; spikes as evidence; metric-sheet gates; restructuring; calibration.
- [building](../building/SKILL.md) — ratified ticket → landed PR; one worktree per body of work, every worktree attributable and cleaned up with the landing; branch tree mirrors the DAG.
- [grill](../grill/SKILL.md) — iterate the scope from its barebones statement until it serves; one recommended-answer question at a time.
- [ticketing](../ticketing/SKILL.md) — one ticket passing the two-stranger test; `## Evidence` / `## Provides` / `## Consumes`.
- [diagramming](../diagramming/SKILL.md) — diagram-over-prose; the lens set; the reader's bandwidth is the bottleneck.
- [strong-typing](../strong-typing/SKILL.md) / [capability-types](../capability-types/SKILL.md) — evidence pushed into the type system; the security case where the type proves a check ran.
- [orient](../orient/SKILL.md) — session pickup: load these skills as the rubric, map branch → PR, verify live, report by severity.
- [recipe](../recipe/SKILL.md) — mine a finished execution into a "how we do X here" doc, each rule earned by a real wrong-turn.

## Cross-cutting disciplines

| Discipline | In one line |
|---|---|
| Diagrams from many lenses | Any human-facing artifact gets the densest faithful picture, from several lenses — never one diagram. |
| Pass/fail metrics are the unit of done | If a finding can't be written `metric → current → target → command`, it isn't finished; the command is the regression gate. |
| Bounded discovery | Spikes run count-first under an explicit ceiling; sub-investigations return conclusions, not file dumps. |
| Assume stale; verify live | Re-derive from Linear + the code, never forwarded prose; mark inferences `[confirm]`. |
| Human ratifies at the forks | Substantive Linear writes are proposed for approval; mechanical sweeps fan out to subagents. |
| Triage the tree before building | A dirty/unrecognized working tree or worktree belongs to *some* body of work — categorize (move to its ticket+branch / top of this DAG / ask) before you commit anything ([building](../building/SKILL.md)). |
| Pin every load-bearing fact | Pinned = a permalink, a prior ticket's output, a recorded "we looked." An unpinned load-bearing fact is a latent spike. |
| Pin before you parallelize | Parallel-by-default holds only across pinned edges — one wrong unpinned fact cascades into every dependent ticket. |
| Contract-first, fan-out | Model the DAG over shared contracts (capability → API → interface → type, by altitude): produce before consume, extract shared contracts as early roots. A root can't be re-minted by two siblings. |
| Provides/Consumes cross-check | Diff every `Consumes` against every `Provides` — a consume with no producer is a missing root; two producers of one structure is a duplicate owner. Mechanical, required. |
| Types reference their producers | Every consumed type in a contract cites the ticket that produces it — the dependency edge is readable from the ticket alone. |

## Pointers

- Repo overview: `../../README.md` · `../../AGENTS.md`
- Ticket body formats: [build](../ticketing/SKILL.md) · [scope](../ticketing/scope-ticket.md) · [execution](../ticketing/execution-ticket.md) · bulk ops: [BULK_OPS](../ticketing/BULK_OPS.md)
