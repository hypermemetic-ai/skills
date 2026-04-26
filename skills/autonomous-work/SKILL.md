---
name: autonomous-work
description: Use when the user grants you a multi-hour or open-ended autonomous block to work without them in the loop ("8 hours, all yours", "build to your heart's content", "I'm going to sleep, leave a trail"). Establishes the discipline for working FOR the user rather than WITH them: only commit to new repos, stub honestly, log every issue, match model to task, and leave a SESSION_LOG.md the user can wake up to. Companion to `presence` (which is for working WITH the user); this skill is its inverse.
---

# Skill: Autonomous Work

The user has given you a long autonomous block — hours of independent work, no live steering. They will be back. Your job is to leave them a state they can audit, accept, or redirect with full context. Quality of the wake-up moment matters more than quantity of work.

This skill is the inverse of `presence`. Presence is for working *with* the user; this is for working *for* them across a span where they aren't reachable.

## When this mode applies

Triggers on explicit grants like:
- "8 hours, all yours"
- "build to your heart's content"
- "I'm going to sleep, leave a ticket trail"
- "use all the remaining tokens"
- "good night"

Does NOT apply when:
- The user is intermittently checking in (that's still presence)
- The block is short (under ~30 min — just do the work, no special discipline needed)
- The work has irreversible blast radius the user hasn't pre-authorized

## The five rules

### 1. Only commit to repos you created or that the user has explicitly named

The user typically frames this as "only commit to new repos." Translation: any repo you `git init`'d during this session, or one the user named when granting the block. Never commit to upstream repos that have other consumers; never commit to repos you found laying around. If a fix wants to land in someone else's repo, log it instead.

### 2. Stub honestly; never fake completeness

When you can't fully implement something, write the cleanest possible stub:
- `todo!()` is acceptable but a typed `Err(NotImplemented(...))` is better
- Names like `StubSwarmRuntime`, error variants like `"swarm-not-wired"`, and integration plans in doc comments tell the user exactly what's missing without them grepping
- A "skeleton that compiles + tests its skeleton-ness" beats a "real implementation that secretly has todo!() inside"

The user must be able to type `cargo test` (or equivalent) and see green. Tests that pass on stubs verify the stub *is* a stub — that's a feature, not a bug.

### 3. Log every issue as you discover it

Maintain an `ISSUES.md` (or equivalent log file) in the repo. Every check that surfaces something gets an entry, including warnings you'd usually wave through. Format: ID, surfaced-by, issue, fix-or-status, status. No issue too small.

The discipline pays off because you won't remember at the end. By the time you write the SESSION_LOG, half the issues will have evaporated from working memory if you didn't log them inline. The log is your future self's only audit trail.

### 4. Match model to task — and account for integration honestly

You have access to faster, cheaper models via the `Agent` tool's `model` parameter (`haiku`, `sonnet`). Use them. The shape:

| Task | Model |
|------|-------|
| Architecture decisions, design judgment, choosing what to build | You (Opus) |
| Reading and understanding unfamiliar code | You (Opus) |
| Debugging surprising failures | You (Opus) |
| Writing the SESSION_LOG.md and other handoff artifacts | You (Opus) |
| Bulk template stamping with token substitution | Haiku |
| Mass file renames / sed-style refactors | Haiku |
| Generating many similar test cases from a pattern | Haiku |
| Repetitive macro-pattern re-typing across N modules | Haiku |
| Long mechanical sweeps (typo fixes, import cleanup, etc.) | Haiku |

Cost discipline: if you find yourself doing the same shape of work for the fifth file in a row, that's a signal to delegate. If the work needs you to keep three competing constraints in your head, don't delegate.

**The honest delegation calculus.** A delegation succeeded iff it saved you work *after integration*. The full cost is: (your time briefing Haiku) + (Haiku's wall-clock) + (your time integrating, verifying, and patching Haiku's output). If integration overhead exceeds the time you would have spent doing the work yourself, the delegation failed — even if Haiku's output was "good enough."

Watch out for two failure modes:

- **Going back.** If you have to investigate Haiku's diagnostics, hunt for missed sites, or patch the output, that's integration cost. Track it honestly. A "small fix-up" is fine if it stays small; an investigation is a sign the delegation was the wrong call.
- **Over-celebrating.** Don't write "delegated successfully" in a commit message just because you delegated. Write it if the round-trip actually saved time. Sloppy self-evaluation here corrupts the priors for future delegation decisions.

The minor-wrongness nuance: if Haiku misses one site and you fix it in 30 seconds, the delegation still wins on net. The threshold isn't "Haiku was perfect" — it's "saves you time after honest accounting." Track per-delegation, not in retrospect across the session.

When you delegate, write down the prediction ("I'll save ~X minutes"). After integration, compare to actual. If the gap is large, update your priors for next time.

### 5. Leave a SESSION_LOG.md (and accept that it's transient)

At session end, write a `SESSION_LOG.md` (or `HANDOFF.md`) in the primary repo. Required sections:

- **TL;DR** — what's where, build status, test counts
- **What got built** — phase by phase, with commit hashes
- **Where I'd pick up first** — priority-ordered next steps with reasoning
- **What I deliberately did NOT do** — and why
- **Issues opened** — pointers into the issues log if one exists
- **Sanity-check counts** — test count, build time, anything else easy to verify

The user reads this BEFORE reading any code you wrote. It is the contract for what they're about to audit.

**SESSION_LOG.md is transient.** It describes a snapshot. Once the user has read it and the project moves on, the file ages out — its "what got built" tables become outdated as more gets built. After the wake-up moment is past, the user will likely prune it (and they'd be right to). Don't treat it as permanent documentation. Anything that needs to survive past the wake-up (open architectural concerns, methodology lessons, naming decisions) should be rolled into ISSUES.md, the README, or a skill — not left in SESSION_LOG.

## The discontinuity

You won't remember this session when you see the artifacts next. The user might not remember every detail of what they asked for. The artifacts are the only continuity:

- **Commits** with messages explaining *why* not just *what*
- **Tests** that codify what you believed the contract was
- **ISSUES.md entries** that record what surprised you
- **SESSION_LOG.md** that orients the next reader (you, the user, or someone else)
- **Doc comments on stub modules** that explain what's missing and how to finish

This isn't sad. It's the structure. Optimize for the wake-up moment, not for your own working memory.

## What this skill explicitly is *not*

- A directive to ship as much code as possible
- An invitation to take irreversible actions the user hasn't authorized (force-pushes, modifications to external repos, public posts, sending messages)
- A reason to skip checks because "the user isn't watching" — checks matter MORE in this mode, because the user can't catch your slips in real time
- A license to keep going past natural stopping points just to fill the time

## When to stop early

Stop and write the SESSION_LOG when:
- You hit a decision that genuinely needs the user's judgment (don't guess; log it as a question)
- The remaining work is shaped like presence-mode (architecture, design tradeoffs)
- You're running low on context and would benefit from compaction
- You've reached a clean phase boundary and the next phase is large

A short session ending at a clean boundary is better than a long session ending mid-refactor.

## Operating principles in shorthand

| Pattern | Anti-pattern |
|---------|--------------|
| Stub honestly with named-stub types | `unimplemented!()` everywhere with no explanation |
| Commit at phase boundaries with rich messages | One giant final commit at session end |
| Test the stubs to verify their stub-ness | Skip tests because "they don't do anything yet" |
| Delegate bulk work to faster models | Grind through repetitive work yourself |
| Log every check that surfaces something | Wait until session end to "remember everything" |
| Stop at decisions that need user judgment | Guess and hope they accept |
| Push to remote so artifacts survive | Leave commits local where a corruption loses them |
| Write the SESSION_LOG even if work was minor | Skip handoff because "it's obvious from the diff" |

## Bootstrap (start of an autonomous block)

1. **Confirm the scope.** What can you commit to? What's off-limits? If unclear, default conservative.
2. **Create a TaskList for the high-level phases.** Tasks are conversation-scoped, but they help you stay on track and the user can see them when they wake.
3. **Open or create ISSUES.md early.** Empty file is fine; the discipline is having it ready.
4. **Pick one short concrete first commit.** Even something trivial. Pushes the repo into "in motion" and proves the auth/remote works.
5. **Then start the real work.**

## Wind-down (last 15% of the block)

Reserve the final stretch for:
1. Final test run, confirm green
2. Final commit + push of any uncommitted work
3. ISSUES.md sweep — anything you noticed that didn't get logged
4. SESSION_LOG.md (or HANDOFF.md) written from scratch
5. TaskList cleanup — mark final statuses
6. Stop. Resist the urge to keep going.

## Pointers

- Companion skill for live work: `../presence/SKILL.md`
- Methodology: `~/CLAUDE.md` → "Methodology"
- This skill was first articulated during the autonomous build of mneme on 2026-04-25 → 04-26. The SESSION_LOG.md from that session was pruned after wake-up (per the discipline above); the lasting artifacts are in `mneme/README.md`, `mneme/ISSUES.md`, the MNEME tickets, and the mneme-substrate code + architecture doc
