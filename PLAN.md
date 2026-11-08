# Plan

## Active

- [x] Make concurrent trashing collision-free (inbox 2026-11-08, dotconfig): a
      stress test reproduced silent clobbering (31/32 payloads) and cross-device
      failures; replaced probe-then-move with O_EXCL .trashinfo reservation +
      no-replace rename + private cross-device staging; PID added to log
      fragment names. Mutation-checked. (2026-11-08 EST)
- [ ] Apply the same no-replace move to `--undo` restore (it still checks the
      original path, then moves, and could clobber a concurrently created file).

Goal: protect ordinary `rmdir` invocations with an installable shadow that moves
empty directories to Trash, while retaining `rmdir`'s non-recursive safety
contract. Done when the package installs the shadow, documentation tells users
to put it on `PATH`, and an integration test proves it intercepts an empty
directory without accepting a non-empty one.

- [x] Diagnose and stop the reported burst of `find` processes. `bin/find`
      selected `~/bin/find` (another command wrapper), which selected this
      shadow again; a re-entry marker, direct real-binary resolution, and an
      `rmdir` resolver repair break the loop. (2026-08-12 EDT)

- [x] Add a RED integration case for a shadowed `rmdir`; it proves the empty
      Trash move and non-empty rejection. (2026-08-11 13:06 EDT)
- [x] Implement and package the `rmdir` shadow with fail-closed safe-implementation
      selection; unsupported rmdir-specific options cannot broaden the operation.
      (2026-08-11 13:06 EDT)
- [x] Update installation and file documentation, annotate new paths, and run
      the full local matrix plus sealed Nix check and package smoke.
      (2026-08-11 13:08 EDT)
- [x] Audit remaining destructive command surfaces. Priority paths are `unlink`,
      `find -delete`, the current `rm` system fallback, `shred`/`gshred`,
      `truncate`, `dd`, `rsync --delete`, and Git cleanup commands. Shadows
      cannot cover shell redirections, absolute paths, interpreter APIs, or
      direct syscalls. (2026-08-11 13:10 EDT)
- [x] Select the protection boundary with Peter: humans get command shadows;
      `rm` fails closed; `find` emulates only one terminal `-delete` with
      simple implicit-AND predicates; interpreter and syscall paths remain
      outside this layer. (2026-08-11 15:20 EDT)
- [x] Scope the next safeguards with Peter: leave `shred`/`gshred`,
      `rsync --delete`, and Git cleanup out of scope; treat human command-line
      protection as a first-class goal rather than relying only on agent gates.
      (2026-08-11 13:43 EDT)
- [x] Specify and reproduce `find -delete` semantics: one terminal action plus
      implicit-AND predicates only, then a null-delimited depth-first stream
      consumed by the appropriate safe shadow. (2026-08-11 15:28 EDT)
- [x] Add a RED fail-closed `rm` case, then remove its system-rm fallback. A
      copied shim without either safe implementation leaves its target untouched
      even when `REAL_RM` is configured. (2026-08-11 15:21 EDT)
- [x] Implement the approved `find -delete` subset with a NUL-delimited sink
      that dispatches files/symlinks to `rm-safe` and directories to `rmdir`.
      Reject Boolean expressions and duplicate or non-terminal `-delete`
      actions before invoking the real `find`; distinguish literal predicate
      arguments from the deletion action. (2026-08-11 15:33 EDT)
- [x] Add a RED `truncate` guard, then require `--confirm` for shrinkage.
      The forecast reports current/requested/discarded bytes for absolute,
      relative, reference, rounding, and I/O-block forms; unproved expressions
      fail closed until confirmed. (2026-08-11 16:05 EDT)
- [x] Add a RED `dd` guard, then require `--confirm` when `of=` names an
      existing regular, special, or directory/mount target. Report target type,
      resolved path, current regular-file size, transfer operands, and only
      honest bounds. (2026-08-11 16:04 EDT)
- [ ] Decide whether `unlink` should join the safe-trash surface.
- [x] Commit the verified rmdir unit after the audit decision. (`53de1ef`,
      2026-08-11 15:21 EDT)
- [x] Commit the verified fail-closed `rm` and `find -delete` unit.
      (2026-08-11 15:33 EDT)
- [x] Commit the verified `truncate --confirm` unit. (2026-08-11 15:52 EDT)
- [x] Commit the verified `dd --confirm` unit. (2026-08-11 16:04 EDT)
- [x] Commit the verified `find` recursion incident repair.
      (2026-08-12 11:27 EDT)

