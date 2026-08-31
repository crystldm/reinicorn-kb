---
type: plan
title: 'Execution Plan: process-as-config stage 2 — relations + literal sweep'
slug: feat-process-as-config-stage2
lifecycle: active
status: in-progress
created: 2026-08-31
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-process-as-config-stage2
ticket: N/A
spec: reinicorn/specs/process-as-config-doc-type-registry-overlay-and-declarative.md
---

# Execution Plan: process-as-config stage 2 — relations + literal sweep

## Goal
Stage 2 of `process-as-config-doc-type-registry-overlay-and-declarative`
(spec §7.2): declarative `depends_on`/`closes` relations on registry rows,
graph queries replacing every literal `REGISTRY["plan"]`-style lookup, the
type-name literal sweep of the engine, and a generic `complete`. Strictly
behavior-preserving — with the default registry, every command, lint and
gate behaves exactly as today. New enforcement (gates, lints, CI) is
stage 3.

## Acceptance Criteria
- [ ] `DocType` gains `depends_on: {field, type, status}` and
      `closes: {type, required}`; `validate()` gains the relation
      invariants (targets exist and are enabled, `depends_on.field` is a
      declared field, `closes` pairs both branch-addressed, one enabled
      closer per closee, no closer chains).
- [ ] Graph queries on the effective registry — `closer_of`,
      `closable_types`, `dependencies_of`, `rows_with(field)` — with
      None/empty meaning "skip the behavior".
- [ ] `spec_refs.py` dissolves: type-agnostic resolution machinery moves
      to `refs.py` serving `depends_on` generically (draft-refs lint +
      pre-push gate become relation-driven); spec-specific functions and
      the module name are gone. `staging.py` owns `closes` stage logic.
- [ ] Closable filenames use the `{stage}` placeholder
      (`plan: "{stage}/{branch}/plan.md"`); `doc_path(dt, ident,
      stage=None)` is the single path contract for every addressing mode;
      closer paths derive via `closer_of` (today's `_branch_target`
      special case deleted).
- [ ] Generic `complete`/`status` (`doc_lifecycle.py`) generated for every
      closable type; post-merge archival iterates `closable_types()`.
- [ ] `{seq}` create-time allocation: max-existing + 1 over the type's dir
      (and `drafts/` when gated), stamped into `id`; `show` resolves id or
      slug, erroring with the matches when ambiguous.
- [ ] Literal sweep per §2b: outside the `doc_types.py` defaults table,
      no identifier or user-facing string in `src/reinicorn` names a
      built-in type; enforced by an AST-walking test.
- [ ] Phantom-pair test: a synthetic closable/closer pair (with
      `depends_on` on the closable row) asserts create placement,
      complete, archival and the lint/gate paths all fire from registry
      rows alone; the defaults assert nothing changed.
- [ ] Full suite green; no behavior change with the default registry.

## Approach
Per spec §2/§2b/§2c: relations are plain fields read by the events that
already exist; shared logic gets `refs.py` (depends_on: lint + push gate)
and `staging.py` (closes: stage resolution, move, filled-check). No base
class, no hook bus. Sweep is verified by an executable test, not review
eyeballs. PR targets the `feat-process-as-config` integration branch.

## Tasks
- [ ] `DocType.depends_on`/`closes` + relation invariants in
      `_validate_rows`; overlay schema picks the new keys up
      automatically; disabled rows still drop their own relations.
- [ ] Graph queries in `doc_types.py`.
- [ ] `{stage}` placeholder: plan row filename becomes
      `{stage}/{branch}/plan.md`; `doc_path` grows `stage=`;
      `iter_branch_dirs` and the structure lint follow.
- [ ] `refs.py`: move resolution machinery from `spec_refs.py`;
      `declared_dependency(doc, dt)` reads `depends_on.field`; port
      draft-refs lint and the pre-push gate
      (`ensure_dependencies_approved`); delete `spec_refs.py`.
- [ ] `staging.py`: stage resolution, dir move, closer-filled check;
      `plan.py` becomes generic `doc_lifecycle.py`; CLI generates
      `complete` for closable types.
- [ ] `{seq}` allocation in `doc_create.py` + id/slug resolution in
      `doc_show.py`.
- [ ] Sweep the thirteen files; add the AST literal-sweep test.
- [ ] Phantom-pair test + overlay round-trip tests for relations.

## Notes
(filled as work proceeds)

## Dependencies
Stage 1 merged into `feat-process-as-config` (PR #65). Spec approved via
reinicorn-kb PR #18. Stage 3 (gates) and stage 4 (defaults flip) build on
this.
