# Execution Plan: fix/kb-remote-resolution-and-publish-errors

**Ticket:** N/A
**Spec:** N/A (fixes two ideas; must survive [[remove-the-kb-submodule]])
**Author:** Michael Biehl
**Created:** 2026-07-27
**Status:** in-progress
**Origin:** ai-assisted
**Human-validated:** false

## Goal

Fix two defects found on 2026-07-27:

1. **[[worktree-kb-init-uses-the-gitmodules-https-url-and-drops-the]]** — a kb
   created by the post-checkout hook takes its remote from `.gitmodules`
   (HTTPS), discarding the main checkout's SSH override. With `gh` set to
   `git_protocol: ssh` for `github.com` and no credential helper, every
   `rcorn kb publish` from that worktree fails with
   `could not read Username for 'https://github.com'`.
2. **[[rcorn-kb-publish-reports-any-post-retry-push-failure-as-conf]]** —
   `push_main_with_retry`'s callers report every non-zero push as "kb has
   conflicting changes", masking the auth failure above and suggesting a retry
   that loops forever.

## Acceptance Criteria

- [x] A kb clone created by the post-checkout hook inherits `remote.origin.url`
      from the main checkout's kb when one exists, so local overrides (SSH,
      mirror, internal host) follow into every worktree.
- [x] With no kb to inherit from, the remote is resolved from
      `REINICORN_KB_REMOTE` (then `.gitmodules` as a transitional fallback) and
      adapted to the user's configured git protocol for `github.com`.
- [x] Protocol detection reads the **host-scoped** `gh` setting; the global
      `gh config get git_protocol` is not authoritative.
- [x] HTTPS is never rewritten to SSH without positive evidence of an SSH
      preference; an SSH URL is never rewritten to HTTPS.
- [x] `rcorn kb publish` and the review lane classify a failed push into
      auth / non-fast-forward / protected-branch / unknown, and print git's
      stderr verbatim for `unknown` rather than guessing.
- [x] The auth message names the remote URL and its protocol and offers a
      concrete `rcorn kb git remote set-url` next step.
- [x] No resolution logic is submodule-specific: everything lives behind one
      seam that [[remove-the-kb-submodule]] can keep unchanged.
- [x] Suite stays green and total coverage stays at or above the 87% floor.

## Approach

**Defect 1 — one resolution seam, `kb_remote.resolve_kb_remote_url(root)`.**
New module `src/reinicorn/kb_remote.py`. Resolution order:

1. `remote.origin.url` of the **main checkout's** kb clone (found by walking up
   from `git rev-parse --git-common-dir`). This is the user's own working
   configuration and is protocol-agnostic — it is what fixes the reported bug.
2. `REINICORN_KB_REMOTE` from `.reinicorn-config`.
3. The `.gitmodules` kb `url =` — transitional only, deleted with the submodule.

Whatever comes from (2) or (3) then goes through `adapt_url_to_git_protocol`,
which rewrites `https://github.com/o/r(.git)` to `git@github.com:o/r.git`
**only** when `gh config get -h github.com git_protocol` reports `ssh`. The
rewrite is deliberately one-way: `gh`'s default for an unconfigured host is
`https`, so treating that as evidence would break SSH-only users.

`post_checkout` resolves the URL *before* creating the kb and applies it with
`git remote set-url origin` afterwards. Post-[[remove-the-kb-submodule]] the
same call feeds `git clone <url>` directly — step 1 and step 2 both survive
verbatim; only step 3 is deleted with `.gitmodules`.

**Defect 2 — classify, then report.** `classify_push_failure(stderr)` in
`kb.py` returns `auth` / `non-fast-forward` / `protected` / `unknown`;
`report_push_failure(push, kb_dir)` prints the matching error and next step.
`commands/publish.py` and `commands/review.py::_push_kb_main` both call it, so
the two lanes cannot drift apart again.

## Tasks

- [x] Red: tests for `resolve_kb_remote_url` (inherit, config, `.gitmodules`,
      precedence) and `adapt_url_to_git_protocol` (ssh preference, https
      preference, non-GitHub host, `gh` missing).
- [x] Green: add `src/reinicorn/kb_remote.py`.
- [x] Red: test that `cmd_post_checkout` in a worktree gives the new kb the main
      checkout's origin URL, not the `.gitmodules` one.
- [x] Green: wire `resolve_kb_remote_url` into
      `commands/internal/post_checkout.py`.
- [x] Red: tests for `classify_push_failure` over real git stderr strings and
      for `report_push_failure` output per class.
- [x] Green: add both to `kb.py`; call from `publish.py` and `review.py`.
- [x] Verify: pytest, ruff, pyright, `bash tests/run-all.sh`, coverage >= 87%.

## Dependencies

- [[remove-the-kb-submodule]] (kb PR #8) merges before this lands. It deletes
  `.gitmodules`, `stage_kb_pointer()`, and `.gitmodules` parsing in
  `get_kb_dir()`. Nothing here adds new `.gitmodules` coupling; the only
  `.gitmodules` read is an explicitly-labelled fallback step that #8 removes.
- Concurrent work on `src/reinicorn/frontmatter.py` / kb doc frontmatter — not
  touched here.
