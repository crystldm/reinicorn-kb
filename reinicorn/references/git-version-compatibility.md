# Git Version Compatibility

**Date:** 2026-02-22
**Author:** Michael Biehl
**Origin:** ai-assisted
**Human-validated:** false

## Minimum Version

**Git 2.28+** (released 2020-07-27)

Required for `git init -b main` (explicit default branch name).

## Known Compatibility Issues

### `protocol.file.allow` — Git 2.38+ (CVE-2022-39253)

Git 2.38 introduced a security hardening that blocks local `file://`
transport by default in sub-transport contexts (clone, submodule add,
fetch, push to local paths).

**Affected reins commands:**
- `reins attach` (mode 2: local bare repo)
- `reins kb publish` (push to local remote)
- `reins kb sync` (fetch from local remote)
- Test fixtures that create local bare repos

**Fix pattern:** Pass `-c protocol.file.allow=always` on the command
line for git operations involving local file paths. The centralized
helper `file_transport_args()` in `src/reins/git.py` detects
local remotes and returns the needed args.

**Important:** Config-based `protocol.file.allow` (set via `git config`)
is unreliable on git 2.52+. The `-c` command-line flag is the only
method that works across all versions. For tests, use the
`GIT_ALLOW_PROTOCOL` environment variable set at module level.

### Default Branch Name — Git 2.28+

Older git versions default to `master`; newer versions and many CI
runners default to `main`. Reins assumes `main` for the kb
submodule branch.

**Fix pattern:** Always use `git init -b main` instead of bare
`git init` to ensure consistent branch naming across environments.

## CI Environment (GitHub Actions)

- **Runner git version:** 2.52.0 (as of 2026-02-22)
- **No global git config:** Runners have no `user.name` or `user.email`
  set. Test fixtures must configure identity per-repo.
- **`actions/checkout@v4`** with `submodules: recursive` rewrites
  submodule URLs to HTTPS and may set its own protocol env vars.

## Test Infrastructure

All git compatibility workarounds for tests live in `tests/conftest.py`:

- `_git_init()` — uses `-b main`, sets user identity
- `GIT_ALLOW_PROTOCOL` — set at module level to allow file transport
- `submodule_repo` fixture — sets identity and protocol config in the
  kb submodule directory
