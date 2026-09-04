---
type: plan
title: 'Execution Plan: process-as-config stage 1 — registry loader + corpus'
slug: feat-process-as-config-stage1
lifecycle: done
status: complete
created: 2026-08-28
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-process-as-config-stage1
ticket: N/A
spec: reinicorn/specs/process-as-config-doc-type-registry-overlay-and-declarative.md
---

# Execution Plan: process-as-config stage 1 — registry loader + corpus

## Goal
Stage 1 of `process-as-config-doc-type-registry-overlay-and-declarative`
(spec §7.1): make the doc-type registry loadable from a kb-side overlay and
give the engine one shared corpus walker. Strictly behavior-preserving —
with no overlay present, every command, lint and gate behaves exactly as
today.

## Acceptance Criteria
- [x] `registry(root)` returns the effective registry: built-in defaults
      overlaid by an optional `kb/<scope>/doc-types.yaml`, resolved once
      per process into the same frozen `DocType` objects.
- [x] Overlay semantics per spec §1: override (listed fields replace
      wholesale), add (`dir_path`/`filename`/`addressing` mandatory,
      derived `help_text`/`template_body`), remove (`disabled: true`,
      dropped before invariant checks along with its own relations;
      refused only when an enabled row targets a disabled one).
- [x] Load-time validation fails closed with file path + offending key;
      all composition invariants live in one `validate()`.
- [x] `fields`/`required_fields` on the row absorb
      `frontmatter.PER_TYPE`/`PER_TYPE_REQUIRED`; branch-addressed rows
      get `branch` auto-added, `{seq}` rows get `id` auto-added.
- [x] `rcorn doc-types show` prints the effective registry as YAML with
      `built-in`/`overlay` annotations; `--schema` emits a JSON Schema
      derived from the dataclass if cheap, else dropped with a note here.
- [x] `corpus.py`: `Doc(path, dt, meta, body)`, `iter_docs(kb, scope=None)`,
      and `doc_path(dt, ident, stage=None)` covering every addressing mode;
      existing lint rules ported onto `iter_docs` (no rule hand-walks
      `kb/*/exec-plans/active` anymore).
- [x] `doc-types.yaml` added to `EXCLUDED_FILENAMES` (a non-doc kb file).
- [x] Full test suite green; new tests for overlay load/validate/annotate
      and for `iter_docs`/`doc_path` parity with today's paths.
- [x] No behavior change without an overlay: existing CLI surface,
      messages, and lint results identical (covered by existing suite).

## Approach
Per spec §2c: no base class, no plugin bus. `doc_types.py` gains the
loader, `validate()`, and the overlay schema derived from the dataclass by
field name. New `corpus.py` owns the only kb-layout walk. Module-level
`REGISTRY` import sites become `registry(root)` call sites (mechanical
sweep, this stage only where needed for the loader; the full literal sweep
of type-named code paths is stage 2). PR targets the
`feat-process-as-config` integration branch, not main.

## Tasks
- [x] `DocType`: add `fields`/`required_fields` (absorbing
      `frontmatter.PER_TYPE*`), keep frozen; defaults table updated.
- [x] `registry(root)` loader: locate `doc-types.yaml` in the kb scope
      dir, parse, apply override/add/disable semantics, memoize per
      process.
- [x] `validate()`: existing invariants + disable-atomicity + auto-added
      `branch`/`id` required fields; fail-closed errors naming file + key.
- [x] `corpus.py` with `Doc`, `iter_docs`, `doc_path`; port lint rules'
      directory walks onto it.
- [x] `rcorn doc-types show` (+ `--schema` if cheap) wired into the CLI.
- [x] `EXCLUDED_FILENAMES` += `doc-types.yaml`.
- [x] Tests: overlay round-trip, validation failures, annotation output,
      `doc_path` parity for slug/branch/singleton/`{seq}`.

## Notes
- `doc_path` is `doc_path(repo_dir, dt, ident)`; the `stage` parameter
  arrives in stage 2 when closable filenames gain the `{stage}`
  placeholder (today's patterns embed `active/`/`completed/` literally).
  `{seq}`/`{username}` filenames resolve through `iter_docs` — create-time
  seq allocation ships in stage 2 with the phantom-type test.
- `--schema` proved cheap and is included (`rcorn doc-types show --schema`).
- Registry-generated dispatch rows moved from cli import time to dispatch
  time so a broken overlay fails closed as a clean error, not a traceback.
- `spec_refs.py` is untouched in stage 1 (review call on PR #65): rather
  than half-convert a module stage 2 dissolves anyway, its constants stay
  built-in-only — overlay-added type dirs are invisible to the draft-refs
  prose matcher until stage 2 replaces the module with generic relation
  resolution.

## Dependencies
Spec approved via reinicorn-kb PR #18. Integration branch
`feat-process-as-config` (targets main only when all four stages land).
Stages 2–4 (relations + literal sweep; gates; defaults flip) build on
this.
