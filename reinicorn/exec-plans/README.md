# Execution Plans

## What Are Execution Plans?

Execution plans are first-class artifacts that describe what a branch is doing, why, and what parts of the codebase it touches. They serve two critical purposes:

1. **Agent autonomy** — An agent working on a feature branch has a structured plan it can follow without constant human direction. The plan captures the goal, approach, acceptance criteria, and current progress in one place.
2. **Cross-branch awareness** — Other branches (and their agents) can read what's in flight elsewhere in the repo, detect overlapping work, and avoid conflicts before they happen.

Every feature branch should have an execution plan. Plans are cheap to create and expensive to skip.

## Lifecycle: Active vs Completed

Execution plans move through a simple lifecycle:

- **Active** (`active/{branch-name}/`) — The branch is in progress. The plan, progress log, and decisions files are actively maintained here.
- **Completed** (`completed/{branch-name}/`) — The branch has been merged. The plan directory is moved here as a historical record. Completed plans are useful for understanding past decisions and for onboarding.

Plans may also carry a status field within `plan.md` itself: `planning`, `in-progress`, `complete`, or `abandoned`.

## Branch Namespacing Convention

Each branch gets its own directory under `active/`:

```
exec-plans/
  active/
    feat/add-auth/
      plan.md
      progress.md
      decisions.md
    fix/header-overflow/
      plan.md
      progress.md
      decisions.md
  completed/
    feat/user-profiles/
      ...
  _template/
    plan.md
    progress.md
    decisions.md
```

The directory name matches the branch name (with `/` preserved as path separators). This makes it trivial to find the plan for any branch.

## Auto-Creation from Issue Tracker Context

When a new branch is created and a ticket ID is detected in the branch name (e.g., `feat/PROJ-1234-add-auth`), agents can auto-generate an execution plan by:

1. Pulling the ticket title, description, and acceptance criteria from the issue tracker.
2. Copying the `_template/` files into `active/{branch-name}/`.
3. Populating `plan.md` with the ticket context.
4. Creating an initial `progress.md` entry.

This removes the friction of plan creation. The agent starts with structure from day one.

## Cross-Branch Awareness via git

Overlap between active branches is computed on demand by querying git directly. When `reins plan status` or `reins kb sync` runs, the CLI enumerates active plan directories under `exec-plans/active/`, then for each one runs `git diff --name-only <merge-base>..<branch>` to determine the files that branch has changed. Overlap is the set intersection across branches.

This is not a lock — it's awareness. Two branches can intentionally touch the same area, but they do so knowingly rather than discovering the conflict at merge time. Because git is the source of truth, there is no per-branch file to maintain and no staleness risk.

## Plan Lifecycle on Merge

When a branch is merged:

1. The branch's directory is moved from `active/` to `completed/`.
2. The `plan.md` status is updated to `complete`.
3. A final `progress.md` entry is added noting the merge.

This happens as part of the merge workflow (manually or via automation).

## Stale Branch Cleanup

The doc-gardening agent periodically reviews `active/` directories and:

- Flags plans whose branches no longer exist in the remote.
- Marks abandoned plans (no progress entries for an extended period).
- Moves stale plans to `completed/` with status `abandoned`.

This keeps the `active/` directory a reliable view of what's actually in flight.
