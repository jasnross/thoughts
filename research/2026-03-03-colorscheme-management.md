---
date: 2026-03-03T07:20:21+0000
git_commit: 4db70aa79da20fde869989e99ebc59f1cd955017
branch: master
repository: dotfiles
topic: "How are colorschemes managed? (kitty, neovim, light/dark mode)"
tags: [research, codebase, colorscheme, neovim, kitty, dark-mode, light-mode, launchd, heirline, gruvbox-material]
status: complete
last_updated: 2026-03-03
last_updated_by: Claude (research-codebase skill)
---

# Research: Colorscheme Management

**Date**: 2026-03-03T07:20:21+0000
**Git Commit**: 4db70aa79da20fde869989e99ebc59f1cd955017
**Branch**: master
**Repository**: dotfiles

## Research Question

How are colorschemes managed in the dotfiles? Include information on kitty, neovim, light/dark mode, etc.

## Summary

Colorscheme management spans three layers. **macOS system appearance** is the single source of truth — dark or light mode is detected by reading `AppleInterfaceStyle` via `defaults read`. **Kitty** responds to appearance changes through two parallel mechanisms: its native `*.auto.conf` feature (handles colors) and a launchd polling daemon (handles font-rendering overrides). **Neovim** has its own Lua polling module that shells out every 5 seconds to detect mode, applies the corresponding colorscheme (`gruvbox-material` for both modes currently), and runs a cascading hook system for per-theme highlight overrides. **k9s** and **Firefox** also participate via the same launchd daemon and an AppleScript app respectively.

The active dark colorscheme is `gruvbox-material` and the active light colorscheme is also `gruvbox-material` (nine other plugins are installed but commented out as alternatives).

---

## Detailed Findings

### 1. macOS System Appearance — The Source of Truth

macOS sets (or clears) the user default `AppleInterfaceStyle = "Dark"` whenever the system dark mode state changes. All components in this system read this single signal — either directly or indirectly.

**How to read it:**
```sh
defaults read -g AppleInterfaceStyle
# Returns "Dark" when in dark mode; exits with an error (key not found) in light mode
```

**Shell toggle alias** (`~/.zshrc-osx:4`):
```sh
alias dark-mode-toggle="osascript -e 'tell app \"System Events\" to tell appearance preferences to set dark mode to not dark mode'"
```
This is the manual toggle — it flips the OS appearance setting for the entire system.

---

### 2. Kitty Terminal Theme Switching

Kitty has a two-layer theme system for colors and font rendering respectively.

#### Layer A — Color Palettes via `*.auto.conf` (Native Kitty Feature)

Kitty automatically loads one of three files from the config directory based on detected OS appearance, with no `include` directive needed:

| File | When Loaded | Theme |
|------|-------------|-------|
| `kitty/dark-theme.auto.conf` | macOS dark mode | Gruvbox Material Dark Hard |
| `kitty/light-theme.auto.conf` | macOS light mode | Gruvbox Material Light Hard |
| `kitty/no-preference-theme.auto.conf` | No OS preference | Identical to dark theme (fallback) |

Key color differences:
- **Dark**: bg `#1d2021`, fg `#d4be98` (dark brown-gray background, warm tan text)
- **Light**: bg `#f9f5d7`, fg `#654735` (warm off-white background, dark brown text)

These files define only the 16 ANSI terminal colors plus background/foreground/cursor. No other settings.

#### Layer B — Font Rendering via launchd Daemon + Symlink

`kitty/kitty.conf:190` includes a symlink-managed override file:
```
include ignored/autodarkmode-overrides.conf
```

The symlink at `~/.config/kitty/ignored/autodarkmode-overrides.conf` points to one of two files in `kitty/modes/`:

| File | `text_composition_strategy` | `inactive_text_alpha` |
|------|-----------------------------|----------------------|
| `kitty/modes/dark.conf` | `1 0` | `0.5` |
| `kitty/modes/light.conf` | `5 0` | `0.8` |

