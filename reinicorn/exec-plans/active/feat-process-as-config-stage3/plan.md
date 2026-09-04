---
type: plan
title: 'Execution Plan: process-as-config stage 3 — gates'
slug: feat-process-as-config-stage3
lifecycle: active
status: in-progress
created: 2026-09-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-process-as-config-stage3
ticket: N/A
spec: reinicorn/specs/process-as-config-doc-type-registry-overlay-and-declarative.md
---

# Execution Plan: process-as-config stage 3 — gates

## Goal
Stage 3 of `process-as-config-doc-type-registry-overlay-and-declarative`
(spec §3, §7.3): the enforcement events read the relation graph. Four lint
rules, `complete` refusal with an `--abandon` escape, the `_process-gate`
pre-merge CI job, and a kb made clean in the same PR — every merged
branch's plan completed, the retro-less ones backfilled. Defaults do not
flip here (`retro.closes.required` stays false; `Spec Drift` is stage 4),
so with the shipped registry the only new red is `kb/lifecycle` on a
merged-but-active plan.

## Acceptance Criteria
- [ ] `kb/required-sections`: every doc of every type with a non-empty
      `required_sections` carries the headers, except docs already in a
      completed stage (spec non-goal: no retroactive sections). Keeps the
      missing-doc-in-active-dir and `depends_on`-field-present checks.
      `kb/plan-structure` stays accepted as a config alias (stage 2's
      compat decision) and is reported as such.
- [ ] `kb/closer-filled`: an active closee whose row has a required closer,
      with the closer missing or placeholder-only, is an error naming the
      closer's `create_hint`.
- [ ] `kb/lifecycle`: an active closee whose branch is merged or deleted is
      an error "merged/deleted but still active — rcorn <type> complete".
      Three signals in spec order (published-then-deleted with evidence,
      ancestor of `origin/HEAD`, merged PR by head); every network fact
      fails open as "cannot verify"; only the project's own scope is
      judged (other scopes belong to other repos).
- [ ] `kb/draft-refs` unchanged.
- [ ] `<type> complete` exits 1 without a filled required closer, next-step
      from the closer row's `create_hint`; `--abandon` stamps
      `status: abandoned` / `lifecycle: dropped` and needs no closer. The
      post-merge sweep's refusal is visible in merge output.
- [ ] `rcorn _process-gate <branch>`: exactly required-sections, draft-refs
      and closer-filled, scoped to that branch's governed docs; no docs →
      pass. New "Process gate" job in `lint-kb.yml` that also prints
      `rcorn doc-types show`. The lint job gets `GH_TOKEN` so the merged-PR
      signal works in CI.
- [ ] Phantom-pair test extended: the synthetic closer with
      `required: true` drives refusal, abandon, closer-filled and the gate;
      the defaults assert nothing changed but the lifecycle rule.
- [ ] Kb clean: every active plan whose PR merged is completed; the 8
      without a retro get one backfilled from the PR record (Spec Drift
      stated where determinable); `rcorn kb lint` green with the new rules
      at error severity.
- [ ] Full gate green (pytest, ruff, pyright, coverage floor).

## Approach
Per spec §2c: the rules are plain functions over `iter_docs` / the stage
dirs; `staging.py` grows the one "is the closer filled" check that
`complete`, the lint and the sweep all call, so they can never disagree.
Merged-ness detection lives in the lifecycle rule (single consumer); the
gh contract (`gh pr list --json headRefName`) is mirrored once in
`github.py`. The gate reuses the rule classes and filters their
diagnostics to the branch's doc paths — no second walk. PR targets the
`feat-process-as-config` integration branch.

## Tasks
- [ ] `staging.closer_gap()` + `stage_root()`; `complete` refusal and
      `--abandon` (CLI flag, dispatch, lifecycle command).
- [ ] Generalize the structure rule to required-sections; runner alias
      for `kb/plan-structure`; seeded `linters/.lint-config.json` rows for
      the four rules.
- [ ] `kb/closer-filled` and `kb/lifecycle` rules; `gh_pr_heads` in
      `github.py`.
- [ ] `commands/internal/process_gate.py`, `_process-gate` dispatch,
      `lint-kb.yml` job + `GH_TOKEN`.
- [ ] Tests: lifecycle refusal/abandon, three rules, gate scoping, phantom
      pair, sweep visibility.
- [ ] Docs: README / GETTING-STARTED (`--abandon`, new lints),
      `linters/README.md` rule table.
- [ ] Kb: backfill retros, complete merged plans, publish; verify
      `rcorn kb lint` green.

## Dependencies
Stage 2 merged into `feat-process-as-config` (PR #70). Stage 4 (defaults
flip, AGENTS.md / PR template, ruleset required-check) follows. After
this merges: add "Process gate" to the `main-pr-gate` required checks
(repo-settings action, stage 4 per spec §3).

## Notes
- Cross-scope registry resolution (stage 2's open follow-up) is not
  resolved here; the lifecycle rule sidesteps it by judging only the
  current repo's scope.
- The hook script already lets stdout through (it silences stderr only),
  so the spec's "hook stdout" item is satisfied without a script change.
