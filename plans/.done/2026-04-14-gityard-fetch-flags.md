# gityard fetch: Comprehensive Flag Support — Implementation Plan

## Overview

Add 10 user-controllable flags to `gityard fetch` with a `[fetch]` config
section for persistent global defaults. CLI flags override config values, config
values override hardcoded defaults. All flags are pure git passthroughs — no
pre-flight implications.

## Current State Analysis

- `Operation::Fetch` is a unit variant with no fields (`operation.rs:8`)
- `Command::Fetch(ExecArgs)` uses the shared args directly (`cli.rs:35`)
- `git_args()` returns a hardcoded `vec!["fetch", "--prune"]` (`operation.rs:91`)
- `preflight()` unconditionally returns `Ready` (`operation.rs:45`) — unchanged
  by this work
- Config has three sections (`status`, `scan`, `defaults`) with
  `deny_unknown_fields` on every struct (`config.rs:19-25`)
- The `Push` command demonstrates the pattern: `PushArgs` wraps `ExecArgs` via
  `#[command(flatten)]`, adds `--force`, dispatch destructures into
  `Operation::Push { force }` (`cli.rs:69-77`, `main.rs:41-46`)

### Key Discoveries

- `deny_unknown_fields` on `Config` means adding `[fetch]` to `config.toml`
  today is a parse error — we must add the field to the struct
- `executor.rs:75` calls `operation::git_args(op)` once and shares the result
  across all threads — the args vector is computed once, not per-repo
- `Display` for `Operation` is used in `main.rs:255` (`eprintln!("running:
  gityard {op}")`) — it should reflect all flags being passed to git
- Git's default tag behavior without `--tags` is "auto-following" (fetches tags
  pointing to downloaded objects). Explicit `--tags` fetches ALL tags. These are
  distinct behaviors — `tags` must be three-state, not bool.

## Desired End State

`gityard fetch` accepts these flags, all passed through to the git command:

| Flag | CLI | Config | Default |
|------|-----|--------|---------|
| `--all` | `--all` | `all` | `false` |
| `--tags` / `--no-tags` | `--tags` / `--no-tags` | `tags` | auto (no flag) |
| `--force` / `--no-force` | `--force` / `--no-force` | `force` | `false` |
| `--prune` / `--no-prune` | `--prune` / `--no-prune` | `prune` | `true` |
| `--prune-tags` | `--prune-tags` | `prune_tags` | `false` |
| `--recurse-submodules` | `--recurse-submodules` | `recurse_submodules` | `false` |
| `--dry-run` | `--dry-run` | — | `false` |
| `--jobs=N` | `--jobs <N>` | `jobs` | unset |
| `--depth=N` | `--depth <N>` | `depth` | unset |
| `--unshallow` | `--unshallow` | — | `false` |

Verification:

```sh
# Default behavior preserved (only --prune, tags use git's auto-follow)
gityard fetch

# Explicit tag control
gityard fetch --tags          # fetch ALL tags
gityard fetch --no-tags       # disable tag fetching entirely

# All flags work
gityard fetch --all --force --prune-tags --jobs 4
gityard fetch --no-tags --no-prune
gityard fetch --dry-run --depth 1
gityard fetch --unshallow --recurse-submodules

# Config defaults apply
cat ~/.config/gityard/config.toml
# [fetch]
# all = true
# tags = true
gityard fetch  # equivalent to: gityard fetch --all --tags --prune

# CLI overrides config
gityard fetch --no-tags    # overrides config tags = true
gityard fetch --no-force   # overrides config force = true

# Existing flags still work
gityard fetch --group mygroup --allow-partial
```

```sh
just check  # all tests, clippy, fmt pass
```

## What We're NOT Doing

- Per-group fetch config defaults
- Repo-level parallelism control (already handled by `thread::scope`)
- Refspec support
- `--recurse-submodules=on-demand` mode — the flag is a simple bool toggle, not
  a multi-value option. `on-demand` support can be added later.
