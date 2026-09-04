---
type: retro
title: 'Retro: feat/worktree-aware-kb-resolution'
slug: feat-worktree-aware-kb-resolution
lifecycle: active
status: draft
created: 2026-09-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat/worktree-aware-kb-resolution
---

# Retro: feat/worktree-aware-kb-resolution

## What Went Well

- Six tests against real git repos and worktrees, no git mocking, including
  an assertion on the actual `objects/info/alternates` file; verified live
  end to end (`git worktree add` → hook fires → kb populated with
  alternates) (PR #3).
- `rcorn hooks install` moved to `--git-common-dir`, so hooks land where
  git reads them from every worktree — a fix that outlived the submodule
  design.
- `kb.py` resolution logic stayed untouched, as the spec required.

## What Could Be Improved

- `git worktree remove --force` became a required step (accepted cost per
  spec) and was only documented in a skill note.
- The submodule-based `--reference` borrow was superseded when the kb
  stopped being a submodule (PR #48, `feat-remove-kb-submodule`); the
  auto-init now borrows into a plain clone. The plan stayed `active` for
  six weeks after its merge.

## Lessons Learned

- Test git behavior against git; a mocked `subprocess` cannot tell you where
  hooks are read from.
- A worktree-aware feature must be verified from inside a linked worktree,
  not just from the main checkout.

## Action Items

- None open.

## Spec Drift

Matches spec per the PR record (implements
`worktree-aware-kb-resolution`, review reinicorn-kb#1). Backfilled
2026-09-04; not re-audited against the later submodule removal.
