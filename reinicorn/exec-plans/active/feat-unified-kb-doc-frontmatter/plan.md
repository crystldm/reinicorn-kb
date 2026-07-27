# Execution Plan: feat/unified-kb-doc-frontmatter

**Ticket:** N/A
**Spec:** specs/unified-kb-doc-frontmatter-schema.md
**Author:** Michael Biehl
**Created:** 2026-07-27
**Status:** planning

## Goal

Give every kb doc real fenced YAML frontmatter with a validated schema, and route
all metadata reads/writes through one module so no consumer regex-scans prose
again.

The concrete bug this closes: `doc-gardening.yml` orphan detection has no field
to read a plan's branch from, so it sanitizes remote heads (`tr '/' '-'`) and
compares them to sanitized plan-dir names. That mapping is lossy and many-to-one
— a deleted `feature/mvp` plan is not flagged as orphaned if a literal
`feature-mvp` branch exists.

## Acceptance Criteria

- [ ] `src/reinicorn/frontmatter.py` is the only module that touches the `---`
      block; `docmeta.py` is deleted.
- [ ] Every kb doc (minus the excluded non-docs) carries valid frontmatter.
- [ ] `dumps(*parse(text)) == text` holds for every doc the migration emits.
- [ ] `rcorn kb lint` fails at `error` severity on missing/invalid frontmatter.
- [ ] Orphan detection matches exact refs — a `feature-mvp` branch no longer
      masks a deleted `feature/mvp` plan.
- [ ] `rcorn review start` / `merge` / `cancel` round-trip without disturbing
      doc bodies.
- [ ] Full suite green; coverage stays at or above the 87% floor.

## Approach

Decisions taken (2026-07-27):

| Decision | Choice |
|---|---|
| YAML impl | `pyyaml>=6.0.3` as reinicorn's first runtime dep. Corpus values (`[TICKET-ID or N/A]`, `Tech Debt: Output Discipline`) require real scalar quoting in both directions; a hand-rolled writer would silently corrupt docs. |
| Migration | One-shot, no compat shim. `frontmatter.py` only ever knows YAML. |
| `type:` values | `doc_types.REGISTRY` keys — `plan`, `debt`, not the spec's `exec-plan`/`tech-debt`. Validation is `meta["type"] in REGISTRY`; no second vocabulary. |
| Lint severity | `kb/frontmatter` at `error` on landing; `kb/provenance` rule and its `.sh` deleted. |

Three gaps found in the spec during exploration:

1. **Review-lane fields are missing from the schema.** `review.py` writes
   `Review-PR`, `Approved-by`, `Review-cancelled` today. They become
   `review_pr` / `approved_by` / `review_cancelled`, allowed on gated types.
2. **The API needs a text-level core.** The spec words it as `read(path)` /
   `write(path, …)`, but `review.py` reads candidate content out of `git show`
   and never from a path. Core is `parse(text)` / `dumps(meta, body)`; the
   path forms are thin wrappers.
3. **Round-trip stability is load-bearing, not cosmetic.**
   `push_candidate` asserts the review ref differs from main by exactly one
   added file, and `candidate_matches_draft` compares exact text. Unstable
   serialization breaks every review push.

Measured baseline (2026-07-27, before any change): `rcorn kb lint` reports **89
`kb/provenance` failures** across the corpus. Both docs created for *this branch*
— by `rcorn plan create` and `rcorn idea create` — are among them, each missing
`Origin`. The create paths emit docs that violate the project's own lint rule,
and at `warning` severity nothing has ever failed the build over it. That is the
case for landing `kb/frontmatter` at `error`, and for `validate()` being shared
by the create paths rather than living only in CI.

Assumptions:

- `golden-principles.md` gets file-level `type: principle` frontmatter. The
  spec's per-principle `id`/`category` would need one file per principle, which
  its own Non-Goals rule out — deferred.
- Dates stay `datetime.date` objects in the meta dict. `yaml.safe_dump` of the
  *string* `"2026-07-19"` emits it quoted to preserve type; date objects emit
  bare.

## Tasks

### 1. `frontmatter.py`

- [ ] Add `pyyaml>=6.0.3` to `[project].dependencies`.
- [ ] Core: `parse(text) -> (meta, body)`, `dumps(meta, body) -> str`,
      `get(text, key)`, `set_meta(text, updates)` (value `None` removes).
- [ ] Path wrappers: `read(path)`, `write(path, meta, body)`,
      `validate(meta) -> list[str]`.
- [ ] `dumps` passes `sort_keys=False, allow_unicode=True, width=float("inf"),
      default_flow_style=False`. **`width` is load-bearing** — the default of 80
      folds long values, and tech-debt docs carry full-sentence `Remediation:`
      values. `allow_unicode` keeps existing em-dashes from becoming escapes.
