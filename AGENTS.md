# skills

House-style methodology skills for agentic, multi-session software work. Each skill
is a reusable practice that produces a particular artifact (a doc set, a ticket, a
finding) following a consistent process. This repo is **self-contained** — it
does not depend on any external skills repo.

## Skill file convention

- One skill = one `SKILL.md` under `skills/<skill-name>/`. Supporting files
  (e.g. `diagramming/lenses.md`) live in the same directory.
- Frontmatter has exactly `name` and `description` (used by the harness for
  skill registration). `description` is one specific paragraph: when to use it
  and what it produces.
- Body sections, at minimum: **When to use**, **Inputs**, **Output**,
  **Process**, **Rules**, **Examples**. Match the shape of
  `skills/orient/SKILL.md`.

## Adding or changing a skill

1. Create `skills/<name>/SKILL.md` in the house format above.
2. Add a row to the **Skills** table in `README.md` (Skill / Produces / Use when).
3. Skills are methodology only — process and rules, not code. Keep them
   project-agnostic; project-specific practices belong in the worked-on repo's
   own skills.

## Self-contained rule

Do **not** add references to external skill repositories or absolute paths into
another repo's skills. If a skill here needs another skill, either reference it
by relative path within this repo (`../<name>/SKILL.md`) or vendor that skill
in. Examples inside the methodology skills stay **fictional/generic** — never a
pointer to a specific live milestone or its tickets (the methodology must not
cite the work it governs; that circularity is what recipes and ADRs are for).

## Conventions (canonical rules)

The vendored methodology skills *carry the methodology*; this section *carries the rule* they defer to. When a skill says "see `../../AGENTS.md` → Conventions › X", this is X.

### Strong Typing Rule

Pass domain values as named types, not raw `string` / `int64` / `uuid`. A function taking a `FormSlug` doesn't re-validate that the string is a slug — the evidence is in the type. Two parameters of the same raw type that mean different things must be different named types; constrained value sets are enums, not strings. Validate at the wire boundary, once; inside the system, the type **is** the proof. (In Go: named types + constructors at the handler / gRPC entry; in Svelte/TS: branded types at the API boundary.)

### Capability Types

A capability type carries evidence that a *check has occurred* — its existence in scope is itself proof that the auth / validation / transactional check it represents has run. Distinct from a plain newtype; it has all four:

1. **Constructor is package-private** — only the module owning the check can mint instances.
2. **Extraction is gated** — access goes through methods that may take additional context; no naive unwrap.
3. **Default behavior fails closed** — implicit use (serialize, log, format) defaults to a safe value; exposure is opt-in.
4. **Wrong composition doesn't typecheck** — downstream functions accept the capability type, not the raw value.

Reach for one — not just a newtype — when the value represents an authorization decision downstream code makes security/correctness choices on, and a handler-author forgetting the check should be a **compile error**, not a review-time catch. Layer them so each layer's type takes the previous layer's as constructor input: the architecture becomes the type system.

### Direction-of-Impact Calibration (security review)

When triaging a finding of the form "X can be modified in unexpected ways", check the *direction* first:

- Modification *removes* privileges → logic bug, severity at most Medium.
- Modification *grants* privileges → vulnerability, severity per the standard table.

Pattern-matching bias makes any state-machine inconsistency in security-critical code look alarming; calibrate against it. Exception: silent, triggerable privilege *removal* is denial-of-service — track as Medium, don't dismiss.

## Git workflow

Keep `main` clean and its history linear:

- Land every change via a feature branch + PR — don't push to `main` directly.
- Keep history linear (rebase, don't merge-commit feature branches). Pushing to
  feature branches is unrestricted.
