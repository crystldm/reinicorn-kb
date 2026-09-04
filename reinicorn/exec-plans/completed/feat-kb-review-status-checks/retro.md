---
type: retro
title: 'Retro: feat-kb-review-status-checks'
slug: feat-kb-review-status-checks
lifecycle: active
status: draft
created: 2026-09-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-kb-review-status-checks
---

# Retro: feat-kb-review-status-checks

## What Went Well

- The review-ref grammar (`parse_review_ref`) and the draft/candidate
  comparison (`candidate_matches`) were extracted once and shared by the
  new `_review-check` CI command, `_review-cleanup` and the merge-time
  guard, so the check and the guard cannot drift apart (PR #62).
- Ruleset reconciliation merges the required-status-check rule into an
  existing ruleset while preserving user rules, contexts, strictness and
  bypass actors; a stale ruleset is reported as outdated without `--force`.
- The bundled workflow reused the cleanup workflow's conventions
  (`__REINICORN_REPO__` placeholder, install-token fallback, head-ref env
  indirection, `contents: read` only) — no new secrets, no new patterns.

## What Could Be Improved

- +988/−95 across 13 files in one PR: the CI command, the workflow asset,
  the setup/reconcile logic and the tests could have been two stages.
- The plan stayed `active` after the merge because the branch was retained
  on origin.

## Lessons Learned

- A status check is only as trustworthy as the ruleset that requires it;
  installing the workflow without reconciling the ruleset is a false
  sense of safety.
- "Every violation reported in one run, each with its remediation" is the
  right contract for a CI check an agent will read.

## Action Items

- None open. The ruleset required-check for the process gate follows the
  same reconciliation path in process-as-config stage 4.

## Spec Drift

Matches spec per the PR record (implements
`kb-review-prs-get-real-github-status-checks-doc-lint-candida`, approved in
reinicorn-kb#17; merge-time guards kept as the spec required). Not
re-audited in this backfill.
