---
type: debt
title: Committed skillset-wiring.md can drift from its generator
slug: committed-skillset-wiring-md-can-drift-from-its-generator
lifecycle: active
status: draft
created: 2026-10-02
author: Michael Biehl
origin: ai-assisted
human_validated: false
category: skills
severity: low
remediation: planned
---

# Committed skillset-wiring.md can drift from its generator

## Impact

PR #61 commits the generated `.agents/skills/using-reinicorn/references/skillset-wiring.md`. Nothing verifies that the committed copy matches what `reinicorn.skillset.wiring.write_wiring` would produce, so a generator change (or a doc-type registry change) can leave the committed doc stale with no failing signal. Found while preparing PR #61 (2026-10-02).

## Remediation Plan

Add a structural test or lint rule that regenerates the wiring doc into a temp dir from the in-repo code and diffs it against the committed file; fail with the regeneration command in the message.
