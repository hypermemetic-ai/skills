---
name: security-review
description: Use when the user asks for a security audit, SOC2 readiness check, or wants to verify that auth/tenant/encryption controls are complete. Produces findings ranked by severity (Critical/High/Medium/Low), grouped by SOC2 control families (CC6, CC7, CC8, A1, PI1, C1), each carrying a structured `## Evidence` section with file:line citations and a calibrated severity that accounts for known reviewer overconfidence.
---

# Skill: Security Review

Perform a structured security review of a codebase, grouped by SOC2 control families. Each finding carries the **evidence trail** that justifies the severity — not just "this is bad," but the reasoning a reviewer of your review can audit.

A single security pass is systematically overconfident. Where stakes are high, run two independent passes and aggregate; where stakes are merely informational, calibrate against the known bias toward over-ranking.

## When to use

When the user asks for a security audit, SOC2 readiness check, or wants to verify that auth/tenant/encryption controls are complete.

## Inputs

| Input | Description |
|-------|-------------|
| Codebase root | The directory to review |
| Auth model | How authentication works (JWT, session, API key) |
| Tenant model | How tenant isolation works (row-level, schema-level, none) |
| Deployment topology | Where TLS terminates, what proxies exist, what's public |
| Prior findings | (Optional) Previous review output, so this pass aggregates rather than duplicates |

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

### 4. Check failure modes

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

### 6. Rank findings (raw severity)

| Severity | Criteria |
|----------|----------|
| Critical | Exploitable now. Data exposure or auth bypass with no mitigating control. |
| High | Exploitable with effort or in specific conditions. Defense-in-depth gap. |
| Medium | Theoretical risk or requires insider access. Hardening opportunity. |
| Low | Best practice violation. No immediate exploit path. |

### 7. Calibrate severity

You are systematically overconfident in your raw severity rankings. Apply this shrinkage before publishing:

- **If you have ≥3 Criticals after step 6**, re-examine each: is the exploit chain *actually* end-to-end, or are you assuming an attacker has prerequisites you haven't verified? Downgrade any that depend on unverified prerequisites.
- **If a finding's exploit requires a layer you didn't independently audit** (e.g., "if the WAF lets this through" or "if the proxy doesn't strip this header"), drop it one severity level and note the dependency in `## Evidence`.
- **If you found the bug by pattern-matching** rather than tracing data flow end-to-end, drop one level. Pattern matches produce false positives at predictable rates.
- **Direction of impact.** For findings of the form "X gets silently overwritten" or "Y can be modified in unexpected ways," classify by direction:
  - Modification *removes* privileges, scope, or access → **logic bug**, not vulnerability. System fails toward less access. Annoyance/disruption surface, not exploitation. Severity at most Medium.
  - Modification *grants* privileges, scope, or access → **vulnerability**. Severity per the table above.
  Pattern-matching bias makes any state-machine inconsistency in a security-critical area look alarming. Calibration: check the direction first. If only downgrades are reachable, severity is bounded. The exception is **availability** — silent, triggerable privilege removal can be a denial-of-service even when it can't escalate. Track as Medium-A1 rather than dismissing.

This is the security-review analogue of Platt scaling: your prior is "I tend to over-rank," and the calibration step shrinks toward Medium where evidence is partial.

### 8. Multi-trial aggregation (when stakes are high)

For releases with significant blast radius (auth model changes, new public endpoints, key handling), run two independent passes. The second reviewer must not see the first reviewer's findings until they have produced their own.

Then aggregate:
- **Found by both, same severity** → keep severity, mark `confidence: high` in the report.
- **Found by both, different severity** → take the lower severity, note the disagreement.
- **Found by one only** → drop one severity level (single-trial findings are noisier than agreed findings).

For routine reviews, single-trial is fine — but record it as `confidence: single-pass` so the consumer knows.

### 9. Fact-check yourself

For every finding, verify:
- Does the code actually do what you think? (read it, don't assume)
- Is the library's default behavior what you assumed? (check docs)
- Is the "vulnerability" actually mitigated by another layer you didn't check?

Common false positives:
- "No token expiry check" — jsonwebtoken validates `exp` by default
- "Command injection via subprocess" — `Command::new().arg()` passes args as OS arguments, not shell-interpolated
- "No HTTPS" — TLS is terminated at the ingress, not the application

#### The evidence ledger

A finding's `### Evidence` section ends with a per-claim ledger that separates what was *observed by execution* from what was *inferred from source*. This is the artifact of step 9: it forces an explicit commitment to which side of the line each claim sits on, rather than letting confident prose elide the difference.

| Claim | Status |
|---|---|
| Function X produces output Y when called with input Z | **Observed** — test asserts state after `f(z)` returns |
| Endpoint A reaches function X in production | **Inferred from source** — handler is a 1-line passthrough at `path/file.go:NN` |
| Library Z routes the request to handler H | **Inferred — framework correctness** |

Three values, in decreasing strength:

- **Observed** — code was run, output was watched, a test or repro produced the claimed behavior. The strongest evidence and the one a reviewer can re-execute.
- **Inferred from source** — read the code, traced the call chain, identified a passthrough or a literal value. Reproducible by re-reading the cited line. Cite the file:line.
- **Inferred — framework correctness** — depends on a third-party framework (gRPC dispatch, ORM behavior, HTTP routing) doing what its docs say. Cheap to assert; cheap to be wrong about.

The ledger is *required* for any finding scored High or above. For Medium and below, it's recommended; the absence is itself evidence that the calibration didn't get sharp enough.

What survives the ledger gets reported. What doesn't either gets re-run to convert inference to observation, or gets downgraded with the inference status visible in the report.

A finding that survives steps 7, 8, and 9 is calibrated. Findings that don't survive get dropped or downgraded — silently. The report is the calibrated set.

## Output format

```markdown
## Finding N — Title (Severity)

**Location:** file:line
**Control:** CC6 (Access Control)
**Confidence:** high | medium | single-pass
**Gap:** what's wrong (one sentence)
**Impact:** what an attacker can achieve

### Evidence
The reasoning that justifies the severity. What code path leads to the
exploit, what assumptions had to hold, what mitigating layers were checked
and ruled out. This is what the next reviewer audits.

| Claim | Status |
|---|---|
| <claim 1> | Observed / Inferred from source / Inferred — framework correctness |
| <claim 2> | … |

The ledger is required for High and Critical findings. Each row is a
single claim and its evidence status. "Observed" means the behavior was
produced by running code; "Inferred from source" means it was read from a
specific file:line; "Inferred — framework correctness" means it depends on
a third-party framework doing what its docs say.

### Recommendation
What to fix (behavioral, not code). Optionally a ticket reference if one
has been filed.
```

The `### Evidence` block is what makes the finding auditable. A finding without it is folklore — useful in conversation, useless in a SOC2 packet six months later.

## Pointers

- Convert findings into tickets: `../ticketing/SKILL.md` — each finding's Gap becomes the ticket's Problem, the Evidence carries forward into the ticket's Evidence
- Tenant-isolation type system: `../strong-typing/SKILL.md`
- Methodology: `~/CLAUDE.md` → "Methodology" section
