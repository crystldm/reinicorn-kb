---
type: retro
title: 'Retro: test/lint-and-runner-coverage'
slug: test-lint-and-runner-coverage
lifecycle: active
status: draft
created: 2026-09-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: test/lint-and-runner-coverage
---

# Retro: test/lint-and-runner-coverage

## What Went Well

- Coverage measurement (PR #20) exposed that `commands/lint.py` was never
  imported by the suite — reachable only through a lazily imported
  dispatch entry — and that `linter/runner.py`'s external-rule loop was
  dead to the fixture, which shipped no `linters/rules/` dir (PR #22).
- Both gaps were closed with tests only: 0% → 100% and 45% → 100%, no
  production change, stacked cleanly on #20.

## What Could Be Improved

- A 0% module went unnoticed until a coverage floor existed; the
  lazy-import dispatch table hides untested commands from import-time
  errors as well as from coverage.

## Lessons Learned

- Fixtures must mirror the shipped layout; a missing directory silently
  turns a live loop into dead code for the whole suite.
- A string assertion about a workflow file is not a test of the command
  the workflow runs.

## Action Items

- None open.

## Spec Drift

Matches spec per the PR record (P1 and P2 of `test-coverage-reports`;
scope was the two gaps). Not re-audited in this backfill.