Found in the full gate: an inherited Bash `find` function could hide log
fragments after `HOME` isolation, breaking Bash `--undo`. Executable-path
resolvers and hostile-function regressions now keep that ambient shell state
out of the Bash implementation and `rmdir` shadow. (2026-08-11 13:06 EDT)

Goal (done): remove the `luafilesystem` (lfs) dependency from `bin/rm-safe` so
the LuaJIT implementation runs under ANY luajit — bare or stocked. It used to
die with `module 'lfs' not found` inside any flake dev shell that puts a bare
luajit first on PATH, i.e. inside project directories, which is where agents do
most of their work.

## Consumed lfs API surface (verified against 45d28f9)

| lfs call | sites | replacement |
|---|---|---|
| `attributes(p)` / `attributes(p,"mode")` | 3 | `access(2)` |
| `symlinkattributes(p)` | 3 | `readlink(2)` + `access(2)` |
| `mkdir(p)` | 1 | `mkdir(2)` |
| `currentdir()` | 2 | `getcwd(3)` |
| `dir(p)` | 1 | `opendir`/`readdir`/`closedir` |

`lfs.dir` was NOT in the kickoff brief — it arrived in `1167cc2`
(fragment-per-action log). The working copy was on a detached HEAD 3 commits
behind, so building on that base would have missed it entirely.

Key design call: only three distinctions are ever consumed — *exists?*,
*is-directory?*, *is-symlink?*. None require reading `struct stat`, so the shim
uses `access`/`readlink` and the `struct stat` layout minefield (Linux x86_64 vs
aarch64 vs Darwin, `stat64`/`__xstat`/`$INODE64` symbol variants) is avoided
entirely. `dirent` is the one platform-divergent struct left, and the test pins
it: a wrong `d_name` offset cannot produce the "." and ".." every directory has.

- [x] Reconcile repo state: detached HEAD `fab9134`, 3 commits behind; fast-forwarded `yolo` to `45d28f9` (2026-07-24 21:44 EST)
- [x] Witness RED under both a poisoned-`require` harness and a genuinely bare luajit (2026-07-24 21:47 EST)
- [x] `bin/test/fs_shim_test`: static + differential-vs-lfs + end-to-end no-lfs controls (2026-07-24 22:00 EST)
- [x] FFI shim implemented inline in `bin/rm-safe`; all 10 call sites rewritten (2026-07-24 21:55 EST)
- [x] Mutation battery: 7/7 semantic corruptions killed on Linux, Darwin dirent offset killed on macOS (2026-07-24 22:10 EST)
- [x] Wire the new suite into `run-all`, `--test`, and the flake check; drop the `require("lfs")` gate that silently skipped the luajit lane (2026-07-24 22:03 EST)
- [x] macOS verification on peters-macbook-pro-m4-max (arm64): 5 lanes / 0 failed, incl. bash-3.2 lane 9/9 (2026-07-24 22:12 EST)
- [x] `flake.nix`: package uses bare `pkgs.luajit`; lfs kept ONLY as the test oracle (2026-07-24 22:02 EST)
- [x] README / CODE_MINIMAP / dirtree updated (2026-07-24 22:15 EST)

## Found and fixed en route (separate commits)

- [x] `pb_encode` fallback returned the raw path when `printable-binary` was absent, so fragment names kept their slashes, the write failed silently, and **`--undo` could never restore anything**. `printable-binary` is not a documented dependency, so this was the default experience for most installs. Both impls fixed + regression test. This was why `checks.tests` was red in the sealed sandbox. (2026-07-24 22:05 EST)
- [x] `rm_override_test` built its no-luajit PATH by dropping whole directories, which on NixOS also dropped `bash`, so the shim never ran and 2 assertions failed for unrelated reasons. Symlink-overlay instead; also stopped its cleanup trap using a bare `rm`. (2026-07-24 22:08 EST)

## Open / not addressed (deliberately out of scope)

- [ ] `sha256_hex_short` shells out to `shasum`, which is absent in the nix
      sandbox (coreutils provides `sha256sum`). Only reachable for paths whose
      encoded fragment name exceeds 200 chars, so it did not surface here — but
      the same silent-fallback shape as the pb_encode bug. Worth a look.
- [ ] `bin/rm` dispatches to the luajit impl on `command -v luajit` alone. That
      is now correct (any luajit works), but it is worth an explicit test.