Both set `macos_thicken_font 0.0`. The higher `text_composition_strategy` value (5 vs 1) produces bolder/thicker text strokes for readability on light backgrounds. The higher `inactive_text_alpha` (0.8 vs 0.5) keeps inactive windows more visible in light mode.

#### The Daemon (`launchd/jasonr.autodarkmode.sh` + `.plist`)

- **Trigger**: launchd runs the script every 10 seconds (`StartInterval = 10`) and at load
- **Detection** (line 13): `defaults read -g AppleInterfaceStyle | tr -d '\n'`
- **Symlink management** (lines 25-44): Creates or updates `~/.config/kitty/ignored/autodarkmode-overrides.conf` to point to `dark.conf` or `light.conf`
- **Kitty reload** (lines 17-20, 85-87): If the symlink changed, sends `SIGUSR1` to all kitty processes (`pgrep -a kitty`), triggering config reload
- **Idempotent**: Only reloads if the symlink actually changed

The `ignored/` directory is gitignored; it holds machine-local runtime state.

---

### 3. Neovim Colorscheme System

#### Active Colorschemes (`nvim/lua/jasonr/colorscheme/init.lua:13-14`)

```lua
dark  = 'gruvbox-material'
light = 'gruvbox-material'
```

Lines 16–57 contain commented-out alternatives for every installed plugin: edge, melange, rose-pine (main/moon/dawn), catppuccin (frappe/macchiato/mocha/latte), zenbones, vimbones, forestbones, neobones, everforest, nightfox variants, and kanagawa (dragon/default).

#### OS Detection (`nvim/lua/jasonr/colorscheme/detect.lua:48-61`)

The `detect_mode()` function shells out via `vim.system()`:
```sh
defaults read -g AppleInterfaceStyle || echo "light"
```
Output is lowercased. `"dark"` → `Mode.dark`; anything else → `Mode.light`. Special case: `"unknown"` returns the current Neovim background instead (handles edge cases during Neovim headless mode).

#### Startup Sequence — Cache-First

`detect.lua:125-131`:
1. **Synchronous** (instant): Reads a disk cache file at `vim.fn.stdpath('data') .. '/jasonr_colorscheme_cache_mode'`. Applies cached mode immediately so there's no flash of wrong colors on startup.
2. **Async** (`vim.uv` timer, `delay=0`): Shells out to verify real OS state, then begins 5-second polling loop.

The cache file contains a single word: `"dark"` or `"light"`. Falls back to `Mode.light` on cache miss.

#### Polling Loop

- Timer is one-shot; `run()` re-arms it each cycle with `delay=5000ms`
- On mode change: updates `vim.o.background`, calls `before_colorscheme()`, runs `vim.cmd.colorscheme`, which triggers Neovim's `ColorScheme` event, which fires `after_colorscheme()`

#### Hook Architecture

Two callbacks are threaded through via `detect.setup(opts)`:

**`before_colorscheme()`** (`init.lua:89-117`) — sets theme-specific globals before the colorscheme loads:
- `gruvbox-material`: `background = 'hard'`, foreground = `'original'` (light) or `'material'` (dark)
- `everforest`: `background = 'soft'` (light) or `'hard'` (dark)
- `edge`: `style = 'neon'`, `better_performance = 1`
- `zenbones` family: darkness/lightness variants

**`after_colorscheme()`** (`init.lua:119-381`) — applies highlight group overrides after load:

