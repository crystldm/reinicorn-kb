# Tech Debt: Code Quality

**Category:** Patterns, consistency, correctness, maintainability
**Date catalogued:** 2026-02-17

---

## Medium

### QUAL-04. `docs_freshness` uses filesystem mtime, not git commit time

**File:** `linter/rules/docs_freshness.py:60`
**Severity:** medium
**Impact:** `st_mtime` resets on clone/checkout. Freshly cloned repo shows all docs as fresh. Makes the rule useless in CI.
**Remediation:** Use `run_git("log", "-1", "--format=%at", "--", str(full_path))`, consistent with `status.py`.

### QUAL-07. `reinicorn_root()` assumes fixed directory structure

**File:** `git.py:131-133`
**Severity:** medium
**Impact:** `Path(__file__).resolve().parent.parent.parent` fails if installed as a wheel in site-packages.
**Remediation:** Add sanity check (verify `AGENTS.md` or `src/` exists), fall back or raise clear error.

---

## Low

### QUAL-09. Idea collision handling only appends `-2`

**File:** `commands/idea.py:42`
**Severity:** low
**Impact:** Third idea with same slug on same day silently overwrites the second.
**Remediation:** Loop through `range(2, 100)` until unused filename found.

### QUAL-10. Magic number `86400` repeated

**Files:** `commands/status.py:109`, `linter/rules/docs_freshness.py:53,62`
**Severity:** low
**Impact:** `86400` (seconds per day) appears as an unextracted constant in multiple files.
**Remediation:** Define `SECONDS_PER_DAY = 86400` in a shared location.

### QUAL-13. Missing docstrings on public command functions

**Files:** Several `cmd_*` functions across `commands/*.py` (e.g. `status.py`, `publish.py`, `idea.py`)
**Severity:** low
**Impact:** Per PEP 257, all public functions should have docstrings. (Partial: some `cmd_*` functions now have docstrings; several still lack them.)
**Remediation:** Add one-line docstrings to each.

### QUAL-14. `from __future__ import annotations` missing in `__init__.py`, `__main__.py`

**Severity:** low
**Impact:** Harmless (no annotations used), but breaks the "every file" convention.
**Remediation:** Add to both files.
