# How we propagate merges up a stacked-PR chain

When a stacked chain's base moves (main merged into the bottom, or any level gains commits) and the branches above must be brought current. **Merge, never rebase — the history is published**; rebasing orphans every PR above the rewrite.

## Steps (per level, bottom-up)

1. In the level's worktree (`per recipe _common/clean-up-worktrees` if one must be created): `git pull` the branch's own remote first, verify `git status --porcelain` is clean — stop and triage if not.
2. `git merge <base-branch>` (no edit). Resolve conflicts by class:
   - **Source files** (Go, `.proto`): UNION — keep both sides' additions; on a genuine same-symbol conflict, the incoming base wins for the chain's shared machinery, the branch's own side wins for the feature it adds. No duplicate proto field numbers, no duplicate Go symbols.
   - **Generated files** (`proto/gen/**`, swagger `docs/`): NEVER hand-merge. Resolve the source, then regenerate (`make proto`; `bash backend/cmscripts/swag.sh` for the gitignored swagger gap in fresh worktrees).
3. **Hook discipline — the step that prevents the evil merge:** if the pre-commit hook fails, do NOT reflex to `--no-verify`. First inspect what the hook already *did*: formatter steps (lint-staged/prettier) may have rewritten staged files before the failing step aborted. `git status` + spot-`diff` the touched paths; restore any hook churn to the merge's intended content (`git checkout MERGE_HEAD -- <paths>` for base-side files). Only then conclude with `--no-verify`, and say so in the report.
4. Gate before pushing: build + the affected package tests + any named tripwire tests (e.g. a predicates-pin regression) **passing unmodified**.
5. **The diff-size check — the detector:** after concluding the merge, `git diff --shortstat <base>...<branch>` and compare against the PR's intended scope (its file list). A merge should change a PR's diff by ≈nothing. If the count balloons, you made an evil merge — find the unintended paths (`git diff --name-only base...branch` minus the intended list) and restore them verbatim from the base side before pushing.
6. Push; verify `gh pr view <n> --json mergeable,mergeStateStatus` (allow ~15s for GitHub to recompute; `UNSTABLE` = CI pending, not a conflict).
7. Repeat at the next level up. A fix discovered late (like step 5's restore) propagates the same way: commit at the lowest affected level, then clean auto-merges upward.

**When delegating a level to a subagent:** the briefing must include steps 3 and 5 explicitly — a gate of "build + tests green" alone passed a 29k-line evil merge, because formatting churn builds and tests fine.

## Gotchas (each earned)

- **Hook half-run + `--no-verify` = evil merge** — *earned by:* a stack-bottom merge where lint-staged's prettier pass rewrote 291 of the base's incoming test files (semicolon stripping) before eslint failed; the `--no-verify` conclusion baked it in, ballooning a 16-file PR to 307 files / ±29k — and every level above inherited it silently through faithful auto-merges.
- **Build/test gates don't catch formatting churn** — *earned by:* the same merge passing `go build` + three package suites; only the diff-size check (step 5) detects it. The human caught it from the PR header numbers; the check makes that mechanical.
- **GitHub's PR diff is `base...head` (merge-base three-dot)** — the local `git diff --shortstat base...head` reproduces exactly what the PR page will show; run it locally instead of pushing and squinting at the UI.
- **Mergeability recomputes asynchronously** — *earned by:* a freshly-pushed fix still showing `CONFLICTING` for ~15s; poll once before concluding the merge didn't take.
- **Pre-existing hook failures are not your license, they're your warning** — *earned by:* the "123 pre-existing ESLint errors" justification being true *and* the churn still happening; the errors being pre-existing says nothing about what the formatter half-applied.

## Divergence notes
*(append-only — when a later run diverges, record what and why)*

- **2026-06-12 (first two runs, app-cm #997–#1025):** run 1 produced the evil merge this recipe exists to prevent; run 2 (the revert propagation) validated steps 5–7 — one verbatim restore at the bottom, three clean auto-merges up, all four PRs back to honest sizes.
