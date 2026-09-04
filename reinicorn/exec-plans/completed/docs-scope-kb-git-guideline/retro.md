---
type: retro
title: 'Retro: docs/scope-kb-git-guideline'
slug: docs-scope-kb-git-guideline
lifecycle: active
status: draft
created: 2026-09-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: docs/scope-kb-git-guideline
---

# Retro: docs/scope-kb-git-guideline

## What Went Well

- One clarifying sentence in AGENTS.md fixed the misreading for every reader
  (CodeRabbit, any other agent, a new contributor) instead of a
  CodeRabbit-only path instruction or learning (PR #40, docs-only).
- The public-repo process held even for a three-line change: plan, branch,
  PR, green suite.

## What Could Be Improved

- The plan stayed `active` for three weeks after the merge. The branch was
  retained on origin, so the post-merge sweep — which only archives docs
  whose remote branch is gone — never touched it.
- Plan-plus-PR ceremony outweighed the change; a docs-only fix needs the
  audit trail, not the full planning template.

## Lessons Learned

- Guideline prose that bans a tool ("never raw git") must say who it binds.
  Reviewers, human or bot, read literally.

## Action Items

- Merged-but-active plans are now a `kb/lifecycle` lint error
  (process-as-config stage 3); this retro is part of that backfill.

## Spec Drift

None. (No spec — `spec: N/A`.)
