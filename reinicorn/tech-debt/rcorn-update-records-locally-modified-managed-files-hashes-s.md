---
type: debt
title: rcorn update records locally-modified managed files' hashes, so the next update overwrites them
slug: rcorn-update-records-locally-modified-managed-files-hashes-s
lifecycle: active
status: draft
created: 2026-10-02
author: Michael Biehl
origin: ai-assisted
human_validated: false
category: cli
severity: medium
remediation: planned
---

# rcorn update records locally-modified managed files' hashes, so the next update overwrites them

## Impact

`rcorn update` hashes managed files from disk when it writes the update manifest. A file it skipped because the user modified it locally gets its *modified* hash recorded, so the next update sees it as untouched and overwrites the user's changes. Found by the PR #66 agent (2026-10-02) while deliberately keeping `.rumdl.toml` out of the manifest for the same reason.

## Remediation Plan

Record the shipped (template) hash for skipped files rather than the on-disk hash, or keep the previous manifest entry when a file is skipped as locally modified. Red/green test: modify a managed file, update twice, assert the modification survives.
