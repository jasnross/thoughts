# nvim-treesitter Main Branch Upgrade Implementation Plan

## Overview

Migrate the Neovim config from the frozen `nvim-treesitter` master branch to the rewritten `main` branch, keep Treesitter highlighting disabled, preserve existing textobject keymaps by moving them into `nvim-treesitter-textobjects`, and remove incremental selection for now while documenting future replacement options.

## Current State Analysis

- Treesitter is configured via the legacy module API (`require('nvim-treesitter.configs').setup`) with `highlight.enable = false`, `incremental_selection` enabled, and inline `textobjects` config. `nvim/lua/jasonr/plugins/treesitter/init.lua:5`
- The treesitter plugins are pinned to `master` in the lazy lockfile. `nvim/lazy-lock.json:70`
- Folding already uses Neovim's `vim.treesitter.foldexpr()` with a legacy fallback. `nvim/lua/jasonr/folding/init.lua:37`
- DAP helpers call `nvim-treesitter.ts_utils`, which is not present in the rewrite. `nvim/lua/jasonr/plugins/dap/typescript.lua:1`
- Custom Scala textobjects are defined in after/queries. `nvim/after/queries/scala/textobjects.scm:1`

## Desired End State

- `nvim-treesitter` uses the `main` branch, is not lazy-loaded, and installs parsers via the new API without the old module config.
- Treesitter highlighting remains off unless explicitly enabled later.
- Textobjects keep the same keymaps and behavior, but are configured through `nvim-treesitter-textobjects` with explicit keymaps.
- No incremental selection mappings or module config remain, with future replacement options documented.
- `nvim-treesitter.ts_utils` usage is replaced with supported Neovim 0.11 treesitter APIs.

### Key Discoveries:

- The new `main` branch is a full incompatible rewrite and does not support lazy-loading. (GitHub README for `main`.)
- Treesitter features in the rewrite must be enabled via Neovim APIs (e.g., `vim.treesitter.start()` and `vim.treesitter.foldexpr()`), not by `nvim-treesitter` modules. (nvim-treesitter docs.)
- `nvim-treesitter-textobjects` now expects explicit keymaps; the old `keymaps` table in module config is no longer supported. (textobjects docs.)

## Preflight Audit (Completed)

- Neovim version: `NVIM v0.11.6`. (Command: `nvim --version`.)
- tree-sitter CLI: `tree-sitter 0.26.5`. (Command: `tree-sitter --version`.)
- Neovim treesitter APIs confirmed in runtime docs:
  - `vim.treesitter.get_node()` and `vim.treesitter.get_node_text()` exist. `/Users/jasonr/.local/share/mise/installs/neovim/0.11.6/nvim-macos-arm64/share/nvim/runtime/doc/treesitter.txt:960`
  - `vim.treesitter.get_node_text()` returns a string. `/Users/jasonr/.local/share/mise/installs/neovim/0.11.6/nvim-macos-arm64/share/nvim/runtime/doc/treesitter.txt:998`
  - `vim.treesitter.get_parser()` is available for explicit parsing when highlighting is off. `/Users/jasonr/.local/share/mise/installs/neovim/0.11.6/nvim-macos-arm64/share/nvim/runtime/doc/treesitter.txt:1014`

## What We're NOT Doing

- We are not enabling Treesitter highlighting or adding a FileType autocmd to start it.
- We are not installing an incremental selection replacement (we will document options only).
- We are not refactoring unrelated plugins or keymaps beyond treesitter-related changes.

## Implementation Approach

- Migrate the core treesitter config to the rewrite API by swapping `configs.setup` for `require('nvim-treesitter').setup` and explicit `install()` calls.
- Move textobject configuration to `nvim-treesitter-textobjects` and recreate keymaps using the plugin's APIs.
- Update any code that depends on `nvim-treesitter.ts_utils` to use Neovim core treesitter APIs, and explicitly parse if `vim.treesitter.get_node()` returns nil (highlighting is off).
- Keep folding behavior as-is and validate it still works without enabling highlighting.

## Phase 1: Config Migration

### Overview

Switch to `nvim-treesitter` main branch, remove module setup, migrate textobjects, and replace `ts_utils` usage.

### Changes Required:

#### 1. Update nvim-treesitter plugin spec and config

