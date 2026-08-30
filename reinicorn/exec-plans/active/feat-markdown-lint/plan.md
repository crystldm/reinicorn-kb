---
type: plan
title: 'Execution Plan: feat-markdown-lint'
slug: feat-markdown-lint
lifecycle: active
status: planning
created: 2026-08-30
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-markdown-lint
ticket: 'https://github.com/crystldm/reinicorn/issues/26'
spec: specs/markdown-linting-a-docs-markdownlint-rule-and-a-config-basel.md
---

# Execution Plan: feat-markdown-lint

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Local markdown linting: a `docs/markdown` lint rule backed by rumdl, with a tuned config baseline that produces zero violations from house style, registered at `warning` severity so it lands green.

**Architecture:** A new external `.sh` rule at `linters/rules/docs/markdown.sh` (auto-discovered by the existing runner — no runner change), a `.rumdl.toml` config at the repo root shipped as a Reinicorn asset and seeded by `rcorn init`, and `rumdl` as a dev dependency. The rule degrades to a skip (exit 0) when the tool is absent.

**Tech Stack:** bash (rule script), rumdl >= 0.2.44 (pip wheel, no node), pytest (tests via subprocess against the real script).

## Global Constraints

- The rule is named `docs/markdown` — the thing checked, never the vendor name.
- Registered at `"severity": "warning"`; `rcorn kb lint` must exit 0 against the current violation backlog.
- Absent-tool behavior prints exactly `rumdl not found — skipping. Install with: pip install rumdl` and exits 0.
- Violation output format is exactly `path:line — [MDxxx] message` (em dash, rule ID in brackets), matching the framework convention set by `scripts/shellcheck.sh`.
- `.rumdl.toml` disables exactly `["MD013", "MD025", "MD033", "MD036", "MD041"]` and excludes exactly `["node_modules", ".claude/worktrees", "presentation"]` — no more, no less (spec §1).
- rumdl goes in `[dependency-groups] dev` (next to ruff), version floor `>=0.2.44`, pinned via `uv.lock`. It is NOT a runtime dependency.
- The kb (`kb/`) IS linted. It is gitignored, so it is invisible to a gitignore-respecting pass over the repo root; the rule runs a second pass inside `kb/` when one exists (see Approach).
- File selection and exclusion come from `.rumdl.toml`, not from `find` — the rule builds no file list of its own.
- Do NOT touch the root `README.md` on this branch (concurrent hand-edit in flight elsewhere).
- Gate for every task: `uv run pytest tests/ -q` (coverage floor 87%), `uv run ruff check src/reinicorn tests`, `uv run pyright src/reinicorn`.
- Tests assert positive behavior only — no change-detector tests (golden principles).

## Approach

The spec (`specs/markdown-linting-a-docs-markdownlint-rule-and-a-config-basel.md`, approved) fixes tool, config, rule name, severity, and absent-tool contract. Two refinements made here, both forced by the same fact — `kb/` is gitignored and rumdl honours `.gitignore` (`respect-gitignore` defaults to true), so a single root pass silently skips the kb the spec most wants linted:

1. **Two passes.** The rule lints `$PROJECT_ROOT`, then `$PROJECT_ROOT/kb` when `kb/.git` exists, prefixing the second pass's paths with `kb/`. Inside `kb/` (its own git repo) rumdl respects the kb's own `.gitignore`, and config discovery walks up to the root `.rumdl.toml`.
2. **`uv run --no-sync --project "$PROJECT_ROOT" rumdl`** as the fallback runner, not the spec's bare `uv run --no-sync rumdl` — the kb pass runs with a different working directory, and `--project` keeps uv resolving the project env instead of hunting from cwd.

The behavioral contract (kb violations reported with `kb/` prefix, gitignored/excluded paths silent, exact skip message) is pinned by tests; if rumdl's actual CLI behavior differs from what a plan step assumes (config discovery, concise-output path shapes), adjust the script to satisfy the tests, and record the deviation in the task report.

