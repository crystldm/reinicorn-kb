# Reinicorn-managed git hooks for golden-principle enforcement

**Date:** 2026-07-22
**Author:** Michael Biehl
**Status:** new

## Description

Golden principles need enforcement. Sometimes that's a lint rule; sometimes
it's a git hook. Today Reinicorn ships only a fixed, hardcoded trio of git
hooks (post-checkout, post-merge, pre-push — all kb-submodule logic) and has no
general capability to manage git hooks on the user's behalf. This idea: give
Reinicorn a first-class **git-hook management** subsystem so a golden principle
can declare an enforcement mechanism (hook or lint) and Reinicorn wires it into
the user's repo.

**Key insight (resolves the "is this the product's job?" tension):** Reinicorn
manages the hook *lifecycle* — install, chain/compose, dedupe, uninstall — while
the enforcement *command* is declared by the principle, not baked into the
product. Reinicorn is not a test runner; it wires `uv run pytest tests/` into
pre-push because a principle asked it to. Hook lifecycle is a legitimate product
concern; the dev command stays out of the product surface.

This generalizes the meta-principle already in `golden-principles.md` (step 4:
"encode it in a linter or structural test") to: **enforcement is a lint rule OR
a managed hook**, and each principle declares which.

## Dependency

Waits for the frontmatter schema
([[unified-kb-doc-frontmatter-schema]], currently in review). Enforcement is
declared in a principle's **frontmatter**, so this should design against a
settled schema, not a moving one. Sketch:

```yaml
type: principle
id: P14
enforcement:
  kind: hook          # or: lint
  event: pre-push     # pre-commit | commit-msg | pre-push | ...
  command: uv run pytest tests/
  scope: this-repo    # vs universal (like kb-submodule push)
```

## First consumer (dogfood)

Golden principle #14 ("Verify against the full gate, not a subset") is the first
thing this feature should enforce: a Reinicorn-managed pre-push hook running the
CI gate on the Reinicorn repo itself. Until this exists, #14 is enforced only by
CI required checks.

## Prior art to study (research task)

Existing git-hook managers — capture how each handles install, config/declaration
format, multiple hooks per event, composition with pre-existing hooks,
cross-platform, monorepo/worktree, and escape hatches:

- **Husky** (Node ecosystem)
- **Lefthook** (Go, language-agnostic)
- **pre-commit** (Python, the framework)
- **simple-git-hooks**
- **Overcommit** (Ruby)
- **Git native** `core.hooksPath`

## Open design questions (for the eventual spec)

- Declaration format and where it lives (principle frontmatter vs a central
  config).
- Hook composition/ordering: how declared hooks compose with Reinicorn's
  built-in kb hooks via the existing chain-below-MARKER mechanism.
- Scope: per-repo-scope (kb is already repo-scoped) vs universal enforcement.
- Escape hatches: `--no-verify`, opt-out, CI-only mode.
- Uninstall / drift detection when a principle's enforcement changes.
- Cross-platform + git worktree behavior (ties into recent worktree-aware work).

## Notes

Prior-art research report to be attached below once complete.
