---
date: 2026-03-02T11:32:24Z
git_commit: 9ac92f185fc71d578b080b685482a2c74abf41e8
branch: master
repository: dotfiles
topic: "How is ctrl+(h/j/k/l) window navigation configured for kitty and neovim?"
tags: [research, codebase, kitty, neovim, smart-splits, window-navigation, keymaps]
status: complete
last_updated: 2026-03-03
last_updated_by: claude
last_updated_note: "Added follow-up investigation — Claude Code navigation bug root cause and fix"
---

# Research: Ctrl+H/J/K/L Window Navigation (Kitty + Neovim)

**Date**: 2026-03-02T11:32:24Z
**Git Commit**: 9ac92f185fc71d578b080b685482a2c74abf41e8
**Branch**: master
**Repository**: dotfiles

## Research Question

How is ctrl+(h/j/k/l) window navigation configured for kitty and neovim in my dotfiles?

## Summary

Ctrl+H/J/K/L provides seamless window navigation across both kitty terminal
splits and neovim splits, powered by the `smart-splits.nvim` plugin. The system
works through a two-layer architecture: kitty intercepts the keypress first and
uses a `pass_keys.py` kitten to decide whether to move between kitty windows or
forward the key to neovim. Inside neovim, `smart-splits.nvim` handles the
cursor movement across vim splits. A fallback path in `mappings.lua` provides
basic `<C-w>h/j/k/l` navigation when smart-splits is not loaded.

## Detailed Findings

### Layer 1: Kitty Terminal — `pass_keys.py` Kitten

**File**: `kitty/kitty.conf` (lines 173-188)

The kitty config defines a `smart-splits.nvim` integration block:

```conf
# smart-splits.nvim {{{
# See: https://github.com/mrjones2014/smart-splits.nvim#credits
map ctrl+j kitten pass_keys.py neighboring_window bottom ctrl+j
map ctrl+k kitten pass_keys.py neighboring_window top    ctrl+k
map ctrl+h kitten pass_keys.py neighboring_window left   ctrl+h
map ctrl+l kitten pass_keys.py neighboring_window right  ctrl+l

# the 3 here is the resize amount, adjust as needed
map alt+j kitten pass_keys.py relative_resize down  3 alt+j
map alt+k kitten pass_keys.py relative_resize up    3 alt+k
map alt+h kitten pass_keys.py relative_resize left  3 alt+h
map alt+l kitten pass_keys.py relative_resize right 3 alt+l

allow_remote_control yes
listen_on unix:/tmp/mykitty
# }}}
```

**How it works:**

1. When Ctrl+H/J/K/L is pressed in kitty, the `pass_keys.py` kitten intercepts
   it.
2. The kitten checks whether the active kitty window is running a process that
   handles these keys (e.g., neovim).
3. If neovim is in the foreground, the keypress is passed through to neovim.
4. If neovim is NOT in the foreground (e.g., a plain shell), kitty performs its
   own `neighboring_window` action to move focus between kitty splits.

**Required kitty settings** (also in this block):
- `allow_remote_control yes` — enables the kitten to query kitty's state
- `listen_on unix:/tmp/mykitty` — the socket smart-splits uses to communicate
  with kitty

**The `pass_keys.py` kitten itself** is NOT stored in this dotfiles repo. It is
installed by the smart-splits.nvim plugin's build step
(`./kitty/install-kittens.bash`), which copies it into the kitty config
directory.

### Layer 2: Neovim — `smart-splits.nvim` Plugin

**File**: `nvim/lua/jasonr/plugins/smart-splits/init.lua`