**File**: `nvim/lua/jasonr/plugins/treesitter/init.lua`
**Changes**:
- Set `branch = 'main'` and `lazy = false`
- Remove `require('nvim-treesitter.install').compilers` line
- Replace `require('nvim-treesitter.configs').setup({...})` with:
  - `require('nvim-treesitter').setup({})` if needed
  - `require('nvim-treesitter').install({ ... })` for the parser list
- Remove `incremental_selection` config block

```lua
{
  'nvim-treesitter/nvim-treesitter',
  branch = 'main',
  lazy = false,
  build = ':TSUpdate',
  config = function()
    local ts = require('nvim-treesitter')
    ts.setup({})
    ts.install({
      'bash', 'comment', 'css', 'diff', 'go', 'gomod', 'helm', 'html',
      'java', 'json', 'jsonc', 'lua', 'markdown', 'markdown_inline', 'mermaid',
      'query', 'scala', 'scss', 'tsx', 'typescript', 'typst', 'vim', 'vimdoc', 'yaml',
    })
  end,
}
```

#### 2. Move textobjects config to nvim-treesitter-textobjects

**File**: `nvim/lua/jasonr/plugins/treesitter/init.lua`
**Changes**:
- Set textobjects plugin to `branch = 'main'`
- Add `config` to `nvim-treesitter-textobjects` with setup options
- Add explicit keymaps for `select`, `move`, and `swap` using the plugin APIs

```lua
{
  'nvim-treesitter/nvim-treesitter-textobjects',
  branch = 'main',
  config = function()
    require('nvim-treesitter-textobjects').setup({
      select = {
        lookahead = true,
      },
      move = {
        set_jumps = true,
      },
    })

    local select = require('nvim-treesitter-textobjects.select')
    local move = require('nvim-treesitter-textobjects.move')
    local swap = require('nvim-treesitter-textobjects.swap')

    vim.keymap.set({ 'x', 'o' }, 'af', function()
      select.select_textobject('@function.outer', 'textobjects')
    end)
    vim.keymap.set({ 'x', 'o' }, 'if', function()
      select.select_textobject('@function.inner', 'textobjects')
    end)
    vim.keymap.set({ 'x', 'o' }, 'ac', function()
      select.select_textobject('@class.outer', 'textobjects')
    end)
    vim.keymap.set({ 'x', 'o' }, 'ic', function()
      select.select_textobject('@class.inner', 'textobjects')
    end)
    vim.keymap.set({ 'x', 'o' }, 'as', function()
      select.select_textobject('@local.scope', 'locals')
    end)
    vim.keymap.set({ 'x', 'o' }, 'aV', function()
      select.select_textobject('@val.outer', 'textobjects')
    end)
    vim.keymap.set({ 'x', 'o' }, 'iV', function()
      select.select_textobject('@val.inner', 'textobjects')
    end)
    vim.keymap.set({ 'x', 'o' }, 'aA', function()
      select.select_textobject('@anything', 'textobjects')
    end)

    vim.keymap.set({ 'n', 'x', 'o' }, '<c-s-]>', function()
      move.goto_next_start({
        '@class.outer',
        '@object.outer',
        '@trait.outer',
        '@object_member.outer',
        '@class_member.outer',
        '@trait_member.outer',
      }, 'textobjects')
    end)
    vim.keymap.set({ 'n', 'x', 'o' }, '<c-s-[>', function()
      move.goto_previous_start({
        '@class.outer',
        '@object.outer',
        '@trait.outer',
        '@object_member.outer',
        '@class_member.outer',
        '@trait_member.outer',
      }, 'textobjects')
    end)

    vim.keymap.set('n', '<leader>a', function()
      swap.swap_next('@argument')
    end)
    vim.keymap.set('n', '<leader>A', function()
      swap.swap_previous('@argument')
    end)
  end,
}
```

#### 3. Replace ts_utils usage

**File**: `nvim/lua/jasonr/plugins/dap/typescript.lua`
**Changes**:
- Replace `ts_utils.get_node_at_cursor()` with the Neovim 0.11 API, and ensure a parse occurs if no node is returned.
- Replace `ts_utils.get_node_text(child)[1]` with `vim.treesitter.get_node_text(child, 0)` (string return value).

```lua
local function get_node_at_cursor()
  local node = vim.treesitter.get_node()
  if node then
    return node
  end

  local parser = vim.treesitter.get_parser(0, nil, { error = false })
  if parser then
    parser:parse()
  end

  return vim.treesitter.get_node()
end

local function node_text(node)
  return vim.treesitter.get_node_text(node, 0)
end
```

