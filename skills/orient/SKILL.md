---
name: orient
description: Use at the start of a session — typically from a cleared context — to orient on the work in flight and produce a grounded "where it's at" read on the active PR. Loads the governing methodology + project skills FIRST (they are the review rubric), finds the in-progress ticket(s), maps the current branch/worktree to its pull request, reads the diff against the tickets and any review threads, then verifies the PR's own claims by building and running the infra-free tests rather than trusting them. Produces a severity-ranked review grounded in the skills and live state — not a restatement of the PR description.
---

# Skill: Orient (session pickup)

**In one line:** dropped onto an in-flight branch cold, rebuild the picture and review it against live state — load the rules, map branch → PR, verify before you report.

You've just cleared context. There is a branch, a PR, a few tickets in flight, and a body of conventions the work is supposed to honor — and none of it is in your head yet. Orienting is the practice of **rebuilding that picture from live state in a fixed order: load the lenses, find the work, map it to its PR, read the diff against the tickets, then verify before you report.**

The output is a grounded "where it's at" review — ticket-intent vs. what the diff did, findings ranked by severity, each one cited to a skill rule or a verified code path. It is explicitly *not* a summary of the PR description; the description is one of the things you are checking.

## When to use

- Starting (or resuming) a session cold: "where am I", "review the current PR", "orient on the work in flight", "what's active for me".
- Before reviewing a PR — your own or a teammate's — when you need the surrounding context (tickets, conventions, stacking) loaded, not just the diff.
- Any time the conversation has decayed and you need conversation-state rebuilt from artifact-state.

Don't use for:
- A known, single-file change you already have context on (just read the file).
- Producing a milestone-level status across a whole pitch (a separate status-reporting task, not in this repo).
- A non-technical manager update (a separate status-reporting task, not in this repo).

## Inputs

| Input | Description | Example |
|-------|-------------|---------|
| Trigger | Usually a cleared context + a "where am I" intent | `/orient`, "review the current PR off this branch" |
| Repo root | The project being worked on | `~/dev/cn/app-cm/` |
| Whose work | The assignee to orient around (default: you) | `me` |
| Target PR (optional) | A specific PR; otherwise inferred branch → PR | `#975`, or inferred from the worktree HEAD |

## Output

A grounded review delivered in chat (no artifact written unless asked):

