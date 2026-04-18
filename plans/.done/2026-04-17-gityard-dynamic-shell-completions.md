# Dynamic Shell Completions Implementation Plan

## Overview

Replace gityard's static shell completions (via `clap_complete::generate()`) with
dynamic completions using `CompleteEnv` + `ArgValueCompleter`. This enables
tab-completion of repo aliases and group names from the registry at completion
time, across all commands that accept these arguments.

## Current State Analysis

- Static completions generated via `clap_complete::generate()` at `src/main.rs:24-29`
- The `Completions` subcommand (`src/cli.rs:51-54`) accepts a `Shell` argument and
  is handled before config/registry loading
- `clap_complete = "4"` without the `unstable-dynamic` feature (`Cargo.toml:11`)
- Registry loads from `repos.toml` via `Registry::load()` (`src/registry.rs:40-48`)
- Config dir resolved via `config::config_dir()` (`src/config.rs:15-22`):
  `$GITYARD_CONFIG_DIR` or `~/.config/gityard`

### Fields that need completers

**Repo alias fields** (complete with registered repo aliases):
- `Command::Remove { repos }` — `cli.rs:30`
- `GroupCommand::Add { repos }` — `cli.rs:181`

**Group name fields** (complete with registered group names):
- `ExecArgs::group` — `cli.rs:65` (used by `Fetch`, `Pull`)
- `StatusArgs::group` — `cli.rs:155`
- `WorktreesArgs::group` — `cli.rs:206`
- `BranchesArgs::group` — `cli.rs:230`
- `TrimArgs::group` — `cli.rs:243`
- `GroupCommand::Add { name }` — `cli.rs:179`
- `GroupCommand::Remove { name }` — `cli.rs:188`

### Key Discoveries

- Completers must be `'static` — they cannot capture shared app state, so each
  completer must load the registry itself
- `CompleteEnv::with_factory(cli).complete()` must be the first line of `main()`
- The factory takes a `Fn() -> Command` (not `Cli`), so a helper function that
  calls `Cli::command()` is needed
- `config_dir()` returns a `Result`, so completers must handle the error case
  gracefully (return empty completions on failure)

## Desired End State

- `gityard remove <tab>` suggests registered repo aliases
- `gityard group remove <tab>` suggests registered group names
- `gityard status --group <tab>` suggests registered group names
- All commands/flags accepting repo aliases or group names have completions
- The `completions` subcommand no longer exists
- Users configure completions with a one-liner in their shell rc:
  - bash: `source <(COMPLETE=bash gityard)`
  - zsh: `source <(COMPLETE=zsh gityard)`
  - fish: `COMPLETE=fish gityard | source`

### Verification

- `just check` passes (format + lint + build + test + license check)
- Manual verification in a shell confirms tab-completion suggests aliases/groups

## What We're NOT Doing

- Fancy UX extras (group member descriptions, filtering already-added repos,
  filesystem path completion for `add`)
- PowerShell or elvish support (unless trivially supported by `CompleteEnv`)
- Keeping the static `completions` subcommand as a fallback
- Caching or lazy-loading optimizations for registry reads

## Implementation Approach

The completers live in `cli.rs` next to the fields they annotate, following the
project's "colocate code with its consumer" principle. Two completer functions
are needed: one for repo aliases and one for group names. Both load the registry
via `config::config_dir()` + `Registry::load()` and return `CompletionCandidate`
values. Failures (missing config dir, malformed registry) return empty
completions silently — tab-completion should never error.

## Implementation

### Overview

Enable the `unstable-dynamic` feature, add two completer functions, annotate all
repo/group fields with `ArgValueCompleter`, wire up `CompleteEnv` in `main()`,
and remove the static `Completions` subcommand.

### Changes Required

#### 1. Enable `unstable-dynamic` feature

**File**: `Cargo.toml`

- [x] Add `unstable-dynamic` feature to `clap_complete`:
  ```toml
  clap_complete = { version = "4", features = ["unstable-dynamic"] }
  ```

#### 2. Add completer functions and annotate fields

**File**: `src/cli.rs`

- [x] Add imports for `clap_complete::engine::{ArgValueCompleter, CompletionCandidate}`
- [x] Add a `complete_repo_alias` function that loads the registry and returns
      aliases as `CompletionCandidate` values:
  ```rust
  fn complete_repo_alias(current: &std::ffi::OsStr) -> Vec<CompletionCandidate> {
      let Some(current) = current.to_str() else {
          return Vec::new();
      };
      let Ok(config_dir) = gityard::config::config_dir() else {
          return Vec::new();
      };
      let Ok(registry) = gityard::registry::Registry::load(&config_dir) else {
          return Vec::new();
      };
      registry
          .list()
          .iter()
          .filter_map(|entry| {
              let name = entry.display_name();
              name.starts_with(current).then(|| CompletionCandidate::new(name))
          })
          .collect()
  }
  ```
