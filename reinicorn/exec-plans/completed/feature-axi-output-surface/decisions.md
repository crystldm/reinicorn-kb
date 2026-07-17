# Decisions: feature-axi-output-surface

Record architectural and implementation decisions made during this work.

## 2026-07-02 — Compact dashboard omits spec §3's "lint failures, stale count"

**Context:** Spec §3 (`kb/reins/specs/agent-native-output-surface-axi-principles.md`)
describes the `kb status --compact` dashboard as including lint failure
counts and a stale-doc count, and it's injected on every session start via
`session-start.sh`.

**Decision:** Ship the compact dashboard without a live lint-failure count
or a live staleness count. Those numbers stay behind explicit commands
(`reins kb lint`, per-doc inspection) rather than being computed on every
session start.

**Alternatives considered:** Compute a per-doc git-log-based staleness scan
and run the lint suite on every session start, folding both counts into the
compact dashboard as spec'd.

**Rationale:** A per-doc git-log stale scan is too slow to run on every
session start (it's O(docs) git-log invocations); running the full lint
suite on every session start has the same problem plus false urgency from
pre-existing warnings. The compact dashboard is meant to be near-instant
ambient context, not a full health report. Deep data (lint results,
staleness) stays reachable via explicit commands instead of being computed
eagerly and cheapened by session-start latency.

## 2026-07-02 — `doc show` truncates the whole file at 1500 chars, not spec §1's "front-matter + first ~1500 chars of body"

**Context:** Spec §1 describes preview truncation as "front-matter +
first ~1500 chars of body" — i.e. front-matter provenance is exempt from
the character budget and truncation only applies to body content.

**Decision:** `doc_show` truncates the whole file (front-matter included)
at 1500 chars, rather than truncating only the body after an untruncated
front-matter block.

**Alternatives considered:** Parse out front-matter separately, always
show it in full, and apply the 1500-char budget only to the remaining body
text.

**Rationale:** Provenance front-matter blocks in this codebase are ~7 lines
(Author/Status/Origin/etc.), a small, roughly constant overhead against a
1500-char budget. Treating the whole file as the truncation unit produces
behavior equivalent to the spec'd split for real documents, with
significantly simpler implementation (no front-matter/body parsing step in
the preview path).

## 2026-07-02 — Errors dual-write rejected

**Context:** During spec review, dual-writing `error:` lines to both
stdout and stderr was considered, so that human operators tailing stderr
would still see failures.

**Decision:** Rejected. Errors go to stdout only, verbatim per the axi
spec.

**Alternatives considered:** Dual-write errors to stdout (for agents) and
stderr (for humans/log aggregators watching stderr).

**Rationale:** The axi spec is explicit that stdout carries all
agent-consumed output including errors; dual-writing muddies the channel
contract the structural tests enforce (`tests/test_output_conventions.py`
asserts stderr is empty on `console.error`) and reintroduces the exact
ambiguity — "where do I look for the real answer" — the spec exists to
remove.

## 2026-07-02 — No `internal/` stderr exclusions needed

**Context:** Task 11's structural sweep (`tests/test_output_conventions.py::
test_no_direct_stderr_prints_in_commands`) checks all of
`src/reins/commands/` (including `internal/`, the git-hook command modules)
for raw `file=sys.stderr` prints, since git-hook protocols sometimes
legitimately require raw stderr writes that the plan allowed excluding if
needed.

**Decision:** No exclusion was needed. `internal/` (hook_check.py,
post_checkout.py, post_merge.py, pre_push.py) contains zero raw
`file=sys.stderr` prints as written, so the sweep runs unmodified over the
whole `commands/` tree.

**Alternatives considered:** Pre-emptively excluding `internal/` from the
sweep.

**Rationale:** Excluding a directory from a lint before it's shown to
violate the rule adds an unexplained gap; since the hook modules already
comply, the sweep is left unconditional so any future raw stderr print
anywhere in `commands/` (hooks included) will fail the test and require a
deliberate, documented exclusion at that time.