| Override Category | What It Does |
|-------------------|-------------|
| `edge` diagnostic fix (lines 123-158) | Copies `sp` color to `fg` for all 4 diagnostic groups |
| gruvbox-material dark (lines 162-170) | Lightens `CursorLine`, `StatusLine`, `Visual` relative to `Normal.bg` |
| melange dark/light (lines 174-187) | Lightens/darkens `CursorLine`, `NormalFloat`, `Visual` by 20-50% |
| kanagawa (lines 192-203) | Lightens `Normal.bg` by 30 (dark) or darkens `CursorLine`/`Visual` (light) |
| zenbones family (lines 207-270) | Uses Lush color objects from each theme for `ColorColumn`, `CursorLine`, `Visual` |
| Universal: TODO highlights (lines 294-303) | Defines `Todo`/`Hack`/`Fixme` from diagnostic colors with `matchadd()` patterns |
| Universal: Search highlights (lines 306-333) | `IncSearch`/`Search`/`CurSearch` from inverted diagnostic colors |
| Universal: Diff highlights (lines 344-358) | Different calcs for light vs dark: light darkens, dark uses hardcoded hex |
| Universal: GitSigns (lines 360-379) | 6 gitsigns groups adjusted for light vs dark |

The `after_colorscheme()` also fires a `User after_colorscheme` autocmd (`detect.lua:146-148`) so other modules can hook in.

#### Installed Colorscheme Plugins (`plugins/colorschemes/init.lua`)

All lazy-loaded:
- `sainnhe/gruvbox-material` — **currently active**
- `sainnhe/edge`
- `rebelot/kanagawa.nvim` — opts: `bg_gutter = 'none'`
- `catppuccin/nvim` — priority 1000
- `sainnhe/everforest`
- `mcchrish/zenbones.nvim` — requires `rktjmp/lush.nvim`
- `EdenEast/nightfox.nvim`
- `rose-pine/neovim`
- `savq/melange-nvim`

#### Corresponding Kitty Themes Library (`kitty/themes/`)

23 `.conf` files mirroring the Neovim plugins for manual use:
- **Dark**: zenbones_dark_warm, forestbones_dark, forestbones_dark_custom, kanagawa_dark, kanagawa_dragon, melange_dark, GruvboxMaterialDarkHard, GruvboxMaterialDarkMedium, catppuccin-mocha/macchiato/frappe, nightfox_terafox
- **Light**: kanagawa_light, melange_light, GruvboxMaterialLight(Hard/Medium/Soft), catppuccin-latte, vimbones_custom

These files are not automatically loaded — they serve as reference/manual override options.

---

### 4. Heirline Statusline — Color Derivation

Heirline derives its palette from live Neovim highlight groups, making it colorscheme-agnostic.

**`nvim/lua/jasonr/heirline/colors.lua`** builds a named palette on every `ColorScheme` event:
- `bright_bg` ← `StatusLine.bg`
- `bright_fg` ← `Folded.fg`
- `red` ← `DiagnosticError.fg`
- `green` ← `String.fg`
- `blue` ← `Function.fg`
- `git_del/add/change` ← `DiffDelete/DiffAdd/DiffChange.fg`
- `dark_bg` ← `Normal.bg`
- `diag_*` ← diagnostic highlight groups

**Kanagawa special case** (`heirline/colors.lua:22-23`): Loads colors from `kanagawa.colors.setup()` directly instead of deriving from highlight groups, then merges with the generic palette using `vim.tbl_deep_extend('force', ...)` where kanagawa values win.

The `ColorScheme` autocmd at `heirline/colors.lua:66-75` fires `heirline.utils.on_colorscheme(colors)` which live-updates the statusline palette without restart.

---

### 5. Tint.nvim — Inactive Window Dimming

`nvim/lua/jasonr/plugins/tint/init.lua` dims unfocused windows with a direction-aware transform:

- **Base config**: `tint = -45` (darken), `saturation = 0.7` (desaturate)
- **Background detection** (lines 14-29): Reads `hl.NormalNC().background` then `hl.Normal().background` (live, via the highlights module)
- **Light mode** (line 31): Inverts tint to `+45` (lighten), fallback `#ffffff`, threshold 100
- **Dark mode** (line 34): Uses `-45` (darken), fallback `#000000`, threshold 100