The plugin is loaded via lazy.nvim with:
- **Source**: `mrjones2014/smart-splits.nvim`
- **Version**: `~1.4.0` (pinned, with a note about issue #209)
- **Build step**: `./kitty/install-kittens.bash` (installs the kitty kitten)

Keymaps defined in the `init` function (runs before plugin loads):

| Keymap | Action |
|---|---|
| `<C-h>` | `smart-splits.move_cursor_left` |
| `<C-j>` | `smart-splits.move_cursor_down` |
| `<C-k>` | `smart-splits.move_cursor_up` |
| `<C-l>` | `smart-splits.move_cursor_right` |
| `<A-h>` | `smart-splits.resize_left` |
| `<A-j>` | `smart-splits.resize_down` |
| `<A-k>` | `smart-splits.resize_up` |
| `<A-l>` | `smart-splits.resize_right` |
| `<leader><leader>h` | `smart-splits.swap_buf_left` |
| `<leader><leader>j` | `smart-splits.swap_buf_down` |
| `<leader><leader>k` | `smart-splits.swap_buf_up` |
| `<leader><leader>l` | `smart-splits.swap_buf_right` |

The plugin uses `opts = {}` (default configuration), which means smart-splits
auto-detects the kitty multiplexer backend via the `listen_on` socket.

### Layer 3: Fallback Keymaps (when smart-splits is absent)

**File**: `nvim/lua/jasonr/mappings.lua` (lines 93-100)

```lua
-- Window navigation {{{
if vim.fn.exists(':SmartSplitsLog') == 0 then
  map('n', '<C-h>', '<C-w>h')
  map('n', '<C-j>', '<C-w>j')
  map('n', '<C-k>', '<C-w>k')
  map('n', '<C-l>', '<C-w>l')
end
-- }}}
```

This guard checks if the `:SmartSplitsLog` command exists (indicating
smart-splits is loaded). If it is NOT loaded, these fallback keymaps provide
basic neovim-native window navigation using `<C-w>h/j/k/l`. This ensures
Ctrl+H/J/K/L always works for window navigation even without the plugin.

### Supporting Configuration: Zsh Terminal Flow Control

**File**: `.zshrc-common` (lines 103-105)

```zsh
# Disable terminal flow control commands
# This allows for mapping <C-j> and whatnot in Command-Line mode in Vim
stty start undef stop undef
```

This disables `XON/XOFF` terminal flow control so that `Ctrl+J` (and others)
are not consumed by the terminal before reaching vim/neovim.

### Legacy Vim Configuration

**File**: `vim/.vimrc` (lines 234-237)

```vim
nnoremap <C-h> <C-w>h
nnoremap <C-j> <C-w>j
nnoremap <C-k> <C-w>k
nnoremap <C-l> <C-w>l
```

The old vimrc has simple `<C-w>` remaps without any multiplexer integration.

## Code References

- `kitty/kitty.conf:173-188` — Kitty-side smart-splits integration block
- `nvim/lua/jasonr/plugins/smart-splits/init.lua:1-28` — Smart-splits plugin spec + keymaps
- `nvim/lua/jasonr/mappings.lua:93-100` — Fallback window navigation keymaps
- `.zshrc-common:103-105` — Terminal flow control disabled for Ctrl key passthrough
- `vim/.vimrc:234-237` — Legacy vim window navigation

## Architecture Documentation

### Request Flow for Ctrl+H/J/K/L

```
User presses Ctrl+H
        │
        ▼
┌─────────────────────┐
│   Kitty Terminal    │
│   kitty.conf:175    │
│   pass_keys.py      │
│   kitten intercepts │
└─────────┬───────────┘
          │
    ┌─────┴──────┐
    │            │
    ▼            ▼
┌────────┐  ┌──────────────┐
│ Neovim │  │ Non-Neovim   │
│ active │  │ (shell, etc) │
└───┬────┘  └──────┬───────┘
    │              │
    ▼              ▼
┌────────────┐  ┌───────────────────┐
│smart-splits│  │kitty              │
│move_cursor │  │neighboring_window │
│ _left()    │  │ left              │
└────────────┘  └───────────────────┘
```

### Key Design Decisions

1. **Two-layer interception**: Kitty handles the "which application?" routing;
   neovim handles the "which vim split?" movement. This allows seamless
   navigation across both kitty and neovim splits.

2. **Graceful degradation**: The `mappings.lua` fallback ensures Ctrl+H/J/K/L
   works even if smart-splits fails to load, using native `<C-w>` commands.

3. **Build-time kitten installation**: The `pass_keys.py` kitten is installed by
   the plugin's build script rather than being committed to the dotfiles repo,
   keeping the external dependency managed by the plugin itself.

4. **Version pinning**: smart-splits is pinned to `~1.4.0` due to a known
   issue (#209).

## Open Questions

- The `pass_keys.py` kitten is installed by `./kitty/install-kittens.bash` from
  the smart-splits.nvim plugin repo. The exact installation target directory is
  determined by that script (likely `~/.config/kitty/`).

## Follow-up Research 2026-03-03T07:12:12Z

### Bug: Ctrl+H/J/K/L navigation broken in Claude Code windows

**Symptom**: Pressing Ctrl+H/J/K/L in a kitty window running Claude Code did not
move focus to an adjacent kitty window. Claude Code consumed the keys instead.

**Diagnostic**: Added temporary logging to `~/.config/kitty/pass_keys.py` to
record `foreground_processes` and `is_window_vim` result on each keypress.

**Root cause**: `is_window_vim` was returning `True` for the Claude Code window
due to Mason-installed LSP servers appearing in the window's foreground process
group. The process list included:

```
/Users/jasonr/.local/share/nvim/mason/packages/lua-language-server/libexec/bin/lua-language-server
```

The regex `re.search('nvim', path, re.I)` matched `/nvim/` as a **directory
component** in the Mason installation path — not the nvim binary. Because
`is_window_vim` returned `True`, pass_keys.py forwarded the keys to Claude Code
(as if it were a neovim window) rather than calling `neighboring_window`.

This only manifests on machines where Mason-installed packages (lua-language-server,
or any other package under `~/.local/share/nvim/mason/`) appear in the foreground
process group of the Claude Code window.

**Fix applied** (`kitty/kitty.conf`): Changed vim_id argument from `nvim` to
`(^|/)n?vim$` in all 8 `pass_keys.py` kitten calls:

```conf
# Before
map ctrl+h kitten pass_keys.py neighboring_window left ctrl+h nvim

# After
map ctrl+h kitten pass_keys.py neighboring_window left ctrl+h (^|/)n?vim$
```

The anchored regex `(^|/)n?vim$` requires `nvim` or `vim` to appear as the
**terminal path segment** (the binary name), not a directory component:

| Path | `nvim` | `(^|/)n?vim$` |
|---|---|---|
| `/usr/local/bin/nvim` | match ✓ | match ✓ |
| `/path/to/nvim/mason/lua-ls` | **match (wrong!)** | no match ✓ |
| `nvim` | match ✓ | match ✓ |

**Also fixed** (`agent-config/claude/keybindings.json`): Replaced `ctrl+j`/`ctrl+k`
in the `Autocomplete` context with `ctrl+n`/`ctrl+p` (consistent with the
`Settings` and `Select` contexts).
