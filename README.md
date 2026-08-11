rm-safe
======

[![CI](https://github.com/pmarreck/rm_safe/actions/workflows/ci.yml/badge.svg?branch=yolo)](https://github.com/pmarreck/rm_safe/actions/workflows/ci.yml)
[![Garnix](https://img.shields.io/endpoint.svg?url=https%3A%2F%2Fgarnix.io%2Fapi%2Fbadges%2Fpmarreck%2Frm_safe%3Fbranch%3Dyolo)](https://garnix.io/repo/pmarreck/rm_safe)

Safe `rm`, `rmdir`, and `find -delete` shadows that move files to Trash instead
of permanently deleting them. Includes small shims and guidance for macOS,
common Linux sudoers setups, and NixOS.

Features
- Moves files/directories to OS trash (XDG trash on Linux, Finder Trash on macOS).
- Moves verified-empty directories invoked through `rmdir` to Trash too.
- Converts a restricted terminal `find -delete` expression into safe Trash moves.
- Refuses to touch protected locations (`/`, `/etc`, `/nix/store`, etc.).
- Works with sudo via a wrapper, not shell aliases.
- Restores manual trash entries with `--undo` or `--undo-picker` (gum or fzf).
- Built-in test suite: `bin/rm-safe --test`.

## Quick install (no dependencies)

rm-safe runs on a stock macOS or Linux box with nothing to install:

```sh
git clone https://github.com/pmarreck/rm-safe.git rm-safe
# put the shim + universal bash implementation on your PATH
ln -s "$PWD/rm-safe/bin/rm"           ~/.local/bin/rm
ln -s "$PWD/rm-safe/bin/rmdir"        ~/.local/bin/rmdir
ln -s "$PWD/rm-safe/bin/find"         ~/.local/bin/find
ln -s "$PWD/rm-safe/bin/rm-safe.bash" ~/.local/bin/rm-safe.bash
```

That's the whole install. `rm` moves files to Trash, and `rmdir` moves empty
directories to Trash. Ordinary `find` searches pass through unchanged; the
accepted `find -delete` form moves matches to Trash. All work on the bash that
ships with macOS (3.2) and on Linux.

The `rmdir` shadow retains the non-recursive boundary: it rejects non-empty
directories, `--parents`/`-p`, and other rmdir-specific options it cannot carry
through a single safe trash operation. `-v`/`--verbose` and `--` are supported.

The `find` shadow accepts exactly one terminal `-delete` action with ordinary
implicit-AND predicates, for example `find . -type f -mtime +30 -delete`. It
rejects Boolean operators, `-prune`, duplicate deletion actions, and any
non-terminal `-delete` because replacing a Boolean deletion predicate could
change the result of the original expression. It traverses depth-first, routes
files and symlinks to `rm-safe`, and routes real directories to the `rmdir`
shadow so unmatched children cannot be moved recursively. If it cannot
unambiguously parse a `-delete` expression, it refuses to run it rather than
guess whether `-delete` was an action or an argument.

### Optional upgrades

- **Speed:** install **luajit** and the shim automatically uses the faster
  `bin/rm-safe`. Any luajit will do — no LuaFileSystem or other Lua rocks
  needed, since `bin/rm-safe` reaches the filesystem through its own FFI shim.
  Without luajit you'll see a one-time note (silence it with `RM_SAFE_QUIET=1`).
- **Nix:** `nix develop` for a managed dev/test environment, or
  `nix build .#rm-safe` / `.#rm-safe-bash` for packaged builds.

### Choosing an implementation

Set `RM_SAFE_BIN=/path/to/rm-safe.bash` (or the luajit `bin/rm-safe`) to force a
specific implementation for `rm`, `rmdir`, and the tests.

Files
- `bin/rm-safe.bash` — universal bash implementation; works on macOS bash 3.2 and Linux with no extra dependencies.
- `bin/rm-safe` — faster LuaJIT implementation; used automatically when any luajit is available (no Lua rocks required).
- `bin/rm` — wrapper shim: dispatches `RM_SAFE_BIN` override → luajit → bash fallback; if neither safe implementation is available, it refuses the operation rather than call system rm.
- `bin/rmdir` — wrapper shim: verifies directories are empty, then dispatches to `rm-safe -r`; it refuses a system-rmdir fallback so a partial install cannot delete permanently.
- `bin/find` — wrapper shim: passes ordinary searches to real find; rewrites the accepted terminal `-delete` form into NUL-delimited safe Trash moves and reports any failed deletion.
- `bin/test/rm-safe_test` — undo/restore tests.
- `bin/test/rm_override_test` — shim tests.
- `bin/test/fs_shim_test` — proves `bin/rm-safe` needs no LuaFileSystem, and diffs its FFI filesystem shim against real lfs.
- `bin/test/run-all` — runs the full test suite across luajit / bash-4+ / bash-3.2 lanes.

Platform notes
Why sudo setup differs by OS:
- macOS `sudo` preserves the user `PATH` by default, so a user-level shim earlier in `PATH` (e.g., `~/bin/rm`) will still be found under sudo.
- Most Linux distros set `secure_path` in sudoers, which **replaces** the user `PATH` with a fixed, root-owned path for safety. That means user-level shims are ignored unless you place a root-owned shim in the secure path.

macOS (defaults keep user PATH under sudo)
- Symlink or copy `bin/rm`, `bin/rmdir`, and `bin/find` into `~/bin` (or any user directory already ahead of `/bin` on PATH).
- `sudo rm …`, `sudo rmdir …`, and `sudo find … -delete` will hit the shims because PATH is preserved by default.

Linux with sudo `secure_path`
- Create a root-owned override dir, e.g. `/usr/bin/overrides` (0755, root:root).
- Install the shims there as `rm`, `rmdir`, and `find`, and ensure `rm-safe` is reachable (the same dir is simplest).
- In sudoers (via `visudo`), prepend the override dir:
  ```
  Defaults secure_path="/usr/bin/overrides:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
  ```
- Avoid adding broad dirs like `/usr/local/bin` just to get the shim; overrides keeps blast-radius small.

NixOS (declarative)
- Build/install `rm-safe` and the `rm`/`rmdir`/`find` shims into a root-owned path on PATH, e.g. `/run/wrappers/bin`.
- Example snippet:
  ```
  # configuration.nix
  {
    security.sudo.extraConfig = ''
      Defaults secure_path=/run/wrappers/bin:/usr/bin:/bin
    '';
    environment.systemPackages = [ rm-safe-package ];
    environment.pathsToLink = [ "/run/wrappers/bin" ];
  }
  ```
- Ensure `rm` wrapper in that path execs `rm-safe`; shim already uses `command -p rm` to reach the real rm when needed.

Claude Code / AI agent integration
- Tools like `nix develop` prepend `/nix/store/.../bin` to `PATH`, shadowing user-level `rm` shims with GNU `rm`. AI coding agents (Claude Code, Codex, etc.) run shell commands through these environments and can permanently delete files without realizing the safe wrapper is bypassed.
- To prevent this in Claude Code, add a global `PreToolUse` hook that blocks bare `rm` in Bash calls:

  **1. Create `~/.claude/hooks/block-rm/block-rm.sh`:**
  ```bash
  #!/bin/bash
  set -u
  COMMAND=$(jq -r '.tool_input.command')
  if echo "$COMMAND" | grep -qE '\brm\b' && ! echo "$COMMAND" | grep -qE 'rm-safe'; then
    echo "BLOCKED: Use rm-safe instead of rm." >&2
    exit 2
  fi
  exit 0
  ```
  ```
  chmod +x ~/.claude/hooks/block-rm/block-rm.sh
  ```

  **2. Add to `~/.claude/settings.json` under `hooks.PreToolUse`:**
  ```json
  {
    "matcher": "Bash",
    "hooks": [
      {
        "type": "command",
        "command": "~/.claude/hooks/block-rm/block-rm.sh",
        "timeout": 2
      }
    ]
  }
  ```

  Any Bash tool call containing `rm` (but not `rm-safe`) will be rejected before execution. The agent sees the block message and retries with `rm-safe`.

Cautions
- Log-rotate/cleanup scripts that expect permanent deletion will move files to trash when the shim is first in PATH; they should call `/bin/rm` explicitly if deletion is intended.
- The `rm` and `rmdir` shadows fail closed if no safe implementation is available. Repair the installation rather than expecting a system-command fallback.
- The `rmdir` shadow checks emptiness before moving a directory. Coordinate concurrent writers around that check; a filesystem cannot atomically test that a directory is empty and rename it into Trash.
- The `find` shadow rejects `-delete` expressions outside its documented subset before real find runs. A concurrent writer can still change a selected path between traversal and its safe move.

Testing
- Run all lanes (luajit / bash-4+ / bash-3.2): `bin/test/run-all`
- Shim: `bin/test/rm_override_test`
- Main tool: `bin/rm-safe --test`
- Undo/restore: `bin/test/rm-safe_test`

License
- MIT (see LICENSE)
