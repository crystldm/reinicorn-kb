---
type: spec
title: post-merge sweep safety
slug: post-merge-sweep-safety
lifecycle: active
status: in-review
created: 2026-10-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/19
---

# post-merge sweep safety

## Problem

The post-merge hook's stale-plan sweep (`rcorn _post-merge`) is the one kb
operation that runs unattended, and it is destructive: it moves plans from
`active/` to `completed/` and commits. Four defects (debt
`post-merge-plan-sweep-archives-plans-it-cannot-vouch-for`) let it act without
grounds or leave a half-done move, and the hook discards everything it says:

1. It walks every kb scope but checks them against the current repo's
   branches, so on a shared kb it archives other repos' plans.
2. It reads branch liveness from local remote-tracking refs with no fetch, so
   a teammate's freshly pushed branch reads as deleted.
3. `plan complete` edits and moves files before `commit_kb` checks the kb is
   on an up-to-date main; when that check fails the move stays uncommitted.
4. `hooks/post-merge` sends stderr to `/dev/null`.

A fifth, adjacent gap makes all of the above worse: branch→directory encoding
maps `feature/good` and `feature-good` to the same plan dir, and nothing
checks that the plan in a dir belongs to the branch acting on it.

## Design Goals

- The sweep never archives a plan outside the current repo's kb scope.
- The sweep archives only on an authoritative "gone from origin" answer; any
  failure to get that answer archives nothing.
- `plan complete` either completes fully (edit, move, commit) or changes
  nothing on disk.
- Whatever the sweep reports reaches the terminal of the person who merged.
- No mutating plan command acts on a plan dir whose recorded `branch:` names a
  different branch.

## Design

### 1. Sweep the current scope only

`_archive_stale_plans(root)` resolves `kb_scope(root)` and sweeps
`<kb>/<scope>/exec-plans/active/` only. Other scopes are owned by other repos'
hooks.

### 2. Ask origin, not the local refs

`_live_remote_branches` runs `git ls-remote --heads origin` and takes the
names after `refs/heads/` (output format: `<sha>\trefs/heads/<name>`, per
git-ls-remote(1)). A non-zero exit or exception returns `None`, which already
means "archive nothing". This costs one network round trip per merge, the
same as the fetch the kb commit already does.

### 3. Check, then mutate

`cmd_plan_complete` calls `ensure_kb_on_main(kb_dir)` before it touches any
file. On failure it reports and returns 1 with the plan still in `active/` and
the kb working tree unchanged. `commit_kb` keeps its own check (it guards
every other caller).

### 4. Let the hook speak

`hooks/post-merge` becomes `rcorn _post-merge || true`: still never fails the
merge, but warnings and the "Plan archived" line are visible. Existing installs
pick it up through `rcorn update` / `rcorn hooks install`.

### 5. Ownership check on the plan dir

One helper, `plan_branch_mismatch(pdir, branch)`, reads `branch:` from
`pdir/plan.md` and returns the recorded branch when it is set and differs
from `branch`. Callers:

- `plan create`: an existing dir owned by another branch is an error naming
  both branches (today it says "Plan already exists" and adopts it).
- `plan complete`: refuses to archive another branch's plan.
- the sweep: passes the recorded `branch:` to `plan complete` (not the dir
  name), so a dir whose name is not the encoding of its own `branch:` is
  skipped by the same check.

## Non-Goals

- An injective branch→directory encoding and the migration of existing dirs.
  It touches the process gate on `feat-process-as-config`; it follows that
  feature's merge to main. This spec only makes the collision loud.
- Syncing the kb clone before the sweep decides: the decision reads only
  `branch:` fields, and `ensure_kb_on_main` runs before any archive.
- Sweeping plans whose branch exists locally but was never pushed (see the
  "push right after `rcorn plan create`" convention); unchanged.
