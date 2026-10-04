---
type: plan
title: 'Execution Plan: fix-seed-principles-frontmatter'
slug: fix-seed-principles-frontmatter
lifecycle: active
status: in-progress
created: 2026-10-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: fix-seed-principles-frontmatter
ticket: N/A
spec: N/A
---

# Execution Plan: fix-seed-principles-frontmatter

## Goal
Bug fix (reported 2026-10-04): `rcorn init` seeds `golden-principles.md`
without frontmatter, so a freshly initialized kb fails `rcorn kb lint`
(`kb/frontmatter`). `rcorn principle add` then appends to the bare file and
never repairs it.

## Acceptance Criteria
- [ ] A freshly seeded kb scope passes `kb/frontmatter`.
- [ ] `rcorn principle add` on an existing bare file leaves a lint-clean doc and keeps the existing content.
- [ ] Seed and `principle add` share one rendering of the file's frontmatter.
- [ ] Full gate green.

## Approach
Red-green. Extend the born-passing test to cover the seed tree; one shared
renderer for the principles singleton's header used by both paths.

## Tasks
- [ ] Red: seed tree lint-clean; append heals a bare file
- [ ] Green: shared header renderer in seed + append
- [ ] Gate, PR

## Dependencies
None.