## Acceptance Criteria

- [x] `rcorn kb lint` runs `docs/markdown` and reports violations as `path:line — [MDxxx] message`.
- [x] With rumdl unavailable and no usable uv, the rule prints the install hint and exits 0.
- [x] A fence with no language is reported; a clean file is not.
- [x] Shipped config yields zero `MD036`/`MD013` violations across repo and kb.
- [x] `rumdl` present in `[dependency-groups] dev`, pinned in `uv.lock`.
- [x] No findings from `node_modules/`, `.claude/worktrees/`, `presentation/`, or gitignored paths.
- [x] Review-stamped docs produce no `MD034`; docs with `title:` + body H1 produce no `MD025` (verify only — the frontmatter migration already landed).
- [x] Rule registered at `warning`; `rcorn kb lint` exits 0 against the backlog.
- [x] `rcorn init` seeds `.rumdl.toml` into a fresh project.
- [x] Tests cover: tool-present, tool-absent, violations-found, clean, exclusions.

## Tasks

### Task 1: rumdl dependency + `.rumdl.toml` baseline

**Files:**
- Create: `.rumdl.toml`
- Modify: `pyproject.toml` (dependency-groups only)
- Modify: `uv.lock` (via `uv sync` — never by hand)

**Interfaces:**
- Produces: `.rumdl.toml` at the repo root (Task 2's tests copy it into fixtures; Task 3 ships it as an asset), and `rumdl` invocable via `uv run --no-sync rumdl`.

- [x] **Step 1: Create `.rumdl.toml`** at the repo root, exactly:

```toml
[global]
disable = ["MD013", "MD025", "MD033", "MD036", "MD041"]
exclude = ["node_modules", ".claude/worktrees", "presentation"]
```

- [x] **Step 2: Add the dev dependency.** In `pyproject.toml`, find `[dependency-groups]` and add `"rumdl>=0.2.44",` to the `dev` list, next to `ruff`. Then run:

```bash
uv sync
```

Expected: `uv.lock` gains a pinned `rumdl` entry; `uv run --no-sync rumdl --version` prints a version >= 0.2.44.

- [x] **Step 3: Smoke the config against the repo.** Run:

```bash
uv run --no-sync rumdl check --output-format concise . 2>&1 | grep -oE '\[MD[0-9]+\]' | sort | uniq -c | sort -rn
```

Expected: violations exist (backlog is known, was ~900 across repo+kb in July), but ZERO counts for `MD013`, `MD025`, `MD033`, `MD036`, `MD041`. If any disabled rule appears, the config isn't being read — stop and investigate (rumdl reads `.rumdl.toml` from the current directory). Record the observed rule counts in your report.

- [x] **Step 4: Confirm the kb is invisible to this pass** (motivates Task 2's second pass): the output of Step 3 must contain no `kb/` paths. Also confirm `.venv/` and `.claude/worktrees/` produce no findings. Record both observations.

- [x] **Step 5: Gate and commit.**

```bash
uv run pytest tests/ -q && uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn
git add .rumdl.toml pyproject.toml uv.lock
git commit -m "feat(lint): add rumdl dev dependency and .rumdl.toml baseline"
```

### Task 2: the `docs/markdown` rule + tests

**Files:**
- Create: `linters/rules/docs/markdown.sh` (mode 755)
- Modify: `linters/.lint-config.json`
- Test: `tests/linter/test_markdown_rule.py`

**Interfaces:**
- Consumes: `.rumdl.toml` from Task 1 (tests copy the real shipped file into fixtures).
- Produces: rule name `docs/markdown` (derived by the runner from the path — no runner change), registered in `.lint-config.json`.

- [x] **Step 1: Write the failing tests** at `tests/linter/test_markdown_rule.py`:

```python
"""Tests for the docs/markdown external lint rule (linters/rules/docs/markdown.sh)."""

from __future__ import annotations

import shutil
import subprocess
from pathlib import Path

import pytest

REPO_ROOT = Path(__file__).resolve().parents[2]
SCRIPT = REPO_ROOT / "linters" / "rules" / "docs" / "markdown.sh"
RUMDL_CONFIG = REPO_ROOT / ".rumdl.toml"

BAD_FENCE = "# Doc\n\nText.\n\n```\ncode with no language\n```\n"
CLEAN = "# Doc\n\nA clean paragraph.\n"
HOUSE_STYLE = (
    "# Doc\n\n**Bold lead-in.** This long line goes on and on and on and on "
    "and on and on and on and on and on well past eighty characters.\n"
)


def make_project(tmp_path: Path) -> Path:
    root = tmp_path / "proj"
    root.mkdir()
    shutil.copy(RUMDL_CONFIG, root / ".rumdl.toml")
    subprocess.run(["git", "init", "-q", str(root)], check=True)
    return root


def run_rule(root: Path, env: dict[str, str] | None = None) -> subprocess.CompletedProcess[str]:
    kwargs = {}
    if env is not None:
        kwargs["env"] = env
    return subprocess.run(
        [str(SCRIPT), str(root)], capture_output=True, text=True, check=False, **kwargs
    )


def test_violation_reported_in_framework_format(tmp_path: Path):
    root = make_project(tmp_path)
    (root / "bad.md").write_text(BAD_FENCE)
    result = run_rule(root)
    assert result.returncode == 1
    assert "bad.md:5 — [MD040]" in result.stdout


def test_clean_project_passes(tmp_path: Path):
    root = make_project(tmp_path)
    (root / "good.md").write_text(CLEAN)
    result = run_rule(root)
    assert result.returncode == 0
    assert "[MD" not in result.stdout


def test_house_style_produces_no_violations(tmp_path: Path):
    """MD036 (bold lead-in) and MD013 (line length) are disabled by config."""
    root = make_project(tmp_path)
    (root / "style.md").write_text(HOUSE_STYLE)
    result = run_rule(root)
    assert result.returncode == 0
    assert "[MD" not in result.stdout


def test_excluded_and_gitignored_paths_are_silent(tmp_path: Path):
    root = make_project(tmp_path)
    (root / "node_modules").mkdir()
    (root / "node_modules" / "dep.md").write_text(BAD_FENCE)
    (root / ".claude" / "worktrees").mkdir(parents=True)
    (root / ".claude" / "worktrees" / "wt.md").write_text(BAD_FENCE)
    (root / ".gitignore").write_text("ignored/\nkb/\n")
    (root / "ignored").mkdir()
    (root / "ignored" / "gen.md").write_text(BAD_FENCE)
    (root / "good.md").write_text(CLEAN)
    result = run_rule(root)
    assert result.returncode == 0
    assert "[MD" not in result.stdout


def test_kb_clone_is_linted_with_prefixed_paths(tmp_path: Path):
    """kb/ is gitignored in the outer repo but must still be linted."""
    root = make_project(tmp_path)
    (root / ".gitignore").write_text("kb/\n")
    kb = root / "kb"
    kb.mkdir()
    subprocess.run(["git", "init", "-q", str(kb)], check=True)
    (kb / "doc.md").write_text(BAD_FENCE)
    result = run_rule(root)
    assert result.returncode == 1
    assert "kb/doc.md:5 — [MD040]" in result.stdout


def test_tool_absent_skips_with_hint(tmp_path: Path):
    root = make_project(tmp_path)
    (root / "bad.md").write_text(BAD_FENCE)
    # PATH without the project venv or uv: rumdl unresolvable either way.
    result = run_rule(root, env={"PATH": "/usr/bin:/bin", "HOME": str(tmp_path)})
    assert result.returncode == 0
    assert "rumdl not found — skipping. Install with: pip install rumdl" in result.stdout
```

- [x] **Step 2: Run the tests to verify they fail.**

```bash
uv run pytest tests/linter/test_markdown_rule.py -v
```

Expected: FAIL — the script does not exist yet.

- [x] **Step 3: Write `linters/rules/docs/markdown.sh`:**

```bash
#!/usr/bin/env bash
# lint-markdown — Lint rule: docs/markdown
#
# Runs rumdl over the project's markdown. File selection and exclusions come
# from .rumdl.toml — this rule builds no file list of its own. The kb clone
# at kb/ is gitignored in the host repo (invisible to a gitignore-respecting
# pass), so it gets a second pass of its own.
#
# Exit 0 if clean or rumdl is unavailable. Exit 1 if any violation.

set -uo pipefail

PROJECT_ROOT="${1:-$(cd "$(dirname "$0")/../../.." && pwd)}"

# Resolve the runner: installed rumdl, else the project env via uv.
# --project keeps uv resolving the same env when cwd is the kb pass.
if command -v rumdl &>/dev/null; then
  RUNNER=(rumdl)
elif uv run --no-sync --project "$PROJECT_ROOT" rumdl --version &>/dev/null; then
  RUNNER=(uv run --no-sync --project "$PROJECT_ROOT" rumdl)
else
  echo "rumdl not found — skipping. Install with: pip install rumdl"
  exit 0
fi

FAILED=0

# lint_tree <dir> <prefix> — run rumdl in <dir>, reformat concise output
# (path:line:col: [MDxxx] message) to framework format (path:line — [MDxxx]
# message), prefixing paths with <prefix>.
lint_tree() {
  local dir="$1" prefix="$2" out rc line matched=0
  out=$( (cd "$dir" && "${RUNNER[@]}" check --output-format concise .) 2>&1 )
  rc=$?
  while IFS= read -r line; do
    if [[ "$line" =~ ^(\./)?([^:]+):([0-9]+):[0-9]+:\ (\[MD[0-9]+\]\ .+)$ ]]; then
      echo "${prefix}${BASH_REMATCH[2]}:${BASH_REMATCH[3]} — ${BASH_REMATCH[4]}"
      matched=1
      FAILED=1
    fi
  done <<<"$out"
  # A non-zero exit with no parseable violations is a tool failure (bad
  # config, crash) — surface it instead of silently passing.
  if [ "$rc" -ne 0 ] && [ "$matched" -eq 0 ]; then
    echo "${prefix:-.}: rumdl failed (exit $rc)"
    FAILED=1
  fi
}

lint_tree "$PROJECT_ROOT" ""
if [ -e "$PROJECT_ROOT/kb/.git" ]; then
  lint_tree "$PROJECT_ROOT/kb" "kb/"
fi

exit "$FAILED"
```

Then `chmod 755 linters/rules/docs/markdown.sh`.

- [x] **Step 4: Register the rule.** In `linters/.lint-config.json`, add after the `scripts/shellcheck` entry:

```json
"docs/markdown": { "enabled": true, "severity": "warning" }
```

- [x] **Step 5: Run the tests until they pass.**

```bash
uv run pytest tests/linter/test_markdown_rule.py -v
```

Expected: 6/6 PASS. If a test fails because rumdl's real CLI behavior differs from the script's assumptions (config discovery from cwd, `./`-prefixed paths, kb-pass config lookup), fix the SCRIPT to meet the tests' behavioral contract — the contract itself only changes if rumdl genuinely cannot express it, and that goes in your report.

- [x] **Step 6: Verify against the real repo.** Run:

```bash
bash linters/rules/docs/markdown.sh "$(pwd)" | head -20; echo "exit: ${PIPESTATUS[0]}"
uv run rcorn kb lint 2>&1 | tail -15
```

Expected: real violations in `path:line — [MDxxx] message` format including `kb/`-prefixed paths, script exit 1; `rcorn kb lint` shows `[FAIL:WARNING] docs/markdown` and overall EXIT CODE 0 (warning severity — run `echo $?` to confirm). (`uv run rcorn` is correct here: this exercises uncommitted code.)

- [x] **Step 7: Full gate and commit.**

```bash
uv run pytest tests/ -q && uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn
git add linters/rules/docs/markdown.sh linters/.lint-config.json tests/linter/test_markdown_rule.py
git commit -m "feat(lint): add the docs/markdown rule backed by rumdl, registered at warning"
```

### Task 3: ship `.rumdl.toml` as an asset, seed it on init

**Files:**
- Modify: `pyproject.toml` (`[tool.hatch.build.targets.wheel.force-include]`)
- Modify: `src/reinicorn/commands/init.py` (`_copy_lint_config`)
- Test: extend the existing init test that asserts `linters/` is copied (locate it via `grep -rn "Copied linters" tests/` or `grep -rln "_copy_lint_config\|lint-config" tests/`)

**Interfaces:**
- Consumes: `.rumdl.toml` from Task 1; `get_asset_path()` from `src/reinicorn/assets.py` (resolves `_data/` in wheels, repo root in editable installs — `.rumdl.toml` at the repo root already resolves in dev with no extra work).

- [x] **Step 1: Write the failing test.** Find the existing test covering init's linters copy and add, in the same test or a sibling following its exact fixture pattern:

```python
assert (project / ".rumdl.toml").exists()
```

(where `project` is that test's initialized-project path). Run it; expected: FAIL.

- [x] **Step 2: Bundle the asset.** In `pyproject.toml` under `[tool.hatch.build.targets.wheel.force-include]`, add:

```toml
".rumdl.toml" = "reinicorn/_data/.rumdl.toml"
```

- [x] **Step 3: Seed on init.** In `src/reinicorn/commands/init.py`, extend `_copy_lint_config` to also copy the markdown lint config:

```python
def _copy_lint_config(target_dir: Path) -> None:
    """Copy linters/ directory and the markdown lint config to the target repo."""
    lint_src = get_asset_path("linters")
    if lint_src is None:
        return
    lint_dest = target_dir / "linters"
    lint_dest.mkdir(parents=True, exist_ok=True)
    shutil.copytree(lint_src, lint_dest, dirs_exist_ok=True)
    console.success("Copied linters/ config")
    print()

    rumdl_src = get_asset_path(".rumdl.toml")
    if rumdl_src is not None:
        shutil.copy(rumdl_src, target_dir / ".rumdl.toml")
        console.success("Copied .rumdl.toml")
        print()
```

- [x] **Step 4: Run the test to verify it passes.**

```bash
uv run pytest tests/ -q -k "init"
```

Expected: PASS, including the new assertion.

- [x] **Step 5: Full gate and commit.**

```bash
uv run pytest tests/ -q && uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn
git add pyproject.toml src/reinicorn/commands/init.py tests/
git commit -m "feat(init): ship .rumdl.toml as a bundled asset and seed it on init"
```

## Dependencies

- Spec's hard ordering dependency (frontmatter migration, reinicorn#30 / reinicorn-kb#9) has LANDED — `src/reinicorn/frontmatter.py` exists, `docmeta.py` is gone, migrated docs carry `review_pr:` in frontmatter. Verify-only acceptance items (`MD034`, `MD025`) confirm it.
- Concurrent branches touching nearby code: `test/lint-and-runner-coverage` (linter tests) — this plan only ADDS `tests/linter/test_markdown_rule.py`, no edits to existing linter tests; and an uncommitted hand-edit to root `README.md` in the main checkout — this branch must not touch `README.md`.
- Non-goals deferred per spec: the `rumdl fmt` cleanup sweep (separate PR; fix `MD040` by hand there), promotion to `error` severity, rumdl opt-in extras.
