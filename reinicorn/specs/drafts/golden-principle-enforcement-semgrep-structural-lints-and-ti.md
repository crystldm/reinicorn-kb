---
type: spec
title: 'Golden principle enforcement: semgrep structural lints and TID251 seams'
slug: golden-principle-enforcement-semgrep-structural-lints-and-ti
lifecycle: active
status: in-review
created: 2026-08-08
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/13
---

# Golden principle enforcement: semgrep structural lints and TID251 seams

## Problem

A full mining pass over both repos' PR history (reinicorn #3–#42, reinicorn-kb #1–#12, all review threads) found five recurring bug classes that each drew the same reviewer correction on three or more independent PRs, plus one security-grade single occurrence:

1. **Under-strict matching in safety checks** — 5 instances: the N/A exemption regex matched `n/a/specs/x.md` (#30), the unconditional-exit detector missed `cmd; exit $?` (#37), the stderr-allowlist matched by basename (#29), `is_doc` matched excluded dirs anywhere in the absolute path (#30), bypass-actor reconciliation diffed by full value instead of identity keys (#18).
2. **"Cannot verify" collapsing into a conclusion before destructive actions** — `ls-remote` failure read as "branch gone" would have mass-archived plans (#30); a catch-all mapped every git failure to "not a git repo" (#29); kb#8's migration ran `submodule deinit -f` before its own safety check.
3. **One bad item aborting a whole batch** — an unguarded `plan.md` read aborting archival for every repo (#30); one non-UTF-8 hook aborting install for all hooks (#37); one malformed API entry crashing the command (#18).
4. **Guessed external behavior / naive parsing of structured formats** — invented GitHub role-id ladder (#12), assumed `gh api --paginate` output shape (#19), substring-matched `.gitmodules` (#13).
5. **"All paths do X" claims that held on one path** — the born-valid invariant held for 1 of 3 create paths until `frontmatter.render()` made the seam enforce it (#30); same shape in #13 and #42.
6. **Credential-bearing URLs printed raw** (#29, caught by CodeRabbit and CodeQL).

Per this repo's promotion rule these become golden principles — but a principle without a lint is a convention, and `tests/test_git_error_surface.py` already declares the hand-rolled AST-walker suite at its limit: three walkers exist and the stated threshold is "do not add a fourth by hand — port all three to semgrep instead."

## Design Goals

- Each new principle has enforcement that fails CI, not a convention that has to be remembered.
- Ruff-native (TID251) first; semgrep for what ruff cannot express; runtime pytest checks only where the rule needs imported objects (compiled regexes).
- The three existing AST walkers are ported to semgrep, honoring the threshold note; no fourth hand-rolled walker is ever added.
- Every rule's failure message cites the principle number and a fix (golden principle 4).
- Escape hatches are explicit, tagged, and greppable — never silent.

## Design

### New golden principles (added via `rcorn principle add` in phase 1)

- **P15 — Anchored, identity-precise matching in safety checks.** Allowlists, exemptions, and staleness checks match on anchored patterns and root-relative POSIX paths, keyed by identity — never substring, basename, loose word boundary, or full-value equality.
- **P16 — Cannot-verify never collapses into a conclusion.** Any check gating a destructive or state-changing action distinguishes confirmed-yes / confirmed-no / cannot-verify; cannot-verify fails safe with a warning, and safety checks execute before the first destructive command.
- **P17 — Per-item fault isolation in batch loops.** Loops over independent items (repos, files, hooks, plans) wrap per-item I/O; one bad item is one skipped item plus a warning, never an aborted batch.
- **P18 — External surfaces get one wrapper and a real parser.** Structured formats are parsed with real parsers; each external CLI/API surface has a single owning module; external IDs, enums, and output formats are verified against live behavior or docs, cited in the PR.
- **P19 — Invariants are seams, not conventions.** An "always/never/all paths" claim routes through a single enforcing seam (with the bypass TID251-banned) or enumerates and tests every path.
- **P20 — Credentials never reach output.** URLs are printed only through a redacting display helper; userinfo is stripped before any console/log write.

### Phase 1 — no new tooling (TID251 + trivial grep tests)

- TID251 bans (pyproject.toml), each with a principle-citing msg:
  - `reinicorn.frontmatter.dumps` outside `frontmatter.py` — creation goes through the validating `render()` seam (P19).
  - `shutil.rmtree`, `shutil.copy2`, destructive `Path.unlink`-wrapper equivalents outside the owning destructive seam module (P16). Exact ban list fixed during implementation against real call sites.
  - The raw remote-URL getter outside `git.py`; `git.py` gains `display_url()` that strips userinfo, with a `user:token@host` fixture test (P20).
- Literal-confinement checks in the P2 grep idiom (a small pytest, not an AST walker): `.gitmodules` may only appear in `git.py`; `--paginate` only in `github.py`, whose `gh_api_paginate()` owns page flattening (P18).
- Runtime anchoring test: every module-level `re.Pattern` in gate modules (`linter/`, `hooks_health.py`, `commands/internal/spec_gate.py`, `commands/internal/post_merge.py`) must start `^`/`\A` and end `$`/`\Z`; escape hatch is a module-level `UNANCHORED_OK: dict[str, str]` mapping pattern name → reason (P15). This imports compiled objects, so pytest is the right layer, not semgrep.

### Phase 2 — semgrep adoption and walker port

- Add semgrep as a dev dependency; rules live in `linters/semgrep/`; runs in `tests/run-all.sh` and the CI lint job beside ruff.
- Port the three existing walkers 1:1 (doc-type key comparisons, branch→path ownership, git/gh stderr seam), then delete the hand-rolled walkers. Port is behavior-preserving: each rule must flag the same seeded violation the old test caught (verified by fixture files under `linters/semgrep/tests/`).

### Phase 3 — new semgrep rules

- **P15:** no `.name` equality/membership comparisons in enforcement or allowlist code paths.
- **P16:** destructive filesystem calls confined to the seam module (widens the phase-1 TID251 slice to call shapes ruff cannot express).
- **P17:** known per-item I/O calls (`read_text`, `frontmatter.read`, `run_git`, …) inside a `for` body must be wrapped by a `try` inside that loop; probe-style loops opt out with a tagged `# per-item-ok:` comment.
- Rules adopt with a baseline allowlist of current violations, then ratchet: the allowlist may only shrink (same policy as the coverage floor).

### Phase 4 — tri-state verdicts on destructive paths

- `reinicorn/verify.py` gains `Verdict` (`CONFIRMED` / `ABSENT` / `UNKNOWN`). Destructive seams take `Verdict`, not `bool`, and raise on `UNKNOWN`; pyright then enforces production of a verdict at every caller. Boolean verification helpers on destructive paths are converted, then TID251-banned by name.
- Existing tri-state behavior (#30's "a plan with no `branch:` is never archived") is re-expressed through `Verdict` so the pattern has one home.

## Non-Goals

- The process half of the findings (PR-body conventions, empirical-verification citations, review-response style) — that is AGENTS.md / review-checklist material, handled separately from this spec.
- CodeRabbit configuration or learnings — per #40, shared guidelines are fixed at the source doc, not per-reviewer.
- The managed pre-push principle-enforcement gate (existing idea: reinicorn-managed-git-hooks-for-golden-principle-enforcement) — these rules run in CI and the test suite; hook delivery is that idea's scope.
- Retrofitting every existing violation immediately — phase 3 rules land with a ratcheting baseline, not a big-bang cleanup.
