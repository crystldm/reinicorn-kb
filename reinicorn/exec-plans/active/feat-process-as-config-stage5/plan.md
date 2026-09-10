---
type: plan
title: 'Execution Plan: feat-process-as-config-stage5'
slug: feat-process-as-config-stage5
lifecycle: active
status: in_progress
created: 2026-09-10
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-process-as-config-stage5
ticket: N/A
spec: reinicorn/specs/process-as-config-doc-type-registry-overlay-and-declarative.md
---

# Execution Plan: feat-process-as-config-stage5

## Goal
Ship the manual for process-as-config. Stages 1-4 made the doc process data
(`kb/<scope>/doc-types.yaml`, `rcorn doc-types show`), but no user-facing doc
says the process is configurable or how: the string `doc-types.yaml` appears
in no markdown outside the spec. Spec §1 assumed "copy the row, change one
line" would be *the documented way to customize* and no stage owned writing
it. Docs only; no engine change.

## Acceptance Criteria
- [x] README: doc-type table labelled as the *default* types; new
      "Customizing the process" section covering overlay location, override /
      add / disable semantics, fail-closed validation, `rcorn doc-types show`
      and `--schema`, and the RFC → ADR worked example from spec §5
- [x] GETTING-STARTED: one paragraph pointing at that section
- [x] linters/README: states once that the process rules read the registry,
      so a custom type gets them without new rules
- [x] `upgrades/v0.4.md`: release note for the overlay, required retro and
      Spec Drift, Process gate check
- [ ] `rcorn kb lint` and the process gate pass; gate suite green

## Approach
Product docs keep describing spec → plan → retro as the shipped default
(spec §4 calls it that); they gain one section that says the default is a
config and how to change it. This repo's AGENTS.md and PR template stay
specific — they govern this repo, which runs the default process (spec §6:
repo prose, not engine). The `using-reinicorn` skill is already generic and
is the model. Content is drawn from spec §1/§5 and the loader docstrings, not
invented.

## Tasks
- [x] README "Customizing the process" + table heading
- [x] GETTING-STARTED pointer paragraph
- [x] linters/README registry sentence
- [x] upgrades/v0.4.md
- [x] Retro filled (Spec Drift), lint + gate, PR onto the integration branch

## Dependencies
Branches from `feat-process-as-config-stage4` (PR #74, unmerged) because it
edits the same README paragraphs; the PR targets #74's branch and retargets
to `feat-process-as-config` when #74 merges.

## Spec Drift
- **Accepted**: the spec has no stage 5; this is the documentation §1 assumed
  and no stage scheduled. No engine or default change.
