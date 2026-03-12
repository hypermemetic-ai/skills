# Skill: Create a Plexus RPC Backend

Create a new Plexus RPC backend server — a WebSocket JSON-RPC 2.0 service with streaming responses, auto-generated schemas, and synapse CLI compatibility.

## Prerequisites

- Rust toolchain (edition 2021)
- Access to plexus crates: `plexus-core`, `plexus-macros`, `plexus-transport`
- The crates are published on crates.io but local `[patch.crates-io]` overrides are common during development

## Inputs

Before starting, gather from the user:

| Input | Description | Example |
|-------|-------------|---------|
| `PROJECT_NAME` | Crate/binary name | `plexus-dispatch` |
| `NAMESPACE` | Top-level hub namespace (what synapse sees) | `dispatch` |
| `PORT` | Default WebSocket port | `4450` |
| `DESCRIPTION` | One-line project description | `Cannabis dispatch management backend` |
| `PLUGINS` | List of activations to scaffold | `orders`, `inventory`, `drivers` |

## Step 1: Create Project Structure

```
{PROJECT_NAME}/
  Cargo.toml
  src/
    main.rs                    # CLI args, transport setup, serve
    lib.rs                     # Public exports
    builder.rs                 # build_rpc() → Arc<DynamicHub>
    activations/
      mod.rs                   # Re-exports all activations
      {plugin}/
        mod.rs                 # Re-exports
        activation.rs          # #[hub_methods] impl
        types.rs               # Event types
```

## Step 2: Write Cargo.toml

```toml
[package]
name = "{PROJECT_NAME}"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "{PROJECT_NAME}"
path = "src/main.rs"

[dependencies]
# Plexus framework
plexus-core = "0.3.0"
plexus-macros = "0.3.0"
plexus-transport = "0.1.0"

# Async runtime
tokio = { version = "1", features = ["full"] }
async-stream = "0.3"
futures = "0.3"

# Serialization & schema
serde = { version = "1", features = ["derive"] }
serde_json = "1"
schemars = { version = "1.1", features = ["derive"] }

# JSON-RPC server
jsonrpsee = { version = "0.26", features = ["server", "macros"] }

# CLI & logging
clap = { version = "4", features = ["derive"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "fmt"] }
chrono = "0.4"
anyhow = "1"

# If in the hypermemetic workspace, add local patches:
# [patch.crates-io]
# plexus-core = { path = "../plexus-core" }
# plexus-macros = { path = "../plexus-macros" }
# plexus-transport = { path = "../plexus-transport" }
```

## Step 3: Write main.rs

The entry point handles CLI args, logging, and transport setup. This is boilerplate — adapt the port and name.

```rust
use {PROJECT_NAME_SNAKE}::build_rpc;
use plexus_transport::TransportServer;
use clap::Parser;

#[derive(Parser, Debug)]
#[command(name = "{PROJECT_NAME}")]
#[command(about = "{DESCRIPTION}")]
struct Args {
    /// Run in stdio mode (MCP-compatible)
    #[arg(long)]
    stdio: bool,

    /// WebSocket port
    #[arg(short, long, default_value = "{PORT}")]
    port: u16,

    /// Disable MCP HTTP server
    #[arg(long)]
    no_mcp: bool,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let args = Args::parse();

    let filter = if args.stdio {
        tracing_subscriber::EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| tracing_subscriber::EnvFilter::new("warn"))
    } else {
        tracing_subscriber::EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| tracing_subscriber::EnvFilter::new(
                "warn,{PROJECT_NAME_SNAKE}=debug"
            ))
    };

    tracing_subscriber::fmt()
        .with_env_filter(filter)
        .with_writer(std::io::stderr)
        .init();

    let hub = build_rpc().await;

    // Log activations
    let activations = hub.list_activations_info();
    tracing::info!("Activations ({}):", activations.len());
    for a in &activations {
        tracing::info!("  {} v{} - {}", a.namespace, a.version, a.description);
        for method in &a.methods {
            tracing::info!("    - {}_{}", a.namespace, method);
        }
    }

    let rpc_converter = |arc| {
        use plexus_core::plexus::DynamicHub;
        DynamicHub::arc_into_rpc_module(arc)
            .map_err(|e| anyhow::anyhow!("Failed to create RPC module: {}", e))
    };

    let mut builder = TransportServer::builder(hub, rpc_converter);

    if args.stdio {
        builder = builder.with_stdio();
    } else {
        builder = builder.with_websocket(args.port);
        if !args.no_mcp {
            builder = builder.with_mcp_http(args.port + 1);
        }
    }

    if args.stdio {
        tracing::info!("Starting stdio transport");
    } else {
        tracing::info!("WebSocket: ws://127.0.0.1:{}", args.port);
        if !args.no_mcp {
            tracing::info!("MCP HTTP:  http://127.0.0.1:{}/mcp", args.port + 1);
        }
    }

    builder.build().await?.serve().await
}
```

## Step 4: Write lib.rs

```rust
mod builder;
pub mod activations;

pub use builder::build_rpc;
```

## Step 5: Write builder.rs

The builder creates the `DynamicHub` and registers all activations.

**Simple case** (no cross-plugin references):

