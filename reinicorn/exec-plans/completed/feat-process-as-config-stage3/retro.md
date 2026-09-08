---
type: retro
title: 'Retro: feat-process-as-config-stage3'
slug: feat-process-as-config-stage3
lifecycle: active
status: draft
created: 2026-09-08
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-process-as-config-stage3
---

# Retro: feat-process-as-config-stage3

## What Went Well

- One filled-check for everything: `staging.closer_gap` is what `complete`
  refuses on, what `kb/closer-filled` reports and what the post-merge sweep
  surfaces, so the three can never disagree. The phantom-pair test drove
  refusal, `--abandon`, the lint and the gate with no type names in the
  engine (PR #71, 1512 tests, 89.9% coverage).
- Merged-ness detection shipped with the three spec signals in order and
  every network fact failing open; the `gh pr list` contract was verified
  against live output before it was mirrored in `github.py`.
- The kb was made clean in the same PR: 12 merged plans completed, 8
  retro-less ones backfilled with a Spec Drift section, `rcorn kb lint`
  green with the new rules at error severity.
- Review ran verify-first on all seven findings (three human, four
  CodeRabbit): two applied in one commit (whole-heading match for required
  sections, bounded probe timeouts), one declined with the reason on the
  thread (`GH_TOKEN` on a `pull_request` event is already read-only), one
  deferred into a filed idea (non-injective branch-to-directory encoding).

## What Could Be Improved

- This retro was not written inside the stage-3 PR; it is backfilled from
  stage 4 — exactly the gap the stage-4 gate now closes by requiring the
  retro before merge.
- The spec's whole-kb `kb/closer-filled` would have redded every in-progress
  plan the moment `required` flipped. It surfaced only while planning stage
  4, not while writing the rule.
- CodeRabbit does not auto-review PRs against non-`main` base branches;
  each re-review needed a manual trigger and the plan allows one per hour.

## Lessons Learned

- A required-section match must be the whole heading: `## Design Notes` was
  satisfying `Design` until the regex was anchored.
- Every subprocess that touches the network needs a timeout, or "fail open"
  degrades to "hang" in CI.
- When a lint is specified over "active" docs, ask when the doc is *due*. A
  rule that is red during normal work is a rule people disable.

## Action Items

- Stage 4 (this integration branch): flip `retro.closes.required`, add
  `Spec Drift`, AGENTS.md / PR template, `main-pr-gate` required check
  after the main merge.
- Idea filed: `branch-to-directory-encoding-is-not-injective-feature-good-a`.
- Still open from stage 2: cross-scope registry resolution for a shared kb.

## Spec Drift

- `kb/required-sections` judges only docs still being authored (completed-
  stage, closed-lifecycle and approved gated docs are exempt) rather than
  "every doc of every type" — **accepted**: five legacy specs predate the
  template, and drafts are linted by the review lane's own CI.
- The structure rule was renamed `kb/required-sections` with
  `kb/plan-structure` kept as a config alias — **amended** toward the spec.
- Hook script unchanged (it already lets stdout through) — **accepted**.
- Backfill covered 8 retro-less plans, not the spec's 7 — **accepted**.
