# Progress: feature-axi-output-surface

## Current Status

All 11 tasks complete: axi channel model, doc show/list verbs, content-first
home view, `kb status --compact` + session-start injection, idempotent
no-ops, and structural enforcement tests are all implemented, tested, and
merged into the branch. Full gate (pytest/ruff/pyright/kb lint) is green.
Plan stays `in-progress`; completion/archival happens at merge time.

## Completed

- 2026-07-02/03 — Implemented the agent-native output surface spec
  (`kb/reins/specs/agent-native-output-surface-axi-principles.md`) end to
  end, across 19 commits on this branch:

  - `9b517cf` feat(console): axi channel model — errors to stdout, progress
    channel, next_step helper
  - `597f524` feat(console): gate indentation decoration on TTY
  - `98f7667` refactor(console): dedupe tty check, cover tty-on indent path
  - `2c3420a` feat(show): doc_show module with truncated preview and --full
  - `e089301` feat(show): list verbs with minimal schema and definitive
    empty state
  - `49a57e2` feat(show): plan/retro show addressed by branch
  - `8e50a52` fix(show): guard Path import behind TYPE_CHECKING per ruff
    TC003
  - `dfdc911` test(show): cover debt/idea/retro type-specific paths
  - `eca7938` feat(cli): wire show/list verbs for all doc types
  - `61eb081` test(cli): parser-dispatch parity and parametrized routing
    coverage
  - `3b1f5d9` feat(home): content-first bare-reins view with live state
  - `5cea585` fix(home): survive uninitialized kb submodule; drop vacuous
    no-args test
  - `246b101` feat(status): --compact dashboard wired into session-start
    ambient context
  - `9c15aee` fix(kb): keep check_overlap silent on early-exit paths,
    tri-state overlap data
  - `b9967f4` refactor(kb): shared dashboard helpers; quiet kb rebase in
    session hook
  - `d392589` feat(output): next-step footers on create/status, publish
    progress to stderr
  - `9818596` feat(idempotency): explicit exit-0 no-ops for mode and hooks
    install
  - `4710054` fix(output): correct blocked-publish hint, unify next-step
    convention, truthful hooks no-op
  - `35df186` test: structural enforcement of axi output conventions

- Test count evolution: 314 (branch start, on `main`) → 362 passed (final,
  after structural enforcement tests + I001 import-sort fixes). 0 failures
  throughout; `uv run ruff check` and `uv run pyright` both clean at every
  commit boundary.

- Review findings fixed mid-branch (caught in review, addressed before
  merge-readiness):
  - **Fresh-clone crash** — the home view crashed when the kb submodule
    was uninitialized (no `kb/.git`); fixed in `5cea585` to survive an
    uninitialized submodule and degrade gracefully instead of raising.
  - **`check_overlap` byte-identity** — overlap detection was doing a
    truthy/loose comparison instead of a byte-identical check on early-exit
    paths, which could misreport overlap state; fixed in `9c15aee` to be
    silent (no premature output) on early-exit paths and to report a
    tri-state (yes/no/unknown) instead of a boolean.
  - **Wrong publish hint** — the `next:` suggestion shown after a blocked
    publish pointed at the wrong follow-up command; corrected in `4710054`.
  - **Hooks no-op truthfulness** — `hooks install` on a re-run claimed
    "no-op" even in cases where it had in fact changed something; `4710054`
    made the no-op summary truthful (only claims no-op when nothing
    changed) and unified the `next:` convention used across commands.

## In Progress

- None — all tasks for this plan are complete. Plan remains `in-progress`
  pending merge/archival by the developer.

## Blocked

- None.
