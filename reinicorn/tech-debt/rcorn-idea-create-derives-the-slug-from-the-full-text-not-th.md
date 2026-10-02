---
type: debt
title: rcorn idea create derives the slug from the full text, not the title
slug: rcorn-idea-create-derives-the-slug-from-the-full-text-not-th
lifecycle: active
status: draft
created: 2026-10-02
author: Michael Biehl
origin: ai-assisted
human_validated: false
category: cli
severity: low
remediation: planned
---

# rcorn idea create derives the slug from the full text, not the title

## Impact

`rcorn idea create "Title\n\nBody..."` sets the title from the first line correctly, but the slug (and filename) is cut from the whole text, so it bleeds into the body: e.g. `code-citation-staleness-for-kb-docs-from-kage-kage-github-co`. Slugs become unstable and hard to reference with `[[...]]` links. Observed 2026-10-02.

## Remediation Plan

Derive the slug from the parsed title only (first line), with a regression test asserting the filename for a multi-line idea text. Existing malformed slugs can stay; slugs are identities.
