# Skill: Strong Type a Domain

Introduce newtypes for every distinct domain concept so the compiler catches misuse that string-typing allows silently.

## When to use

When a codebase passes bare `String`, `i64`, or `Uuid` for values that have distinct domain meanings. When ticket contracts reference domain concepts by their raw type instead of a named type. When a bug was caused by passing one kind of identifier where another was expected.

## Inputs

| Input | Description | Example |
|-------|-------------|---------|
| Domain concepts | The distinct identifiers and constrained values in the domain | WizardNodeId, FormSlug, StateId, IdentityRoot |
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

1. Database query return types (change `String` → `FormSlug` in query_as tuples)
2. Domain structs (change `pub slug: String` → `pub slug: FormSlug`)
3. Function parameters (change `fn get(slug: &str)` → `fn get(slug: &FormSlug)`)
4. Wire boundary (parse `String` → newtype at the activation layer)
5. Ticket contracts (reference `FormSlug`, not "a string containing a form slug")

### 4. Update ticket contracts

If tickets reference the domain concepts, update them to use the type names. "The table contains `WizardNodeId` values" not "the table contains text values." This makes contract violations between tickets visible at review time.

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
- Values already uniquely typed (like `uuid::Uuid` or a custom enum)

## Pointers

- Convention: `~/CLAUDE.md` → "Strong Typing Rule"
- Existing types in FormVeritas: `Sub`, `Sid`, `ValidOrigin`, `SecureTransport`, `ClientIp`, `TenantScope`
- Planned types: `FormSlug`, `GoalId`, `ExtractionId`, `FieldPath` (IDN-35)
- Planned types for STF: `WizardNodeId`, `StateId`, `TransitionId`, `IdentityRoot`, `Pathway`, `FormRole`
