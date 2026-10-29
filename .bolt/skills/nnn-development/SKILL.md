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
- Optional features are compile-time toggles. Add `O_<NAME> := 0  # what it does` to the option block at the top of the Makefile, then wire it like the existing ones:

  ```make
  ifeq ($(strip $(O_NOFIFO)),1)
  	CPPFLAGS += -DNOFIFO
  endif
  ```

  Wrap the code in `#ifdef`/`#ifndef` on that macro. This keeps the default binary lean and lets packagers strip features.
- CI compiles with `-Werror` on gcc and clang and also runs `clang-tidy` and `cppcheck`, so treat every warning as a build failure. Unused variables inside an `#ifdef` branch are a common trap; check the build with the option both on and off.
- Avoid heap allocations and extra syscalls on hot paths (directory load, redraw, filtering); nnn's speed is a core feature.
- New keys need three things: a `SEL_*` value in `enum action` and a `{ key, SEL_* }` entry in `bindings[]` (both in `src/nnn.h`), plus a `case SEL_*:` in the main loop in `src/nnn.c`. Check `bindings[]` first so you don't reuse a key that's already taken. The colemak patch remaps keys, so a new binding may also need a matching line there.
- Update the help screen (`helpstr[]` in `show_help()`) and the man page `nnn.1` together, because users discover features through both. `helpstr` is encoded: the first character of each line is a **hex digit giving the number of leading spaces**, and `%NN` inside a line inserts NN (two decimal digits) spaces. Count columns against the neighbouring lines so the two-column layout stays aligned, e.g. `"9g ^A  Top%21J  Jump to entry/offset\n"` means 9 spaces, then the left column, then 21 spaces before the right column.
- Any change to `src/nnn.c` can break the user patches (`patches/*/mainline.diff`), which apply against exact surrounding lines. Avoid gratuitous edits near code the patches touch. `make checkpatches` builds every combination of patches; it runs `git restore src` between builds, so it only works inside a git checkout. Outside one, apply each patch manually with `patch --dry-run --forward --strip=1 --input=patches/<name>/mainline.diff` to check it still applies cleanly.
- When adding a new user patch, add its `O_<NAME>` to the Makefile's "User patches" block, to the prepatch/postpatch targets, and to the `patches=(...)` array in `patches/check-patches.sh` (the README calls it `misc/test/check-patches.sh`, but that path is outdated).

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
- Make the file executable, add it to the alphabetical table in `plugins/README.md`, and run `make shellcheck`. If shellcheck isn't installed, at least run `sh -n` (or `bash -n`) on the plugin and say that shellcheck wasn't run; don't claim it passed.

## Before finishing

1. `make clean && CFLAGS=-Werror make` builds without warnings. For changes behind a toggle, build with it both on and off. Run `make clean` afterwards so no `nnn` binary is left in the tree.
2. The patches still apply if `src/` changed (see above).
3. `make shellcheck` passes if `plugins/` changed.
4. Docs are updated where relevant: `nnn.1`, the help screen, `plugins/README.md`, and the shell completions in `misc/auto-completion/` for new command-line flags.
5. Leave `CHANGELOG` alone unless asked. Maintainers write its release entries (`nnn vX.Y Name`, a date, then `- ` bullets).

Report honestly which of these checks actually ran.

## Example

Request: "Add a way to copy the hovered file's path to the clipboard."

Good: a small POSIX `sh` plugin with a proper header, reading `$2/$1`, using a clipboard tool listed under Dependencies, added to the plugins table, shellcheck-clean.

Poor: new C code in `src/nnn.c` that shells out to `xclip`, adding an X11 dependency to the core for something a plugin handles fine.