This ensures inactive windows always appear "washed out" in the direction of their background — lighter windows dim lighter, darker windows dim darker.

---

### 6. k9s Theme Integration

- `k9s/config.yaml:27`: `skin: autodarkmode`
- The launchd daemon manages a symlink at `~/Library/Application Support/k9s/skins/autodarkmode.yaml`:
  - Dark: → `k9s/skins/gruvbox-dark.yaml` (a commented-out alternative `everforest-dark.yaml` at `jasonr.autodarkmode.sh:61`)
  - Light: → `k9s/skins/gruvbox-light.yaml`
- The k9s skins use Gruvbox color palettes matching the Neovim/Kitty themes

---

### 7. Firefox and Desktop Wallpaper

**`launchd/DarkModeAutomation.applescript`** — a stay-open AppleScript app polling every 5 seconds:
- Uses `appliedDarkMode` as a latch variable to detect transitions
- **Dark**: Opens a Firefox Color URL for the dark theme; sets desktop to `~/Pictures/desktop-dark.jpg`
- **Light**: Opens a Firefox Color URL for the light theme; sets desktop to `~/Pictures/desktop-light.jpg`
- `wasRun` guard ensures settings apply once on first launch regardless of a mode transition

---

## Architecture Documentation

### End-to-End Flow

```
macOS System Appearance (AppleInterfaceStyle)
         │
         ├──[Kitty native]────────────────────────────────────────────────┐
         │   Detects OS appearance instantly                               │
         │   Loads *.auto.conf → full Gruvbox color palette change        │
         │                                                                 │
         ├──[launchd, every 10s]──────────────────────────────────────────┤
         │   jasonr.autodarkmode.sh                                        │
         │   ├─ Kitty: symlink autodarkmode-overrides.conf → modes/{dark,light}.conf
         │   │         SIGUSR1 to kitty → font rendering reload           │
         │   └─ k9s:   symlink autodarkmode.yaml → gruvbox-{dark,light}.yaml
         │                                                                 │
         ├──[Neovim, every 5s]────────────────────────────────────────────┤
         │   detect.lua polling timer                                      │
         │   ├─ defaults read -g AppleInterfaceStyle                       │
         │   ├─ If changed: set vim.o.background                           │
         │   ├─ before_colorscheme() → set theme globals                   │
         │   ├─ vim.cmd.colorscheme → load gruvbox-material                │
         │   ├─ ColorScheme event fires                                     │
         │   │   ├─ after_colorscheme() → highlight overrides              │
         │   │   ├─ User after_colorscheme → external hooks                │
         │   │   └─ heirline ColorScheme listener → palette refresh        │
         │   └─ Cache mode to disk                                          │
         │                                                                 │
         └──[AppleScript, every 5s]───────────────────────────────────────┘
             DarkModeAutomation.applescript
             ├─ Firefox: open Color URL for dark/light theme
             └─ Desktop: set wallpaper to desktop-{dark,light}.jpg
```

### Key Architectural Patterns

1. **Single source of truth**: All components read `AppleInterfaceStyle` independently rather than coordinating through a shared IPC channel.

2. **Polling everywhere**: Both the launchd daemon (10s) and Neovim's detect module (5s) poll rather than use event-driven notification. This means there can be up to 10 seconds of lag for kitty font rendering and 5 seconds for Neovim after an OS appearance change.

3. **Kitty native auto-switching**: The `*.auto.conf` files are handled by Kitty itself with no custom tooling — colors update instantly. Only the font rendering overrides require the launchd daemon.

4. **Symlink-based mode switching**: Both kitty's font overrides and k9s skins use the same pattern: a fixed symlink path that is atomically updated to point to a mode-specific config file.

5. **Cache-first startup**: Neovim reads a disk cache for instant colorscheme on launch, then asynchronously verifies/updates.

