---
type: idea
title: Browsable kb site via mdBook (or similar static doc generator)
slug: browsable-kb-site-via-mdbook-or-similar-static-doc-generator
lifecycle: active
status: new
created: 2026-10-02
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Browsable kb site via mdBook (or similar static doc generator)

## Description

Browsable kb site via mdBook (or similar static doc generator)

The kb is plain markdown across scopes (ideas, specs, plans, retros, principles, debt) and is navigated today through 'rcorn <type> list/show' or a file browser. Humans reviewing intent need an easy way to browse it. Generate a static site from the kb with mdBook (or MkDocs Material, Docusaurus, Starlight; pick by search quality, zero-config build and no-JS fallback): a SUMMARY/nav generated from the doc-type registry (REGISTRY drives the sections, so custom doc types appear automatically), frontmatter rendered as badges (status, lifecycle, review state, author), [[wikilinks]] and depends_on/closes relations turned into links and backlinks, and per-scope index pages. Command surface: 'rcorn kb site build' / 'rcorn kb site serve' for local use, plus an optional GitHub Pages workflow on the kb repo (respecting private kb repos). Related: obsidian-integration (vault-native conventions), html-as-a-kb-doc-type, lavish-axi visual review surface, rendered-markdown PR review via Obsidian. This is the low-effort, tool-agnostic member of that family.

## Notes

_No additional notes yet._
