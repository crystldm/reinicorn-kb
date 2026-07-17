# pre-push hook should verify harness submodule is pushed before allowing outer re

**Date:** 2026-02-16
**Author:** mnbiehl
**Status:** resolved
**Resolved:** 2026-05-05

## Description

pre-push hook should verify harness submodule is pushed before allowing outer repo push — dangling submodule pointers break checkout for other agents and CI

## Resolution

Implemented as part of M5 publish/sync hardening. `hooks/pre-push` delegates
to `reins _pre-push`, which verifies kb commits are pushed before allowing
the parent push and auto-pushes the submodule when possible.

- Hook: `hooks/pre-push`
- Plan: `kb/reins/exec-plans/active/feature-m5-publish-hardening/`
