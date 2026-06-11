---
name: create-plexus-backend
description: Use when the user asks to scaffold a new Plexus RPC backend, create a WebSocket JSON-RPC 2.0 server with synapse CLI compatibility, or set up activations/plugins for a Plexus hub. Drives `axon new` (the plexus-axon scaffolder — compiled, anti-rot-tested templates), then guides extending the scaffold with additional activations.
---

# Skill: Create a Plexus RPC Backend

Scaffold a new Plexus RPC backend — a WebSocket JSON-RPC 2.0 service with
streaming responses, auto-generated schemas, and synapse CLI compatibility.

**The mechanism is `axon new`** (binary `axon`, crate `plexus-axon`, repo
`plexus-cli` in the hypermemetic workspace). Do not hand-assemble the file
tree: axon's templates are embedded in a tested binary whose test suite
scaffolds real crates and `cargo build`s them, so they cannot silently rot
the way copy-paste templates do (this skill's previous template set died
exactly that death — plexus 0.3 pins against a 0.5 world).

## Prerequisites

- Rust toolchain (edition 2021)
- `axon` on PATH, or built from the workspace:
  `cd plexus-cli && cargo install --path .` (not yet on crates.io;
  once published: `cargo install plexus-axon`)
- `synapse` for calling the service (Hackage: `cabal install plexus-synapse`)

## Scaffold

```bash
axon new <name> [--port <p>] [--with-auth] [--dev-layout] [--no-git] [--dir <parent>]
```

Real run (transcript):

```
$ axon new zecho --port 4452
Created ./zecho

  crate:      zecho
  backend:    zecho   (hub root — hyphen-free for synapse)
  activation: greeter   (distinct from the root by construction)
  auth:       ANONYMOUS — provisional default; re-run with --with-auth to wire validation
  git:        initialized (skip with --no-git)

Next:
  cd zecho
  cargo run
  synapse -P 4452 zecho greeter greet --name World
  synapse-cc build        # typed client (server must be running)
```

Flag guide:

- `--port <p>` — default 4444 (synapse's default `-P`). Pick a free port in
  4440–4459 (`synapse _self scan` scans that range).
- `--with-auth` — wires a transport `SessionValidator` (static bearer token
  via `--auth-token` / `PLEXUS_AUTH_TOKEN`) plus an auth-required example
  method. Without it the service is **anonymous and says so in a startup
  WARN**.
- `--dev-layout` — sibling-checkout path deps (`../plexus-core` …) instead of
  crates.io pins. Use inside the hypermemetic workspace; registry pins are
  the default. Note: registry-pinned scaffolds build against published
  `plexus-transport`, which does not yet include boot-time registry
  self-registration — call them with explicit `-P`; dev-layout scaffolds
  self-register and resolve by bare name.
- `--no-git` — skip the default `git init`.

## Verify (all transcript-checked)

```bash
cd zecho
cargo build          # compiles as emitted — zero manual edits
cargo run            # serves ws://127.0.0.1:4452

# In another terminal:
synapse -P 4452 zecho greeter                      # list the activation's methods
synapse -P 4452 zecho greeter greet --name World   # streams one greeting event
```

Expected output of the `greet` call:

```
message: Hello, World! This service was scaffolded by `axon new`.
name: World
type: greeting
```

Typed client: the scaffold ships `synapse.config.json` pre-pointed at the
service, so with the server running `synapse-cc build` works unedited
(generate → install → build → smoke tests).

## Extending the scaffold (the part the skill still owns)

`axon new` emits one example activation (`greeter`). Add your own:

1. Create `src/<plugin>.rs` modeled on `src/greeter.rs`: an event enum with
   `#[serde(tag = "type", rename_all = "snake_case")]` (always include an
   `Error { message: String }` variant), and an impl block annotated
   `#[plexus_macros::activation(namespace = "<plugin>", version = "...", description = "...")]`.
2. Register it in `main.rs`: `.register(YourPlugin)` on the `DynamicHub`.
3. `cargo build && cargo run`, then `synapse -P <port> <backend> <plugin>`.

Conventions (unchanged from the template era — they describe the macro
surface, not the dead templates):

- **Method docs go in `///` comments, not in the macro.** The doc comment
  above each method (and above each parameter, on current plexus-macros) is
  its schema description. The activation-level `description` lives in the
  macro attribute — and must stay **under 15 words** (plexus-macros rejects
  longer; the limit is otherwise undocumented).
- **Method return type:** every method returns
  `impl Stream<Item = YourEvent> + Send + 'static`; single-response methods
  use `stream! { yield ... }`.
- **The hub root name must differ from every activation namespace.**
  `DynamicHub::new("<root>")` panics at registration when an activation
  namespace equals the root (a like-named child would be unreachable —
  Z2H-8). axon enforces this by construction (`<name>` normalized
  hyphen-free for the root, activation `greeter` refused as a crate name);
  preserve the distinction when renaming. A `<name>_hub` root is a fine
  pattern when you want the bare name for an activation.
- **Hub registration:** `.register(activation)` for leaves,
  `.register_hub(...)` for hubs implementing `ChildRouter`. Namespaces must
  be unique within a hub.
- **Strong types at the boundary:** method params and event fields are the
  wire contract — apply the `strong-typing` skill, not bare `String`/`i64`.

For activations that need to call back into the hub, build it cyclically:

```rust
Arc::new_cyclic(|weak_hub: &Weak<DynamicHub>| {
    plugin_a.inject_parent(weak_hub.clone());
    DynamicHub::new("myservice_hub")   // root ≠ any activation namespace
        .register(plugin_a)
})
```

## Checklist

- [ ] `axon new <name>` run (port chosen from 4440–4459, free)
- [ ] `cargo build` succeeds with zero edits
- [ ] `cargo run` + `synapse -P <port> <backend> greeter greet --name World` round-trips
- [ ] Additional activations registered and visible via `synapse -P <port> <backend> <plugin>`
- [ ] At least one method round-trips a real value (not just the greeter)
- [ ] Hub root name ≠ every activation namespace

## Legacy templates

`templates/` in this skill directory is the pre-axon copy-paste scaffold.
It is **unverified and known-rotted** (it shipped plexus 0.3-era pins); it
is kept only as a reference for the file-tree shape and will be removed.
Do not generate new backends from it — use `axon new`. If you must read it,
note the builder was corrected to `DynamicHub::new("{{NAMESPACE}}_hub")`:
a root named identically to an activation namespace panics at registration.

## Pointers

- axon scaffolder: `plexus-cli/README.md` in the hypermemetic root (naming
  rules, anti-rot test design, auth posture, known constraints)
- Runnable minimal server: `plexus-rpc/examples/echo.rs`
- Strong-typing method params: `../strong-typing/SKILL.md`
- Tickets that scaffold a backend: `../ticketing/SKILL.md`