#### 4. Lockfile update

**File**: `nvim/lazy-lock.json`
**Changes**: Update the entries for `nvim-treesitter` and `nvim-treesitter-textobjects` to the `main` branch after a `:Lazy sync`.

### Success Criteria:

#### Automated Verification:

- [x] `:TSUpdate` completes without errors
- [x] `:Lazy sync` completes and updates lock entries

#### Manual Verification:

- [x] Textobject selections (`af`, `if`, `ac`, `ic`, `as`, `aV`, `iV`) work in supported filetypes
- [x] Move keymaps (`<c-s-]>`, `<c-s-[>`) jump as expected
- [x] Swap keymaps (`<leader>a`, `<leader>A`) work where `@argument` exists
- [x] DAP Jest helpers still compute the test name under cursor

**Implementation Note**: After completing this phase and all automated verification passes, pause for manual confirmation before proceeding.

---

## Phase 2: Verification and Cleanup

### Overview

Confirm folding behavior and ensure highlighting remains disabled by default.

### Changes Required:

#### 1. Validate folding without highlight

**File**: `nvim/lua/jasonr/folding/init.lua`
**Changes**: No code changes expected; verify that foldexpr still works and adjust only if needed.

#### 2. Validate custom Scala textobjects

**File**: `nvim/after/queries/scala/textobjects.scm`
**Changes**: No code changes expected; confirm `@val.*` and member captures still map to keymaps.

### Success Criteria:

#### Automated Verification:

- [x] `:checkhealth nvim-treesitter` reports OK for parsers and queries

#### Manual Verification:

- [x] Fold preview and foldtext behavior is unchanged
- [x] Treesitter highlighting stays off by default
- [x] Scala textobject captures `@val`, `@object_member`, `@class_member`, `@trait_member` still work

**Implementation Note**: After completing this phase and all automated verification passes, pause for manual confirmation before proceeding.

---

## Testing Strategy

### Unit Tests:

- None available for dotfiles; rely on targeted manual checks.

### Automated Verification:

- Covered in Phase 1 and Phase 2 success criteria (e.g., `:TSUpdate`, `:Lazy sync`, `:checkhealth nvim-treesitter`).

### Manual Testing Steps:

1. Open a Scala file and test `aV`, `iV`, `<c-s-]>`, `<c-s-[>` mappings.
2. Open a TypeScript Jest test and run the DAP command that uses `get_test_name_at_cursor()`.
3. Confirm folding still works and fold preview behaves as expected.
4. Confirm no Treesitter highlighting is enabled unless explicitly turned on.

## Performance Considerations

- `nvim-treesitter.install()` runs asynchronously; avoid forcing synchronous waits unless bootstrapping.
- Parser installs use `tree-sitter` CLI and may be slower on first run; subsequent runs should be no-ops if already installed.

## Migration Notes

- The rewrite removes module-based config, so `incremental_selection` and `textobjects` must move to standalone plugins or be removed.
- `nvim-treesitter.ts_utils` is not present in `main`; use Neovim's `vim.treesitter` APIs instead.
- Highlighting is now controlled by Neovim (`vim.treesitter.start()`), not by `nvim-treesitter` config.

## Future Considerations (Deferred)

Incremental selection replacements to evaluate later:
- `treesitter-modules.nvim` (re-implements the old module behavior)
- `incr.nvim` (lightweight incremental selection)
- Neovim LSP selection range (`vim.lsp.buf.selection_range`)
- `treemonkey.nvim` or `mini.ai` for alternate selection workflows

## References

- Current treesitter config: `nvim/lua/jasonr/plugins/treesitter/init.lua:5`
- Folding integration: `nvim/lua/jasonr/folding/init.lua:37`
- DAP ts_utils usage: `nvim/lua/jasonr/plugins/dap/typescript.lua:1`
- Scala custom textobjects: `nvim/after/queries/scala/textobjects.scm:1`
- nvim-treesitter docs (main): `doc/nvim-treesitter.txt`
- nvim-treesitter-textobjects docs: `doc/nvim-treesitter-textobjects.txt`
- Main branch README for requirements and non-lazy loading:

```text
https://github.com/nvim-treesitter/nvim-treesitter/blob/main/README.md
https://github.com/nvim-treesitter/nvim-treesitter-textobjects/blob/main/README.md
```
