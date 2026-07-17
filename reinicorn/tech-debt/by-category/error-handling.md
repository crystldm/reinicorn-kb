# Tech Debt: Error Handling

**Category:** Exception handling, exit codes, error reporting
**Date catalogued:** 2026-02-17

---

## High

### ERR-01. `SystemExit` raised deep in stack

**Files:** `mode.py:35` (`set_mode`), `config.py:58` (`kb_scope`), `kb.py:78` (`require_kb_dir`)
**Severity:** high
**Impact:** Violates best practice "Return int exit codes from `main()`, not `sys.exit()` deep in stack." Prevents callers from handling errors gracefully. Makes testing harder.
**Remediation:** Define domain exceptions (`NotInRepoError`, `KbNotFoundError`) in a `reinicorn.exceptions` module. Raise those instead. Catch in `cli.py:main()`.

---

## Medium

### ERR-04. Broad `except Exception` suppression silences errors

**Files:**
- `commands/internal/post_checkout.py:32` — `contextlib.suppress(Exception)` around submodule update
- `commands/internal/post_merge.py:57` — `contextlib.suppress(Exception)` around plan archival

**Severity:** medium
**Impact:** Programming bugs, permission errors, and meaningful failures are invisible. Violates "Never silence broadly — at minimum log a warning."
**Remediation:** Suppress specific exceptions (`subprocess.CalledProcessError`, `OSError`). At minimum add `console.warn()` before continuing.
**Note (re-verify):** Both sites are fail-safe git hooks where a failure must not block the checkout/merge — the broad suppression may be intentional. Confirm intent before narrowing.

### ERR-05. Bare `except Exception` catches too broadly

**Files:**
- `commands/idea.py:27` — git config lookup, should catch `CalledProcessError`
- `commands/plan.py:63` — git config lookup, same
- `commands/hooks_install.py:45` — should catch `(CalledProcessError, FileNotFoundError)`

**Severity:** medium
**Impact:** Catches programming bugs alongside expected failures.
**Remediation:** Replace with specific exception types.

### ERR-06. `status.py` redundant `except (ValueError, Exception)`

**File:** `commands/status.py:107`
**Severity:** medium
**Impact:** `ValueError` is a subclass of `Exception`, so the tuple is redundant. Obscures intent.
**Remediation:** Use `except (ValueError, OSError):` or `except Exception:` depending on intent.

### ERR-07. No SIGPIPE handler

**File:** `cli.py`
**Severity:** medium
**Impact:** Piping output to `head` etc. raises `BrokenPipeError`. Should exit 141.
**Remediation:** Add `signal.signal(signal.SIGPIPE, signal.SIG_DFL)` in `main()` on Unix.

---

## Low

### ERR-10. `_use_color()` called on every output — no caching

**File:** `console.py:15-24`
**Severity:** low
**Impact:** Redundant env var reads and `isatty()` checks on every console call.
**Remediation:** Cache with `functools.lru_cache` or compute once at module load.

### ERR-11. No `TERM=dumb` check in console

**File:** `console.py:15-24`
**Severity:** low
**Impact:** Terminals reporting `TERM=dumb` still get color escape codes.
**Remediation:** Add `if os.environ.get("TERM") == "dumb": return False` before isatty check.
