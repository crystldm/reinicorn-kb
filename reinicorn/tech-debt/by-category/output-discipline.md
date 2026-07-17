# Tech Debt: Output Discipline

**Category:** stdout/stderr routing, console module consistency, formatting
**Date catalogued:** 2026-02-17

---

## High

### OUT-01. Many raw `print()` calls bypass console module

**Files (worst offenders — current counts):**
- `commands/init.py` — ~66 raw prints for interactive prompts and summaries
- `linter/runner.py` — ~29 raw prints, including FATAL errors to stdout
- `commands/update.py` — ~16 raw prints
- `commands/status.py`, `commands/home.py` — ~10 raw prints each
- `commands/review.py`, `commands/plan.py`, `commands/internal/post_checkout.py`, `commands/doc_show.py` — blank-line and info `print()` calls

**Severity:** high
**Impact:** Bypasses color support, `NO_COLOR`/`FORCE_COLOR` handling, and stdout/stderr routing. Lint output does not visually match other commands. (Note: per the axi channel model, `console.*` already routes intentionally — the debt is the *bypass*, not the stdout routing itself.)
**Remediation:** Systematically replace incidental `print()` calls with appropriate `console.*` functions. Consider adding `console.blank()` for spacing. For interactive commands (`init.py`), consider a `console.prompt()` wrapper. Some content-first prints (e.g. `home.py`) may be intentional — triage before replacing.

---

## Medium

### OUT-02. Emoji in `pre_push.py` — only file with emoji

**File:** `commands/internal/pre_push.py:70,81`
**Severity:** medium
**Impact:** Uses a unicorn emoji (`\U0001f984`). No other file uses emoji. Not all terminals render emoji. Not gated by `NO_COLOR` or terminal capability checks. (Given Reinicorn's unicorn branding this may now be intentional — re-verify whether it should be kept as debt.)
**Remediation:** Remove emoji or route through console module with terminal capability check.

---

## Low

### OUT-03. No `--verbose`/`--quiet` flags

**File:** `cli.py`
**Severity:** low
**Impact:** No way to control output verbosity. clig.dev recommends `-v`/`--verbose` and `-q`/`--quiet`.
**Remediation:** Add to root parser, thread through to console module.