- Output verbosity flags (`--quiet`, `--verbose`)
- Changes to `preflight()` — fetch remains unconditionally `Ready`
- `--dry-run` and `--unshallow` in config (these are per-invocation only)

## Implementation Approach

Three flag categories require different merge strategies:

1. **Three-state flags** (`tags`): `Option<bool>` in config and `Operation`.
   `None` = git's default behavior (no flag passed), `Some(true)` = explicit
   `--tags`, `Some(false)` = explicit `--no-tags`. The CLI uses a pair of bool
   args (`--tags` / `--no-tags`) that produce `Some(true)`, `Some(false)`, or
   fall through to config.

2. **Default-on bool with negation** (`prune`): `bool` with default `true`.
   Uses the same CLI pair pattern (`--prune` / `--no-prune`), but the type is
   `bool` because `--prune` is already hardcoded in the current behavior — no
   "auto" mode exists.

3. **Default-off bool with negation** (`force`): `bool` with default `false`.
   The `--no-force` override exists so users can override a config `force = true`
   on the CLI without editing the config file.

4. **Simple default-off bool** (`all`, `prune_tags`, `recurse_submodules`,
   `dry_run`, `unshallow`): `cli_flag || config_value`. No negation needed.

5. **Optional value** (`jobs`, `depth`): `Option<usize>`, CLI takes precedence
   via `.or()`. `--unshallow` clears any config-derived `depth`.

The `Display` impl mirrors `git_args()` — it shows all flags that will be
passed to git, not just deviations from defaults. This keeps the
`"running: gityard fetch --prune"` message transparent and debuggable.

## Implementation

### Overview

Add `FetchConfig` to config, `FetchArgs` to CLI, extend `Operation::Fetch` with
flag fields, update dispatch to merge CLI + config, and update `git_args()` to
build the args vector conditionally.

### Changes Required

#### 1. Config: Add `FetchConfig`

**File**: `src/config.rs`

- [x] Add `FetchConfig` struct with fields: `all: bool`, `tags: Option<bool>`,
  `force: bool`, `prune: bool`, `prune_tags: bool`, `recurse_submodules: bool`,
  `jobs: Option<usize>`, `depth: Option<usize>`
  - `#[serde(default, deny_unknown_fields)]` to match existing config structs
  - `Default` impl: `all=false`, `tags=None`, `force=false`, `prune=true`,
    `prune_tags=false`, `recurse_submodules=false`, `jobs=None`, `depth=None`
  > **Deviation:** Also needed `#[allow(clippy::struct_excessive_bools)]` on
  > `FetchConfig` — same as `FetchArgs`.
- [x] Add `pub fetch: FetchConfig` field to `Config` struct

```rust
/// Configuration defaults for the `fetch` command.
#[derive(Debug, Deserialize)]
#[serde(default, deny_unknown_fields)]
pub struct FetchConfig {
    /// Fetch from all remotes.
    pub all: bool,
    /// Tag fetching mode: `None` = git auto-follow (default), `Some(true)` =
    /// fetch all tags (`--tags`), `Some(false)` = no tags (`--no-tags`).
    pub tags: Option<bool>,
    /// Force-update local refs.
    pub force: bool,
    /// Prune remote-tracking refs that no longer exist on the remote.
    pub prune: bool,
    /// Prune local tags that no longer exist on the remote.
    pub prune_tags: bool,
    /// Recurse into submodules.
    pub recurse_submodules: bool,
    /// Number of parallel children for fetching submodules/remotes.
    pub jobs: Option<usize>,
    /// Limit fetching to the specified number of commits from each remote
    /// branch tip.
    pub depth: Option<usize>,
}

impl Default for FetchConfig {
    fn default() -> Self {
        Self {
            all: false,
            tags: None,
            force: false,
            prune: true,
            prune_tags: false,
            recurse_submodules: false,
            jobs: None,
            depth: None,
        }
    }
}
```

#### 2. CLI: Add `FetchArgs`

