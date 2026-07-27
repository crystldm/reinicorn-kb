---
type: spec
title: 'Markdown linting: a docs/markdownlint rule and a config baseline'
slug: markdown-linting-a-docs-markdownlint-rule-and-a-config-basel
lifecycle: active
status: in-review
created: 2026-07-26
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/6
---

# Markdown linting: a docs/markdownlint rule and a config baseline

Closes #26.

## Problem

Markdown is this project's primary artifact — the kb, the skills, every doc
surface — and nothing lints it. There is no markdownlint config, no lint rule,
and the tool is not installed. The only thing reviewing markdown today is
CodeRabbit: external, advisory, and PR-only.

The gap surfaced on `reinicorn-kb#5`, where CodeRabbit flagged a fenced block
with no language (`MD040`) that no local check would have caught.

`linters/rules/scripts/shellcheck.sh` already establishes the pattern for an
external linter — a `.sh` rule taking `$PROJECT_ROOT`, emitting
`path:line — message`, and skipping with exit 0 when the tool is absent.
Markdown has no equivalent.

### Measurements

`markdownlint-cli2` v0.23.1 over 230 markdown files (repo + kb, excluding
`.venv/`, `htmlcov/`, `node_modules/`, `.claude/worktrees/`):

| Config | Violations |
| --- | --- |
| Default | 4,698 |
| Tuned | 1,384 |
| — auto-fixable via `--fix` | 1,213 |
| — needing human judgment | 171 (168 `MD040`, 3 `MD001`) |

Top offenders under the default config:

```text
2491  MD013/line-length
 712  MD032/blanks-around-lists
 503  MD060/table-column-style
 255  MD031/blanks-around-fences
 248  MD036/no-emphasis-as-heading
 185  MD022/blanks-around-headings
 168  MD040/fenced-code-language
  53  MD033/no-inline-html
```

### The house style is illegal under the default config

The two largest rules fire on deliberate conventions, not defects. `MD036`
(`no-emphasis-as-heading`, 248) hits the `**Bold lead-in.**` paragraph opener
used throughout specs and skills. `MD013` (`line-length` at 80, 2,491) is
unmeetable for tables, code blocks, and long URLs, and at that volume is pure
noise.

Config is therefore a prerequisite, not polish. A rule shipped without one would
report 4,698 violations on a clean checkout and be ignored within a day.

### Reinicorn's own generated frontmatter fails the lint

`rcorn review start` stamps a bare URL into every gated doc:

```text
**Review-PR:** https://github.com/crystldm/reinicorn-kb/pull/5
```

That is an `MD034/no-bare-urls` violation, and it **cannot be fixed in the
document** — the next `review push` rewrites the line. Confirmed live on
`reinicorn-kb#5`, where it is the sole remaining violation in an otherwise clean
spec. Every gated doc the tool touches fails a lint the tool itself would impose.

## Design Goals

- Markdown structure is checked locally, by the same `rcorn kb lint` surface that
  already runs the other rules — not only by an external reviewer on a PR.
- The shipped config produces **zero** violations from the project's own
  documented conventions. A linter that flags house style trains people to ignore
  it.
- The rule is absent-tool-safe: no hard dependency on node or npm to run
  `rcorn kb lint`.
- Reinicorn stops emitting markdown that Reinicorn would reject.
- Landing the rule does not turn `main` red, and does not require a 1,213-file
  diff to be reviewed alongside it.
- Downstream projects that adopt Reinicorn inherit a working baseline, not 4,698
  violations on day one.

## Design

### 1. Config baseline

Add `.markdownlint-cli2.jsonc` at the repo root, disabling the rules that
conflict with house style or are pure volume:

| Rule | Why disabled |
| --- | --- |
| `MD013` line-length | Unmeetable for tables/code/URLs; 2,491 hits |
| `MD036` no-emphasis-as-heading | `**Bold lead-in.**` is house style |
| `MD060` table-column-style | Stylistic; 503 hits, no correctness value |
| `MD033` no-inline-html | Inline HTML is used deliberately in docs |
| `MD024` no-duplicate-heading | Repeated section names across doc parts are fine |
| `MD029` ol-prefix | Both numbering styles are in use and both are readable |
| `MD041` first-line-heading | Not all fragments begin with a heading |

