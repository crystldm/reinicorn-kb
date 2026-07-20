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

The current parse model is also a **toggle** (typed numbers flip defaults
on/off). The only user-visible cue is the word "Toggle", which is easy to
miss; typing `2` is naturally read as "I want Cursor", not "add Cursor while
keeping Claude". Select-set semantics match that intuition better.

Reinicorn has zero runtime dependencies, so a real interactive multi-select
library is out of scope for this fix.

## Design Goals

- The platform prompt must not look like an interactive checkbox TUI.
- Typed numbers mean **select-set**: the listed numbers are the chosen
  platforms (replacing defaults), not toggles. Empty Enter keeps the
  defaults.
- Prompt copy must state that model in plain language (no "toggle").
- Defaults remain visible in the prompt line (today: Claude Code only).
- Zero new runtime dependencies.
- Once a selection is chosen, install behavior for those platforms is
  unchanged.

## Design

### 1. Plain numbered list

Replace checkbox markers with a plain numbered list:

```
Which AI coding platforms do you use?

  1) Claude Code
  2) Cursor
  3) GitHub Copilot
  4) Codex

Enter numbers to select (e.g. 1,2), or Enter for default [Claude Code]:
```

Option keys, labels, and default flags stay as they are today
(`claude` default-on; `cursor`, `copilot`, `codex` default-off).

### 2. Prompt line carries defaults and interaction model

The input line states:

1. That typed numbers are the selection (comma-separated allowed).
2. That bare Enter keeps the default.
3. Which platforms are default-selected, in brackets using human labels
   (e.g. `[Claude Code]`). If multiple defaults ever exist, join labels
   with commas inside the brackets.

No per-row selected/unselected decoration in the list. Do not use the word
"toggle".

### 3. Select-set parse logic

Replace toggle semantics with select-set:

- Empty input (after strip) → return the default-selected platform keys.
- Non-empty input → split on commas (whitespace stripped); collect each
  digit token whose index is in range as a selected option. Deduplicate;
  return keys in option-list order (not input order).
- Non-digit / out-of-range tokens are ignored.
- If the input was non-empty but yielded no valid indices, fall back to
  defaults (same forgiving "ignore junk" spirit as today).

Examples with today’s defaults:

| Input | Result |
|-------|--------|
| `` (Enter) | `["claude"]` |
| `2` | `["cursor"]` |
| `1,2` | `["claude", "cursor"]` |
| `2,4` | `["cursor", "codex"]` |
| `abc` | `["claude"]` (no valid indices → defaults) |

### 4. Tests

Add a focused unit test for `_prompt_platforms` that mocks `input` and
captures stdout, asserting:

- Output does not contain `[x]` or `[ ]` checkbox markers.
- Output prompt uses select wording (e.g. contains `select` / `default`,
  does not contain `toggle` case-insensitively).
- Empty input returns the default selection (`["claude"]` with today’s
  defaults).
- Input `"2"` returns `["cursor"]` only (select-set, not toggle).
- Input `"1,2"` returns `["claude", "cursor"]`.

Existing init tests that mock `_prompt_platforms` remain unchanged.

## Non-Goals

- No interactive TUI (arrow keys, space toggle, cursor libraries).
- No new runtime dependencies (`questionary`, `rich`, etc.).
- No `--platforms` CLI flag or non-interactive init path in this change.
- No rewrite of other init prompts (`_prompt_kb_source`, gh auth, etc.),
  except that this prompt should remain stylistically consistent with them.
- No change to which platform instruction files are generated once a
  selection is chosen.
- No re-prompt loop on invalid input beyond the forgiving fallback above.
