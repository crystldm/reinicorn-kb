---
type: retro
title: 'Retro: fix/path-scoped-create-commits'
slug: fix-path-scoped-create-commits
lifecycle: active
status: draft
created: 2026-09-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: fix/path-scoped-create-commits
---

# Retro: fix/path-scoped-create-commits

## What Went Well

- Issue #35 came with a concrete repro (48 unrelated lines swept into an
  `idea:` commit), and the fix had red tests before green: `commit_kb`
  gained a `paths` parameter that scopes staging, the staged-diff check and
  the commit itself (PR #36).
- Call sites were kept to single-line edits so the concurrent frontmatter
  rewrite (PR #30) rebased trivially.
- `kb publish` and the review flow deliberately kept their
  sweep-everything behavior; the fix changed only the per-artifact commands.

## What Could Be Improved

- An unconditional `git add -A` in a shared helper had been an audit-trail
  hazard since the first commit; it took an in-review draft diverging to
  notice.
- CodeRabbit read AGENTS.md's "never raw git on the kb" literally and
  flagged the helper's own `run_git` calls, which spawned a separate
  docs PR (#40) to scope the guideline.

## Lessons Learned

- A helper that commits should take pathspecs; "commit what you touched"
  is a contract, not an optimization.

## Action Items

- None open.

## Spec Drift

None. (No spec — `spec: N/A`.)
