# Inert kb submodule pointer

**Date:** 2026-07-26
**Author:** Michael Biehl
**Status:** in-review
**Origin:** ai-assisted
**Human-validated:** false
**Review-PR:** https://github.com/crystldm/reinicorn-kb/pull/7

## Problem

The kb is a git submodule at `kb/`, so the parent repo records a specific kb
commit. Nothing wants that commit.

`ensure_kb_on_main()` (`src/reinicorn/kb.py:82`) runs on sync, publish,
review, and post-checkout, and forces the kb onto `main`. `stage_kb_pointer()`
(`kb.py:92`) then runs after those same operations and does `git add kb`, so
the next parent commit carries whatever `main` happened to be at that moment.
The recorded SHA therefore means "kb main as of some arbitrary past commit" —
and anyone who checked out that SHA deliberately would be dragged back to
`main` by the next `rcorn` command. `.gitmodules` already declares
`branch = main`, which is the real intent leaking through a mechanism built
for pinning.

Four concrete costs:

1. **Pointer-bump PRs.** `main` is protected (`main-pr-gate`, `main-safety`;
   PR + status checks required, bypass limited to a repository role), so
   refreshing a stale pointer means a PR whose entire content is a 40-byte
   SHA. #25 is one.
2. **Drift, then dangling pointers.** The pointer only advances when someone
   happens to commit in the parent, so every kb-only publish leaves it behind.
   Worse, doc-creating commands commit to the kb locally without pushing, so a
   parent commit can reference kb commits that exist nowhere else; CI then
   fails at `submodules: recursive` checkout. Filed as #21.
3. **CI lints a stale kb.** All four workflows (`test`, `lint-kb`,
   `lint-architecture`, `doc-gardening`) use `submodules: recursive`, which
   checks out the *pinned* commit. On the #22 run that was four commits
   behind, so `Lint Kb` and doc-gardening never saw the newest spec drafts.
4. **It multiplies per consumer repo.** `rcorn init` runs `setup_submodule`
   for every adopting repo. One shared kb, N independently drifting pointers,
   N streams of bump commits.

Separately, and more seriously, the guard that keeps the kb usable is broken
in a way that is currently latent. `ensure_kb_on_main()` is:

```python
r = run_git("symbolic-ref", "--short", "HEAD", check=False, cwd=kb_dir)
if r.returncode != 0 or r.stdout.strip() != "main":
    run_git("checkout", "main", check=False, cwd=kb_dir)
```

It never fetches, and it discards both the exit code and the output. Two
verified failure modes:

- **It can move the kb backwards.** From a detached HEAD at `origin/main`'s
  tip, `git checkout main` lands on the *local* `main` ref, which may be
  stale. Reproduced: content silently reverted from `v2` to `v1`. Git prints
  "Your branch is behind 'origin/main'"; `check=False` throws it away.
- **It can strand commits off-branch.** If the checkout fails — uncommitted
  kb work conflicting with `main` — it exits 1 and HEAD stays detached.
  `commit_kb()` swallows that and proceeds to `git add -A` and `git commit`
  anyway. Reproduced: the commit lands, is not an ancestor of local `main`,
  and `push origin main` does not carry it. It is recoverable only via the
  reflog. A doc would appear committed and silently never publish.

Nothing detaches the kb today, which is the only reason this has not bitten.
Any move toward `git submodule update --remote` makes detached HEAD the normal
state and turns both modes into everyday paths. That is why this is in scope
here rather than filed separately.

## Design Goals

- Routine work never touches the submodule pointer — not by hand, not by
  accident, not via `git add -A` or `git commit -a`.
- No pointer-bump PRs, and no bot with branch-protection bypass.
- CI always lints the current kb, never a pinned snapshot.
- `commit_kb()` never commits to the kb unless it is genuinely on an
  up-to-date `main`; if it cannot get there, it fails loudly.
- No breaking change to `rcorn init` and no migration for consumer repos.
- A deliberate pointer bump remains possible when someone actually wants one.

## Design

### 1. Freeze the pointer with `ignore = all`

Add to the kb entry in `.gitmodules` (keeping the existing `branch = main`):

```
[submodule "kb"]
	path = kb
	url = https://github.com/crystldm/reinicorn-kb.git
	branch = main
	ignore = all
```

Verified behavior with `ignore = all`:

| operation | result |
|---|---|
| `git status` | clean — no `M kb` |
| `git add -A` | pointer not staged |
| `git commit -a` | pointer not swept in |
| `git add kb` (explicit) | pointer **not** staged either |

