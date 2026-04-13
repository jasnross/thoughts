# Cargo Quickfix Command Implementation Plan

## Overview

Add a `:Cargo` Neovim user command that runs arbitrary `cargo` subcommands and
captures compiler errors and test failures into the quickfix list, following the
same pattern as the existing `:Sbt` command. Cargo's `--message-format=short`
flag normalizes multi-line compiler diagnostics into single-line GNU-style
output (`file:line:col: error[code]: message`) that Vim's `errorformat` can
parse directly.

## Current State Analysis

The existing quickfix command infrastructure lives in two files:

- `nvim/lua/jasonr/utils/quickfix.lua:78` — `populate_from_command(cmd, options)`
  runs a command asynchronously, streams output to the quickfix window, then
  parses the accumulated output with `vim.fn.setqflist` using the provided
  `errorformats` on exit.
- `nvim/lua/jasonr/commands.lua:96-152` — the `:Sbt` command is the closest
  analog: it accepts arbitrary subcommand args via `nargs = '+'`, strips custom
  `--flag` args, and delegates to `populate_from_command`.

The `PopulateQuickFixFromCommandOptions` type accepts:
- `errorformats string[]` — vim errorformat patterns joined with commas
- `cwd string?` — working directory override
- `env table<string, string>?` — environment variable overrides

Rust tooling is already partially configured: `rust-analyzer` with clippy
check-on-save in `nvim/lua/jasonr/lsp-config/lsps/rust_analyzer.lua` and
`rustfmt` via conform.nvim. There are no existing cargo errorformat patterns
anywhere in the codebase.

## Desired End State

`:Cargo build`, `:Cargo check`, `:Cargo clippy`, and `:Cargo test` all populate
the quickfix list with navigable entries. Compiler errors and warnings (from all
subcommands) link to the correct file and line. Test panics (phase 2) also link
to the failing assertion location.

### Key Discoveries

- `--message-format=short` produces single-line output per diagnostic:
  `file:line:col: error[E0308]: message` — all compilation subcommands support it
- `%t` in errorformat captures one char as the error type: `%trror` matches
  `error` and captures `'e'`; `%tarning` matches `warning` and captures `'w'`
- The `[%.%#]` pattern matches a bracket-enclosed error code like `[E0308]` or
  `[dead_code]` (literal `[`, any chars, literal `]`)
- `--message-format=short` only affects compiler diagnostics; test failure output
  (panics) uses a different format and requires separate patterns (phase 2)
- Test panics differ between Rust versions: 1.73+ puts the location first
  (`thread 'name' panicked at file:line:col:`), older Rust puts it last
  (`thread 'name' panicked at 'msg', file:line:col`)
- Cargo caches compiler output — if source files haven't changed, re-running
  `:Cargo` with a different `--message-format` may not force recompilation

## What We're NOT Doing

