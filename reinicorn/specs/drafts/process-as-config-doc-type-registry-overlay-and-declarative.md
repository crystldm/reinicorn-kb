---
type: spec
title: 'Process-as-config: doc-type registry overlay and declarative type relations'
slug: process-as-config-doc-type-registry-overlay-and-declarative
lifecycle: active
status: in-review
created: 2026-08-22
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/18
---

# Process-as-config: doc-type registry overlay and declarative type relations

## Problem

Four retros written 2026-08-20/21 (mattpocock-adapter, registry stage 2,
enforce-review-lane, unified-frontmatter) found the same failures
independently: every audited plan was still `lifecycle: active` weeks after
its PR merged; `rcorn plan complete` only *warns* when the retro is empty and
the post-merge auto-archive completes plans with no retro at all; and spec
drift (stages 3-5 of `registry-driven-doc-types` never shipped, #27 cut
spec §6, #30 skipped the frontmatter back-fill) is recorded nowhere.

The first fix drafted for this (`enforce-plan-completion-pre-merge-retro-
gate-with-spec-drift`, kb PR #16, withdrawn) patched the symptom: a
`Spec Drift` literal on the retro row, a bespoke `_retro-gate` command, a
`kb/retro-structure` lint and a `kb/plan-lifecycle` lint — four more code
paths that name `spec`/`plan`/`retro` by key, on top of the thirteen files
that already do (`_branch_target` carries a comment saying "this coupling
stays code"; `spec_gate.py`, `plan_structure.py`, `post_merge.py`,
`plan.py`, `status.py`, `kb.py`, `spec_refs.py`, `frontmatter.PER_TYPE`).

That is the design gap. `registry-driven-doc-types` made the registry the
single source for *what a doc type is* but explicitly deferred two things:
"the config file itself" (a follow-up spec) and "plan lifecycle as data —
the retro-rides-with-plan coupling stays code". The process — which type
must point at which approved type, which type closes which, what a
reviewer must see before merge — therefore lives only in Python. Opting out
of a retro section, or running an RFC → ADR process instead of
spec → plan → retro, means forking the engine.

## Design Goals

- The process is data: a repo changes its doc process (sections, which
  types depend on or close which, what gates merge) by editing one config
  file, with zero engine changes. Deleting "Spec Drift" from a list is the
  whole opt-out.
- The engine's enforcement *events* stay fixed and few (lint, pre-push,
  pre-merge CI check, `complete`, post-merge archive); what varies is the
  type graph they read. No rule language, no custom events.
- Every built-in behavior today — spec gate, retro placement, plan
  completion — is expressible as a default config row, and the shipped
  defaults reproduce it exactly plus the retro gate and spec-drift
  accounting the retros asked for.
- A second process (RFC → ADR) is a worked example in this spec and a
  phantom-type test in the suite, not a promise.
- Branches without governed docs (dependabot, trivial docs) pass every gate
  untouched; the 11 legacy active plans get a stated migration path.

## Design

### 0. Behaviors, not types

The engine never asks *what* a doc is, only what it can do. A type is a
named bundle of behaviors; the name exists for humans and the CLI. Every
engine branch is `if dt.<behavior>` or `for dt in rows_with(<behavior>)`,
never `if dt is X`. Each behavior is one registry field (or enum choice)
and the events that react to it:

| behavior | field | reacting events |
|---|---|---|
| addressed by slug / branch / singleton | `addressing`, `filename` | CLI arg shape, `doc_path`, validation |
| numbered | `{seq}` in `filename` | create (allocate id), show (resolve id) |
| review-gated | `gated` | create → drafts, review lane, status vocabulary, review stamps |
| protected | `protected` | edit hook |
| structured | `required_sections` | create template, `required-sections` lint |
| indexed | `index_file` | freshness lint, dashboard |
| appendable | `create_mode` | create |
| seeded | `readme_label` | kb seed |
| depends on | `depends_on` | `draft-refs` lint, pre-push gate |
| closable / closer | `closes` | CLI `complete`/`status` verbs, staged `doc_path`, create placement, `closer-filled` + `lifecycle` lints, pre-merge gate, post-merge archive |

`spec` = slug + gated + protected + structured + indexed + seeded; `retro`
= branch + protected + structured + closer-of(plan); an `rfc` is spec's
bundle plus numbered. The invariants in §1 are composition rules ("gated
⇒ slug", "closer ⇒ bare filename"), which is why they carry no type names
either. A new **type** is overlay data; a new **behavior** is an engine
change with its own spec — the overlay composes behaviors, it cannot
define them.

### 1. Registry overlay: `kb/<scope>/doc-types.yaml`

`doc_types.REGISTRY` becomes the **built-in defaults**. The effective
registry is `registry(root)`: defaults overlaid by an optional
`doc-types.yaml` in the kb scope dir, resolved once per process and
validated into the same frozen `DocType` objects, so every consumer keeps
its types and the module-level `REGISTRY` import sites become call sites
(mechanical sweep).

Placement in the kb scope dir, not the code repo: process is a property of
the doc corpus, it is reviewed where the docs are, a shared kb serving
several repos gets per-scope process, and the kb-side checks
(`kb-review-prs-get-real-github-status-checks`) see it without the
consuming repo checked out. The file is a non-doc (added to
`EXCLUDED_FILENAMES`), published with `rcorn kb publish` like any kb file.

Overlay semantics, by `doc_types.<key>`:

- **Override** a built-in row: only the listed fields change
  (`required_sections: [...]` replaces the list wholesale — no merge magic).
- **Add** a row: `dir_path`, `filename` and `addressing` are mandatory;
  enums by their string value. Everything else takes the dataclass default
  plus two derived ones so a minimal row is usable: `help_text` →
  `"<key> doc operations"`, `template_body` → `"{sections}"`.
- **Remove** a built-in row: `disabled: true`; refused if any other row's
  relation targets it.

Load-time validation fails **closed** with the file path and offending key
— a broken process config must not silently revert to defaults. Invariants:
existing ones (`gated` ⇒ slug addressing), relation targets exist, relation
fields are declared fields of the row, `filename` placeholders match
`addressing`, `closes` pairs are both branch-addressed, at most one enabled
closer per closee and a closer is not itself closable (depth one — no
chains), and a branch-addressed row always has `branch` in
`required_fields` (the loader adds it; its value is the git branch name,
path-sanitized through the existing `sanitize_branch`).

**Filename patterns.** Slug-addressed rows may use a `{seq}` placeholder
with a width — `"RFC-{seq:04}-{slug}.md"` — for numbered corpora. The
number is allocated at create time as max-existing + 1, found by matching
the pattern's own regex over the type's dir (and `drafts/` when gated, so
two open drafts never collide), and stamped into the doc's `id` field
(`RFC-0007`). `show` resolves either the id or the slug. The slug stays the
identity for review-lane refs and `depends_on`; the number is a display
prefix, never re-allocated.

The per-type frontmatter vocabulary moves into the row too — `fields` and
`required_fields` absorb `frontmatter.PER_TYPE` / `PER_TYPE_REQUIRED` — so
a custom type can declare the field its `depends_on` uses.

`rcorn doc-types show` prints the effective registry as YAML, each row
annotated `built-in` / `overlay`, so "copy the row, change one line" is
the documented way to customize.

### 2. Two relations, declared on the row

```yaml
doc_types:
  plan:
    depends_on: {field: spec, type: spec, status: approved}
  retro:
    closes: {type: plan, required: true}
    required_sections: [What Went Well, What Could Be Improved,
                        Lessons Learned, Action Items, Spec Drift]
```

- `depends_on: {field, type, status}` — the doc's `field:` must resolve
  (via `spec_refs`, unchanged) to a tracked doc of `type` with `status`, or
  be the N/A sentinel. Generalizes the plan → approved-spec rule.
- `closes: {type, required}` — this type is the closer of `type`. Implies:
  the closer is created *inside* the closee's dir (today's `_branch_target`
  special case, now a graph lookup `closer_of(dt)`); `<closee> complete`
  moves the dir from the active to the completed stage with both docs; and,
  when `required`, the closee cannot complete without a filled closer.

  One path contract, one function. A closable type's `filename` must be
  `{stage}/{branch}/<name>` (`plan` becomes `"{stage}/{branch}/plan.md"`;
  `stage` ∈ the existing `active`/`completed` constants, `branch` through
  `sanitize_branch`). A closer's `filename` is a bare name with no
  placeholders or `/` (`"retro.md"`). `doc_path(dt, branch, stage)` is the
  only path computation: for a closable row it formats the pattern; for a
  closer it is `doc_path(closee, branch, stage).parent / closer.filename`.
  Create, show, complete, archive and the lints all call it. Creating a
  closer resolves `stage` to wherever the closee currently lives; if no
  closee doc exists for that branch, `completed` (a retro without a plan is
  a closed branch, as today).

The `complete` verb is generated for every type that something `closes`
(today only plan), alongside `create`/`show`. Queries on the effective
registry — `closer_of(type)`, `closable_types()`, `dependencies_of(type)`
— replace the literal `REGISTRY["plan"]`/`["retro"]`/`["spec"]` lookups in
all thirteen files. A relation with no match returns `None` and the caller
skips, which is how a repo with no `closes` row gets no retro behavior.

### 2b. No type names in the engine

The sweep is not only `REGISTRY["plan"]` lookups. After stage 2 the only
place a built-in type key may appear in `src/reinicorn` is the defaults
table in `doc_types.py` — no identifiers (`_archive_stale_plans`,
`plan_dir`, `active_plan_names`, `cmd_plan_complete`, `cmd_plan_status`),
no user-facing strings (`"Plan archived:"`, `"No retro captured"`), no
per-type tables (`frontmatter.PER_TYPE`, `status.py`'s debt-index line,
`kb_seed`'s README rows). Each becomes a registry-driven generic:
`stage_dir(dt, stage)`, `cmd_doc_complete`/`cmd_doc_status` for every
closable type, overlap detection over the active stage of every closable
type, dashboard lines over every row with `index_file`, README rows from
`readme_label`, messages from `dt.key`/`dt.create_hint`. The executable
form: a test walks `src/reinicorn` (AST identifiers and string literals,
`doc_types.py` defaults excluded) and fails on any token equal to a
default type key. Tests may still name the defaults — they are the fixture.

### 2c. Code structure: adding a behavior is a field plus event hooks

No `Behavior` base class, no plugin discovery, no hook bus. Behaviors are
plain fields; events are the plain, explicit lists that already exist;
one new shared helper removes the walk that every rule hand-rolls today.

- **`doc_types.py`** — `DocType` fields, the defaults table, `validate()`
  (all composition invariants in one function), `registry(root)` loader,
  and the graph queries (`closer_of`, `dependencies_of`, `closable_types`,
  `rows_with(field)`). The overlay schema is derived from the dataclass by
  field name — there is no second schema to keep in sync; adding a field
  makes it an overlay key and a `doc-types show` line automatically.
- **`corpus.py`** (new) — `Doc(path, dt, meta, body)` and
  `iter_docs(kb, scope=None)`, which yields every governed doc already
  paired with its row and parsed frontmatter, plus `doc_path(dt, branch,
  stage)` from §2. Lint rules, gates and the dashboard become
  `for doc in iter_docs(kb): if doc.dt.<behavior>: …` — the only place the
  kb layout is walked, so no rule can rebuild `kb/*/exec-plans/active` by
  hand.
- **Events keep their homes.** `cli.py` (generation loop), `doc_create.py`,
  `doc_lifecycle.py` (today's `plan.py`, generic `complete`/`status`),
  `linter/rules/<rule>.py` registered in `BUILTIN_RULES`, `pre_push.py`'s
  ordered checks, `post_merge.py`, `process_gate.py`. Each reads the
  field(s) it reacts to and nothing else.
- **Shared behavior logic lives in one module only when ≥2 events need
  it**, following `spec_refs.py` (lint + push gate): `refs.py` for
  `depends_on`, `staging.py` for `closes` (stage resolution, move,
  filled-check). A behavior touching one event lives in that event.

The recipe, kept as the `doc_types.py` module docstring so it is read by
whoever adds the next one:

1. Add the field to `DocType` with an off-by-default value (or a new enum
   member), so every existing row is unaffected, and its composition
   invariant to `validate()`.
2. For each event that reacts: one function or rule reading `dt.<field>`,
   appended to that event's explicit list (and enabled in the seeded
   `linters/.lint-config.json` for a lint rule).
3. Extend the phantom-type test: a synthetic row with the behavior on
   asserts each event fires; the defaults assert nothing changed.
4. Document the field in the spec that introduces it; `doc-types show`
   and the overlay accept it without further work.

### 3. Gates: fixed events, graph-driven

| event | reads | rule |
|---|---|---|
| `rcorn kb lint` | `required_sections` | `kb/required-sections`: every doc of every type with a non-empty list carries the headers (generalizes `plan_structure`, which was `registry-driven-doc-types` stage 4, never shipped) |
| `rcorn kb lint` | `depends_on` | `kb/draft-refs` unchanged in behavior, now iterates every row with `depends_on` |
| `rcorn kb lint` | `closes` | `kb/closer-filled`: an active closee with a required closer whose closer is missing or has only placeholder sections → error (`_retro_is_empty` moves from `commands/plan.py` to the linter as `sections_empty`) |
| `rcorn kb lint` | `closes` | `kb/lifecycle`: active closee whose `branch:` is gone from origin or whose `origin/<branch>` is an ancestor of `origin/main` → error "merged/deleted but still active — rcorn <type> complete". Network facts fail open as "cannot verify", mirroring `_live_remote_branches` |
| pre-push | `depends_on` | `spec_gate.ensure_plan_spec_approved` becomes `ensure_dependencies_approved`: for each pushed branch, every branch-addressed type with `depends_on` is checked. Same fail-open-loud contract |
| pre-merge CI | `closes` + sections | new job "Process gate" in `lint-kb.yml` running `rcorn _process-gate <branch>`: exactly `kb/required-sections`, `kb/draft-refs` and `kb/closer-filled`, scoped to that branch's docs (`kb/lifecycle` is excluded by design — the branch under review is unmerged, and other branches' staleness must not red this PR); no governed docs → pass. The job also prints `rcorn doc-types show` so a process weakened by an overlay edit is visible in the check log. Added to `main-pr-gate` required checks after the implementation merges (repo-settings action, recorded in the plan) |
| `<type> complete` | `closes` | exit 1 without a filled required closer, next-step rendered from the closer row's existing `create_hint` (honors a custom `create_verb`); `--abandon` stamps `status: abandoned` / `lifecycle: dropped` and needs no closer |
| post-merge | `closes` | `_archive_stale_plans` calls the generic `complete`; the hook script stops piping stdout to `/dev/null` so a refusal and its next-step are visible in merge output |

### 4. Shipped defaults

The built-in rows encode the current flow plus what the retros asked for:
`plan.depends_on` = approved spec (unchanged behavior); `retro.closes`
plan, `required: true` (new); `Spec Drift` appended to retro
`required_sections` (new). The `Spec Drift` placeholder states the content
contract: enumerate every deviation between the plan's declared `spec:` and
what shipped, each with a disposition — **amended** (link the spec review
PR), **debted** (link the debt doc), **accepted** (one-line reason) — or
the single word "None."

### 5. Worked example: RFC → ADR

```yaml
doc_types:
  spec:  {disabled: true}
  plan:  {disabled: true}
  retro: {disabled: true}
  rfc:
    dir_path: rfcs
    filename: "RFC-{seq:04}-{slug}.md"
    addressing: slug
    gated: true
    required_sections: [Summary, Motivation, Detailed Design, Drawbacks, Alternatives]
    fields: [superseded_by]
  adr:
    dir_path: decisions
    filename: "{slug}.md"
    addressing: slug
    required_sections: [Context, Decision, Consequences]
    fields: [rfc]
    depends_on: {field: rfc, type: rfc, status: approved}
```

Gets: `rcorn rfc create` through the review lane with `RFC-0001-…`
numbering, `rcorn adr create`, the draft-refs lint and pre-push gate on
`adr.rfc`, required-section lint on both, `help_text`/`template_body`
derived, no retro or completion machinery at all. Zero engine changes. The
"phantom type" test from `registry-driven-doc-types` is extended with a
synthetic closer/dependency pair and asserts placement, gate, lint and
`complete` appear with no other change.

### 6. Review rules: AGENTS.md + PR template

Repo prose, not engine:

- AGENTS.md "Pull requests": the body states spec-drift status — "matches
  spec" or the deviation list with dispositions, mirroring the retro.
- AGENTS.md "Reviewing": the reviewer diffs the change against the plan's
  declared spec; undisclosed drift is a blocking finding; the retro is part
  of the reviewed diff and held to the same bar as code.
- PR template: the stale kb checklist (`progress.md`, `decisions.md`,
  `kb/exec-plans/` paths) is replaced with "retro filled incl. Spec Drift"
  and "drift dispositions recorded".

### 7. Stages (one PR each onto a `feat-process-as-config` integration branch)

1. **Loader + corpus.** `registry(root)`, overlay parsing + `validate()`,
   `fields` absorption of `PER_TYPE`, `rcorn doc-types show`, `corpus.py`
   with existing lint rules ported onto `iter_docs`. Behavior-preserving.
2. **Relations.** `depends_on`/`closes` fields, graph queries, literal sweep
   of the thirteen files, generic `complete`, phantom-pair test.
   Behavior-preserving (defaults unchanged).
3. **Gates.** The four lint rules, `complete` refusal + `--abandon`, hook
   stdout, `_process-gate` + CI job. Lints ship at error severity against a
   kb made clean in the same PR: backfill the 7 retro-less legacy plans
   (Spec Drift where determinable), `plan complete` every merged branch.
4. **Defaults + rules.** Flip `retro.closes.required` and add `Spec Drift`;
   AGENTS.md / PR template; ruleset required-check last.

## Non-Goals

- **No rule language, no rules/policy library.** No conditions, custom
  events, or scripting in the overlay; if a process needs a gate the
  engine lacks, that is an engine change with its own spec. The checks are
  ~6 fixed functions, so a rules engine or policy language (OPA, CUE) and a
  validation library (pydantic) were considered and rejected at this size
  — stdlib dataclass + one `validate()` suffices. Revisit trigger: the
  first request for a *conditional* in the overlay ("required only when
  …"); at that point a policy language beats growing our own (idea:
  `policy-language-for-overlay-conditions`). One zero-dependency nicety is
  in scope for stage 1 if cheap: `rcorn doc-types show --schema` emitting a
  JSON Schema derived from the dataclass, for editor validation of
  `doc-types.yaml`.
- **Overlay is not review-gated.** `doc-types.yaml` ships with
  `kb publish`. The process gate guards a team norm, not an adversary: a
  collaborator with kb push can already edit a retro to say "None." or
  delete the plan outright, so gating the overlay alone would not raise
  the bar. The kb repo's `main-safety` ruleset is the trust boundary;
  the CI gate prints the effective registry (§3) so a weakening is at
  least visible in the check log. Routing overlay edits through the
  review lane is a follow-up if that visibility proves insufficient.
- **Overlay does not live in the code repo.** Rejected above; revisit only
  if a per-repo override of a shared scope is needed.
- No `.coderabbit.yaml` (decided 2026-08-21); no change to the
  solo-maintainer approval gate
  (`review-lane-solo-maintainer-self-review-gate`); no doc-gardening
  consolidation (its orphan step and `kb/lifecycle` dedupe later).
- No retro *content* quality gate beyond structure and non-emptiness; no
  retroactive `Spec Drift` sections in already-completed retros.
- Supersedes only the "plan lifecycle as data" non-goal of
  `registry-driven-doc-types`; that spec's unshipped stages 3 and 5
  (literal sweep, shipped-docs genericization) are absorbed by stage 2 here
  where they overlap and otherwise remain that spec's.
