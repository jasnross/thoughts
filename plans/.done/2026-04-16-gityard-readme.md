# gityard README Implementation Plan

## Overview

Create a user-facing README.md for gityard covering: summary, installation,
quick start walkthrough, commands with stylized example output, and license.

## Current State Analysis

- No README.md exists in the project root.
- CLAUDE.md contains developer-facing documentation (build commands, lint rules,
  module layout) but is not suitable as a user-facing README.
- All command descriptions are defined as doc comments on clap derive macros in
  `src/cli.rs`.
- Golden test examples in `src/presenter.rs` (lines 1069-1186, 1438-1448)
  provide representative output that can serve as templates for stylized
  examples.

### Key Discoveries:

- The app's top-level description is `"Multi-repository git management"`
  (`src/cli.rs:7`).
- 12 commands total: `add`, `list`, `remove`, `status`, `fetch`, `pull`,
  `config` (3 subcommands), `group` (3 subcommands), `worktrees`, `branches`,
  `trim`, `completions`.
- Output uses Unicode symbols (`✓`/`✗`/`─`), dynamic column widths, and
  `·`-separated summary footers.
- Project is dual-licensed: `LICENSE-MIT` and `LICENSE-APACHE` files exist in
  the repo root.

## Desired End State

A `README.md` at the project root that a new user can read to understand what
gityard does, install it, and start using it within minutes. The file renders
well on GitHub and in terminal markdown viewers.

## What We're NOT Doing

- Detailed configuration reference (future work)
- Contributing guide
- CI badges or project status indicators
- Comparison with similar tools
- Per-flag documentation (commands section gives descriptions, not full
  `--help` output)
- Screenshot images (text examples only)

## Implementation

### Overview

Create `README.md` with five sections: summary, installation, quick start,
commands, and license. Command examples use stylized output based on golden
test patterns from `src/presenter.rs`.

### Changes Required:

#### 1. Create `README.md`

**File**: `README.md` (new file at project root)

- [x] **Summary section**: One paragraph explaining what gityard is — a
      multi-repository git management CLI that provides rich summarized status
      across many repos and safe bulk operations with pre-flight validation.
      Mention key capabilities: status overview, bulk fetch/pull, branch and
      worktree management, stale branch cleanup.

- [x] **Installation section**: Single `cargo install --git` command:
      ```
      cargo install --git https://github.com/jasnross/gityard
      ```

- [x] **Quick Start section**: Three-step walkthrough showing the core loop:
      1. `gityard add ~/projects` — register repos (mention directory scanning)
      2. `gityard status` — show a stylized status table example
      3. `gityard fetch` — bulk fetch with stylized output
      
      End with a one-liner mentioning `gityard config show` for customizing
      defaults.

- [x] **Commands section**: Each command with its description (from
      `src/cli.rs` doc comments) and stylized example output for key commands.
      Organize as subsections. Commands to cover:

      **Registry management** (`add`, `remove`, `list`):
      - `add`: description + note about directory scanning behavior
      - `remove`: description only
      - `list`: stylized two-column output example

      **Status & inspection** (`status`, `branches`, `worktrees`):
      - `status`: stylized table based on golden test at `presenter.rs:1070-1077`
        ```
           REPO     BRANCH      STATUS    ↑↓ MAIN    ↑↓ REMOTE
         ✓ api      main        clean     ─          ─
         ✗ web-app  feat/login  3M 1?     +5 -2      +1
         ✓ deploy   develop     clean     -3         ─

         3 repos · 1 dirty · 1 ahead of main
        ```
        Mention default hiding of clean repos and key filter flags
        (`--dirty`, `--all`, `--group`).
      - `branches`: stylized table based on golden test at
        `presenter.rs:1109-1114`. Mention key filters (`--gone`,
        `--no-remote`).
      - `worktrees`: stylized table based on golden test at
        `presenter.rs:1439-1447`. Mention key filters (`--gone`, `--dirty`).

      **Bulk operations** (`fetch`, `pull`):
      - `fetch`: stylized success output with `✓`/`✗` per repo and summary
        footer. Mention notable flags (passthrough to git fetch).
      - `pull`: description, mention pre-flight dirty check and
        `--allow-partial`.

      **Branch cleanup** (`trim`):
      - Stylized output based on golden test at `presenter.rs:1176-1186`.
        Mention `--dry-run` and `--no-remote` flags.

      **Groups** (`group add`, `group list`, `group remove`):
      - Brief descriptions for each subcommand. Mention that most commands
        accept `--group` to scope operations.

      **Configuration** (`config show`, `config path`, `config edit`):
      - Brief descriptions. Mention `--toml` flag on `config show`.

      **Shell completions** (`completions`):
      - Brief description with example: `gityard completions zsh`.

- [x] **License section**: One-liner noting MIT license with link to LICENSE file.
  > **Deviation:** Plan said dual-license MIT + Apache 2.0, but the project is
  > MIT-only (`Cargo.toml` says `license = "MIT"`, single `LICENSE` file).

### Success Criteria:

#### Automated Verification:

- [x] `README.md` exists at the project root
- [x] README renders valid markdown: no broken links, no unclosed code fences
- [x] `just check` still passes (README should not affect build/lint/test)

#### Manual Verification:

- [x] README renders correctly on GitHub (check after pushing)
- [x] Example output is representative of actual command output
- [x] Quick start walkthrough is accurate and followable

## References

- Idea document: `$THOUGHTS_DIR/ideas/2026-04-16-gityard-readme.md`
- CLI definitions: `src/cli.rs`
- Golden test examples: `src/presenter.rs:1069-1186, 1438-1448`
- Executor report format: `src/executor.rs:195-256`
- List output: `src/main.rs:142-155`
