---
type: idea
title: 'Branch-to-directory encoding is not injective: feature/good and feature-good sha'
slug: branch-to-directory-encoding-is-not-injective-feature-good-a
lifecycle: active
status: new
created: 2026-09-08
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Branch-to-directory encoding is not injective: feature/good and feature-good sha

## Description

Branch-to-directory encoding is not injective: feature/good and feature-good share one kb dir

## Notes

Surfaced by CodeRabbit on PR #71 (process-as-config stage 3, 2026-09-07)
at the process gate's branch-directory lookup; deferred there because it
predates the PR. The same seam has come up before: PR #28 confined
`sanitize_branch` to one module with an AST lint after a misuse, and the
tech-debt note `small-duplication-cleanups-from-quality-review` flags the
`kb.branch_dir_name` alias for it. Neither addressed the encoding itself.

### Mechanism

Branch-addressed docs live under a directory named after the branch
(`exec-plans/<stage>/<branch>/plan.md`). Git allows `/` in branch names;
a slash would nest directories, so `git.sanitize_branch` maps `/` and
`\` to `-`. The mapping loses information: `feature/good` and
`feature-good` are two valid, distinct branches that both resolve to
`exec-plans/active/feature-good/`.

Everything that turns a branch into a path goes through that one seam
(`corpus.doc_path` → `kb.branch_dir_name` → `git.sanitize_branch`):
`plan create`, `plan show`, `plan complete`, `check_overlap`, the
`kb/lifecycle` lint and `_process-gate`. If two colliding branches ever
coexist, the second one sees the first one's plan as its own, and the
gate judges one branch by the other's docs. Nothing reports the clash.

### Why it has not bitten

Every branch in this repo is hyphenated, so no two live branches collide
today. The failure needs two concurrent branches whose names differ only
by slash versus hyphen, which is rare but silent when it happens.

### Fix shape

1. Pick an injective encoding for the directory name. Cheapest option
   that stays readable: keep `-` for `/` but escape a literal `-` (or
   `_`) in the source name, or append a short hash of the raw branch
   name only when the sanitized form is ambiguous. Percent-encoding is
   fully injective but ugly in `ls`.
2. Record the raw branch in frontmatter (`branch:` already exists for
   plans) and make lookups match on that field, not on the directory
   name, so the directory becomes a label rather than the identity.
3. Migrate existing dirs in both stages, in kb main, in one commit, with
   a lint that fails on a directory whose name is not the encoding of its
   frontmatter `branch`.
4. Keep the `sanitize_branch` confinement lint from #28; the new
   encoder replaces it inside the same module.

Option 2 is the durable half: once identity comes from frontmatter, the
directory encoding can be anything and a future rename is a plain move.
