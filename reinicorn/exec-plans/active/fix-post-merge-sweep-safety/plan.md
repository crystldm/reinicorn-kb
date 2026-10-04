---
type: plan
title: 'Execution Plan: fix-post-merge-sweep-safety'
slug: fix-post-merge-sweep-safety
lifecycle: active
status: in-progress
created: 2026-10-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: fix-post-merge-sweep-safety
ticket: N/A
spec: specs/post-merge-sweep-safety.md
---

# Execution Plan: fix-post-merge-sweep-safety

## Goal
Make the post-merge stale-plan sweep safe: no cross-scope archiving, no
archiving on a stale remote view, no half-moved plans, visible hook output,
and no acting on a plan dir owned by a different branch.

## Acceptance Criteria
- [ ] Sweep in repo A leaves repo B's active plans untouched on a shared kb.
- [ ] A branch present on origin but absent from local remote-tracking refs keeps its plan.
- [ ] `plan complete` with a kb that cannot fast-forward leaves active/ intact and the kb tree clean.
- [ ] `hooks/post-merge` no longer discards stderr.
- [ ] `plan create` / `plan complete` refuse a plan dir whose `branch:` is another branch.
- [ ] Full gate green.

## Approach
Red-green per spec section, one commit each. See spec
`post-merge-sweep-safety` (review: crystldm/reinicorn-kb#19).

## Tasks
- [ ] 1. Scope-restricted sweep
- [ ] 2. `git ls-remote --heads origin` liveness
- [ ] 3. Check-then-mutate in `plan complete`
- [ ] 4. Hook stderr
- [ ] 5. Plan-dir ownership check (create, complete, sweep)

## Dependencies
None. Injective encoding follows feat-process-as-config's merge to main.
