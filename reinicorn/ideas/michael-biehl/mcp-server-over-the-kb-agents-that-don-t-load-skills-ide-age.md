---
type: idea
title: MCP server over the kb
slug: mcp-server-over-the-kb-agents-that-don-t-load-skills-ide-age
lifecycle: active
status: new
created: 2026-10-02
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# MCP server over the kb

## Description

MCP server over the kb

Agents that don't load skills (IDE agents, Copilot Spaces, Cursor, Kiro, Backstage-over-MCP setups like Spotify's) can't see the team kb today. Expose it via an 'rcorn mcp' server: read-only search/show across doc types first, then doc-type-aware create (routing through the same templates and gates as the CLI so enforcement isn't bypassed). Prior art: AdrMcp / mcp-adr expose ADR folders as search/author/validate tools. Keeps the CLI as the single enforcing seam. Source: landscape research 2026-10-02.

## Notes

_No additional notes yet._
