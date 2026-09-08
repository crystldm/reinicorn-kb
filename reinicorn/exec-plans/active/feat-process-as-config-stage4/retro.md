---
type: retro
title: 'Retro: feat-process-as-config-stage4'
slug: feat-process-as-config-stage4
lifecycle: active
status: draft
created: 2026-09-08
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-process-as-config-stage4
---

# Retro: feat-process-as-config-stage4

## What Went Well

- The flip was data, as spec §2c promised: two row values and one new row
  field (`section_hints`). `render_doc`, the overlay coercion, the JSON
  schema and `doc-types show` picked it up through the same seams
  `extra_meta` already uses; no new engine path.
- The one real decision — what whole-kb `kb/closer-filled` should flag once
  the retro is required — was raised once with a recommendation, settled in
  one exchange, and written into the plan's Spec Drift before any code.
- The gate proved itself on its own PR: `_process-gate` would not pass
  until this retro existed and was filled, scaffolded by the in-repo
  template (`uv run rcorn retro create`, stated as pre-install testing).
- Twelve tests broke on the flip, every one because a fixture had built the
  old default implicitly. One shared `FILLED_RETRO` in `conftest.py` fixed
  them and now doubles as the executable definition of "filled".

## What Could Be Improved

- The stage-3 retro had to be backfilled here: the process that requires it
  did not exist when stage 3 merged.
- The installed `rcorn` lagged the integration branch by three stages until
  it was reinstalled from the stage-4 worktree; mid-feature kb operations
  run on whichever build is installed, and the per-stage plan should say
  which.
- `section_hints` is deliberately not validated against
  `required_sections` (a narrowing override must stay one line), so a typo
  in an overlay hint is silent until the first create.

## Lessons Learned

- "Active" is not "due". A whole-kb rule needs a moment at which the
  artifact is owed, or it is red during normal work and gets disabled.
- A default flip is a fixture audit: reread every test that *built* the old
  default, not only the ones that assert it.
- Prose rules (AGENTS.md, PR template) ship in the same PR as the
  mechanical ones, so a reviewer meets the gate's demand and its rationale
  together.

## Action Items

- After `feat-process-as-config` merges to `main`: add "Process gate" to
  the `main-pr-gate` required status checks, then reinstall `rcorn` from
  `main`.
- Open the integration branch (stages 1–4) against `main`.
- Still open: cross-scope registry resolution (stage 2); non-injective
  branch-to-directory encoding (idea filed in stage 3).

## Spec Drift

- Whole-kb `kb/closer-filled` reports only a closer that exists but is
  placeholder-only; a missing closer is a finding only in the process gate
  (strict mode) and at `complete` — **accepted**: spec §6 puts the retro
  inside the PR diff, so an in-progress plan has none by design, and the
  verbatim §3 rule would red `rcorn kb lint` and the "Run kb lints" CI job
  whenever anyone is mid-work. Decided 2026-09-08 and recorded in the plan.
- The Spec Drift placeholder is delivered through a new overlay-settable
  `section_hints` row field rather than a retro-specific template body —
  **accepted**: `{sections}` stays the one scaffold path for every row.
- The `main-pr-gate` required check waits for the integration branch to
  reach `main` — not drift: the spec says "after the implementation
  merges", and a required check must exist in `main`'s workflow before
  unrelated PRs can satisfy it.
