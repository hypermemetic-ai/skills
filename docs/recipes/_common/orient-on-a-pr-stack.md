# How we orient on a milestone's PR landing shape

When you need "how is this milestone actually landing" — which builds are on main, which ride open PRs, and how the PRs stack — without trusting ticket prose.

## Steps

1. Per build ticket, find its PRs: `gh pr list --state all --search "CMD-XXXX" --json number,state,mergedAt,headRefName,isDraft` — tickets↔PRs are many-to-many, so run it per ticket, not once.
2. For every `MERGED` PR, check **what it merged into**: `gh pr view N --json baseRefName`. Merged-into-a-stack-branch is not merged-into-main.
3. Refresh your view of main: `git fetch origin main`, then `git rev-list --count main..origin/main` — if nonzero, grep `origin/main`, never the local checkout.
4. Verify landed names on unmerged branches **without checking out**: `git fetch origin <branch>` then `git grep "<pattern>" origin/<branch> -- 'backend/**/*.go'`.
5. Draw the stack: each open PR's `baseRefName` is its parent; the chain (main ← A ← B ← C) *is* the landing plan — report it as such.

## Gotchas (each earned)

- A `MERGED` badge does not mean the code is on main — *earned by:* app-cm #983 (B6.1) showed MERGED while `origin/main` had none of its types; its base was the #975 stack branch, and only checking `baseRefName` exposed it.
- Repo-root greps lie in a worktree-heavy repo — `.claude/worktrees/*` checkouts match first and `head` truncates before the real tree appears — *earned by:* a `user_uuid` sweep that "found nothing on main" while listing fifteen worktree hits; scope greps to `git grep <ref> --` instead of the filesystem.
- Quote glob filters under zsh (`--include="*.go"`) or the shell eats them with `no matches found` — *earned by:* an entire verification batch silently returning empty.
- The local default branch is routinely behind origin — *earned by:* local main was 8 commits behind during the same sweep; the fetch+count check is two seconds and prevents false "not landed" conclusions.
- **After concluding any merge commit, verify the PR diff size matches intent** (`git diff --shortstat base...head`) — pre-commit hooks (lint-staged/prettier) can half-run against the merge's staged files and a `--no-verify` conclusion bakes their formatter churn into an *evil merge* — *earned by:* a stack-bottom merge that ballooned its PR from 16 files to 307 (±29k of pure semicolon stripping across the base's incoming test suite), silently inherited by every branch above it. Full procedure + avoidance: `per recipe _common/propagate-merges-up-a-pr-stack`.

## Divergence notes
*(append-only — when a later run diverges, record what and why)*
