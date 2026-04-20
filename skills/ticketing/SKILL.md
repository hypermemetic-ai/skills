# Skill: Write a TDD Ticket

Write an implementation ticket that passes the two-stranger test: two people who have never spoken — one implementing, one verifying — can independently agree on Done using only the ticket text.

## When to use

When the user asks to ticket work, plan a feature, or write acceptance criteria. Also invoked implicitly when any epic needs to be broken into implementable units.

## Inputs

| Input | Description | Example |
|-------|-------------|---------|
| Problem description | What gap, bug, or capability is missing | "matters.stats returns global counts to all users" |
| Domain context | Any external APIs, crate APIs, or domain knowledge needed | USCIS wizard API shape, pdf_oxide 0.3.24 method signatures |
| Upstream tickets | What this ticket depends on | STF-2 must be Complete |
| Downstream consumers | What tickets will read this ticket's output | STF-4 reads state_transitions table |

## Output

A markdown file in `plans/<EPIC>/<EPIC>-<N>.md` with YAML frontmatter. On creation, status is always `Pending` — Claude must never set a ticket to `Ready` on creation. Promotion to `Ready` requires human approval: either the user edits the file themselves, or the user explicitly grants Claude permission to flip the status (e.g., "promote these to Ready", "you have permission to mark X as Ready"). Without explicit permission, Claude leaves status at `Pending`. Implementation must not begin on `Pending` tickets.

## Directory convention

```
plans/
  <EPIC>/
    <EPIC>-1.md    # Epic overview (type: epic)
    <EPIC>-2.md    # Individual ticket
    <EPIC>-3.md    # Individual ticket
```

`<EPIC>` is a short prefix (e.g., IDN, STF, CHILD). `<N>` is sequential within the epic. The epic overview is always `-1`.

## Status values

| Value | Meaning | Who sets it |
|-------|---------|-------------|
| `Pending` | Written, awaiting human review | Claude (on creation) |
| `Ready` | Approved, no blockers, ready for implementation | Human (directly, or Claude with explicit permission) |
| `Blocked` | Approved but waiting on `blocked_by` tickets | Human or Claude |
| `Complete` | Implemented and committed | Implementor (in the same commit as the code) |
| `Idea` | Captured but not ready — open design questions | Claude or human |
| `Epic` | Overview document, not a unit of work | Claude (on epic creation) |
| `Superseded` | Absorbed by another ticket (see `superseded_by` field) | Claude or human |

## Querying tickets

```bash
# All outstanding work
grep -rl '^status: Ready' plans/

# All complete
grep -rl '^status: Complete' plans/

# Pending (awaiting human approval)
grep -rl '^status: Pending' plans/

# Ready tickets with titles (Python)
python3 -c "
import glob, re
for f in sorted(glob.glob('plans/*/*.md')):
    with open(f) as fh:
        text = fh.read()
    if not text.startswith('---'): continue
    block = text.split('---')[1]
    status = re.search(r'^status:\s*(.+)', block, re.MULTILINE)
    tid = re.search(r'^id:\s*(.+)', block, re.MULTILINE)
    title = re.search(r'^title:\s*\"?(.+?)\"?\s*$', block, re.MULTILINE)
    if status and tid and title and status.group(1).strip() == 'Ready':
        print(f'  {tid.group(1).strip():20s} {title.group(1).strip()}')
"
```

## Bulk frontmatter updates

Do not edit ticket frontmatter one file at a time when updating N tickets to the same state. Use a YAML-aware bulk-edit one-liner. It is faster, atomic per file, and avoids the Read-before-Edit friction of tool-based editing when flipping many tickets at once.

```bash
# Flip every Pending ticket in an epic to Ready (bulk promotion).
# Replace <EPIC> with the epic prefix (e.g., IR, CHILD, SYN).
python3 -c "
import re, glob, pathlib
for f in glob.glob('plans/<EPIC>/*.md'):
    p = pathlib.Path(f)
    text = p.read_text()
    new = re.sub(r'^status:\s*Pending\s*$', 'status: Ready', text, count=1, flags=re.MULTILINE)
    if new != text: p.write_text(new)
"

# Flip specific IDs (CHILD-2, CHILD-5, CHILD-6) from Ready to Complete.
python3 -c "
import re, pathlib
for tid in ['CHILD-2', 'CHILD-5', 'CHILD-6']:
    p = pathlib.Path(f'plans/CHILD/{tid}.md')
    text = p.read_text()
    new = re.sub(r'^status:\s*Ready\s*$', 'status: Complete', text, count=1, flags=re.MULTILINE)
    if new != text: p.write_text(new)
"

# Verify what changed (run before committing).
git diff plans/
```

For structured edits beyond simple status flips (e.g., adding `superseded_by` fields, changing `target_repo` across many tickets), prefer a PyYAML round-trip rather than regex to preserve structure:

