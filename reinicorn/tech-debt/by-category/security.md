# Tech Debt: Security

**Category:** Security vulnerabilities and input validation gaps
**Date catalogued:** 2026-02-17

---

## High

### SEC-08. External lint scripts executed without validation

**File:** `linter/runner.py:76,91-95`
**Severity:** high
**Impact:** Every `*.sh` file under `linters/rules/` is executed. A PR adding a malicious script gets executed on `rcorn kb lint`.
**Remediation:** Warn before executing external scripts. Better: require a signed manifest. Best: provide a Python plugin API.

---

## Medium

### SEC-10. Regex from config used without validation

**File:** `commands/plan.py:67-68`, `commands/internal/post_checkout.py:53-56`
**Severity:** medium
**Impact:** `REINICORN_TICKET_PATTERN` is used as a regex without validation. Malformed patterns crash the CLI; crafted patterns cause ReDoS.
**Remediation:** Wrap `re.search()` in `try/except re.error`. Consider pattern length/complexity limits.

### SEC-13. No validation of mode values in `set_mode`

**File:** `mode.py:30-38`
**Severity:** medium
**Impact:** `set_mode` writes any string to `.reinicorn-mode` without validating it's a known value. `get_mode` returns raw file content.
**Remediation:** Validate against allowlist `("enabled", "disabled", "incognito")`.

### SEC-14. `config_get` quote-stripping uses `str.strip()` not pair-matching

**File:** `config.py:36`
**Severity:** medium
**Impact:** `strip("\"'")` removes all quote chars from both ends, not matching pairs. `"'value'"` becomes `value`.
**Remediation:** Match outer quote pairs explicitly: if starts and ends with same quote, strip that one pair.

---

## Low

### SEC-15. Information disclosure via absolute paths in messages

**File:** `commands/init.py:134`, plus assorted user-facing messages that print absolute paths (`status.py`, `submodule.py`, `kb.py` errors)
**Severity:** low
**Impact:** Messages include absolute filesystem paths, revealing directory structure in CI logs or screenshots.
**Remediation:** Use paths relative to repo root in user-facing messages. (path needs re-verify — original agent.py/extensions.py sites are gone; concern persists in surviving modules.)

### SEC-16. Markdown injection via git username in idea files

**File:** `commands/idea.py:26-29,44-48`
**Severity:** low
**Impact:** Git `user.name` (the `author` display value) is written into markdown without escaping. Names containing `](url)` inject clickable links.
**Remediation:** Escape markdown-significant characters in author value.
