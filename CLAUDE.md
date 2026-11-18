# CLAUDE.md

Legacy project (last touched 2021). `zfsync` is a POSIX `/bin/sh` wrapper that replicates ZFS snapshots to a remote host over SSH, using `zfs send | ssh zfs receive`. Usage and remote-host setup are in [README.md](README.md).

## Layout

- `zfsync`: entry point. Handles option parsing (`getopts`), sanity checks, and the main per-filesystem sync loop.
- `lib/common`: usage text, logging (`zf_msg`, `zf_debug`, `zf_warn`, `zf_error`, where `zf_error` exits 1), and `zf_has_binary`.
- `lib/zfs`: all ZFS logic: listing, permission checks (`zfs allow` parsing), snapshot matching, `zf_zfs_sync` (the send/receive pipe), and retention.
- `lib/zpool`: pool feature detection (used for `large_blocks`, which turns on `send -L`).
- `lib/ssh`: `zf_ssh_cmd` wraps every remote command in `sh -c "export PATH=...; <cmd>"`.
- `lib/mbuffer`: builds an optional `mbuffer` pipe stage.
- `zfsync.conf`: sample config, sourced with `-f`.

Each lib file is wrapped in an include guard (`LIB_*_INCLUDED`). Functions use the `zf_<module>_` prefix and config/state globals use `ZF_`.

## Running / developing

- There is no build step, no test suite, and no CI. Libraries are found at `./lib` when `./lib/common` exists, otherwise at `/usr/local/lib/zfsync`. To use the in-tree libs, run `./zfsync` from the repo root.
- You need a real ZFS source host and an SSH-reachable ZFS target to exercise it, so on macOS you can't run it end to end. Use `sh -n zfsync lib/*` for a syntax check. If `shellcheck -s sh` is available, run it too, though the existing code has many pre-existing warnings.
- Use `-d` for debug output, which logs every local and remote command before it runs.

## Key mechanics

- **Local vs. remote execution**: `zf_zfs_cmd "<cmd>" [1]` runs locally through `eval`, or remotely through SSH when the second argument is `1`. It adds the `zfs` prefix if missing. Remote `zfs receive` commands automatically get an `mbuffer` stage when `ZF_MBUFFER_FORCE_REMOTE` is set. Commands are built as strings, so pipes and redirects inside them (`>/dev/null 2>&1`, `| grep`) are interpreted by `eval` locally or by `sh -c` remotely.
- **Recursion**: instead of using replication streams (`send -R`), the script iterates `zfs list -r` and syncs each filesystem on its own, mapping each source child path onto the target by stripping the source prefix. In recursive mode, per-filesystem failures are warnings (`continue`). In single-filesystem mode they are fatal.
- **Sync algorithm**: the script finds common snapshots by name (`zf_zfs_matching_snapshots` writes two temp files and runs `grep -wFf`).
  - With no match, it sends the oldest local snapshot as a full stream, then an incremental from there.
  - With a match, it sends an incremental (`-I` by default, `-i` with the `-i` flag) from the latest match to the newest local snapshot.
  - Receive always uses `-F`.
  - It aborts if the target has snapshots but none match the source.
- **Permissions**: before sending, it checks the `zfs allow` output with a regex. It needs local `hold,send`, remote `receive,create,mount`, `userprop` for `-p`, and `destroy` for retention. `root` is assumed to be allowed. Group permissions are not checked.
- **Retention** (`-k`/`-K`): retention is either a count or an age (`h`/`d`/`w`/`m`). Oldest snapshots are destroyed first, and it never destroys the last common snapshot or anything newer.

## Gotchas / known quirks

- **Portability**: the target is FreeBSD/illumos `/bin/sh`. Keep code POSIX-ish, with no bashisms. `local` is used and works in FreeBSD sh and dash, but breaks on ksh-as-sh (noted in the README). Retention uses BSD `date -v` and `date -r`, so it won't work with GNU date or on Linux. ZFS on Linux is untested.
- **Config precedence**: the config file passed with `-f` is sourced *after* `getopts`, so config values override CLI flags.
- **Usage text drift**: the help text in `lib/common` omits `-B` (no mbuffer) and `-f` (config file). The README lists `-B` but not `-f`. Update both places when changing options.
- **Global leakage**: several functions assign non-`local` variables (`cmd`, `matches`, `fses`, `regex`, `tmp_*`). Be careful when reusing those names in callers, for example the main loop uses `matches`.
- **Temp files**: `zf_zfs_matching_snapshots` resets the `EXIT` trap on each call, so only the last call's temp files are guaranteed to be cleaned up.
- **`set -e`**: it is on globally. Commands whose failure is expected need `|| true` or must be used in a conditional.
- The large-blocks check passes the *target* pool name to `zf_zpool_has_feature` with remote=1. It derives the pool from `-T` by taking everything before the first `/`.
