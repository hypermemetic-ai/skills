# How we reconcile scope/execution tickets with landed code

When a milestone has been building for a while and the tickets' names, diagrams, and DAGs may lag what the code actually says — the "find and resolve inconsistencies" sweep.

## Steps

1. Establish the landing shape first ([orient-on-a-pr-stack](orient-on-a-pr-stack.md)) — you reconcile against branches, not against ticket prose.
2. Inventory the **names**: every type, function, column, edge, and state the scope/execution mentions. `git grep` each one on the branch where it should live; record the landed form.
3. Sweep the scope: diagrams and contract tables get the landed names (the classic class: a diagram label still says the *deprecated* key where the canonical one belongs — check edge labels especially, they encode direction-of-meaning).
4. Sweep the execution: struck builds actually struck from the DAG, decision gates updated to the latest re-decision, the Status section describing the real PR chain.
5. Chase every supersession to its terminal state: archived ticket → pointer comment → parent DAG updated → **orphan draft PRs closed**.
6. Cross-milestone contract check: for each type the milestone consumes, confirm (a) the producer's ticket *names* it (not just a blocking link), and (b) no sibling milestone already landed a type for the same concept.

## Gotchas (each earned)

- An archival comment that promises "the DAG drops this node" is not the DAG dropping the node — *earned by:* an archived concurrency-guard build whose execution ticket still routed `B7 → B8 → B4` two days later, and whose draft PR (#985) stayed open after the comment said it could close.
- Diagram edge labels rot worse than prose — *earned by:* a scope payoff diagram carrying `user_id (immutable)` where `user_id` was the *mutable email being deprecated*; three diagrams in one ticket had the same class of error.
- A dependency edge is not a contract — *earned by:* a milestone blocked on an upstream "output" (`AuthedIdentity`) that no upstream ticket named; the dependency edge existed, the producer-side `Provides` didn't (a missing root, found only by reading the producer's scope).
- Check for duplicate type owners before freezing a surface — *earned by:* a frozen contract minting `TenantID int64` while a sibling stack had already landed a validated `Tenant` newtype in the same package (`per ADR _common/0003`).
- Verify the version/mechanism claims, not just names — *earned by:* a scope table promising a v7 id where the landed migration was DB-default v4 (a recorded, ratified fallback the scope never absorbed).

## Divergence notes
*(append-only — when a later run diverges, record what and why)*