- [x] Add a `complete_group_name` function that loads the registry and returns
      group names as `CompletionCandidate` values:
  > **Deviation:** the closure was written as
  > `|name| CompletionCandidate::new(name.as_str())` to make the `&String` →
  > `&str` coercion explicit (`.keys()` yields `&String`, but
  > `CompletionCandidate::new` takes `impl Into<OsString>` which works through
  > `&str`).
  ```rust
  fn complete_group_name(current: &std::ffi::OsStr) -> Vec<CompletionCandidate> {
      let Some(current) = current.to_str() else {
          return Vec::new();
      };
      let Ok(config_dir) = gityard::config::config_dir() else {
          return Vec::new();
      };
      let Ok(registry) = gityard::registry::Registry::load(&config_dir) else {
          return Vec::new();
      };
      registry
          .list_groups()
          .keys()
          .filter(|name| name.starts_with(current))
          .map(|name| CompletionCandidate::new(name))
          .collect()
  }
  ```
- [x] Annotate `Command::Remove { repos }` with `#[arg(add = ArgValueCompleter::new(complete_repo_alias))]`
- [x] Annotate `GroupCommand::Add { repos }` with `#[arg(add = ArgValueCompleter::new(complete_repo_alias))]`
- [x] Annotate `GroupCommand::Add { name }` with `#[arg(add = ArgValueCompleter::new(complete_group_name))]`
- [x] Annotate `GroupCommand::Remove { name }` with `#[arg(add = ArgValueCompleter::new(complete_group_name))]`
- [x] Annotate `PullArgs::group` and `FetchArgs::group` with `#[arg(long, add = ArgValueCompleter::new(complete_group_name))]`
  > **Deviation:** the plan referenced a single `ExecArgs::group` field, but
  > the codebase has `PullArgs` and `FetchArgs` as separate structs (each with
  > its own `group: Option<String>`). Both are annotated.
- [x] Annotate `StatusArgs::group` with `#[arg(long, add = ArgValueCompleter::new(complete_group_name))]`
- [x] Annotate `WorktreesArgs::group` with `#[arg(long, add = ArgValueCompleter::new(complete_group_name))]`
- [x] Annotate `BranchesArgs::group` with `#[arg(long, add = ArgValueCompleter::new(complete_group_name))]`
- [x] Annotate `TrimArgs::group` with `#[arg(long, add = ArgValueCompleter::new(complete_group_name))]`

#### 3. Wire up `CompleteEnv` and remove static completions

**File**: `src/main.rs`

- [x] Add `use clap_complete::CompleteEnv;` import
- [x] Add a `cli()` factory function that returns `Cli::command()`:
  > **Deviation:** the helper was removed during code review — `Cli::command`
  > already satisfies `Fn() -> clap::Command`, so the indirection wasn't
  > needed. Call site is `CompleteEnv::with_factory(Cli::command).complete();`.
- [x] Insert `CompleteEnv::with_factory(Cli::command).complete();` as the first line of
      `main()`, before `Cli::parse()`
- [x] Remove the `Completions` variant handling block (`main.rs:22-31`)
- [x] Remove the `Command::Completions { .. }` arm from the `unreachable!` match
      at `main.rs:57`

**File**: `src/cli.rs`

- [x] Remove the `Completions` variant from the `Command` enum (`cli.rs:51-54`)
- [x] Remove the `use clap_complete::Shell;` import (`cli.rs:4`)

#### 4. Update README

**File**: `README.md`

- [x] Replace the "Shell Completions" section (`README.md:174-180`) — remove
      the `gityard completions <shell>` usage and replace with the new
      `source <(COMPLETE=<shell> gityard)` pattern for bash, zsh, and fish

#### Tests

- [x] Verify `just check` passes (format + lint + build + test + license check)
- [x] Verify no existing tests reference the removed `Completions` variant

### Success Criteria

#### Automated Verification

- [x] `just check` passes: format + lint + build + test + license check
- [x] `cargo build` succeeds with the `unstable-dynamic` feature enabled
- [x] No compiler warnings about unused imports after removing `Shell`/`Completions`

#### Manual Verification

- [ ] `COMPLETE=zsh gityard` outputs a shell function (not an error)
- [ ] In a shell with completions sourced, `gityard remove <tab>` suggests repo aliases
- [ ] In a shell with completions sourced, `gityard status --group <tab>` suggests group names
- [ ] `gityard completions` is no longer a recognized subcommand
  > **Note (automated):** `COMPLETE=zsh gityard` confirmed to emit a valid zsh
  > completion function; `gityard --help` confirmed to no longer list
  > `completions`. Full in-shell tab-completion test requires user verification.

## Performance Considerations

Each tab press invokes the binary and loads `repos.toml`. For typical registries
(tens to low hundreds of entries), this is a small TOML parse that completes in
single-digit milliseconds — well within acceptable tab-completion latency. No
caching is needed at this scale.

## Documentation Updates

The README's shell completion setup section needs to be updated to document the
new `source <(COMPLETE=<shell> gityard)` pattern. This is included as a task in
the implementation section above.

## References

- Idea document: `thoughts/ideas/2026-04-16-gityard-dynamic-shell-completions.md`
- [clap_complete dynamic completions docs](https://docs.rs/clap_complete/latest/clap_complete/env/index.html)
- [ArgValueCompleter docs](https://docs.rs/clap_complete/latest/clap_complete/engine/struct.ArgValueCompleter.html)
- Current static completions: `src/main.rs:22-31`
- CLI argument definitions: `src/cli.rs`
- Registry data source: `src/registry.rs`
- Config dir resolution: `src/config.rs:15-22`
