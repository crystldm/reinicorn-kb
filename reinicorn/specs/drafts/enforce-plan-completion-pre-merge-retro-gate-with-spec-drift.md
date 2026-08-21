---
type: spec
title: 'Enforce plan completion: pre-merge retro gate with spec-drift accounting'
slug: enforce-plan-completion-pre-merge-retro-gate-with-spec-drift
lifecycle: active
status: in-review
created: 2026-08-21
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/16
---
# Enforce plan completion: pre-merge retro gate with spec-drift accounting

## Problem

Four retros written on 2026-08-20/21 (mattpocock-adapter, registry stage 2,
enforce-review-lane, unified-frontmatter) found the same three failures
independently:

- **Plans are never completed.** Every audited plan was still
  `lifecycle: active` with unticked boxes weeks after its PR merged.
  `rcorn plan complete` is voluntary, the post-merge hook only archives
  plans whose remote branch was *deleted* (and pipes its output to
  /dev/null), and doc-gardening's orphan sweep has the same blind spot:
  a merged branch left on origin looks alive forever.
- **Retros are optional in practice.** `plan complete` merely warns when
  the retro is empty, and the post-merge auto-archive path completes plans
  with no retro at all. The four retros above exist only because of a
  manual sweep.
- **Spec drift is recorded nowhere.** Stages 3-5 of `registry-driven-doc-types`
  never shipped, #27 silently cut spec §6, #30 never back-filled the
  review-lane fields into `unified-kb-doc-frontmatter-schema` — none of it
  recorded in a spec amendment, debt doc, or review thread. Reviewers have
  no contract that says drift is their problem.

## Design Goals

- Every merged feature branch ends with a filled retro and a completed
  plan, with no manual policing.
- Spec drift is accounted for *before* merge: either "none" or an explicit
  disposition per deviation, visible to the reviewer.
- Enforcement is mechanical (lint rule, CI required check, CLI refusal);
  the judgment parts live in AGENTS.md and the PR template, which
  reviewers (human and CodeRabbit) already read.
- Branches without a plan (dependabot, trivial docs) pass untouched.
- The 11 legacy active plans get a stated migration path; the gate cannot
  red every PR on day one.

## Design

### 1. Retro rides the PR (pre-merge)

`rcorn retro create` already places `retro.md` in the *active* plan dir
when one exists. Make that the contract: the retro is written on the
branch, as the last act before merge, and reviewed like code. Post-merge
findings are appended to the completed retro later when they surface.

### 2. `Spec Drift` becomes a required retro section

Add `"Spec Drift"` to the retro row's `required_sections` in
`doc_types.REGISTRY` (template follows automatically via `{sections}`).
Content contract, documented in the section placeholder: enumerate every
deviation between the plan's declared `spec:` and what shipped — in either
direction — each with a disposition: **amended** (link the spec review
PR), **debted** (link the debt doc), **accepted** (one-line reason), or
the single word "None."

### 3. Lint: `kb/retro-structure`

New lint rule (sibling of `plan_structure.py`, registry-driven): every
`retro.md` under `active/*/` or `completed/*/` must carry all
`required_sections` headers, and a retro in an *active* plan dir must not
be empty (reuse `_retro_is_empty`, moved from `commands/plan.py` to a
shared home so both consumers import it). Completed legacy retros are
checked for sections they were created with only (no retroactive
`Spec Drift` requirement — see §7).

### 4. CI required check: "Retro gate"

New job in `lint-kb.yml` (runs on `pull_request`): resolve the PR head
branch's active plan in the kb clone; if no plan exists, pass. If a plan
exists: fail unless `retro.md` exists, is non-empty, and contains all
required sections including `## Spec Drift`. Implemented as
`rcorn _retro-gate <branch>` (internal command, same pattern as
`_post-merge`) so the logic is tested Python, not workflow bash. After the
implementation merges, add `Retro gate` to the `main-pr-gate` ruleset's
required status checks next to `Run pytest` (repo-settings action,
recorded in the plan).

### 5. `rcorn plan complete` refuses without a retro

Exit 1 when `retro.md` is missing or `_retro_is_empty`, with next-step
`rcorn retro create`. New `--abandon` flag for dropped work: stamps
`status: abandoned` / `lifecycle: dropped`, requires no retro. The
post-merge auto-archive keeps calling `cmd_plan_complete` and now simply
fails for retro-less plans: change the hook script to stop discarding
stdout so the refusal and its next-step are visible in the merge output.

### 6. Lint: `kb/plan-lifecycle`

Active plan whose `branch:` is gone from origin **or** whose
`origin/<branch>` tip is an ancestor of `origin/main` → error: "branch
merged/deleted but plan still active — rcorn plan complete <branch>".
Catches the merged-but-undeleted case every retro flagged. Network-derived
facts (ls-remote, merge-base) fail open as "cannot verify", mirroring
`_live_remote_branches`.

### 7. Review rules: AGENTS.md + PR template

- AGENTS.md "Pull requests": the body states spec-drift status — "matches
  spec" or the deviation list with dispositions, mirroring the retro's
  Spec Drift section.
- AGENTS.md gains a "Reviewing" bullet list: the reviewer diffs the change
  against the plan's declared spec; undisclosed drift is a blocking
  finding; the retro is part of the reviewed diff, held to the same bar as
  code.
- PR template checklist: replace the stale kb items (`progress.md`,
  `decisions.md`, `kb/exec-plans/` paths predate the current layout) with:
  retro filled including Spec Drift, and drift dispositions recorded.

### 8. Rollout

Implementation branch backfills first: retros for the remaining 7
retro-less legacy plans (Spec Drift included where determinable),
`plan complete` for every merged branch, then the lint rules land error-
severity from the start against a clean kb. The ruleset change (§4) is
last.

## Non-Goals

- No `.coderabbit.yaml`: CodeRabbit config stays in the UI; the committed
  review rules live in AGENTS.md and the PR template only (decided
  2026-08-21).
- No change to the solo-maintainer approval gate
  (`review-lane-solo-maintainer-self-review-gate` draft owns that).
- No doc-gardening consolidation: the weekly report keeps its own orphan
  step; deduplicating it against `kb/plan-lifecycle` is a follow-up.
- No retro *content* quality gate beyond structure and non-emptiness.
- No retroactive `Spec Drift` sections in already-completed retros.
