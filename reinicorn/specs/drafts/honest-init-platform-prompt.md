# Honest init platform prompt

**Date:** 2026-07-20
**Author:** Rodion Izotov
**Status:** draft
**Origin:** ai-assisted
**Human-validated:** false

## Problem

`rcorn init` asks which AI coding platforms to install instructions for via
`_prompt_platforms()` in `src/reinicorn/commands/init.py`. The prompt prints
checkbox-looking markers (`[x]` / `[ ]`) next to a numbered list, then waits
on plain `input()` for comma-separated numbers (or Enter to keep defaults).

That visual affordance implies an arrow-key / space-to-toggle TUI. Users who
try keyboard navigation discover the prompt is actually typed-input only.
The adjacent kb-source prompt (`Choose [1/2/3]:`) is already honest about
typed selection; the platform prompt is not.

Reinicorn has zero runtime dependencies, so a real interactive multi-select
library is out of scope for this fix.

## Design Goals

- The platform prompt must not look like an interactive checkbox TUI.
- Number-toggle behavior stays: type one or more option numbers (comma-
  separated) to flip those defaults; empty Enter keeps the defaults.
- Defaults remain visible in the prompt line (today: Claude Code only).
- Zero new runtime dependencies.
- Existing callers and install behavior for selected platforms are unchanged.

## Design

### 1. Plain numbered list

Replace checkbox markers with a plain numbered list:

```
Which AI coding platforms do you use?

  1) Claude Code
  2) Cursor
  3) GitHub Copilot
  4) Codex

Toggle by number (e.g. 2,4), Enter to confirm [Claude Code]:
```

Option keys, labels, and default flags stay as they are today
(`claude` default-on; `cursor`, `copilot`, `codex` default-off).

### 2. Prompt line carries defaults and interaction model

The input line states:

1. How to toggle (numbers, optionally comma-separated).
2. That Enter confirms.
3. Which platforms are currently default-selected, in brackets using the
   human labels (e.g. `[Claude Code]`). If multiple defaults ever exist,
   join labels with commas inside the brackets.

No per-row selected/unselected decoration in the list.

### 3. Parse logic unchanged

Keep the current toggle semantics:

- Start from the option default flags.
- If the user typed anything, split on commas (whitespace stripped); each
  digit token in range toggles that option index.
- Non-digit / out-of-range tokens are ignored (same as today).
- Return the list of selected platform keys.

### 4. Tests

Add a focused unit test for `_prompt_platforms` that mocks `input` and
captures stdout, asserting:

- Output does not contain `[x]` or `[ ]` checkbox markers.
- Empty input returns the default selection (`["claude"]` with today’s
  defaults).
- A toggle input (e.g. `"2"`) flips the expected option.

Existing init tests that mock `_prompt_platforms` remain unchanged.

## Non-Goals

- No interactive TUI (arrow keys, space toggle, cursor libraries).
- No new runtime dependencies (`questionary`, `rich`, etc.).
- No `--platforms` CLI flag or non-interactive init path in this change.
- No rewrite of other init prompts (`_prompt_kb_source`, gh auth, etc.),
  except that this prompt should remain stylistically consistent with them.
- No change to which platform instruction files are generated once a
  selection is chosen.
