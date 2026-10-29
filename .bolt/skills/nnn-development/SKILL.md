---
name: nnn-development
description: Conventions for working on the nnn terminal file manager codebase — C source in src/, shell plugins in plugins/, the user patch framework in patches/, and build options in the Makefile. Use whenever the user asks to fix a bug, add a feature or key binding, write or modify an nnn plugin, add a build option, touch a user patch, or update nnn docs, even if they don't mention conventions or name this skill.
---

# nnn Development

nnn is a small, fast, dependency-light terminal file manager written in C, extended through shell plugins. Follow these conventions so changes look like they belong upstream and pass CI (patch sanity check + shellcheck).

## Design first

nnn values minimal footprint, speed, and few dependencies. Before adding a feature to the C core, ask whether it belongs in a plugin instead — anything subjective, OS/DE-specific, or dependent on an external tool usually does. Subjective core changes that upstream rejected live in `patches/`, not mainline.

## C source (`src/`)

- Follow the Linux kernel coding style: TAB indentation, K&R braces, short lowercase names, no C++-style abstractions. Match the surrounding code in `src/nnn.c` rather than imposing a different style.
- Keep it portable (Linux, macOS, BSDs, Haiku, musl). Guard platform-specific code with the existing `#ifdef` patterns instead of assuming glibc.
- Optional features are compile-time toggles. Add a new `O_<NAME> := 0` variable in the Makefile with a trailing comment, wire it to a `-D` flag, and wrap the code in `#ifdef`/`#ifndef`, mirroring options like `O_NOMOUSE` or `O_NOFIFO`. This keeps the default binary lean.
- Avoid heap allocations and extra syscalls on hot paths (directory load, redraw, filtering); nnn's speed is a core feature.
- New keys and actions go through the existing key/action tables in `src/nnn.h`. Update the help text in `src/nnn.c` and the man page `nnn.1` together, because users discover features through both.
- Any change to `src/nnn.c` can break the user patches. Run `make checkpatches` after editing; if a patch no longer applies, regenerate its `mainline.diff` as described in `patches/README.md`.

## Plugins (`plugins/`)

- Prefer POSIX `sh`; use bash only when it clearly simplifies things, and say so in the header. Indent with 4 spaces (not TABs — that's the C convention).
- Start every plugin with an informative header comment, because it is the plugin's only documentation:

  ```sh
  #!/usr/bin/env sh

  # Description: One-line summary of what it does
  #
  # Dependencies: fzf, xsel
  #
  # Shell: POSIX compliant
  # Author: Name
  ```

- Use the inputs nnn provides instead of re-deriving them: `$1` hovered file, `$2` working dir, `$3` picker output, `$NNN_PIPE`, `$NNN_INCLUDE_HIDDEN`, `$NNN_PREFER_SELECTION`, and the selection file at `${NNN_SEL:-${XDG_CONFIG_HOME:-$HOME/.config}/nnn/.selection}`.
- To change directory or list files, source `.nnn-plugin-helper` and call `nnn_cd`, or write `<ctx><opcode><data>` (no trailing newline) to `$NNN_PIPE`.
- Store persistent data in `${XDG_CACHE_HOME:-$HOME/.cache}/nnn` and document it in the header.
- Keep non-portable commands (e.g. `notify-send`) commented out so users on other systems aren't surprised.
- Background any `$NNN_FIFO` reader so it never blocks nnn.
- Make the file executable, add it to the alphabetical table in `plugins/README.md`, and run `make shellcheck`.

## Before finishing

1. `make` builds cleanly (also try `make O_DEBUG=1` for C changes).
2. `make checkpatches` passes if `src/` changed.
3. `make shellcheck` passes if `plugins/` changed.
4. Docs updated: `nnn.1`, in-program help, `plugins/README.md`, `CHANGELOG` as relevant.

## Example

Request: "Add a way to copy the hovered file's path to the clipboard."

Good: a small POSIX `sh` plugin with a proper header, reading `$2/$1`, using a clipboard tool listed under Dependencies, added to the plugins table, shellcheck-clean.

Poor: new C code in `src/nnn.c` that shells out to `xclip`, adding an X11 dependency to the core for something a plugin handles fine.
