# PreToolUse hook to block direct git submodule commands from agents. Agents must 

**Date:** 2026-03-06
**Author:** mnbiehl
**Status:** resolved
**Resolved:** 2026-05-05

## Description

PreToolUse hook to block direct git submodule commands from agents. Agents must use reins CLI (init/publish/sync) instead. Also add a reins passthrough command (e.g. reins git submodule ...) as an escape hatch for custom submodule operations that still go through reins's safeguards.

## Resolution

Both pieces landed in M5:

- Guardrail: `editor-hooks/block-raw-kb-git.sh` — PreToolUse hook on Bash
  blocks raw `git -C kb …`, `git submodule …`, semicolon/pushd evasions.
- Escape hatch: `reins kb-git` passthrough command for legitimate raw
  git operations inside the kb submodule.
