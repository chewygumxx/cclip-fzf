# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`cclip-fzf` is a single Bash script (`cclip-fzf`, no extension) that wraps
[`cclip`](https://github.com/heather7283/cclip)'s Wayland clipboard history
database in an `fzf` picker. It has no build system, package manifest, or test
suite — the entire project is that one file plus `README.md`, `LICENSE`, and
`.repo-metadata.jsonc`.

## Commands

- Lint: `shellcheck cclip-fzf` (available in this environment; there's no
  `.shellcheckrc` or CI wiring for it, so run it manually after edits).
- Run: `./cclip-fzf` (requires `cclip`, `fzf`, `wl-clipboard`, `awk`, `chafa`
  in `PATH`, and a running `cclipd` daemon with data in
  `~/.local/share/cclip/db.sqlite3` or `$CCLIP_FZF_DB_PATH`).
- There is no automated test suite; verify changes by running the picker
  interactively and exercising the keybindings described in `README.md`.

## Architecture

The script is **self-dispatching**: it is both the fzf launcher and the
callback target fzf invokes for every action. `__main` builds the `fzf`
`--bind` table with `reload(...)`, `become(...)`, and the `--preview`
argument all shelling out to `"$__this_file"` (the script's own resolved
path) with a flag:

- `--list` — emit `cclip list id,mime,preview`, tab-delimited, one entry per
  line; this is the data source for the picker and every reload.
- `--delete <ids>` — delete entries via `cclip delete -s <id>`, then fall
  through to `--list` to re-render.
- `--copy <ids>` — pipe `cclip get <id>` for each selected id into
  `wl-copy`, joining multiple entries with `CCLIP_FZF_CONCAT_DELIMITER`.
- `--previewer <line>` — parse one tab-delimited list line back into
  `id`/`mime`/`preview` and render: `chafa` for `image/*`, word-wrapped
  `awk` for `text/*`, otherwise the raw `preview` column.

Because of this dispatch model, any change to the `--list` output columns,
delimiter, or field order must be mirrored in `__previewer`'s `read -r`
line and in fzf's `--delimiter`/`--with-nth`/`--id-nth` flags in `__main` —
they all assume the same tab-separated `id, mime, preview` shape.

Other structural points:

- `__this_file="${BASH_SOURCE[0]}"` is captured once and reused for every
  self-invocation (via `@Q` quoting) so the script keeps working regardless
  of `cwd` or how it was invoked (bare name on `PATH`, relative, absolute).
- `__guard cmd ...` runs a command, and on nonzero exit logs the command,
  its stderr (captured via a single `mktemp`'d `errfile` cleaned up by an
  `EXIT` trap), and exits with the same code. Use it to wrap any command
  whose failure should abort the script loudly rather than continuing
  silently.
- `__log` writes `INFO`/`NOTICE`/`WARN`/`ERROR`/`FATAL` lines to stderr;
  `INFO` is suppressed unless `CCLIP_FZF_VERBOSE=1`. Inside the previewer
  (`CTX_PREVIEWER=1`), log lines are also `tee`'d into the preview pane
  itself, since that's the only visible surface at that point.
- The query field starts hidden (`--no-input`) so single keystrokes act as
  navigation/control keys (`j`/`k`/`q`/etc.) rather than filter text; `/`
  and `ctrl-/` toggle into search mode via fzf's `enable-search`/
  `show-input`/`unbind`/`rebind` transforms. Any new single-key keybind
  must be added to both the normal-mode `--bind` list *and* the
  `unbind(...)`/`rebind(...)` argument lists in the `/` and `esc` binds, or
  it will leak into search-mode text entry (or vice versa).
- All user-facing configuration is environment variables read once near the
  top of the file with `: "${VAR:=default}"` (see `README.md` for the
  table: `CCLIP_FZF_DB_PATH`, `CCLIP_FZF_VERBOSE`, `CCLIP_FZF_PREVIEW_WIDTH`,
  `CCLIP_FZF_CONCAT_DELIMITER`), plus the single `-d <path>` CLI flag which
  overrides `CCLIP_FZF_DB_PATH` for that invocation.

## Conventions

- File header block: every source/config/doc file starts with a `vim:` modeline
  comment, an `SPDX-License-Identifier: GPL-3.0-only` line, then a boxed
  comment giving the repo (`~chewygumxx/cclip-fzf.git`) and the file's
  repo-relative path (`::: :/path/to/file`). Match this exactly in new files.
- Functions that are internal dispatch targets/helpers are prefixed `__`
  (e.g. `__list`, `__guard`, `__previewer_image`) to distinguish them from
  `__main`, the actual entry point.
- `.repo-metadata.jsonc` is the source of truth for repo name/description/
  topics/license and is applied to the GitHub repo settings by the
  `apply-repo-metadata-jsonc` workflow on push — keep it in sync with
  `README.md` if either changes (description, topics, license).
