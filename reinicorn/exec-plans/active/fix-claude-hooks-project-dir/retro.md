---
type: retro
title: 'Retro: fix/claude-hooks-project-dir'
slug: fix-claude-hooks-project-dir
lifecycle: active
status: draft
created: 2026-09-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: fix/claude-hooks-project-dir
---

# Retro: fix/claude-hooks-project-dir

## What Went Well

- One root cause explained a whole class of failures: Claude Code runs hook
  commands via `/bin/sh` from wherever the session started, so
  bare-relative `.reinicorn/hooks/foo.sh` breaks in worktrees and
  subdirectory sessions. Generated entries now carry
  `"$CLAUDE_PROJECT_DIR"` (PR #47).
- The repair covers stale entries too: pre-rename `.reins/hooks/` and
  bare-relative `.reinicorn/hooks/` share a matcher with their
  replacements, so matcher-only dedup kept broken hooks alive; they are
  dropped before dedup and the repaired count is reported.
- Tests pin the preservation of unrelated user hooks on the same matcher.

## What Could Be Improved

- This repo's own `.claude/settings.json` had been hand-fixed in #16 weeks
  earlier while the installer kept emitting broken paths for every other
  repo. A local patch masked a product bug.

## Lessons Learned

- When fixing a generated config in your own repo, fix the generator or
  file the bug the same day.

## Action Items

- None open.

## Spec Drift

None. (No spec — `spec: N/A`.)
