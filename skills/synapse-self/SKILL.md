---
name: synapse-self
description: Use for synapse's per-backend credential store — storing/resolving the token any Plexus backend reads. Triggers "where does synapse keep the token", "set a cookie/header", "synapse _self show/set/resolve", "defaults.json", "credential reference / literal:/env://file://keychain://", "token priority chain", "why is my token not picked up", "redact secrets in _self". This is the UNIVERSAL plumbing shared by every backend and by synapse-cc — it is NOT specific to any one backend. For ISSUING a token use `plexus-idp`; for a backend's auth/authorization rules use that backend's skill (e.g. `trak`).
---

# Skill: synapse `_self` — the per-backend credential store

`_self` is synapse's universal credential layer: how the CLI stores and resolves the token (and any cookies/headers) it attaches to a backend on the WebSocket upgrade. It is **identical for every backend** (trak, gamma, substrate, …) and shared byte-for-byte with `synapse-cc` (the `Synapse.Self` module family is the canonical implementation). It does **not** issue tokens (that's `plexus-idp`) and it does **not** define a backend's auth rules (that's the backend's validator + gate).

## Where credentials live

Per-backend store at `~/.plexus/<backend>/defaults.json` — e.g. `~/.plexus/trak/defaults.json`. Each cookie/header value is a **credential-reference URI**, not (usually) a raw secret:

| Scheme | Resolves to | Notes |
|---|---|---|
| `literal:<value>` | the value verbatim | the escape hatch; the secret sits in the file. `_self show` redacts these (see below) |
| `env://VAR` | the env var's value | |
| `file:///path` | file contents | trailing newline stripped |
| `keychain://svc/acct` | OS keychain item | macOS `security` CLI; Linux/Windows stubbed. Encryption at rest + commit-safe manifest, but NOT per-process sandboxing — any process as your user can read it |

`defaults.json` shape: `defaults.cookies.access_token` is the one synapse sends as `Cookie: access_token=<jwt>` on the upgrade. `defaults.headers.*` for custom headers. There may also be an `oidc` block (issuer/audience/refresh_token) used by login tooling.

## The `_self` commands

```bash
synapse _self <backend> show                         # inspect; decodes JWTs, flags expired tokens, REDACTS literal: secrets
synapse _self <backend> set cookie access_token "eyJ..."   # bare value auto-wraps as literal:
synapse _self <backend> set header X-API-Key "env://MY_API_KEY"
synapse _self <backend> resolve cookie access_token        # resolve a ref; secret MASKED unless you pass --reveal
synapse _self <backend> import-token ~/jwt.txt             # import a raw JWT file
synapse _self <backend> clear --yes                        # wipe the store
synapse _self <backend> upgrade-to-keychain                # SELF-8, macOS only
```

> **Secret redaction (current behavior).** `_self show` masks `literal:` secret bodies (`literal:<first6>...<last4>`) so tokens don't leak to terminals/logs; non-`literal:` refs (pointers) show in full. `_self resolve` masks the resolved value by default and prints it only with `--reveal`. (env/file/keychain refs are pointers and display as-is.)

## Priority chain (which token wins)

On each invocation, first match wins:

1. `--token <jwt>` flag
2. `SYNAPSE_TOKEN` env var
3. `--token-file <path>`
4. stored `cookies.access_token` in `defaults.json` ← **the native path; no flag needed**

Per-key `--cookie KEY=VALUE` / `--header KEY=VALUE` flags override stored values for that key. So if a call ignores your stored token, check for a `--token`/`SYNAPSE_TOKEN`/`--token-file` shadowing it higher in the chain.

## Native auth = store it, then drop `-t`

The point of `_self`: put a valid token in `defaults.json` once and every later `synapse -P <port> <backend> …` call authenticates with no `-t`. The `-t`/`--token-file` flags are overrides for one-off or scripted use.

```bash
# store a token natively (so subsequent calls need no -t):
synapse _self trak set cookie access_token "$(cat /path/to/jwt)"
synapse -P 44107 -j trak facet list      # authenticates from the store
```

## Gotchas

- **Legacy migration:** a pre-`_self` `~/.plexus/tokens/<backend>` file auto-migrates to `defaults.json` (as `literal:<jwt>`) on first read.
- **Tokens expire (~1 h for plexus-idp access tokens).** `_self` stores; it does not refresh. Re-issue via `plexus-idp` and re-`set` (or re-run the login flow).
- **`literal:` means the raw secret is on disk** — prefer `env://`/`keychain://` for anything sensitive; `_self show` redaction protects the terminal, not the file.
- A stored value in `defaults.json` can be silently overridden by `--token` / `SYNAPSE_TOKEN` / `--token-file` (priority chain) — a common "my token isn't being used" cause.

## Relationship to other skills

- **Issue a token:** [`plexus-idp`](../plexus-idp/SKILL.md) (the identity provider).
- **Store/resolve it:** this skill (synapse `_self`) — universal across backends.
- **A backend's auth/authorization rules** (what needs auth, tenancy/visibility): that backend's skill — e.g. [`trak`](../trak/SKILL.md).
- Source of truth: synapse's `CLAUDE.md` "Credentials & `_self`" section and the `Synapse.Self.*` modules; the redaction behavior is the `2e18a00e` fix.
