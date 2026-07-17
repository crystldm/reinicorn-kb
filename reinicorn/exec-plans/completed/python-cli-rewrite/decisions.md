# Decisions: python-cli-rewrite

Record of architectural and implementation decisions made during the rewrite.

## 2026-02-15 — Use `src/` layout with pyproject.toml and uv
**Context:** Plan originally specified `lib/reins/` with sys.path insert in entrypoint.
**Decision:** Use standard `src/` layout with `pyproject.toml`, `hatchling` build backend, and `uv` for dependency management.
**Alternatives considered:** `lib/` with sys.path hack (original plan), flat layout without src/
**Rationale:** Proper Python packaging; pyproject.toml + uv are the project standard. Eliminates the need for a `bin/reins` shim script.

## 2026-02-15 — console_scripts only, no bin/ shim
**Context:** Original plan had a `bin/reins` Python shim with sys.path insert.
**Decision:** Use `console_scripts` entry point only (`reins = "reins.cli:main"`). Access via `uv run reins`.
**Alternatives considered:** Keep bin/ shim alongside console_scripts for backward compat
**Rationale:** This is all greenfield — nothing needs to be preserved. Clean packaging wins.

## 2026-02-15 — dependency-groups for dev deps
**Context:** Need pytest as dev-only dependency.
**Decision:** Use `[dependency-groups]` (PEP 735) with `dev = ["pytest>=8.0"]`.
**Alternatives considered:** `[project.optional-dependencies]` extras
**Rationale:** PEP 735 is the modern approach, supported by uv. Cleaner than extras for dev tooling.

## 2026-02-15 — Centralized git.py as single mock point
**Context:** Many modules need to call git commands.
**Decision:** All git subprocess calls go through `git.run_git()`. Other modules import from `reins.git`.
**Alternatives considered:** Each module calling subprocess directly
**Rationale:** Single mock point in tests. Consistent error handling. Easy to add logging/debugging later.

## 2026-02-15 — Wrap parse_args in try/except SystemExit
**Context:** argparse calls `sys.exit(2)` on unrecognized commands, bypassing our dispatch logic.
**Decision:** Wrap `parser.parse_args(argv)` in try/except SystemExit to catch argparse errors gracefully.
**Alternatives considered:** Using `parse_known_args`, custom error handler
**Rationale:** Simplest fix. Preserves argparse's error messages while letting main() return the exit code instead of raising.
