---
type: debt
title: rcorn update does not sync .rumdl.toml — updated installs get a false-green docs/markdown PASS
slug: rcorn-update-does-not-sync-rumdl-toml-updated-installs-get-a
lifecycle: active
status: draft
created: 2026-08-30
author: Michael Biehl
origin: ai-assisted
human_validated: false
category: tooling
severity: medium
remediation: planned
---

# rcorn update does not sync .rumdl.toml — updated installs get a false-green docs/markdown PASS

## Impact

`rcorn update` syncs `linters/` (so an existing install picks up the new
`docs/markdown` rule and its `enabled: true` registration) but
`_collect_package_files` in `src/reinicorn/commands/update.py` probes
directories only — the root-level `.rumdl.toml` asset added by
feat-markdown-lint is never synced. The rule's missing-config guard then
skips with exit 0, and the runner renders the skip as
`[PASS] docs/markdown`: every pre-existing install reports markdown as
clean while nothing was checked. Fresh `rcorn init` runs are unaffected
(init seeds the config).

The spec scoped seeding to `rcorn init` deliberately; the update path was
not considered. Flagged by the feat-markdown-lint final whole-branch
review (2026-08-30).

## Remediation Plan

Teach `_collect_package_files` about root-level file assets: extend the
`asset_probes` table (or add a sibling table) so a single file like
`.rumdl.toml` maps source → destination the same way directories do, and
add it to the probe list. Then `rcorn update` offers the config like any
other bundled file, and `rcorn update --diff .rumdl.toml` works.
Alternatively (or additionally), make the runner print skip reasons on
PASS so a skipped tool is distinguishable from a clean check — that also
covers the pre-existing shellcheck-absent case.