```python
# Requires: pip install ruamel.yaml  (preserves YAML formatting + comments)
from ruamel.yaml import YAML
import pathlib

yaml = YAML()
for p in pathlib.Path('plans/IR').glob('IR-*.md'):
    text = p.read_text()
    if not text.startswith('---\n'): continue
    _, front, body = text.split('---\n', 2)
    data = yaml.load(front)
    # Edit here — e.g.:
    if data.get('target_repo') == 'synapse-cc':
        data['target_repo'] = 'hub-codegen'
    # Write back.
    import io; buf = io.StringIO()
    yaml.dump(data, buf)
    p.write_text(f'---\n{buf.getvalue()}---\n{body}')
```

**Rule:** reaching for a bulk update tool is itself a signal that the plan may be mid-rework. If you're flipping a dozen tickets to Superseded, capture *why* in one commit message rather than spreading the rationale across many individual edits.

## Format

```yaml
---
id: EPIC-N
title: "Short description"
status: Pending
type: implementation    # implementation | analysis | spike | epic
blocked_by: []
unlocks: []
severity: Medium        # Critical | High | Medium | Low (optional)
---
```

Body sections in order:

```markdown
## Problem
One paragraph. The gap, the bug, the missing capability.

## Context
(Optional) External API shapes, domain knowledge, crate API refs.
NOT implementation instructions. This is the terrain, not the route.

## Required behavior
Input/output tables. Observable behaviors. "When X, then Y."
No code. No SQL. No file paths.

## Risks
(Optional) Places where the output might not be achievable.
Each risk maps to a spike, a fallback, or a replanning trigger.

## What must NOT change
Regression criteria. Existing behaviors that must survive.

## Acceptance criteria
Numbered items. Each checkable by running a command, reading a
response, or counting rows. No criterion requires reading source
code to evaluate.

## Completion
What the implementor delivers: test command + output, status flip.
```

## Rules

1. **One decision per ticket.** Multiple options = not ready = write a spike.
2. **Specify behavior, not code.** Use input/output tables.
3. **No implementation in acceptance criteria.** Criteria describe what the system does, not how.
4. **State what does NOT change.** Regression criteria are as important as new behavior.
5. **Scope is one diff.** Multi-repo changes that can't be tested together = split.
6. **Verification is high-level.** "Run tests; they pass." Not "grep for this string."
7. **Fixtures are committed.** If criteria reference a test fixture, it exists in the repo.
8. **Downstream tickets can read the contract.** If ticket B depends on A's output, A's criteria pin the shape.
9. **Strong types in contracts.** Reference `WizardNodeId`, not "a string that's a node ID."
10. **Concurrency is file-bounded.** Two tickets that write to the same file cannot run in parallel — they collide at commit time, regardless of what the `blocked_by` DAG says. When planning an epic, check file boundaries across tickets in addition to logical dependencies: "can ticket X and ticket Y run simultaneously?" requires "do they modify any of the same files?" to be answered first. Reading the same file is fine; writing is not.
11. **Split tickets along file boundaries to expose parallelism.** If a single ticket would modify multiple independent files — and those files have no data dependency on each other within the ticket — split the ticket into per-file units so the pieces can be implemented concurrently. A ticket containing internally-parallelizable work is a serial chain disguised as one unit. Prefer narrow tickets that each own one file (or one tightly-coupled file set) to wide tickets that block an entire directory.

## Meta-acceptance criteria (check before writing to disk)

- [ ] Two-stranger test passes
- [ ] No open design decisions
- [ ] Acceptance criteria are observable
- [ ] Fixtures are committed (or described precisely enough to create)
- [ ] Regression section is complete
- [ ] Downstream tickets can read the contract
- [ ] File-write boundaries are disjoint from any sibling ticket marked concurrent in the DAG

## Examples

**Good acceptance criteria:**

| Caller | Operation | Expected result |
|---|---|---|
| Unauthenticated client | `clients.list` | Auth error |
| Solo user A | `clients.get` with Solo B's id | "Not found" error |
| Admin user | `clients.list` | Returns all clients |

**Bad acceptance criteria:**

> `cargo test --test tenant_isolation::solo_cannot_see_other` passes

(Pins test name, file, framework. The contract should say what the system does, not which test to run.)

**Good context section:**

> The USCIS wizard API returns JSON at `GET /rest/wizard-page/{id}`:
> ```json
> [{"id": "95941", "options": [{"referenced_content": "95940", "wizard_answer": "Alien"}]}]
> ```

(Documents external contract. Not implementation.)

**Bad context section:**

> Use `reqwest::get()` to fetch. Parse with `serde_json`. Store in a HashMap.

(Implementation prescription. The ticket should say what to achieve, not how.)

## Pointers

- Planning epics: `skills/planning/SKILL.md`
- Spikes: `skills/planning/SKILL.md` → "Spikes" section
- Strong typing in contracts: `skills/strong-typing/SKILL.md`
- Methodology: `~/CLAUDE.md` → "Methodology" section
- Example tickets: FormVeritas `plans/IDN/`, `plans/STF/`, `plans/PDF_EXTRACTION/`
