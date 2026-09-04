---
type: retro
title: 'Retro: feat-process-as-config-stage2'
slug: feat-process-as-config-stage2
lifecycle: active
status: draft
created: 2026-09-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-process-as-config-stage2
---

# Retro: feat-process-as-config-stage2

## What Went Well

- Behavior-preserving with the default registry, as the plan promised: the
  1,418 pre-existing tests passed with only mechanical adjustments for
  moved internals (PR #70, +2131/−795 in 46 files).
- The literal sweep is an executable rule, not review eyeballs:
  `tests/test_no_type_literals.py` walks the AST of `src/reinicorn` and
  fails on any type-key word outside the defaults table, with a documented
  allowlist.
- Review response ran verify-first: all nine CodeRabbit findings were
  checked against the code before acting — eight applied in one commit,
  one deferred with a stated reason (required-closer enforcement is a
  stage-3 gate; enforcing it early would strand the post-merge sweep).
  CodeRabbit verified and resolved every thread within a minute.

## What Could Be Improved

- CodeRabbit auto-review is disabled for non-`main` base branches, so the
  fix commit on the integration-branch PR got no re-review; the full pass
  only happens when the integration branch opens against `main`.
- The registry cache is keyed by cwd when no root is passed, which cost a
  round of test-patching confusion; tests now `chdir` instead of patching
  `repo_root` in two modules.
- Cross-scope registry resolution (a shared kb's other scopes judged by the
  current repo's overlay) was raised and left open.

## Lessons Learned

- A rename of a lint rule is a silent disable in deployed configs; the
  compat answer is an alias, not a frozen name (stage 3 aliases
  `kb/plan-structure` → `kb/required-sections`).
- The `_template` constant and the post-merge sweep's per-type remote query
  are small duplications worth a follow-up, not a blocker.

## Action Items

- Decide cross-scope registry resolution (per-scope overlays for a shared
  kb) before a second repo shares this kb.
- Stage 3: gates, `complete` refusal + `--abandon`, process-gate CI job.

## Spec Drift

- `kb/required-sections` shipped under the name `kb/plan-structure` —
  **accepted** in stage 2 for config compatibility; **amended** in stage 3
  by renaming with a config alias.
- `rows_with(field)` graph query dropped (callers filter inline) —
  **accepted**: a wrapper added nothing.
- `closes.required` is declared but not enforced — not drift: the spec's
  own staging puts enforcement in stage 3.