6. **Hook cascading**: `before_colorscheme` and `after_colorscheme` form a bracket around `vim.cmd.colorscheme`. The `after_colorscheme` additionally fires a `User` autocmd, enabling other plugins to extend the pipeline.

7. **Highlight-group derivation**: Heirline and tint.nvim derive colors from live highlight groups at runtime, making them work with any colorscheme without per-theme configuration.

8. **Light mode inverts tinting direction**: `tint.nvim` flips its tint sign based on `vim.o.background` so inactive windows always dim in the "right" direction regardless of whether light or dark mode is active.

---

## Code References

- `nvim/lua/jasonr/colorscheme/init.lua:13-14` — Active dark/light colorscheme variables
- `nvim/lua/jasonr/colorscheme/init.lua:59-74` — Theme-specific global variable setup
- `nvim/lua/jasonr/colorscheme/init.lua:89-117` — `before_colorscheme` hook
- `nvim/lua/jasonr/colorscheme/init.lua:119-381` — `after_colorscheme` highlight overrides
- `nvim/lua/jasonr/colorscheme/detect.lua:48-61` — `detect_mode()` via `defaults read`
- `nvim/lua/jasonr/colorscheme/detect.lua:25-39` — Disk cache read/write
- `nvim/lua/jasonr/colorscheme/detect.lua:101-131` — Polling loop + startup sequence
- `nvim/lua/jasonr/colorscheme/detect.lua:135-151` — `setup()` and `ColorScheme` autocmd
- `nvim/lua/jasonr/colorscheme/highlights.lua:6-56` — Lazy highlight group getters
- `nvim/lua/jasonr/plugins/colorschemes/init.lua:1-19` — All 9 lazy colorscheme plugin specs
- `nvim/lua/jasonr/heirline/colors.lua:19-64` — `get_colors()` palette derivation
- `nvim/lua/jasonr/heirline/colors.lua:66-75` — `ColorScheme` autocmd for live palette updates
- `nvim/lua/jasonr/plugins/tint/init.lua:14-35` — Direction-aware tint transform
- `nvim/lua/jasonr/utils/colors.lua:17-46` — `luminance()`, `lighten()`, `darken()` color math
- `kitty/kitty.conf:190` — `include ignored/autodarkmode-overrides.conf`
- `kitty/dark-theme.auto.conf` — Gruvbox Material Dark Hard palette
- `kitty/light-theme.auto.conf` — Gruvbox Material Light Hard palette
- `kitty/modes/dark.conf` — `text_composition_strategy 1 0`, `inactive_text_alpha 0.5`
- `kitty/modes/light.conf` — `text_composition_strategy 5 0`, `inactive_text_alpha 0.8`
- `launchd/jasonr.autodarkmode.sh:13` — `defaults read -g AppleInterfaceStyle`
- `launchd/jasonr.autodarkmode.sh:22-45` — Kitty symlink management
- `launchd/jasonr.autodarkmode.sh:17-20,85-87` — SIGUSR1 kitty reload
- `launchd/jasonr.autodarkmode.sh:47-80` — k9s symlink management
- `launchd/jasonr.autodarkmode.plist:16` — `StartInterval = 10` seconds
- `launchd/DarkModeAutomation.applescript:18-60` — Firefox + wallpaper idle handler
- `k9s/config.yaml:27` — `skin: autodarkmode`
- `.zshrc-osx:4` — `dark-mode-toggle` alias

## Open Questions

- The `k9s/config.yaml:27` `skin: autodarkmode` references a file that is not present in the dotfiles repo — it is a runtime symlink managed by the launchd daemon at `~/Library/Application Support/k9s/skins/autodarkmode.yaml`. The mechanism relies on k9s loading skins by name from that directory.
- The `kitty/themes/` library of 23 theme files appears to be a manual reference collection rather than part of the automated switching pipeline. It is unclear whether there is a keybinding or command to manually switch to these themes.
