---
name: strong-typing
description: Use when a codebase passes bare `String`, `i64`, or `Uuid` for distinct domain concepts; when ticket contracts reference domain values by their raw type instead of a named one; or when a bug was caused by passing one identifier where another was expected. Introduces newtypes (with `#[serde(transparent)]`) and enums so the compiler catches misuse the type system was previously allowing silently.
---

# Skill: Strong Type a Domain

Newtypes are the type system's version of a sufficient statistic. A function taking `FormSlug` doesn't re-validate that the string is a slug — the evidence that it *is* a slug is carried in the type. Every downstream caller can condition on that evidence without re-deriving it.

This is the same principle that ticket `## Evidence` sections apply at the documentation layer, but enforced by the compiler. Where ticket evidence asks reviewers to check the reasoning, type evidence asks the compiler to check it on every call site.

## When to use

When a codebase passes bare `String`, `i64`, or `Uuid` for values that have distinct domain meanings. When ticket contracts reference domain concepts by their raw type instead of a named type. When a bug was caused by passing one kind of identifier where another was expected.

## Inputs

| Input | Description | Example |
|-------|-------------|---------|
| Domain concepts | The distinct identifiers and constrained values in the domain | `FormSlug`, `WizardNodeId`, `IdentityRoot`, `ActivationNamespace` |
| Current representation | What Rust type they're currently stored as | `String`, `Uuid`, `i64` |
| Where they flow | Which modules pass these values around | auth → storage → activation → wire |

## Process

### 1. Inventory the domain concepts

For each `String` (or `Uuid`, `i64`) field that represents a specific domain entity, ask: "could I accidentally swap this with another field of the same raw type?" If yes, it needs a newtype.

| Pattern | Needs a newtype |
|---------|----------------|
| Two `String` parameters in a function where swapping them silently compiles | Yes |
| A database ID that could be confused with another table's ID | Yes |
| A constrained set of values ("alien", "citizen", "employer") | Yes — an enum, not a newtype |
| An opaque external identifier (OAuth token, session ID) | Yes |
| A human-readable free-text field (name, description) | Usually no |

### 2. Define the types

**Newtypes** for identifiers:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(transparent)]
pub struct FormSlug(pub String);

impl FormSlug {
    pub fn new(s: impl Into<String>) -> Self { Self(s.into()) }
    pub fn as_str(&self) -> &str { &self.0 }
}

impl std::fmt::Display for FormSlug {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.write_str(&self.0)
    }
}
```

**Enums** for constrained values:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum IdentityRoot {
    Alien,
    GreenCardHolder,
    Citizen,
    Employer,
    Attorney,
}
```

### 3. Apply bottom-up

Start at the storage layer (closest to the database), then work outward:

1. Database query return types (change `String` → `FormSlug` in `query_as` tuples)
2. Domain structs (change `pub slug: String` → `pub slug: FormSlug`)
3. Function parameters (change `fn get(slug: &str)` → `fn get(slug: &FormSlug)`)
4. Wire boundary (parse `String` → newtype at the activation layer)
5. Ticket contracts (reference `FormSlug`, not "a string containing a form slug")

### 4. Update ticket contracts

If tickets reference the domain concepts, update them to use the type names. "The table contains `WizardNodeId` values" not "the table contains text values." This makes contract violations between tickets visible at review time and lets `## Evidence` sections cite type names rather than re-explain shape.

## Required traits for every newtype

| Trait | Why |
|-------|-----|
| `Debug` | Logging, error messages |
| `Clone` | Value semantics |
| `PartialEq`, `Eq` | Comparison (HashMap keys, dedup) |
| `Hash` | HashMap/HashSet keys |
| `Serialize`, `Deserialize` | JSON wire format |
| `#[serde(transparent)]` | Serializes as bare string, not `{"0": "value"}` |
| `Display` | tracing fields, string interpolation |
| `new()`, `as_str()` | Construction and access |

## What NOT to wrap

- Free-text human-readable fields (names, descriptions, titles)
- Temporary/local values that never cross a function boundary
- Values already uniquely typed (`uuid::Uuid` for one specific entity, a custom enum)

## The boundary check

Newtype validation belongs at the wire boundary — once. Inside the system, the type IS the proof. If you find yourself re-validating a `FormSlug` deep in the call stack, either:

- The wire boundary isn't really validating (fix that), or
- The newtype is too permissive and needs an invariant the constructor enforces (`FormSlug::try_new` returning `Result<Self, ParseError>`).

A newtype that demands re-validation by every caller has lost its claim to be a sufficient statistic.

## Pointers

- Convention: `~/CLAUDE.md` → "Strong Typing Rule"
- How tickets reference types in contracts: `../ticketing/SKILL.md` → Rule 9
- Where to introduce them in a Plexus backend: at the activation method boundary in `<plugin>/types.rs` — see `../create-plexus-backend/SKILL.md`