```rust
use std::sync::Arc;
use plexus_core::plexus::DynamicHub;
use crate::activations::*;

pub async fn build_rpc() -> Arc<DynamicHub> {
    Arc::new(
        DynamicHub::new("{NAMESPACE}")
            .register({Plugin1}::new())
            .register({Plugin2}::new())
            // ... more activations
    )
}
```

**Complex case** (activations need to reference each other via the hub):

```rust
use std::sync::{Arc, Weak};
use plexus_core::plexus::DynamicHub;
use crate::activations::*;

pub async fn build_rpc() -> Arc<DynamicHub> {
    let plugin_a = PluginA::new().await;
    let plugin_b = PluginB::new(plugin_a.shared_state()).await;

    Arc::new_cyclic(|weak_hub: &Weak<DynamicHub>| {
        plugin_a.inject_parent(weak_hub.clone());
        plugin_b.inject_parent(weak_hub.clone());

        DynamicHub::new("{NAMESPACE}")
            .register(plugin_a)
            .register(plugin_b)
    })
}
```

## Step 6: Write an Activation (Plugin)

Each activation needs three files. Here's the pattern:

### types.rs — Event Types

Event types are plain Rust enums. The `#[serde(tag = "type")]` attribute creates discriminated unions that synapse can template.

```rust
use schemars::JsonSchema;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum {Plugin}Event {
    /// Description of this variant
    {Variant1} {
        field1: String,
        field2: u64,
    },
    /// Another variant
    {Variant2} {
        message: String,
    },
    /// Error variant (convention)
    Error {
        message: String,
    },
}
```

### activation.rs — The Activation Implementation

The `#[hub_methods]` macro generates all Plexus boilerplate: RPC trait, Activation trait impl, method enum, JSON schema.

```rust
use super::types::{Plugin}Event;
use async_stream::stream;
use futures::Stream;

#[derive(Clone)]
pub struct {Plugin} {
    // state fields
}

impl {Plugin} {
    pub fn new() -> Self {
        Self { /* ... */ }
    }
}

#[plexus_macros::hub_methods(
    namespace = "{plugin_snake}",
    version = "1.0.0",
    description = "{Plugin description}"
)]
impl {Plugin} {
    /// Method description shown in synapse help
    #[plexus_macros::hub_method(
        description = "What this method does",
        params(
            param1 = "Description of param1",
            param2 = "Description of param2 (optional)"
        )
    )]
    async fn method_name(
        &self,
        param1: String,
        param2: Option<u32>,
    ) -> impl Stream<Item = {Plugin}Event> + Send + 'static {
        stream! {
            yield {Plugin}Event::{Variant1} {
                field1: param1,
                field2: param2.unwrap_or(0) as u64,
            };
        }
    }

    /// Streaming method (yields multiple events)
    #[plexus_macros::hub_method(
        streaming,
        description = "Streams multiple results",
        params(query = "Search query")
    )]
    async fn search(
        &self,
        query: String,
    ) -> impl Stream<Item = {Plugin}Event> + Send + 'static {
        stream! {
            // Yield multiple events over time
            for item in do_search(&query).await {
                yield {Plugin}Event::{Variant1} { /* ... */ };
            }
        }
    }
}
```

### mod.rs — Module Re-exports

```rust
mod activation;
mod types;

pub use activation::{Plugin};
pub use types::{Plugin}Event;
```

### activations/mod.rs — Top-level Re-exports

```rust
pub mod {plugin1};
pub mod {plugin2};

pub use {plugin1}::{Plugin1};
pub use {plugin2}::{Plugin2};
```

## Key Rules

### Method Return Types
- **All methods** return `impl Stream<Item = YourEvent> + Send + 'static`
- Even single-response methods use `stream! { yield ... }` — the framework handles it
- Mark multi-yield methods with `#[plexus_macros::hub_method(streaming)]`

### Event Type Conventions
- Use `#[serde(tag = "type", rename_all = "snake_case")]` — this creates discriminated unions
- Include an `Error { message: String }` variant for in-band errors
- Each variant becomes a distinct JSON object with `"type": "variant_name"`

### Hub Registration
- `.register(activation)` — leaf activation (no children)
- `.register_hub(activation)` — hub activation that implements `ChildRouter` for nested paths
- Namespace must be unique within a hub

### Synapse Integration
Once running, the backend is accessible via synapse:
```bash
synapse -P {PORT} {NAMESPACE} {plugin} {method} --param value
```

Schema introspection works automatically:
```bash
synapse -P {PORT} {NAMESPACE}                    # List plugins
synapse -P {PORT} {NAMESPACE} {plugin}            # List methods
synapse -P {PORT} {NAMESPACE} {plugin} {method}   # Show help (if required params missing)
```

### Port Convention
Plexus backends use ports in the 4440-4459 range (synapse `_self scan` scans this range). Pick an unused port.

## Checklist

- [ ] `Cargo.toml` with plexus dependencies (and `[patch.crates-io]` if in hypermemetic)
- [ ] `main.rs` with CLI args, tracing, transport setup
- [ ] `lib.rs` exporting `build_rpc`
- [ ] `builder.rs` constructing `DynamicHub` with all activations registered
- [ ] At least one activation with `types.rs` + `activation.rs` + `mod.rs`
- [ ] `activations/mod.rs` re-exporting all plugins
- [ ] `cargo build` succeeds
- [ ] `cargo run -- -p {PORT}` starts and responds to `synapse -P {PORT} {NAMESPACE}`
