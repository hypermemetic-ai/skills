---
name: plexus-idp
description: Use when working with identity/auth for ANY Plexus backend — "register a user", "get a token / log in", "password grant", "refresh token", "manage roles", "rotate keys", "which issuer/audience", "JWKS/discovery", or wiring a backend to validate tokens. plexus-idp is THE identity provider (OIDC, RS256) for the Plexus stack; it supersedes the per-backend identity hubs (e.g. trak's legacy `identity` activation). For storing a token into a backend's config, see `synapse-self`; for a backend's own validation/authorization rules, see that backend's skill (e.g. `trak`).
---

# Skill: plexus-idp — the Plexus identity provider

plexus-idp is the single OIDC identity provider for the whole Plexus stack. It issues **RS256** access tokens that any backend validates from the discovery document + JWKS alone — no shared secret. It **supersedes** the per-backend identity hubs (the old `trak identity` HS256 minting is dead; backends now point their validator at this IdP). A real Auth0 tenant can drop in where plexus-idp sits by changing two config values (issuer + audience).

## Surfaces & ports

- **RPC** (Plexus WebSocket): `ws://127.0.0.1:4460` — backend `idp`, activation `identity`.
- **OIDC HTTP**: `http://127.0.0.1:4461` — discovery, JWKS, token endpoint.
- **Issuer**: `http://127.0.0.1:4461` by default — note **`127.0.0.1`, not `localhost`** (it's the `iss` claim every validator checks; a `localhost` mismatch silently fails validation).
- SQLite at `~/Library/Application Support/plexus-idp/idp.db` (or XDG); RSA keys in `<config>/plexus-idp/keys`.

Run it:
```bash
plexus-idp                      # or: cargo run --release
# flags: --port 4460 --http-port 4461 --db <path> --key-dir <path>
#        --issuer <url> --audience <aud>
```
Confirm it's up: `curl -s http://127.0.0.1:4461/.well-known/openid-configuration`.

## Get a token

**OIDC password grant (HTTP)** — the portable path, no synapse needed. Request the audience of the backend you're targeting (`plexus:trak` for trak, etc.):
```bash
curl -s -X POST http://127.0.0.1:4461/oauth/token \
  --data-urlencode grant_type=password \
  --data-urlencode username=<user> \
  --data-urlencode password=<pw> \
  --data-urlencode audience=plexus:<backend>
# -> {"access_token": "<RS256 JWT>", "refresh_token": "<opaque>", "expires_in": 3600}
```

**RPC (via synapse)** — same `IdentityCore` underneath:
```bash
synapse -P 4460 -p '{"username":"alice","password":"s3cret"}' idp identity login
synapse -P 4460 -t "$TOKEN" idp identity me
```

Register a user (`org_id` = the tenant; it becomes the `org_id` claim and drives backend tenancy):
```bash
synapse -P 4460 -p '{"username":"alice","password":"s3cret","org_id":"hypermemetic"}' idp identity register
```

## Tokens (what's in them, how they expire)

- **Access**: RS256, `kid` header, **1 h TTL**. Claims: `iss`, `sub`, `aud`, `exp`, `iat`, plus `org_id` (only when the user has a tenant) and the plexus extensions `username` + `roles` (optional — no backend may *require* them).
- **Refresh**: opaque 64-char hex, 30-day TTL, **single-use — rotated on every refresh grant**. ⚠️ Don't spend a refresh token twice: the grant returns a *new* one and invalidates the old; if you don't persist the rotation you've burned the chain and must do a fresh password grant.
- **Audience**: per-backend (`plexus:trak`, …). A grant requests it via the `audience` param; else the server `--audience` default. A token minted for `plexus:trak` won't validate at a backend expecting a different `aud`.

## Roles (R-6)

Roles are what the IdP administers; each backend expands roles → its own scopes locally (the IdP knows no backend's scope vocabulary).
```bash
synapse -P 4460 -t "$ADMIN" -p '{"user_id_or_username":"alice"}' idp identity get_user_roles
synapse -P 4460 -t "$ADMIN" -p '{"user_id_or_username":"alice","roles":["user","operator"]}' idp identity set_user_roles
```
- `set_user_roles` **replaces** (not merges). Names: `[A-Za-z0-9._:-]`, ≤64 chars, ≤32 roles.
- The last admin can't drop `admin` (lockout guard).
- **Revocation latency = access-token TTL by design.** No revocation list: a role/permission change only takes effect for tokens minted *after* it; tokens in the wild keep their claims until they expire (≤1 h).

## OIDC surface (what a validator consumes)

```bash
curl http://127.0.0.1:4461/.well-known/openid-configuration   # discovery
curl http://127.0.0.1:4461/jwks.json                          # RSA public keys, kid-tagged (current + previous)
```
A backend's entire coupling to identity is **issuer + audience**. Its validator does: issuer → discovery → `jwks_uri` → RS256 by `kid` → check `iss`/`aud`/`exp` → read `sub` + `org_id`. That's the Auth0 swap story — point the issuer at an Auth0 tenant, set `aud` to the API identifier, done; zero backend code change.

## Keys & rotation

RSA-2048 generated on first boot into `--key-dir` (PKCS#8 PEM, 0600). The `kid` = SHA-256 of the public-key DER → **stable across restarts** (so a token minted before a restart still validates after). Rotate by dropping a newer `idp-rsa-<nanos>.pem` into the key dir and restarting; JWKS publishes current + previous, so old tokens validate against the previous `kid` until they expire. `identity.rotate` RPC is an admin stub (key-custody decision pending).

## API keys

A plexus extension that sits *beside* the OIDC surface, in each backend's `SessionValidator` chain (not in the IdP's token flow). `idp identity create_api_key` mints one. ⚠️ Backend-specific caveat: on some backends (e.g. the trak cutover) API-key callers carry an empty session id and are treated as **anonymous (read-public, no writes)** — check the backend before relying on API keys for writes.

## Gotchas (the ones that waste a session)

- **Issuer `127.0.0.1` ≠ `localhost`** — the target backend's `--oidc-issuer` must byte-match the `iss` the IdP stamps.
- **Audience must match the backend** — request `plexus:<backend>` in the grant.
- **Refresh tokens are single-use** (rotate on use) — capture-and-store atomically.
- **Roles/perm changes lag by ≤1 h** (TTL-bound, no revocation list).
- **SRP** is NOT here — it remains a plexus-trak-only extension (deferred, UT-S01 Q5).

## Relationship to other skills / backends

- This skill = identity provider operations (issue/validate tokens, roles, keys) — reusable across all backends.
- **To authenticate to a specific backend**, get a token here, then store it with synapse's universal credential store — see [`synapse-self`](../synapse-self/SKILL.md). A backend's own auth (token validation + its authorization/tenancy rules) lives in that backend's skill, e.g. [`trak`](../trak/SKILL.md).
- Background: `plexus-idp/README.md`; trak facets `UT-2` (`be9fb1d5`, this crate) and `UT-W3` (`ae9ff54b`, the trak cutover onto it).