**File**: `src/cli.rs`

- [x] Add `FetchArgs` struct that wraps `ExecArgs` via `#[command(flatten)]` and
  adds all fetch-specific flags
  - Three-state and default-on flags (`tags`, `prune`, `force`) use a pair of
    bool args with `conflicts_with`
  - `--dry-run` and `--unshallow` are simple bool flags (not in config)
  - `--jobs` and `--depth` are `Option<usize>`
- [x] Change `Command::Fetch(ExecArgs)` to `Command::Fetch(FetchArgs)`
- [x] Add `#[allow(clippy::struct_excessive_bools)]` to `FetchArgs` (same
  pattern as `StatusArgs` at `cli.rs:90`)

```rust
/// Arguments for the `fetch` subcommand.
#[derive(Debug, Parser)]
#[allow(clippy::struct_excessive_bools)]
pub struct FetchArgs {
    #[command(flatten)]
    pub exec: ExecArgs,

    /// Fetch from all remotes
    #[arg(long)]
    pub all: bool,

    /// Fetch all tags (by default, git auto-follows tags on fetched commits)
    #[arg(long, conflicts_with = "no_tags")]
    pub tags: bool,

    /// Disable tag fetching entirely
    #[arg(long = "no-tags", conflicts_with = "tags")]
    pub no_tags: bool,

    /// Force-update local refs
    #[arg(long, conflicts_with = "no_force")]
    pub force: bool,

    /// Disable force-updating local refs (overrides config)
    #[arg(long = "no-force", conflicts_with = "force")]
    pub no_force: bool,

    /// Prune remote-tracking refs (on by default; use --no-prune to disable)
    #[arg(long, conflicts_with = "no_prune")]
    pub prune: bool,

    /// Disable pruning of remote-tracking refs
    #[arg(long = "no-prune", conflicts_with = "prune")]
    pub no_prune: bool,

    /// Prune local tags that no longer exist on the remote
    #[arg(long)]
    pub prune_tags: bool,

    /// Recurse into submodules
    #[arg(long)]
    pub recurse_submodules: bool,

    /// Show what would be fetched without actually fetching
    #[arg(long)]
    pub dry_run: bool,

    /// Number of parallel children for submodule/remote fetching
    #[arg(long)]
    pub jobs: Option<usize>,

    /// Limit fetch depth to N commits from each remote branch tip
    #[arg(long)]
    pub depth: Option<usize>,

    /// Convert a shallow repository to a complete one
    #[arg(long, conflicts_with = "depth")]
    pub unshallow: bool,
}
```

#### 3. Operation: Extend `Operation::Fetch`

**File**: `src/operation.rs`

- [x] Change `Operation::Fetch` from a unit variant to a struct variant with
  resolved flag fields. `tags` is `Option<bool>` (three-state); all other bools
  are resolved to their final values.

```rust
pub enum Operation {
    Fetch {
        all: bool,
        tags: Option<bool>,
        force: bool,
        prune: bool,
        prune_tags: bool,
        recurse_submodules: bool,
        dry_run: bool,
        jobs: Option<usize>,
        depth: Option<usize>,
        unshallow: bool,
    },
    Pull,
    Push { force: bool },
    Checkout { branch: String },
}
```

- [x] Update `Display` impl for `Fetch` to show all flags being passed to git
  (mirrors `git_args()` output):

```rust
Operation::Fetch {
    all,
    tags,
    force,
    prune,
    prune_tags,
    recurse_submodules,
    dry_run,
    jobs,
    depth,
    unshallow,
} => {
    write!(f, "fetch")?;
    if *all { write!(f, " --all")?; }
    match tags {
        Some(true) => write!(f, " --tags")?,
        Some(false) => write!(f, " --no-tags")?,
        None => {}
    }
    if *force { write!(f, " --force")?; }
    if *prune { write!(f, " --prune")?; }
    if *prune_tags { write!(f, " --prune-tags")?; }
    if *recurse_submodules { write!(f, " --recurse-submodules")?; }
    if *dry_run { write!(f, " --dry-run")?; }
    if let Some(n) = jobs { write!(f, " --jobs={n}")?; }
    if let Some(n) = depth { write!(f, " --depth={n}")?; }
    if *unshallow { write!(f, " --unshallow")?; }
    Ok(())
}
```

