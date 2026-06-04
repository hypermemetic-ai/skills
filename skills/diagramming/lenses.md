# Diagramming · the lens set

A system is understood through *many* diagrams, each a different lens; no single one captures it. Reach for the lenses the work has — most non-trivial work wants several. Each below is *what it answers* + a small drawn starting shape (all vertical). Best practices + gotchas live in [SKILL.md](SKILL.md).

## Provenance / resolution flow
*Where does a value come from, through what hops?*
```mermaid
graph TD
  JWT["Auth0 token (sub, connection)"] --> TEN["tenant mw<br/>connection → tenant"]
  TEN --> RES["user-service<br/>resolve (sub, conn) → user_id"]
  RES --> AC["access context"]
```

## Graph abstraction
*What are the entities and how are they linked?*
```mermaid
graph TD
  U["user (sub, connection)"] -->|"BOUND_TO"| P["Person"]
  P -->|"has_participant_role"| PA["Participant"]
  U -.->|"cross-tenant · inter-tenant only"| U2["user · other tenant"]
```

## Ownership / boundary partition
*Where does the source of truth live, and what crosses a boundary?*
```mermaid
graph TD
  subgraph cm["cm_common · user-service"]
    DB[("users.user_id — CANONICAL")]
  end
  subgraph org["graph · org-service"]
    UV["user vertex — non-canonical reference"]
  end
  DB -->|"user_id (immutable)"| UV
```

## DAG — work / dependencies
*What's the work and in what order?*
```mermaid
graph TD
  S1["spike"] --> DEC{"decision gate"}
  DEC --> B1["build"]
  S2["spike"] --> B2["build"] --> B1
  B1 --> OUT["delivered → next milestone"]
```

## State / lifecycle
*What states exist and what moves between them?* (A state machine, not a flow.)
```mermaid
stateDiagram-v2
  [*] --> pending: invite
  pending --> active: first login
  active --> disabled: admin disable
  disabled --> active: re-enable
  active --> [*]: delete
```

## Payoff / contribution chain
*Why does this matter and where does it land?*
```mermaid
graph TD
  F["M2a · canonical user_id"] --> C["M2b · RBAC + surface users"]
  C --> PP["✅ users on the People page"]
```

## Decision gate
*What choice is open, on what evidence?*
```mermaid
graph TD
  Q{"remap vs fix?"}
  Q ==>|"0 FKs · email-load-bearing (S1)"| R["REMAP — chosen"]
  Q -.->|"if mostly opaque + FK'd"| FX["fix in place"]
```

## Before → after
*What exactly changes?*
```mermaid
graph TD
  OLD["today · query-time email match<br/>(fragile)"] ==>|"materialize"| NEW["explicit BOUND_TO edge<br/>(durable — email can change)"]
```

## Failure mode
*What goes wrong without this work — the path a value escapes on?*
```mermaid
graph TD
  Q2["query without row scope"] --> ALL["fetches all rows in tenant"]
  ALL -.->|"before the handler filter"| LEAK["leaks via logs / trace"]
  ALL --> POST["post-filter in handler"]
```

The set is open — add a lens when the work has one these don't capture.
