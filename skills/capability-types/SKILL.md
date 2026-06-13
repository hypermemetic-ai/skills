---
name: capability-types
description: Use when a value passed between modules represents an authorization, validation, or transactional decision that downstream code makes security or correctness choices on. Distinct from a plain newtype: the type's existence in scope is itself proof that an external check has occurred. Constructor is package-private, extraction is gated, default behavior fails closed, and wrong composition doesn't typecheck.
---

# Skill: Capability Types

**In one line:** a type whose very existence proves a check already ran — holding the value *is* the permission, and there's no way to construct one without passing the check.

A newtype carries evidence that a value is *well-formed* — `FormSlug` instead of `String` says "this string is a slug, not just any string." A capability type carries evidence that a *check has occurred* — `VerifiedUser` instead of `String` says "this value's existence in scope means the system has validated the user's identity."

The compiler is the witness. Holding the value is the capability. There is no path to producing the value that bypasses the check the type represents.

This is the type-system version of capability-based access control: instead of "does this caller have permission to do X," the question becomes "is this caller in possession of a value of type X-Capability?" The latter is decidable at compile time on every call site.

## When to use

Reach for a capability type rather than a plain newtype when:

- The value represents an **authorization** decision, not a parsing decision (the caller has been authenticated, has permission, holds a valid session)
- Downstream code makes security or correctness decisions based on whether a value of this type exists
- A handler-author forgetting to use it should be a **compile error**, not a review-time catch
- The wrong path through the system is "use the raw value before the check has happened" — and you want that path to be unreachable

If the answer to "what's the worst case if a junior dev skips this check" is "they leak data, escalate privileges, or violate an invariant," the check belongs in a capability type.

## Inputs

| Input | Description | Example |
|-------|-------------|---------|
| Capability concept | The check or decision being captured | "user identity has been verified" |
| Constructor authority | Which module is the only one allowed to mint instances | The auth middleware, the validation layer |
| Extraction protocol | How callers access the wrapped value (and what they must pass to do so) | `Unwrap(authzContext)`, `VerifiedSub()` |
| Default behavior | What happens when the wrapped value is used implicitly (serialized, converted, logged) | Mask, zero, error |

## Defining properties

A capability type has all four. Missing any one weakens the guarantee.

### 1. Constructor is package-private

Only the module that owns the check can mint an instance. Handler authors, downstream code, or library consumers cannot fabricate one. In Go, this is an unexported struct field plus an unexported constructor; in Rust, a private field plus a `pub(crate)` constructor; in TypeScript, a class with a private constructor and a static factory called only from the auth/validation module.

```go
package auth

type VerifiedUser struct {
    sub string  // unexported
}

// Package-private constructor. The auth middleware is the only caller.
func newFromValidatedJWT(sub string) VerifiedUser { ... }
```

### 2. Extraction is gated

The wrapped value isn't naively unpackable. Access goes through methods that may take additional context as input. The method's signature documents what's required to extract.

```go
// Returns the verified subject claim.
func (u VerifiedUser) VerifiedSub() string { return u.sub }

// Or, more strongly: extraction takes context.
func (f Field[T]) UnwrapFor(ac AccessContext) T { ... }
```

A getter that takes nothing and returns the raw value is the same as making the field public. If extraction must depend on caller context, encode that in the signature.

### 3. Default behavior fails closed

Implicit usage — JSON serialization, string formatting, logging, debug printing — defaults to a safe value. Exposure is opt-in via an explicit method, not opt-out via remembering to mask.

```go
// MarshalJSON refuses to serialize the raw value.
func (f Field[T]) MarshalJSON() ([]byte, error) {
    return json.Marshal("********")
}
```

The contrapositive: a handler that does nothing produces masked output. Forgetting to mask becomes "no output," not "leaked output." The failure mode is loud, not silent.

### 4. Wrong composition doesn't typecheck

Downstream functions accept the capability type, not the raw value. Calling them with a string or a different newtype is a compile error.

```go
// CS-prone:
func ResolveAccess(userEmail string, ...) { ... }

// CS-impossible: signature won't accept a string.
func ResolveAccess(user VerifiedUser, ...) { ... }
```

The change is invasive on purpose. Every call site that passed the raw value now has to either obtain a capability type (going through the gated path) or fail to compile. There is no third option.

## Process

### 1. Identify the check

What's the prerequisite that downstream code is currently assumed-but-unverified to depend on? "User has been authenticated." "Tenant scope has been resolved." "Input has been sanitized." "Transaction has been opened."

Each of these is a capability candidate.

### 2. Locate the natural constructor site

Where does the check actually happen? Auth middleware, validation layer, transaction begin, deserialization boundary. That module owns the constructor.

### 3. Define the type

Unexported wrapped field. Package-private constructor in the owning module. Public extraction method(s) — gated by context where appropriate. Custom serialization or default-behavior overrides that fail closed.

### 4. Migrate signatures bottom-up

Start at the most-downstream consumer of the value. Change its signature from `string` to the capability type. Follow compile errors upstream. Each error is a forced decision: either the caller already has the capability (good), or it has a raw value and needs to obtain the capability (the migration's whole point).

### 5. Delete the unsafe path

The old function that took a raw value either gets removed or marked deprecated. If the migration is partial, the unsafe variant lingers — and so does the bug class. Finishing the migration is what closes the door.

### 6. Tests assert the invariant, not the implementation

Property-based or invariant tests asserting "for all inputs, the access decision is a function of the capability type only" are stronger than tests asserting specific call paths work. The capability type's value is that *any* caller's wrong path doesn't compile; the test should encode the rule, not enumerate paths.

## What NOT to capability-type

- **Free-text human-readable fields** (names, descriptions, titles)
- **Values that don't carry authorization meaning** (timestamps, sizes, quantities, content) — these are newtype candidates, not capability candidates
- **Local-only intermediate values** that never cross a function boundary or a security/trust boundary
- **Cases where the check is genuinely cheap and idempotent and re-validation is fine** — a pure parse function is a newtype, not a capability

The cost of a capability type is the migration plus the architectural commitment ("this check happens here, never there"). The benefit is that a class of bugs becomes uncompilable. Weigh accordingly.

## Layered capability types

A useful pattern in security-critical systems: each layer has a capability type, and each layer's type takes the previous layer's type as its constructor input.

| Layer | Capability type | Constructor takes |
|-------|----------------|-------------------|
| Auth | `VerifiedUser` | Validated JWT |
| Tenant | `TenantScope` | `VerifiedUser` |
| Row scope | `AccessContext` | `VerifiedUser`, `TenantScope` |
| Field | `MaskedField[T]` | `AccessContext` (for unwrap) |

The downstream layers cannot exist without the upstream layer's check having occurred. The compiler enforces composition correctness. A handler that doesn't go through the auth middleware can't construct an `AccessContext`; therefore it can't unwrap a `MaskedField`. The wrong path doesn't compile, and the architecture is the type system.

## The boundary check

Capability validation belongs at exactly one place — the constructor's module. Inside the system, the type IS the proof. If you find yourself re-checking the capability deep in the call stack, either:

- The constructor isn't really validating (fix that), or
- The capability is too coarse and needs to be split (e.g., `VerifiedUser` and `VerifiedAdmin` as distinct types).

A capability type that demands re-validation by every caller has lost its claim to be a sufficient statistic. The whole point is that downstream code conditions on the type and stops asking.

## Pointers

- Convention: `../../AGENTS.md` → "Conventions › Capability Types"
- The plain-newtype version of this idea: `../strong-typing/SKILL.md`
- How tickets reference these in contracts: `../ticketing/SKILL.md`
