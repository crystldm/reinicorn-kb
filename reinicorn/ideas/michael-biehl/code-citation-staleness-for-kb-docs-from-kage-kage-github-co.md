---
type: idea
title: Code-citation staleness for kb docs (from Kage)
slug: code-citation-staleness-for-kb-docs-from-kage-kage-github-co
lifecycle: active
status: new
created: 2026-10-02
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Code-citation staleness for kb docs (from Kage)

## Description

Code-citation staleness for kb docs (from Kage)

Kage (github.com/kage-core/Kage) stores agent memories as git-tracked files that cite code paths/symbols; each citation is verified at write time and a memory is withheld once the cited code changes. Reinicorn has 211 docs untouched >30 days and no mechanism tying a doc to the code it describes.

Proposal: let principles, architecture docs and specs carry a 'cites:' frontmatter list (path, optional symbol, commit SHA at write time). A kb lint rule / session-start check flags docs whose cited code changed since the recorded SHA, and 'rcorn kb status' reports them as stale-with-cause instead of stale-by-age. Complements the 2026-07-04 docs-gardening ideas (GSD SHA stamping) and the converge idea; this is the cheap, mechanical half of drift detection.

Source: landscape research 2026-10-02.

## Notes

_No additional notes yet._
