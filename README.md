<!-- vim:set expandtab shiftwidth=2 filetype=markdown: -->

<!--
   -
   - ~chewygumxx/cclip-fzf.git
   - ::: :/README.md
   -
   -->

# cclip-fzf

[`fzf`][fzf] front-end for [`cclip`][cclip] to further assist browse, preview, and
restoration of Wayland clipboard history.

Derived from cclip's own [example picker].

## Overview

`cclip-fzf` wraps `cclip`'s clipboard history in an `fzf` picker. Selecting one
or more entries copies them back onto the clipboard via `wl-copy`; the preview
pane renders images with `chafa` and word-wraps text with `awk`.

The script is a single self-dispatching file. With no arguments it opens the
picker, and `fzf`'s own binds call the same file back with `--list`, `--delete`,
`--copy`, or `--previewer` to do the actual work.

## Dependencies

- [`cclip`][cclip]
- [`fzf`][fzf]
- `bash`
- `wl-clipboard`
- `awk`
- `chafa`

## Installation

### Arch Linux

```bash
sudo pacman -S fzf chafa wl-clipboard gawk
yay -S cclip
```

```bash
curl "https://github.com/chewygumxx/cclip-fzf/releases/download/v1.0.0/cclip-fzf" > ~/.local/bin/cclip-fzf
chmod 755 ~/.local/bin/cclip-fzf
```

## Configuration

Everything is read from the environment, with one CLI override.

### Environment variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `CCLIP_FZF_DB_PATH` | `$XDG_DATA_HOME/cclip/db.sqlite3` (or `~/.local/share/cclip/db.sqlite3`) | Database path passed to every `cclip` call. Matches cclip's own default. |
| `CCLIP_FZF_VERBOSE` | `0` | Set to `1` to also print `INFO`-level log lines. |
| `CCLIP_FZF_PREVIEW_WIDTH` | `70` | Column width for the preview pane and the text word-wrap. |
| `CCLIP_FZF_CONCAT_DELIMITER` | newline | Inserted between entries when a multi-selection is copied at once. Set to an empty string for no separator. |

### `-d` flag

`cclip-fzf -d <path>` overrides `CCLIP_FZF_DB_PATH` for that invocation only.

## Usage

Ensure `cclipd` daemon is running before launching the picker (e.g. via
`exec-once` in Hyprland's config).

Navigation and the query field are keyboard-driven; the query field starts
hidden so single keys act as controls instead of typing into a
filter, until you explicitly open search with `/`.

### Keybindings

| Key(s) | Action |
| --- | --- |
| `p`, `Ctrl-P`, `k`, `Ctrl-K`, `Up` | move up |
| `n`, `Ctrl-N`, `j`, `Ctrl-J`, `Down` | move down |
| `Tab` / `Shift-Tab` | mark / unmark an entry for multi-select |
| `/` | open the search field (keys then type as search text instead of acting as controls) |
| `Ctrl-/` | toggle the search field |
| `Backspace`, `Del`, `Ctrl-D` | delete the selected entry/entries and reload |
| `Ctrl-R` | reload the list |
| `Enter` | copy the selected entry/entries to the clipboard and exit |
| `Esc` | close the search field if it's open, otherwise quit |
| `Ctrl-C`, `Ctrl-G`, `Ctrl-Q`, `Ctrl-W`, `q` | quit without copying |

### Hyprland example

A floating popup picker bound to a hotkey, using WezTerm as the host terminal:

```lua
hl.window_rule({
    name = "terminal-clipboard-manager-cclip-fzf",
    match = { class = "cclip-fzf", },

    float = true,
    size = { 1100, 652 },
})

hl.bind("SUPER + V",
    hl.dsp.exec_cmd("wezterm start --always-new-process --class cclip-fzf 'cclip-fzf'"))
```

`SUPER + V` spawns a new WezTerm window tagged with the `cclip-fzf` class; the
window rule matches on that class to float and size it, so the picker appears
as a small popup instead of a full-screen terminal. This relies on `cclip-fzf`
resolving by bare name, as such, ensure the install directory is on `$PATH` in
the environment Hyprland spawns processes with.

## Credits

Derived from cclip's own [example picker]

[fzf]: <https://github.com/junegunn/fzf>
[cclip]: <https://github.com/heather7283/cclip>
[example picker]: <https://github.com/heather7283/cclip/tree/master/examples>