Ship it as a Reinicorn asset (`assets.py` / `_data/`) alongside
`linters/.lint-config.json`, so `rcorn init` seeds it into downstream projects.

### 2. The lint rule

Add `linters/rules/docs/markdownlint.sh`, modelled directly on
`scripts/shellcheck.sh` — same shape, same contract, same output format:

1. Take `$PROJECT_ROOT` as `$1`, defaulting to the script's grandparent as
   shellcheck does.
2. Resolve the tool: prefer a `markdownlint-cli2` on `PATH`, fall back to
   `npx --yes markdownlint-cli2`. If neither is available, print an install hint
   and **exit 0** — matching shellcheck's absent-tool behavior exactly, so
   `rcorn kb lint` never hard-fails on a machine without node.
3. Scan `**/*.md`, excluding `.venv/`, `htmlcov/`, `node_modules/`,
   `.claude/worktrees/`, and `.git/`. Unlike `shellcheck.sh`, `kb/` is **not**
   excluded — the kb is the main thing worth linting here.
4. Reformat `path:line:col error MDxxx/name message` into the framework's
   `path:line — [MDxxx] message`.
5. Exit 1 if any violation, 0 otherwise.

The runner already discovers `linters/rules/**/*.sh` and derives the rule name
from the relative path, so `docs/markdownlint.sh` becomes `docs/markdownlint`
with no runner change.

### 3. Registration at warning severity

`linters/.lint-config.json` gains:

```json
"docs/markdownlint": { "enabled": true, "severity": "warning" }
```

Warning, not error, so the rule lands green against the existing 1,384-violation
backlog. Promotion to `error` happens after the cleanup, as a separate change —
the same warning-then-error sequencing `kb/draft-refs` is going through in the
review-lane spec.

### 4. Fix the MD034 generator defect

Change the `Review-PR` stamp from a bare URL to an angle-bracket autolink:

```text
**Review-PR:** <https://github.com/crystldm/reinicorn-kb/pull/5>
```

This is valid CommonMark, renders identically on GitHub, and satisfies `MD034`.
Fixing the generator is better than exempting `MD034` permanently — it fixes the
defect rather than lowering the bar.

The cost is that the stamp is round-tripped: `review start` writes it,
`review link` rewrites it, `review status` reads it, and `docmeta.get_field`
parses it. The field parser must tolerate **both** forms, because docs already in
flight carry the bare form and must keep resolving. So: write the bracketed form,
strip surrounding `<>` on read, and cover both shapes in `docmeta` tests.

## Non-Goals

- The `--fix` cleanup sweep over the ~1,213 mechanical violations. It touches
  nearly every markdown file in the repo and kb, and that diff must not be
  reviewed alongside the rule that motivates it. Separate PR, after this one.
- Promotion to `severity: error`. Follows the cleanup, not this change.
- The 168 `MD040` and 3 `MD001` violations needing human judgment — part of the
  cleanup, not of landing the rule.
- Prose and grammar linting (Vale, LanguageTool). Structure only.
- Linting markdown in `.venv/`, `htmlcov/`, `node_modules/`, or
  `.claude/worktrees/`.
- Making node a hard dependency of Reinicorn. The rule degrades to a skip.

## Acceptance Criteria

- `rcorn kb lint` runs `docs/markdownlint` and reports violations in the
  framework's `path:line — message` format.
- With `markdownlint-cli2` unavailable and no `npx`, the rule prints an install
  hint and exits 0; `rcorn kb lint` still succeeds.
- With the tool available, a file containing a fence with no language is
  reported; a clean file is not.
- The shipped config yields zero violations from `MD036` and `MD013` across the
  existing repo and kb.
- A doc stamped by `rcorn review start` produces no `MD034` violation.
- `docmeta.get_field` returns the same URL for `**Review-PR:** <url>` and
  `**Review-PR:** url`, so docs stamped before this change still resolve.
- `review status` and `review merge` work unchanged against a doc carrying either
  stamp form.
- The rule is registered at `warning` and `rcorn kb lint` exits 0 against the
  current backlog.
- `rcorn init` seeds `.markdownlint-cli2.jsonc` into a fresh project.
- Tests cover: tool-present, tool-absent, violations-found, clean, the exclusion
  set, both stamp forms in `docmeta`, and the `review start` output shape.
