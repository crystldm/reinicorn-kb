---
type: debt
title: post-merge plan sweep archives plans it cannot vouch for
slug: post-merge-plan-sweep-archives-plans-it-cannot-vouch-for
lifecycle: active
status: draft
created: 2026-10-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
category: kb
severity: high
remediation: planned
---

# post-merge plan sweep archives plans it cannot vouch for

## Impact

The `post-merge` hook runs `rcorn _post-merge`, which archives active plans
whose branch is gone from origin. Four defects make it archive plans it has no
basis to judge, or leave the kb half-moved, and hide all of it:

1. **Cross-scope sweep.** `_archive_stale_plans` walks every scope directory in
   the kb but compares their `branch:` values against the *current* repo's
   remote branches. On a shared multi-repo kb, merging in repo A archives
   every active plan of repo B whose branch name A's origin lacks — i.e. all
   of them.
2. **Stale remote view.** Liveness comes from `git branch -r`, the local
   remote-tracking refs, without a fetch. A teammate's freshly pushed branch
   looks deleted until the next fetch, so their plan gets archived.
3. **Mutate before the up-to-date check.** `cmd_plan_complete` rewrites the
   frontmatter and moves `active/<b>` to `completed/<b>`, *then* calls
   `commit_kb`, whose `ensure_kb_on_main` refuses when the kb clone cannot
   fast-forward. The result is an uncommitted half-move in the kb working
   tree (observed in the `feat-markdown-lint` worktree, 2026-10).
4. **Silent hook.** `hooks/post-merge` runs `rcorn _post-merge 2>/dev/null ||
   true`; the warnings above never reach the user.

Related: the branch→directory encoding is not injective (`feature/good` and
`feature-good` share a dir; idea
`branch-to-directory-encoding-is-not-injective-feature-good-a`), so a plan
dir can be claimed by the wrong branch.

## Remediation Plan

Spec `post-merge-sweep-safety`: sweep only the current scope, ask origin
directly (`git ls-remote --heads`), verify the kb is on an up-to-date main
before touching any file, let hook output through, and refuse to act on a plan
dir whose `branch:` is a different branch than the one asking.