The last row is the load-bearing one: the flag does not merely hide drift, it
freezes the pointer. That means `stage_kb_pointer()` becomes a no-op the
moment this lands, with no code change — the four call sites that generate all
the churn are neutralized by one line.

### 2. Delete the now-dead pointer staging

With the pointer frozen, `stage_kb_pointer()` and its call sites in
`commands/sync.py`, `commands/publish.py`, `commands/internal/post_checkout.py`,
and `kb.py:commit_kb` are dead code that silently does nothing. Remove the
function and its callers rather than leaving code whose stated purpose no
longer happens. This is hygiene, not correctness — step 1 alone stops the
churn.

### 3. Fix `ensure_kb_on_main()`

Rewrite it to mean "the kb is on an up-to-date `main`", and to report failure:

- `git fetch origin main` first, so "main" means `origin/main` rather than a
  possibly-stale local ref.
- If HEAD is not `main`, `git checkout main`; then fast-forward local `main`
  to `origin/main`. Never move the working tree backwards.
- Return a success/failure result instead of discarding it. Keep uncommitted
  kb work intact — abort rather than force.

`commit_kb()` must check that result and refuse to stage or commit when the kb
is not on a clean, current `main`, printing what is wrong and how to fix it
(per golden principle 4). Committing into a detached HEAD is never acceptable:
the work looks saved and is not.

Fetch cost: one network round trip per kb-writing command. `sync` and
`publish` already hit the network; `doc create` and `plan create` do not, so
this makes them slower. If that proves annoying, the fetch can be skipped when
`origin/main` was fetched within the last N seconds — deliberately left out of
the first cut.

### 4. CI takes the branch tip, not the pin

In `test.yml`, `lint-kb.yml`, `lint-architecture.yml`, and
`doc-gardening.yml`, keep `submodules: recursive` and add a step immediately
after checkout:

```yaml
      - name: Use current kb main
        run: git submodule update --remote kb
```

Verified: `--remote` follows the `branch = main` already in `.gitmodules`,
fetches, and checks out the branch tip, ignoring the recorded pin. It also
refuses to clobber uncommitted submodule changes — it aborts with "Please
commit your changes or stash them" — so it is safe in a scripted context.

Note it leaves the kb in **detached HEAD**. That is fine for CI, which only
reads files, and is precisely why §3 must land alongside this.

### 5. Post-checkout should fetch

`commands/internal/post_checkout.py` currently calls `ensure_kb_on_main()`,
which under §3 gains the fetch. Once §3 lands this needs no separate change,
but the fixed guard must be verified against a fresh-clone path where the kb
has never been fetched.

### 6. The pin becomes a bootstrap floor

The recorded SHA stops being maintained and becomes the commit a fresh
`git clone --recurse-submodules` lands on before its first sync. It will get
old. That is acceptable because every `rcorn kb` command and every CI job
moves off it immediately.

When someone does want to move the floor — say once per release — the escape
hatch is verified to work:

```bash
git config submodule.kb.ignore none && git add kb && git config --unset submodule.kb.ignore
```

Whether to wrap that as `rcorn kb pin` is left open; it is a rare, deliberate
operation and a documented two-liner may be enough.

## Non-Goals

- **Removing the submodule** (gitignored clone managed entirely by `rcorn`).
  It would delete the stale pin outright rather than making it inert, but it
  is a breaking change to `init` plus a migration for every consumer repo,
  and `upgrades/` is currently just a README. Revisit if the inert pin proves
  insufficient.
- **A CI job that bumps the pointer** on merge or on a schedule. Redundant
  once CI uses `--remote`: nothing would read the value it maintains. Per-merge
  also races with human merges (the race that produced #25), moves commit
  noise into `main`'s history rather than removing it, and pushing to a
  protected `main` needs a bypass actor or an admin PAT stored in a public
  repo.
- **Historical docs-with-code reproducibility.** Making the pin meaningful
  would mean removing `ensure_kb_on_main()`'s force-to-main entirely — a
  different philosophy, not a tweak. Out of scope.
- **Fetch-throttling `ensure_kb_on_main()`.** Noted in §3 as a possible
  follow-up; not in the first cut.
- **Fixing the review-lane enforcement gap** (#23) or the unpushed-kb
  dangling-pointer warning (#21). Related and adjacent, tracked separately.
  This spec makes #21's failure mode rarer by removing pointer commits, but
  does not add the pre-push check that issue asks for.
