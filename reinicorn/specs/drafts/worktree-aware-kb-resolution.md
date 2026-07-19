# Worktree-aware kb resolution

**Date:** 2026-07-19
**Author:** Michael Biehl
**Status:** in-review
**Origin:** ai-assisted
**Human-validated:** false
**Review-PR:** https://github.com/crystldm/reinicorn-kb/pull/1

## Problem

`git worktree add` never initializes submodules, so a fresh linked worktree has an
empty `kb/` directory and every `rcorn` kb command dies with "No kb submodule found."
The obvious fix — `git submodule update --init` per worktree — creates a full
independent kb clone under `.git/worktrees/<name>/modules/kb` (git shares nothing
between worktree submodule clones), and an initialized submodule makes the worktree
immovable (`git worktree move` refuses) and removable only with `--force`.

None of that cost buys anything here: the kb deliberately ignores its gitlink pin
(session-start rebases it onto `origin/main` unconditionally) and kb docs are
branch-scoped internally, so every worktree wants the *same* kb state. Symlinking
`kb/` into the worktree was tested and rejected: git reports a `T` typechange,
`git add -A` would commit a symlink over the gitlink, and `reset --hard` silently
deletes the link. `get_kb_dir` would also reject the symlink target as outside the
repo root.

## Design Goals

- Every `rcorn` kb command works from a linked worktree with an uninitialized
  (empty) `kb/` directory, operating on the main checkout's kb.
- No git state change in the worktree: `kb/` stays uninitialized, `git status`
  stays clean, `git worktree move`/`remove` keep working without `--force`.
- Behavior in the main checkout (and in submodules of other projects) is unchanged.
- A worktree that *has* initialized its own kb keeps using it (explicit local
  state wins over redirection).

## Design

In `src/reinicorn/kb.py`, `get_kb_dir` learns to redirect to the main checkout:

1. Resolve the kb candidate path from `.gitmodules` as today.
2. If `candidate / ".git"` exists (file or dir), return it — initialized local
   kb always wins, in main checkouts and worktrees alike.
3. Otherwise, detect a linked worktree: `git rev-parse --git-dir` differs from
   `git rev-parse --git-common-dir` **and** `git rev-parse
   --show-superproject-working-tree` is empty (the submodule guard — inside a
   submodule those dirs also differ).
4. In a linked worktree, the main checkout root is the parent of the resolved
   `--git-common-dir`. Re-run the `.gitmodules` path resolution against that
   root, including the existing containment check (path must resolve inside the
   *main* root), and return the main checkout's kb dir if it is initialized.
5. If the main checkout's kb is also uninitialized, fall through to the current
   behavior (return the local candidate / error from `require_kb_dir`), with the
   error message extended to mention the worktree case per golden principle 4
   (what/where/how-to-fix).

`session-start.sh` needs no change: its `kb/.git` guard already skips
uninitialized worktrees, and `rcorn kb status --compact` starts working there
via this redirection.

### Opt-in: real per-worktree kb via `--reference`

When a worktree genuinely needs its own kb checkout (e.g. testing kb-pinned
behavior), the sanctioned way is to borrow objects from the main checkout's
module clone instead of re-fetching over the network:

```sh
git submodule update --init \
  --reference "$(git rev-parse --path-format=absolute --git-common-dir)/modules/kb" -- kb
```

The `using-git-worktrees` skill (Step 2, project setup) documents this as the
opt-in path. `--reference` for `git submodule add/update` dates to git 1.6.4
(2009), so there is no meaningful version floor. Caveat: the borrowed clone
uses alternates, so aggressive `git gc` in the main module could prune objects
a borrower still needs — acceptable here because both sides track the same
`origin/main`; pass `--dissociate` for a standalone (still network-free) copy
if that ever matters. Rule 2 above means such a worktree keeps using its own
kb automatically.

## Non-Goals

- No automatic `git submodule update --init` in worktrees (that is the cost we
  are avoiding, not a fallback to add).
- No symlinking of `kb/` (tested; fights git — see Problem).
- No automatic `--reference` init during worktree creation — the opt-in command
  above stays a deliberate, documented choice, not default behavior.
- No locking/coordination for concurrent kb writes across worktrees — the same
  race already exists between any two sessions today and is handled by
  rebase-on-publish.
