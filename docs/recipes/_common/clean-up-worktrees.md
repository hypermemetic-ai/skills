# How we clean up worktrees

When `git worktree list` has grown past what the live work explains — after a stack lands, a milestone closes, or agent runs leave residue. (The standing discipline — one worktree per body of work, always attributable — is [`skills/building`](../../../skills/building/SKILL.md); this is the removal procedure. First full run: app-cm 2026-06-12, 23 worktrees → 3.)

## Steps

### Mark — derive the live set (read-only)

1. Census, one command: [`bin/worktree-census`](../../../bin/worktree-census) — every worktree → branch → ticket id (naming convention) → **open-PR relation** (`#N` = branch heads that PR, a live root; `⊂#N` = HEAD contained in that PR's branch, content live), with `PR-OPEN`/`COLLECT?`/`ARCHIVE?`/`RESCUE?` flags.
2. Resolve the `?`s the script can't: ticket states by querying the tracker.
3. For never-pushed branches that *look* like dead copies of landed work, run the **patch-equivalence check**: `git cherry <pushed-stack-ref> <head>` — `0` unique (`+`) commits proves the content is patch-contained in the stack, which downgrades `ARCHIVE?` → collect. (Commit-containment can't see this; `cherry` compares patches.)
4. Build a **dossier per `RESCUE?` row** — the human can't ratify "dirty" in the abstract: unpushed-commit log (`git log --oneline origin/main..HEAD`), dirt inventory (`status --porcelain`, untracked included), and the head of any interesting file. Present each with a recommendation.

### Ratify — the human gate

5. Present the disposition table; **nothing is removed before sign-off.** `COLLECT` rows can go as a batch with a one-line report; `ARCHIVE`/`RESCUE`/unattributable rows get explicit calls.

### Sweep — execute the dispositions

6. `RESCUE` worth saving → **the archive choreography, in this order** (so every artifact can reference the previous one):
   1. create the tracker ticket (state it's an archive, where it's parked and why — "parked under M<N> because it needs a home, not because it's committed scope" is fine);
   2. push the branch — for untracked-files-only rescues, branch from the worktree's HEAD, commit the files, push (an old detached base is fine: the PR diff shows just the new commit);
   3. `gh pr create` with the ticket id in the title and a body that says **"archive PR — opened to be closed; do not merge"** + recovery instructions; then `gh pr close` with a comment;
   4. archive the ticket **with the PR attached as a link** and the supersession pointer in the body;
   5. only now remove the worktree.
7. `RESCUE` dropped without archive → the drop still gets **recorded on a live ticket** (a planning ticket's "prior art (dropped)" section naming the files) — never silent.
8. Dangling-commit worktrees (clean, on no remote, no PR): local tag (`git tag archive/<name> <sha>`) before removal — belt-and-braces, no remote noise.
9. Remove: `git worktree unlock <path>` first if locked (verify the lock isn't a *running* agent's); `git worktree remove <path>`, adding `--force` **only for ratified RESCUE rows whose dirt was triaged in step 4** — force past untriaged dirt is the recipe being skipped.
10. Delete the local branches the removals orphaned — `git branch -D` (not `-d`: patch-contained branches aren't ancestors of main, so `-d` refuses); worktrees before branches. `git worktree prune` after any manual deletion.
11. Re-run the census — exit condition: **every remaining entry attributable in one sentence** (e.g. `here · PR-OPEN #975 · PR-OPEN #998`).

## Gotchas (each earned)

- Worktrees rot invisibly and at volume — *earned by:* one repo carrying 12 manual worktrees + 11 `.claude/worktrees/agent-*` leftovers (mostly detached HEAD, one locked) from past milestones and agent runs, none tracked by anything.
- Stale worktrees actively corrupt searches, not just disk — *earned by:* a filesystem grep for a migration that "found nothing on main" because every hit was a sibling worktree and `head` truncated before the real tree (`per recipe _common/orient-on-a-pr-stack`).
- Containment-in-a-remote-branch isn't the whole liveness story — patch-equivalence is — *earned by:* four B5 sub-build branches showing `NOREMOTE`/unmerged that `git cherry` proved 0-unique against the pushed stack; without it they'd have been archived instead of collected.
- A `MERGED`-into-a-stack branch and an open-PR head are both live roots main knows nothing about — *earned by:* the root set being derived from session memory until a ratification review asked "aren't we missing open PRs?"; `gh` is shell-available, so the check folded into the census.
- Untracked files in a worktree are one cleanup from gone — *earned by:* deliberately-uncommitted conformance tests living untracked in a main checkout; a routine `git clean`/remove would have eaten them. Dirty-check means *untracked included* (`--porcelain` shows them).
- A `locked` agent worktree may belong to a *running* agent — *earned by:* live sessions holding locked entries while background work was in flight; attribute before unlocking, never unlock-and-remove as one reflex.
- The worktree pins its branch — you can't check the branch out elsewhere or delete it while the worktree exists — *earned by:* landed-stack branch deletion failing until the worktrees went first; order is worktrees → branches.
- Ticket state is not derivable from git — the census can't be fully single-command — *earned by:* the shortcut attempt: branch names carry the ticket id, but resolving its state needs the tracker. If the tracker becomes queryable from the census, fold that lookup into `bin/worktree-census` and the `?` flags resolve themselves.
- An archive isn't reachable if only git knows about it — *earned by:* the first run's rescues (CMD-418/PR #1022 rig, CMD-419/PR #1023 phi-downgrade tests): the archived ticket + closed PR pair is what makes shelved work findable from the tracker; a bare branch on origin is invisible.

## Divergence notes
*(append-only — when a later run diverges, record what and why)*

- **2026-06-12 (first full run, app-cm):** executed beyond the v1 text — added the patch-equivalence step, rescue dossiers, the five-step archive choreography, the locked/unlock handling, the dangling-commit local tag, and the `-D` branch sweep; v1's "never `--force`" sharpened to "never `--force` past *untriaged* dirt." Steps above are the as-executed form.
