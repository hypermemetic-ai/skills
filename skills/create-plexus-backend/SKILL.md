---
name: create-plexus-backend
description: Use when the user asks to scaffold a new Plexus RPC backend, create a WebSocket JSON-RPC 2.0 server with synapse CLI compatibility, or set up activations/plugins for a Plexus hub. Generates the file tree from `templates/` by substituting `{{PROJECT_NAME}}`, `{{NAMESPACE}}`, `{{PORT}}`, `{{PLUGIN}}`, etc.; supports the 4440–4459 port convention and MCP HTTP bridge.
---

# Skill: Create a Plexus RPC Backend

Scaffold a new Plexus RPC backend — a WebSocket JSON-RPC 2.0 service with streaming responses, auto-generated schemas, and synapse CLI compatibility. The skill is a navigation guide; the actual code lives in `templates/` and is produced by token substitution.

## Prerequisites

- Rust toolchain (edition 2021)
- Access to plexus crates: `plexus-core`, `plexus-macros`, `plexus-transport` — published on crates.io, but local `[patch.crates-io]` overrides are common during development inside the hypermemetic workspace

## Inputs (collect from the user before generating)

| Token | Description | Example |
|-------|-------------|---------|
| `{{PROJECT_NAME}}` | Crate / binary name (kebab-case) | `plexus-dispatch` |
| `{{PROJECT_NAME_SNAKE}}` | Crate name as a Rust identifier | `plexus_dispatch` |
| `{{NAMESPACE}}` | Top-level hub namespace (what synapse sees) | `dispatch` |
| `{{PORT}}` | Default WebSocket port (4440–4459 range) | `4450` |
| `{{DESCRIPTION}}` | One-line project description | `Cannabis dispatch management backend` |
| `{{PLUGIN}}` | Activation type name (PascalCase, per plugin) | `Orders` |
| `{{PLUGIN_SNAKE}}` | Plugin namespace (snake_case, per plugin) | `orders` |
| `{{PLUGIN_DESCRIPTION}}` | One-line plugin description | `Order intake and routing` |

## Templates

All scaffolding lives in `templates/`. Token names match the table above.

| Template | Destination | Purpose |
|----------|-------------|---------|
| `Cargo.toml.template` | `<project>/Cargo.toml` | Dependencies + `[patch.crates-io]` block (commented) |
| `main.rs.template` | `<project>/src/main.rs` | CLI args, tracing, transport setup (stdio + WebSocket + MCP HTTP) |
| `lib.rs.template` | `<project>/src/lib.rs` | Public exports |
| `builder.rs.template` | `<project>/src/builder.rs` | `build_rpc()` → `Arc<DynamicHub>`, registers activations |
| `activations_mod.rs.template` | `<project>/src/activations/mod.rs` | Re-exports each plugin |
| `plugin_mod.rs.template` | `<project>/src/activations/<plugin>/mod.rs` | Per-plugin re-exports |
| `activation.rs.template` | `<project>/src/activations/<plugin>/activation.rs` | `#[plexus_macros::activation]` impl with one `#[plexus_macros::method]` example |
| `types.rs.template` | `<project>/src/activations/<plugin>/types.rs` | Event enum with `Ok` and `Error` variants |

## Resulting structure

```
{{PROJECT_NAME}}/
  Cargo.toml
  src/
    main.rs
    lib.rs
    builder.rs
    activations/
      mod.rs
      {{PLUGIN_SNAKE}}/
        mod.rs
        activation.rs
        types.rs
```

For multiple plugins: repeat the per-plugin templates and add each to `activations/mod.rs` and `builder.rs`.

## Builder patterns

`builder.rs.template` covers the simple case (independent activations). For activations that need to call back into the hub, use `Arc::new_cyclic`:

```rust
Arc::new_cyclic(|weak_hub: &Weak<DynamicHub>| {
    plugin_a.inject_parent(weak_hub.clone());
    plugin_b.inject_parent(weak_hub.clone());
    DynamicHub::new("{{NAMESPACE}}")
        .register(plugin_a)
        .register(plugin_b)
})
```

## Conventions

- **Method docs go in `///` comments, not in the macro.** The doc comment above each method is its description in the generated schema. The `#[plexus_macros::method(...)]` attribute carries only `params(name = "...")` per parameter, and `streaming` for multi-yield. Do NOT pass `description = "..."` to the method macro — the doc comment IS the description. (The activation-level `description` on the impl block does live in the macro, since impl blocks can't carry a `///` comment as a description.)
- **Method return type:** every method returns `impl Stream<Item = YourEvent> + Send + 'static`. Even single-response methods use `stream! { yield ... }` — the framework handles it. Mark multi-yield methods with `#[plexus_macros::method(streaming, params(...))]`.
- **Event variants:** `#[serde(tag = "type", rename_all = "snake_case")]` produces discriminated unions synapse can template. Always include an `Error { message: String }` variant for in-band errors.
- **Hub registration:** `.register(activation)` for leaves, `.register_hub(activation)` for hubs that implement `ChildRouter` for nested paths. Namespaces must be unique within a hub.
- **Port range:** Plexus backends use 4440–4459 (synapse `_self scan` scans this range). Pick an unused port.
- **Strong types at the boundary:** activation method parameters are part of the wire contract — apply the `strong-typing` skill to method params and event fields, not bare `String`/`i64`.

## Verification

```bash
cd <project>
cargo build
cargo run -- -p {{PORT}}                              # start the server
synapse -P {{PORT}} {{NAMESPACE}}                     # list plugins
synapse -P {{PORT}} {{NAMESPACE}} {{PLUGIN_SNAKE}}    # list methods
synapse -P {{PORT}} {{NAMESPACE}} {{PLUGIN_SNAKE}} hello --message "test"
```

If the MCP HTTP bridge is enabled (default), the MCP endpoint is at `http://127.0.0.1:{{PORT}}+1/mcp`.

## Checklist

- [ ] All input tokens collected from the user
- [ ] File tree generated from templates with substitutions applied
- [ ] `[patch.crates-io]` block uncommented if inside hypermemetic workspace
- [ ] `cargo build` succeeds
- [ ] `cargo run -- -p {{PORT}}` starts and responds to `synapse -P {{PORT}} {{NAMESPACE}}`
- [ ] At least one method round-trips a real value (not just `hello`)

## Pointers

- Strong-typing method params: `../strong-typing/SKILL.md`
- Tickets that scaffold a backend: `../ticketing/SKILL.md`
- Reference standalone backend: `hyperforge/src/bin/hyperforge.rs` in the hypermemetic root