- The **branch/worktree → PR** mapping, with stacking noted (draft? base? what it's stacked on).
- **Ticket intent → how the diff accomplished it**, per ticket.
- **Findings ranked by severity**, separating **code findings** from **artifact/process findings** (stale PR body, missing PR↔ticket link, ticket-state lag) — both are real.
- Each finding **cited** to a skill rule or a **verified** code path (`file:line`).
- What was and was **not** test-verified.

Any follow-on tracker writes (new tickets, status moves) follow the normal **Pending** / human-ratify flow — see `../ticketing/SKILL.md` and `../planning/SKILL.md` → "The milestone in the tracker".

## Process

Do these in order. The order is the point — step 1 before any code.

1. **Load the lenses first (before forming any opinion).** Read the methodology skills that govern the work — `../methodology/SKILL.md`, `../planning/SKILL.md`, `../ticketing/SKILL.md` (especially the build vs. execution vs. scope ticket kinds) — and the project's own skills for the area the diff touches: read the repo's `AGENTS.md`, then the relevant `.claude/skills/*`. *(In app-cm: `graph-nodes` for any vertex/edge work, `go-backend`, `generated-code`, `test-planning`.)* These skills are the **rubric you review against**; reading them *after* forming an opinion means re-reviewing.
2. **Establish git ground truth — including the workspace census.** Current branch / worktree (`git branch --show-current` — note a detached HEAD), recent commits (`git log --oneline`), the remote — and **one command for the worktrees**: `bin/worktree-census` (this repo). It maps every worktree → branch → ticket id (naming convention) and flags `COLLECT?`/`ARCHIVE?`/`RESCUE?` candidates. The `?` marks what git can't know — ticket state — resolved by querying the tracker for the ticket's state. Flagged rows go in the report as **workspace findings**; execution follows the [cleanup recipe](../../docs/recipes/_common/clean-up-worktrees.md) after ratification.
3. **Find the work in flight.** Query the tracker for tickets assigned to me in an in-progress state (**active**, and adjacent in-progress states). Separate the **execution/scope epics** (they stay in-progress by design) from the **build tickets** — the builds are the actual work.
4. **Map branch → PR.** `git branch -a --contains HEAD` shows which branches hold the current commit; `gh pr list` (sorted desc) and `gh pr list --head <branch>` resolve the PR. Pick the active one and cross-check it against the in-progress ticket (its branch name / attachments). In a stacked PR, note the base — line references are against the head.
5. **Pull the PR whole.** `gh pr view` (metadata: draft? base? mergeable? stacking), the body, `gh pr diff` (full), and existing review threads (`--json reviews,comments`).
6. **Pull the tickets behind it.** Read every ticket the PR claims — and **follow the trail**: a ticket like "corrective build surfaced in review of #N" means the PR may already be *past* its own description.
7. **Reconcile code vs. prose.** The diff is ground truth; the PR body and ticket prose lag it. Diff the **commit log** against the description — a build whose commits are in the PR but whose ticket still says **active**, or a body describing a shape the code has already superseded, **is a finding** (`../planning` → the doc/tracker is a projection; evidence must travel with the code, not only commit messages).
8. **Verify, don't trust.** Check the PR branch out into the review worktree; build the touched packages and run the infra-free tests yourself; spot-check the load-bearing claims — a "soft delete", an "index-backed point lookup", a "v7 id" — against the actual implementation. Never restate the PR's "tests green" without running what you can; name what needs live infra and was skipped.
9. **Report against the rubric.** Lead with ticket-intent → how-accomplished, then findings by severity, each grounded in a skill rule or a verified path. Calibrate any security-shaped finding by direction-of-impact (`../../AGENTS.md` → Conventions › Direction-of-Impact).

## Rules

- **Skills before opinions.** Load the governing + project skills before reading the diff. They are the review rubric, not background reading. (This is the most common miss: reviewing on ticket prose + code, then discovering the conventions afterward.)
- **Ground truth is the diff + a build, not the description.** Verify every load-bearing claim and name the exact path. If you didn't run a tier, say so.
- **Map the branch to its PR explicitly.** In a stacked PR, note the base and that `file:line` references are against the head, not `main`.
- **Know the ticket kinds.** Distinguish build vs. execution vs. scope (`../ticketing`). An execution/scope epic sitting in-progress is *correct*, not a finding.
- **Artifact findings are findings.** A stale PR body, a missing PR↔ticket link (`../ticketing` Rule 17), or a ticket whose state lags its PR'd code are real gaps — report them beside the code findings, not instead of them.
- **Don't overstate.** Precise claims only; name what each guard does and does not cover. (`../../AGENTS.md` and the project conventions both forbid the silent overclaim.)

## Examples

**Good orientation finding** (grounded, verified, precise):

> Loaded `graph-nodes` first, so I reviewed the mint against its pre-merge checklist: `system_from` set, v7 `SystemID` (`uuid.NewV7()`), reads filter live, tenant anchor edge present — all pass. But the canonical id column is **v4** (`gen_random_uuid()`, pinned to the migration's line), so the "always v7" rule holds only for the node's own `SystemID`; the domain newtype correctly stays version-agnostic to accept existing v4 ids. The mint's check-then-act race is the *house pattern* (the skill's canonical `Create` has it too) — not a defect of this PR.

**Bad orientation finding** (restates the body, verifies nothing, cites no rule):

> The PR adds a user identity read-seam and a graph node; the description says tests pass. Looks good to me.

**Good code-vs-prose reconciliation:**

> The commit log shows the corrective build already pushed (`feat(CMD-XXXX): collapse the node to the canonical id …`), but the PR body still describes the superseded design and the ticket is still **active**. Code is ahead of both artifacts — that's the headline, not the diff.

## Pointers

- The operating shape (goal → milestone → scope → execution DAG → spikes → builds): `../methodology/SKILL.md`
- Ticket kinds, the two-stranger test, PR-linking rule: `../ticketing/SKILL.md`
- Tracker-is-a-projection, status conventions, supersession trail: `../planning/SKILL.md`
- Severity calibration: `../../AGENTS.md` → Conventions › Direction-of-Impact
- Project-specific review rubrics live in the worked-on repo's `.claude/skills/*` (read its `AGENTS.md` first).
