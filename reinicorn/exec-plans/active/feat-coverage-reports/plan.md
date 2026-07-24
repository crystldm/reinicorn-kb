# Execution Plan: feat/coverage-reports

**Ticket:** N/A
**Author:** Michael Biehl
**Created:** 2026-07-24
**Status:** planning

## Goal
Measure test coverage of `src/reinicorn` on every local and CI test run,
publish the per-file breakdown to the GitHub Actions job summary, and fail
CI below an 84% floor. Spec: `specs/test-coverage-reports.md`.

## Acceptance Criteria
- [ ] `uv run pytest` prints a coverage summary with no extra flags
- [ ] Total coverage below 84% fails the test run (locally and in CI)
- [ ] CI job summary shows the per-file coverage table, including on failure
- [ ] CI uploads a browsable `htmlcov/` artifact
- [ ] No third-party coverage service, no secrets, no new runtime dependency
- [ ] Coverage artifacts are gitignored
- [ ] `pytest`, `ruff`, and `pyright` all green

## Approach
- `pytest-cov` in the `dev` group; all config in `pyproject.toml` (no new
  dotfile), sitting next to the existing ruff/pyright/pytest sections.
- Measure by import name (`source = ["reinicorn"]`) rather than by path so
  source-tree and installed-wheel runs report identically.
- Branch coverage on — the CLI is conditional-heavy and statement-only
  coverage would over-report.
- `fail_under = 84` is a ratchet just under today's measured 84.78% (87%
  statement-only, but branch coverage is the honest number): catches a real
  regression, tolerates sub-1% noise. Raise over time, never lower.
- CI reporting stays in-repo (job summary + artifact). Codecov was
  considered and rejected: token, bot comment, third-party data path, and a
  fork-PR failure mode in exchange for a badge.
- Report steps run `if: always()` — the table matters most when the build
  just went red.

## Tasks
- [ ] Add `pytest-cov` to the `dev` dependency group; `uv sync`
- [ ] Add `[tool.coverage.run]` / `[tool.coverage.report]` and pytest
      `addopts` to `pyproject.toml`
- [ ] Gitignore `.coverage`, `coverage.xml`, `htmlcov/`
- [ ] Add xml/html report flags to the CI test step
- [ ] Add job-summary step (`coverage report --format=markdown`)
- [ ] Add `htmlcov/` artifact upload step, SHA-pinned like the other actions
- [ ] Document coverage and the 85% floor in `CONTRIBUTING.md`
- [ ] Verify: full suite green, threshold enforced, xml/html/markdown all
      generated

## Dependencies
None. Touches `pyproject.toml`, `.github/workflows/test.yml`, `.gitignore`,
and `CONTRIBUTING.md` only — no `src/` or `tests/` changes.
