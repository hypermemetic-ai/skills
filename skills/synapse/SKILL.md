---
name: synapse
description: Use whenever you drive a Plexus backend through the `synapse` CLI (hyperforge/`lforge`, trak, substrate, …) and need to actually form a correct invocation — the flag/backend/navigation grammar that makes agents fumble. Covers where flags go, which registered name a daemon answers to, how to path-navigate to a method, how to list methods and discover a method's parameters from the CLI itself, and the dirty-tree guard. Read this before issuing synapse commands; it is the invocation contract, not a backend's auth or domain rules.
---

# Skill: Synapse (the invocation contract)

**In one line:** `synapse [OPTIONS] <backend> <path...> [--param value]` — flags go *before* the backend, a daemon answers to its *registered* name (not its product name), and you reach a method by *path-navigating* the hub, not by guessing.

The single most common failure is a misplaced flag or a wrong backend name producing the giant ASCII banner + usage dump, which reads like "the command is broken" when it actually means "synapse never saw your intent." This skill is the grammar that avoids that. It is tool plumbing — shared by every Plexus backend — and deliberately says nothing about any backend's auth model or domain methods (those live with the backend).

## When to use

- Any time you're about to run `synapse …` against a backend and want the call to land first try.
- Driving hyperforge (registered as `lforge`), trak, substrate, or any Plexus hub: listing methods, finding a method's parameters, navigating namespaces.
- Debugging a synapse call that printed the usage banner, "Backend not found", "'X' at root", "Unknown parameter", or a handshake timeout.

Don't use for:
- A backend's **auth / token** plumbing (who can call what, where the token is stored) — that's the credential layer and the backend's own rules.
- A backend's **domain semantics** (what a trak facet *means*, what a hyperforge org *is*) — that's the backend's skill.

## Inputs

| Input | Description | Example |
|-------|-------------|---------|
| Backend | The **registered** name, not the product name | `lforge` (the hyperforge daemon), `trak`, `substrate` |
| Path | The namespace-navigated method path under the hub root | `hyperforge repos pull`, `hyperforge orgs list` |
| Params | `--name value` pairs after the path | `--org hypermemetic-ai`, `--path ~/dev/x` |
| Global flags | Output/connection flags, placed **before** the backend | `-j` (json), `-i` (IR), `-s` (schema), `-H/-P` |

## Output

A correctly-formed `synapse` invocation that reaches the intended method — or, when exploring, the method list / parameter spec read straight from the live backend.

## Process

1. **Put global flags before the backend.** Usage is `synapse [OPTIONS] <backend> <path...>`. `synapse lforge -i` places `-i` *after* the backend, so synapse swallows it and prints the help banner. Correct: `synapse -i lforge`, `synapse -j lforge …`, `synapse -s lforge …`. **If you see the ASCII banner + usage block, your flag was misplaced or your backend name was wrong** — that's the tell, not a real error.
2. **Use the registered backend name, not the product name.** A daemon registers itself under a name that may differ from what it's called: the **hyperforge** daemon answers to **`lforge`**. `synapse hyperforge` → "Backend not found / not registered". Run bare `synapse` (no backend) to print the registry table — every backend, its host:port, and reachability (`[OK]`/`[UNREACHABLE]`).
3. **Navigate the hub by path; don't call a method at the root.** Methods sit under the hub's root plugin. For `lforge` that root is `hyperforge`, so it's `synapse lforge hyperforge <plugin> <method> --param value`. `synapse lforge orgs list` fails with "'orgs' at root" — go through the root plugin: `synapse lforge hyperforge orgs list`. Navigating one level at a time (`synapse lforge`, then `synapse lforge hyperforge`) prints the child plugins/methods available there.
4. **List every method with the IR.** `synapse -i <backend>` emits IR JSON; `irMethods` is a dict keyed by dotted path (`hyperforge.repos.pull`, `hyperforge.orgs.list`, …). This is the fastest full map of what a backend exposes.
5. **Discover a method's parameters by calling it bare.** Invoke the method with no params and it streams back its parameter spec — name, required/optional, default, doc. E.g. `synapse lforge hyperforge repos pull` prints `--path` (required), `--branch` (optional), `--remote` (default `origin`). Then fill them in. A wrong param name returns "Unknown parameter".
6. **Prefer the registry over raw host:port.** `synapse <backend> …` resolves through the registry. `-H host -P port` bypasses it but must speak the Plexus handshake; pointing `-P` at a raw daemon port often yields "Protocol handshake failed / _info did not complete". Use the registered name unless you have a specific reason not to.

## Rules

- **Flags before the backend, always.** `synapse -j lforge …`, never `synapse lforge -j …`. The banner is the symptom of breaking this.
- **Registered name, not product name.** Confirm against the bare-`synapse` registry table when unsure (hyperforge ⇒ `lforge`).
- **Navigate, don't guess the leaf.** Reach a method through its namespace path from the hub root; "'X' at root" means you skipped a level.
- **Params are individual `--flags`, not a JSON blob.** Each parameter is its own `--kebab-case-flag value` after the method path (`--parent-id X --status active`, list flags repeat: `--tags a --tags b`). There is no need to assemble a `-p '{json}'` argument — reach for an inline `'{…}'` value only when a *single* param is itself a nested JSON object (e.g. `--meta-extra '{"k":"v"}'`). Treat synapse as a normal CLI.
- **Let the backend tell you the shape.** `-i` for the method map, bare-call (or `--help`) for the parameter spec — don't hand-author params from memory.
- **Respect the dirty-tree guard.** hyperforge `repos.pull` refuses a dirty working tree (`code: dirty_tree`); commit or stash first, or fall back to raw `git pull` (the sanctioned fallback when hyperforge has no clean equivalent). Default `--remote` is `origin` — pass the remote explicitly when the target isn't origin.
- **This skill is invocation only.** Token/credential resolution and a backend's auth/domain rules are out of scope by design — keep them in the credential layer and the backend's own skill.

## Examples

**Good — listing what a backend exposes, then calling a method correctly:**

> `synapse -i lforge` → read `irMethods` keys → `hyperforge.repos.pull` exists. `synapse lforge hyperforge repos pull` (bare) prints its params. Then: `synapse lforge hyperforge repos pull --path ~/dev/skills --remote github --branch main`.

**Bad — flag after backend, wrong name, root-level leaf:**

> `synapse lforge --schema repos.pull` (flag after backend → prints the usage banner), or `synapse hyperforge orgs list` ("Backend not found" — the daemon is `lforge`), or `synapse lforge orgs list` ("'orgs' at root" — skipped the `hyperforge` namespace).

**Good — reading the registry when a call won't connect:**

> Bare `synapse` shows `lforge  127.0.0.1:44104 [OK]` and `trak 127.0.0.1:44107 [OK]` while others are `[UNREACHABLE]` — so the target is up and the issue is the invocation grammar, not the daemon.
