---
type: retro
title: 'Retro: feat-process-as-config-stage5'
slug: feat-process-as-config-stage5
lifecycle: active
status: draft
created: 2026-09-10
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-process-as-config-stage5
---

# Retro: feat-process-as-config-stage5

## What Went Well

- The gap was found by reading the shipped docs against spec §1's own claim
  ("copy the row, change one line is the documented way to customize") rather
  than by a retro; a docs-only stage closed it before the integration branch
  reached main, so the feature lands with its manual.
- Every claim in the new README section and upgrade note was checked against
  `cli.py`, `doc_types.py` and `hooks/post-merge` before commit; two claims
  were corrected on the way (`list` is slug-only; the post-merge hook never
  piped stdout).

## What Could Be Improved

- First cut appended a "Customizing" section to a README that still
  presented the default types as the product. Michael's review: document
  the generic API with placeholders first, then show spec → plan → retro
  as one configured instance. The rewrite followed; the right order was
  knowable from spec §0 ("behaviors, not types") before writing.
- The spec's stage list (§7) had no docs stage. "Documented way to customize"
  in §1 was an assumption with no owner. Stage plans should carry a
  "user-facing docs" acceptance criterion whenever a stage adds a config
  surface.

## Lessons Learned

- Product docs describing the shipped default is fine (spec §4 calls it the
  default); what must not be missing is the one paragraph saying the default
  is a config and where the config lives. Check for that paragraph, not for
  type-specific vocabulary counts.
- `using-reinicorn` was already written generically ("registered doc
  types", the wiring doc) and is the model for any future doc surface.

## Action Items

- [ ] Idea: a plan-template hint or lint that flags a stage adding a CLI
      surface (`cli.py` parser) with no README change in the same PR.

## Spec Drift

- **Accepted**: the spec schedules four stages; this fifth is the
  documentation §1 assumed and no stage owned. Docs only, no engine or
  default change.