- No custom `--vim-flag` stripping (unlike `:Sbt`'s `--warn` flag) — no need
  has been identified
- No special handling for `cargo run` output
- No integration or deduplication with LSP/rust-analyzer diagnostics — the
  commands serve different workflows (LSP is ambient; `:Cargo` is on-demand
  for full-project checks)
- No global `[build]` config in `~/.cargo/config.toml` — the flag is injected
  by the command

---

## Phase 1: Core Cargo command with compiler diagnostics

### Overview

Add the `:Cargo` command to `commands.lua`. It injects `--message-format=short`
after the subcommand (before any `--` separator) so that compiler errors and
warnings are normalized to single-line format and parsed into the quickfix list.

### Changes Required

#### 1. New command in `nvim/lua/jasonr/commands.lua`

Add after the `:Sbt` command block (after line 152):

- [x] Add a `-- Runs cargo and populates errors to the quickfix list` comment
- [x] Register a `Cargo` user command with `nargs = '+'`
- [x] Build the command string with `--message-format=short` inserted after the
  first farg (the subcommand) and before any remaining args, so that
  `:Cargo test -- --nocapture` becomes
  `cargo test --message-format=short -- --nocapture`
  > **Deviation:** `--message-format=short` is only injected when the subcommand
  > is in an allowlist of compilation-driving subcommands (`build`, `check`,
  > `clippy`, `run`, `test`, `bench`, `fix`, `doc`, `rustc`). Other subcommands
  > (`fmt`, `add`, `update`, etc.) don't accept the flag and would fail. The
  > allowlist is a module-level local `cargo_format_subcommands`.
- [x] Pass the following `errorformats` to `populate_from_command`:

```lua
local efm = {
  '%f:%l:%c:\\ %trror[%.%#]:\\ %m',
  '%f:%l:%c:\\ %tarning[%.%#]:\\ %m',
  '%f:%l:%c:\\ %trror:\\ %m',
  '%f:%l:%c:\\ %tarning:\\ %m',
  '%-G%.%#',
}
```

Pattern explanation:
- Patterns 1–2 match errors/warnings with error codes (`[E0308]`, `[dead_code]`)
  and strip the code from `%m` so the message is clean
- Patterns 3–4 match the same without a code (e.g., `error: linker not found`)
- `%-G%.%#` is the catch-all that suppresses all unmatched lines

Full command implementation:

```lua
-- Runs cargo and populates errors to the quickfix list
utils.vim.create_user_command({
  name = 'Cargo',
  opts = {
    desc = 'Run cargo with provided arguments and populate quickfix list with results',
    nargs = '+',
  },
  callback = function(tbl)
    local fargs = tbl.fargs
    local args = vim.list_extend({ fargs[1], '--message-format=short' }, vim.list_slice(fargs, 2))
    local cmd = 'cargo ' .. table.concat(args, ' ')

    local efm = {
      '%f:%l:%c:\\ %trror[%.%#]:\\ %m',
      '%f:%l:%c:\\ %tarning[%.%#]:\\ %m',
      '%f:%l:%c:\\ %trror:\\ %m',
      '%f:%l:%c:\\ %tarning:\\ %m',
      '%-G%.%#',
    }

    utils.quickfix.populate_from_command(cmd, {
      errorformats = efm,
    })
  end,
})
```

### Success Criteria

#### Automated Verification

- [x] Neovim starts without errors: `nvim --headless -c 'lua require("jasonr.commands")' -c 'qa'`

#### Manual Verification

- [x] `:Cargo build` in a Rust project with a compilation error populates the
  quickfix list with entries that jump to the correct file and line
- [x] `:Cargo clippy` with lint warnings shows warning entries in the quickfix
  list with correct locations
- [x] `:Cargo check -- --all-targets` (with `--` separator) does not break
  argument passing and still populates the quickfix list
- [x] Subcommands with no errors show an empty or raw-output quickfix list (the
  `%-G%.%#` catch-all suppresses unparseable lines, so no false entries appear)

---

## Phase 2: Test failure patterns

### Overview

Extend the `efm` pattern list in `:Cargo` to capture test panic locations.
`cargo test` output includes lines from the test binary itself, which are
unaffected by `--message-format=short`. Two panic formats must be handled,
matching modern (Rust 1.73+) and older Rust versions.

### Changes Required

#### 1. Update errorformat list in `nvim/lua/jasonr/commands.lua`

The test panic patterns use Vim's multi-line errorformat mechanism for modern
Rust (since the file location and message are on separate lines), plus a
single-line pattern for old Rust.

- [x] Insert the following patterns **before** the `'%-G%.%#'` catch-all in `efm`:

```lua
-- Modern Rust 1.73+: location is on the panicked-at line, message on the next line
-- Input line 1: "thread 'test_name' panicked at src/lib.rs:11:9:"
-- Input line 2: "assertion `left == right` failed"
"thread\\ '%*[^']'\\ panicked\\ at\\ %f:%l:%c:",  -- %E: multi-line start, captures location
'%Z%m',                                            -- %Z: end of multi-line, captures message

-- Old Rust (<1.73): message is inline, location is at the end
-- Input: "thread 'test_name' panicked at 'assertion failed', src/lib.rs:11:9"
"thread\\ '%*[^']'\\ panicked\\ at\\ '%m',\\ %f:%l:%c",
```
> **Deviation (code review):** The `%C%.%#` continuation pattern was removed.
> Both `%Z%m` and `%C%.%#` match any line, making `%C` unreachable dead code.
> Since the panic message is exactly one line after `panicked at`, dropping `%C`
> is correct.
>
> **Deviation (runtime testing):** The actual `cargo test` output includes a
> thread ID between the thread name and `panicked at`:
> `thread 'name' (7915993) panicked at src/lib.rs:7:9:`
> The `'\\ panicked` literal match was replaced with `'%.%#panicked` so `%.%#`
> absorbs whatever sits between the closing `'` and `panicked` (thread ID or just
> a space). Verified correct matching with `vim.fn.setqflist` against real output.

Note: the modern-format pattern uses `%E`/`%Z`/`%C` multi-line markers. In the
`efm` array these must appear in the order `%E`, `%C`, `%Z` for Vim to
correctly associate continuation lines. The `%*[^']` scanf pattern matches and
discards any characters that are not a single quote, which captures the thread
name without consuming the closing `'`.

Updated full `efm` list:

```lua
local efm = {
  '%f:%l:%c:\\ %trror[%.%#]:\\ %m',
  '%f:%l:%c:\\ %tarning[%.%#]:\\ %m',
  '%f:%l:%c:\\ %trror:\\ %m',
  '%f:%l:%c:\\ %tarning:\\ %m',
  "%Ethread\\ '%*[^']'\\ panicked\\ at\\ %f:%l:%c:",
  '%Z%m',
  '%C%.%#',
  "thread\\ '%*[^']'\\ panicked\\ at\\ '%m',\\ %f:%l:%c",
  '%-G%.%#',
}
```

### Success Criteria

#### Manual Verification

- [x] `:Cargo test` with a failing `assert_eq!` populates the quickfix list
  with an entry pointing to the assertion's file and line (not the test runner
  boilerplate line)
- [x] Entries from both a current Rust toolchain (1.73+) and the old panic
  format (if accessible via `rustup` with an older toolchain) are captured
  correctly — or verify by constructing a `panic!("msg")` at a known location
  and confirming the quickfix jumps there
- [x] Passing test runs produce no spurious quickfix entries
