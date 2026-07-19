# Tech Debt: Missing Tests

**Category:** Test coverage gaps
**Date catalogued:** 2026-02-17
**Reference:** Golden Principle 7 — "Tests for every new behavior"

---

## Medium — Thin or missing coverage

### TEST-10. `KeyboardInterrupt` and `SystemExit` handling in `main()` untested

**File:** `cli.py:381-386` (`main()` exception handling)
**Severity:** medium
**Impact:** The exit code mapping for `KeyboardInterrupt` (130) and `SystemExit` passthrough are untested.
**Remediation:** Mock a dispatched command raising `KeyboardInterrupt`/`SystemExit(42)`, assert correct return codes.

### TEST-12. `linter/runner.py` — external shell rules, severity levels untested

**File:** `linter/runner.py:73-117`
**Severity:** medium
**Impact:** `tests/linter/test_runner.py` covers no-config, invalid-JSON, and all-pass only. The external `.sh` rule execution block, error-vs-warning severity behavior, and disabled-rule skipping remain untested.
**Remediation:** Test each remaining runner code path with configured rules.

### TEST-13. `docs_freshness` stale detection never exercised

**File:** `linter/rules/docs_freshness.py`
**Severity:** medium
**Impact:** Tests cover the fresh-docs pass path only; no test creates a file old enough to trigger staleness, so the positive detection path is never exercised.
**Remediation:** Use `os.utime` to set mtime to 60 days ago, assert diagnostic returned.

### TEST-14. `__main__.py` untested

**File:** `src/reinicorn/__main__.py`
**Severity:** low
**Impact:** `python -m reinicorn` support not verified in tests.
**Remediation:** Import with mocked `main()`, assert `SystemExit(0)`.