- [ ] Key order per spec: `type, title, slug, lifecycle, status, created,
      updated, author, origin, human_validated, <per-type>, tags, related`.
      Unknown keys appended sorted, never dropped, so `validate` can report them.
- [ ] Per-type fields keyed by REGISTRY key: `plan` → `branch` (required,
      unsanitized), `ticket`, `spec`, `retro` · `retro` → `branch` (required),
      `plan` · `idea` → `promoted_to` · `spec`/`prd` → `supersedes`,
      `superseded_by`, `implemented_by` · `debt` → `id`, `category`, `severity`,
      `remediation` · `principle` → none. Gated types also allow `review_pr`,
      `approved_by`, `review_cancelled`.
- [ ] `validate()`: required core present; `type in REGISTRY`; `lifecycle in
      {active, done, dropped}`; `origin in {human, ai-assisted}`;
      `created`/`updated` are dates; `human_validated` bool; `tags`/`related`
      string lists; per-type required present; no unknown keys.
- [ ] Single exclusion list for non-docs (currently duplicated in
      `provenance.sh`): `README.md`, `index.md`, `ATTRIBUTION.md`,
      `quality-scores.md`, `cleanup-queue.md`, `progress.md`, `decisions.md`,
      anything under `_template/`.
- [ ] `tests/test_frontmatter.py` replaces `tests/test_docmeta.py`, including a
      round-trip stability test and the raw-literal pinning `test_docmeta.py`
      does on purpose.

### 2. Migration

- [ ] `rcorn kb migrate-frontmatter [--dry-run]` in
      `commands/kb_migrate.py` — a real command, not a throwaway script, since
      adopting repos have kbs with the same legacy blocks. Register in `cli.py`
      (`kb_sub` parser ~line 151, dispatch table ~line 282).
- [ ] Scan **all** leading `**Key:** value` runs, not just the first —
      `_header_span` stops at a blank line, but tech-debt docs carry a second
      block (`**Severity:** / **Domain:** / **Remediation:**`).
- [ ] Map `Date`/`Created` → `created`, `Domain` → `category`; derive
      `lifecycle` from `status` per the spec's table, keeping `status` verbatim;
      `slug` from filename stem; `title` from H1 (H1 stays in the body).
- [ ] `# Execution Plan: <branch>` → `branch:` field + a generic H1.
- [ ] Unmapped legacy keys are reported and fail the run, never silently dropped.
- [ ] 113 of 134 kb `.md` files carry a legacy block; the rest are excluded
      non-docs plus a few header-less plans that get frontmatter synthesized
      from path + git history.

### 3. Repoint consumers

- [ ] `commands/doc_show.py:86` — `_title_and_status` reads from frontmatter.
- [ ] `commands/status.py` — reads `lifecycle`; `collect_gated_drafts` rows too.
- [ ] `commands/plan.py:180` — drop the `re.sub` on `**Status:**`.
- [ ] `review.py:55,129,245-250` — `candidate_text` / `_finalize_tree` use
      `set_meta` / `get`.
- [ ] `commands/review.py:129` — `_stamp_draft`'s remove/set loop collapses to
      one `set_meta(text, fields)`.
- [ ] `linter/rules/draft_refs.py:54` — `get(text, "status") == "in-review"`.
- [ ] `commands/doc_create.py:27` (`_provenance`), `commands/idea.py`,
      `kb_seed.py` — emit frontmatter; `kb_seed`'s plan template gains `branch:`.
- [ ] Delete `docmeta.py`.

### 4. Lint + CI

- [ ] `linter/rules/frontmatter.py` → `FrontmatterRule`, registered as
      `kb/frontmatter` in `linter/rules/__init__.py`.
- [ ] `linters/.lint-config.json` — replace the `kb/provenance` entry with
      `kb/frontmatter` at `error`; delete `linters/rules/kb/provenance.sh`.
- [ ] `doc-gardening.yml` "Check for orphaned plans" — read `branch:` from each
      active plan, then `git ls-remote --exit-code --heads origin
      "refs/heads/$branch"`; flag only on exact-ref miss. Keep the existing
      env-var passing so branch names reach the report as data, never
      interpolated into shell source.

## Dependencies

`spec/remove-kb-submodule` (kb PR #8, in-review) turns kb into an ordinary clone.
It rewrites the same `submodules: recursive` checkout in `doc-gardening.yml` that
task 4 edits, and collapses task 2's two-PR shape (kb PR + submodule pointer
bump) into one. No logical conflict — task 4 changes *how the branch is read*,
not how kb is checked out — but whichever lands second rebases. **Decide merge
order before task 4 opens.**
