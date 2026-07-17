# Tech Debt: Security

**Category:** Security vulnerabilities and input validation gaps
**Date catalogued:** 2026-02-17

---

## High

### SEC-06. Unvalidated kb URL passed to `git submodule add`

**File:** `submodule.py:83` (`setup_submodule` calls `validate_git_url`); `kb.py:37-67` (`get_kb_dir` path check)
**Severity:** high
**Impact:** User-supplied kb URL passed to `git submodule add`. An unvalidated URL can use git's `ext::` transport helper for command execution or `file://`/local paths for unintended access.
**Remediation:** Validate URL matches `https://`, `ssh://`, or `git@host:path`. Reject `ext::`, `fd::`, `git://`, and option-like values.
**Note (re-verify):** `submodule.py:83` now calls `validate_git_url()` before `git submodule add`, and `kb.py:get_kb_dir` rejects `.gitmodules` paths that escape the repo root — this concern appears largely closed. Kept per the release-audit anchor list; confirm no other unvalidated `submodule add` path exists before removing.

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

### SEC-11. `branch` name in `plan complete` flows into paths

**File:** `kb.py:191` (`plan_dir`), `commands/plan.py:147-190` (`cmd_plan_complete` → `branch_doc_path` → `shutil.move`)
**Severity:** medium
**Impact:** The `plan complete` CLI argument flows through `branch_doc_path()` into `shutil.move()`. A crafted branch value could operate on unexpected paths.
**Remediation:** Validate with `Path.resolve().is_relative_to()` or a regex allowlist.
**Note (re-verify):** `sanitize_branch()` now replaces `/` with `-`, neutralising slash-based `../` traversal, so exploitability is much reduced. Kept because a literal `..`-style CLI argument is not explicitly rejected — confirm whether any residual path escape remains.

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