- [x] Update `preflight()` — change the pattern match from `Operation::Fetch` to
  `Operation::Fetch { .. }` (behavior stays the same: `PreflightResult::Ready`)

- [x] Update `git_args()` to build the args vector conditionally:

```rust
Operation::Fetch {
    all,
    tags,
    force,
    prune,
    prune_tags,
    recurse_submodules,
    dry_run,
    jobs,
    depth,
    unshallow,
} => {
    let mut args = vec!["fetch".to_string()];
    if *all { args.push("--all".to_string()); }
    match tags {
        Some(true) => args.push("--tags".to_string()),
        Some(false) => args.push("--no-tags".to_string()),
        None => {}
    }
    if *force { args.push("--force".to_string()); }
    if *prune { args.push("--prune".to_string()); }
    if *prune_tags { args.push("--prune-tags".to_string()); }
    if *recurse_submodules { args.push("--recurse-submodules".to_string()); }
    if *dry_run { args.push("--dry-run".to_string()); }
    if let Some(n) = jobs { args.push(format!("--jobs={n}")); }
    if let Some(n) = depth { args.push(format!("--depth={n}")); }
    if *unshallow { args.push("--unshallow".to_string()); }
    args
}
```

#### 4. Dispatch: Merge CLI + Config in `main.rs`

**File**: `src/main.rs`

- [x] Update the `Command::Fetch` arm to destructure `FetchArgs`, merge with
  config, and construct `Operation::Fetch { ... }`
- [x] Add `FetchArgs` to the import from `cli`
- [x] Extract the merge into a testable function:
  `resolve_fetch_op(args: &FetchArgs, config: &FetchConfig) -> Operation`
- [x] The merge logic for each flag type:
  - **Three-state** (`tags`): CLI `--tags` → `Some(true)`, CLI `--no-tags` →
    `Some(false)`, unspecified → config value (which defaults to `None`)
  - **Default-on bool with negation** (`prune`): CLI positive → `true`, CLI
    negative → `false`, unspecified → config value
  - **Default-off bool with negation** (`force`): same three-way pattern as
    prune
  - **Simple default-off bool** (`all`, `prune_tags`, `recurse_submodules`):
    `cli_flag || config_value`
  - **Option<usize>** (`jobs`, `depth`): `cli_value.or(config_value)`, with
    `depth` cleared when `unshallow` is true
  - **CLI-only bool** (`dry_run`, `unshallow`): pass through directly

```rust
Command::Fetch(args) => {
    let op = resolve_fetch_op(&args, &config.fetch);
    cmd_exec(&config_dir, &config, &args.exec, &op)
}
```

```rust
fn resolve_fetch_op(args: &FetchArgs, fc: &FetchConfig) -> Operation {
    Operation::Fetch {
        all: args.all || fc.all,
        tags: if args.tags {
            Some(true)
        } else if args.no_tags {
            Some(false)
        } else {
            fc.tags
        },
        force: if args.force {
            true
        } else if args.no_force {
            false
        } else {
            fc.force
        },
        prune: if args.prune {
            true
        } else if args.no_prune {
            false
        } else {
            fc.prune
        },
        prune_tags: args.prune_tags || fc.prune_tags,
        recurse_submodules: args.recurse_submodules || fc.recurse_submodules,
        dry_run: args.dry_run,
        jobs: args.jobs.or(fc.jobs),
        depth: if args.unshallow {
            None
        } else {
            args.depth.or(fc.depth)
        },
        unshallow: args.unshallow,
    }
}
```

#### Tests

**File**: `src/operation.rs` (inline tests)

