---
type: plan
title: 'Execution Plan: process-as-config stage 4 — defaults + rules'
slug: feat-process-as-config-stage4
lifecycle: active
status: in-progress
created: 2026-09-08
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-process-as-config-stage4
ticket: N/A
spec: reinicorn/specs/process-as-config-doc-type-registry-overlay-and-declarative.md
---

# Execution Plan: process-as-config stage 4 — defaults + rules

## Goal
Stage 4 of `process-as-config-doc-type-registry-overlay-and-declarative`
(spec §4, §6, §7.4), the last stage on the `feat-process-as-config`
integration branch: the shipped defaults encode what the retros asked for
(`retro.closes.required: true`, `Spec Drift` in the retro), the repo's
review rules say how drift is disclosed and reviewed (AGENTS.md, PR
template), and the "Process gate" check becomes required on `main` once
the integration branch lands there.

## Acceptance Criteria
- [x] `retro.closes.required` is `true` in the built-in row: with no
      overlay, `rcorn plan complete` refuses without a filled retro and
      `--abandon` is the recorded escape.
- [x] `Spec Drift` is the retro's fifth required section. A new retro
      scaffolds it with a placeholder stating the content contract (every
      deviation from the plan's declared `spec:` with a disposition —
      amended / debted / accepted — or the single word "None."). The
      placeholder comes from a `section_hints` row field so an overlay can
      set its own; `sections_empty` still treats it as unfilled.
- [x] Whole-kb `kb/closer-filled` reports only an empty closer stub (retro
      present, placeholder-only). A missing retro on an in-progress plan
      is not a whole-kb finding; the process gate and `complete` enforce
      it where the retro is due (see Spec Drift).
- [x] `rcorn _process-gate` still fails on a missing required closer, so
      a PR cannot merge without its retro.
- [x] AGENTS.md: "Pull requests" states spec-drift status ("matches spec"
      or the deviation list with dispositions); a "Reviewing" section
      makes undisclosed drift a blocking finding and holds the retro to the
      code's bar.
- [x] PR template: the stale kb checklist (`progress.md`, `decisions.md`,
      `kb/exec-plans/` paths) is replaced with "retro filled incl. Spec
      Drift" and "drift dispositions recorded".
- [x] Docs follow the defaults: README, GETTING-STARTED, `linters/README.md`.
- [x] Kb: the stage 3 plan gets its retro (with Spec Drift) and is
      completed; this branch's retro is filled in this PR because the gate
      now demands it; `rcorn kb lint` is green under the flipped defaults.
- [x] Full gate green (pytest, ruff, pyright, coverage floor).
- [ ] After `feat-process-as-config` merges to `main`: "Process gate" added
      to the `main-pr-gate` required status checks (repo-settings action —
      only then, because a required check must exist in `main`'s workflow
      for unrelated PRs to run it).

## Spec Drift
- Whole-kb `kb/closer-filled` narrowed to empty stubs — **accepted**. The
  spec's §3 row says "closer missing or placeholder-only", but §6 puts the
  retro inside the reviewed PR diff, so every in-progress plan legitimately
  has no retro until its PR is up; the verbatim rule would red `rcorn kb
  lint` (and the "Run kb lints" CI job on unrelated PRs) whenever anyone
  is mid-work. The gate keeps the strict pre-merge check and `complete`
  keeps the refusal, so a plan still cannot merge or close without a
  filled retro. Decided with Michael on 2026-09-08.
- Ruleset required-check deferred past this PR — not drift: the spec's own
  wording is "after the implementation merges"; the implementation reaches
  `main` when the integration branch does.

## Approach
Per spec §2c the flip is data: two row values and one new row field. The
`section_hints` field rides the same overlay coercion as `extra_meta`
(string mapping) and is rendered by `render_doc`'s `{sections}` expansion;
the retro's hint is the only built-in use. `CloserFilledRule` gains a
`presence` mode: the gate constructs it strict (missing counts), the
whole-kb runner uses the default (stubs only). Prose changes are repo
files only. PR targets `feat-process-as-config`.

## Tasks
- [x] Defaults: `retro.closes.required=True`, `Spec Drift` section,
      `section_hints` field + coercion + validation + render + `doc-types
      show`, schema output.
- [x] `kb/closer-filled` presence mode; gate strict.
- [x] Tests: defaults graph, retro scaffold, render hints, overlay
      coercion, closer-filled modes, gate, lifecycle default refusal.
- [x] AGENTS.md, `.github/PULL_REQUEST_TEMPLATE.md`, README,
      GETTING-STARTED, `linters/README.md`.
- [x] Kb: stage 3 retro + complete; stage 4 retro; lint green; publish.
- [ ] Post-main-merge: `main-pr-gate` required check.

## Dependencies
Stage 3 merged into `feat-process-as-config` (PR #71). Closes the staged
feature; the integration branch then opens against `main`.

## Notes
- Existing completed-stage retros without `Spec Drift` are exempt from
  `kb/required-sections` (stage 3's authoring-scope decision); no backfill.
- Open follow-ups carried from earlier stages: cross-scope registry
  resolution; non-injective branch-to-directory encoding (idea filed).
