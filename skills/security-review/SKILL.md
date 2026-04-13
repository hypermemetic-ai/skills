# Skill: Security Review

Perform a structured security review of a codebase, grouped by SOC2 control families. Produces findings ranked by severity with specific code citations, not generic advice.

## When to use

When the user asks for a security audit, SOC2 readiness check, or wants to verify that auth/tenant/encryption controls are complete.

## Inputs

| Input | Description |
|-------|-------------|
| Codebase root | The directory to review |
| Auth model | How authentication works (JWT, session, API key) |
| Tenant model | How tenant isolation works (row-level, schema-level, none) |
| Deployment topology | Where TLS terminates, what proxies exist, what's public |

## Process

### 1. Map the auth chain

Trace a request from the client through every layer:
- Transport (TLS termination, proxy headers)
- Middleware (token extraction, validation)
- Dispatch (per-method auth gates)
- Data access (scoped queries, audit logging)

For each layer, ask: "what happens if this layer is bypassed?"

### 2. Enumerate every endpoint

List every public method/route. For each:
- Does it require authentication? (check for auth decorators/middleware)
- Does it require authorization/scoping? (check for tenant filters)
- What data does it access? (read the DB calls)
- What data does it return? (could it leak cross-tenant data?)

### 3. Check tenant isolation

For every database query that touches tenant data:
- Is the query filtered by the user's scope?
- Is the filter at the SQL level (safe) or application level (bypassable)?
- Can the filter be skipped by calling an unscoped variant?
- Is there a TOCTOU gap between the scope check and the data access?

### 4. Check the failure modes

For each security control:
- What happens when it's misconfigured? (fail-open vs fail-closed)
- What happens when the dependency is unavailable? (DB down, Keycloak unreachable)
- What's logged when it fails? (debug vs audit level)
- Can an attacker observe the failure mode? (error messages leak info?)

### 5. Group findings by SOC2 control family

| Control | What to check |
|---------|---------------|
| CC6 (Access Control) | Auth gates, tenant scoping, role checks, origin validation |
| CC7 (Operations) | Audit logging, monitoring, incident detection |
| CC8 (Change Management) | Committed secrets, default credentials, migration safety |
| A1 (Availability) | Rate limiting, size limits, resource exhaustion |
| PI1 (Processing Integrity) | Input validation, command injection, temp file handling |
| C1 (Confidentiality) | Token handling, TLS enforcement, error message leakage |

### 6. Rank findings

| Severity | Criteria |
|----------|----------|
| Critical | Exploitable now. Data exposure or auth bypass with no mitigating control. |
| High | Exploitable with effort or in specific conditions. Defense-in-depth gap. |
| Medium | Theoretical risk or requires insider access. Hardening opportunity. |
| Low | Best practice violation. No immediate exploit path. |

### 7. Fact-check yourself

For every finding, verify:
- Does the code actually do what you think? (read it, don't assume)
- Is the library's default behavior what you assumed? (check docs)
- Is the "vulnerability" actually mitigated by another layer you didn't check?

Common false positives:
- "No token expiry check" — jsonwebtoken validates exp by default
- "Command injection via subprocess" — `Command::new().arg()` passes args as OS arguments, not shell-interpolated
- "No HTTPS" — TLS is terminated at the ingress, not the application

## Output format

```markdown
## Finding N — Title (Severity)
**Location:** file:line
**Gap:** what's wrong
**Impact:** what an attacker can do
**Recommendation:** what to fix (behavioral, not code)
```

## Pointers

- Example review: FormVeritas `docs/architecture/16670821463464200703_auth-and-tenancy-flow.md`
- Tenant isolation model: FormVeritas `CLAUDE.md` → "Roles and Tenant Scoping"
- Ticket the findings: use the ticketing skill with each finding as a Problem statement