- [x] Update `test_fetch_always_ready` to use the new struct variant pattern
- [x] Add `test_fetch_git_args_defaults` — verify that default flags
  (`tags=None`, `prune=true`, all others off/None) produce
  `["fetch", "--prune"]` (preserves current behavior)
- [x] Add `test_fetch_git_args_all_flags` — verify that all flags enabled
  produces the full args vector with correct flag strings
- [x] Add `test_fetch_git_args_tags_three_state` — verify `None` omits tag
  flags, `Some(true)` emits `--tags`, `Some(false)` emits `--no-tags`
- [x] Add `test_fetch_git_args_negations` — verify `prune=false` omits
  `--prune`, `force=false` omits `--force`
- [x] Add `test_fetch_git_args_unshallow_without_depth` — verify
  `unshallow=true, depth=None` produces `--unshallow` without `--depth`
- [x] Add `test_fetch_git_args_jobs_and_depth` — verify `--jobs=N` and
  `--depth=N` formatting
- [x] Add `test_fetch_display_defaults` — verify `Display` output for default
  flags is `"fetch --prune"` (only prune is default-on)
- [x] Add `test_fetch_display_with_flags` — verify `Display` output includes
  all active flags like `"fetch --tags --force --prune"`
- [x] Add `test_fetch_display_no_tags` — verify `Display` output shows
  `"fetch --no-tags"` when tags is `Some(false)`

**File**: `src/config.rs` (inline tests)

- [x] Add `test_fetch_config_defaults` — verify `FetchConfig::default()` has
  `tags=None`, `prune=true`, all others off/None
- [x] Add `test_fetch_config_parses` — verify a `[fetch]` TOML section parses
  correctly with `tags = true`, `tags = false`, and omitted `tags`
- [x] Add `test_fetch_config_rejects_unknown_fields` — verify
  `deny_unknown_fields` works on `FetchConfig`

**File**: `src/main.rs` (merge logic — `resolve_fetch_op`)

- [x] Add `test_merge_cli_tags_overrides_config` — verify `--tags` overrides
  config `tags = None` and `tags = false`; `--no-tags` overrides config
  `tags = true`; unspecified falls through to config value
- [x] Add `test_merge_cli_force_overrides_config` — verify `--force` overrides
  config `force = false`; `--no-force` overrides config `force = true`;
  unspecified falls through to config
- [x] Add `test_merge_cli_prune_overrides_config` — verify same three-way
  pattern for prune
- [x] Add `test_merge_unshallow_clears_config_depth` — verify that
  `--unshallow` with config `depth = 5` results in `depth: None`
- [x] Add `test_merge_default_off_bools_or_with_config` — verify simple
  `||` merge for `all`, `prune_tags`, `recurse_submodules`

**File**: `tests/cli.rs` (integration tests)

- [x] Add test that `gityard fetch --help` lists all new flags
- [x] Add test that `--tags` and `--no-tags` conflict (exits with error)
- [x] Add test that `--force` and `--no-force` conflict
- [x] Add test that `--prune` and `--no-prune` conflict
- [x] Add test that `--depth` and `--unshallow` conflict

### Success Criteria

#### Automated Verification

- [x] `just check` passes (format + lint + build + test + license check)
- [x] `cargo test` — all new and existing tests pass
- [x] `cargo clippy --all-targets` — no warnings
- [x] `cargo fmt --check` — properly formatted
- [x] `gityard fetch --help` shows all new flags with descriptions
- [x] `gityard fetch` with no flags produces `git fetch --prune` (preserves
  current behavior, verified by test)

## References

- Idea document: `thoughts/ideas/2026-04-14-gityard-fetch-flags.md`
- Existing pattern: `PushArgs` in `cli.rs:69-77`, `Operation::Push` in
  `operation.rs:93-98`
- Config structure: `config.rs:19-25`
- Dispatch: `main.rs:34-61`
- Executor: `executor.rs:75` (git_args computed once, shared across threads)
