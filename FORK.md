# Fork notes

This is a personal fork of [aqua5230/usage](https://github.com/aqua5230/usage). We use only the Claude Code
status line, `usage_statusline.py`. The menu bar app is not built or installed from this fork.

## Branches

- `ks-main`: upstream tag `v0.31.4` plus the changes below. It is the default branch, and we install from it.
- `main`: an untouched copy of upstream `main` from fork time (2026-10-27). It has not been reviewed, so never
  install from it.

## What we changed

The base is v0.31.4, which a static audit on 2026-10-27 found to be clean: no telemetry, no credential reads,
and no network or subprocess calls in the status line.

- Removed the "🆕 vX available" hint from the status line. It came from the menu bar app's upstream update
  check, which it wrote to `~/.claude/usage-preferences.json`. `_read_update_hint` is left in place but
  unused, to keep merges small.
- `git_branch()` now reads `.git` and `HEAD` only if they are regular files, and at most 4 KiB. Before this, a
  FIFO or a symlink to `/dev/zero` in the working directory could hang the render or exhaust its memory.
- Added tests for both changes in `tests/test_usage_statusline.py`.

The status line has to stay stdlib-only and run under macOS `/usr/bin/python3` 3.9 (see `CLAUDE.md`).

## Installing the status line

The installer is `addons/usage-statusline/install.sh` in
[claude-code-tools](https://github.com/ksreenivasan/claude-code-tools). It downloads `usage_statusline.py`
from this repo at a pinned commit, checks its sha256, copies it to `~/.claude/usage-statusline.py`, and points
`statusLine` at it. To install by hand instead:

```bash
install -m 0755 usage_statusline.py ~/.claude/usage-statusline.py
```

Don't install upstream's menu bar app, and don't run upstream `usage-cli`, on a host that uses this status line.
Each run of either repeats setup if `statusLine` is missing. If the installed script's version differs from its
own, it silently overwrites `~/.claude/usage-statusline.py` with upstream's copy.

## Pulling changes from upstream

Upstream releases often, several times a day at times. Pull only what you have reviewed.

```bash
git fetch upstream --tags
git diff v0.31.4 vX.Y.Z -- usage_statusline.py usage_hooks/usage_statusline.py   # what we run
git log --oneline v0.31.4..vX.Y.Z                                                # everything else
```

1. Review the diff of the status line script. From v0.32 on it lives at `usage_hooks/usage_statusline.py`.
   Look for new imports, network or subprocess use, new files read or written, and anything that changes the
   `~/.claude/usage-status.json` schema, which `claude-usage-watch` reads.
2. Merge on a review branch: `git switch -c review/vX.Y.Z ks-main && git merge vX.Y.Z`. Keep our changes when
   resolving conflicts, and keep the update hint out.
3. Run `uvx --python 3.13 --with pytest pytest -q -p no:cacheprovider tests/test_usage_statusline.py`. Then
   check the script still compiles under `/usr/bin/python3`.
4. Merge the review branch into `ks-main` and push.
5. Bump the pinned commit, sha256 and (after v0.32) script path in claude-code-tools' `addons/usage-statusline/install.sh`, then rerun it
   on each host.

Never take a release wholesale through Homebrew or PyPI. The tap's cask token `usage` now also collides with
an unrelated Homebrew cask, `usage-app`.

## License

AGPL-3.0-only, unchanged. Our changes are under the same license, and their source is this public repository.
