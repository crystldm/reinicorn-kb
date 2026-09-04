---
type: retro
title: 'Retro: fix/retro-show-branch-discovery'
slug: fix-retro-show-branch-discovery
lifecycle: active
status: draft
created: 2026-09-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: fix/retro-show-branch-discovery
---

# Retro: fix/retro-show-branch-discovery

## What Went Well

- TDD fix for an agent-discoverability dead-end: `retro show` / `plan show`
  on another branch now lists the branches that have the doc instead of
  suggesting a create command that only works on the current branch
  (PR #17). Four new tests; two tests that had pinned the buggy behavior
  were replaced.
- An obsolete idea (`plan create` ignoring a name argument) was
  investigated and resolved in the kb without a code change.

## What Could Be Improved

- The dead-end existed because next-step hints were unconditional; the
  hint text was never checked against the context it appeared in.

## Lessons Learned

- A `next:` hint must be valid where it is shown. Discovery at the point
  of failure ("branches with a retro: …") beats a separate list command
  the agent has to know about.

## Action Items

- `plan list` / `retro list` remain optional future work.

## Spec Drift

None. (No spec — `spec: N/A`.)
