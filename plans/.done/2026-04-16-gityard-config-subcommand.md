# gityard config subcommand — Implementation Plan

## Overview

Add a `config` subcommand to gityard with three sub-subcommands: `show`, `path`,
and `edit`. This surfaces the currently invisible config system so users can
discover available settings, find the config file, and edit it conveniently.

Idea document: `$THOUGHTS_DIR/ideas/2026-04-16-gityard-config-subcommand.md`

## Current State Analysis

- Config is loaded in `main()` via `Config::load(&config_dir)` and passed by
  reference to command functions (`src/main.rs:33-34`)
- `Config` derives only `Deserialize` (no `Serialize`) — `src/config.rs:19`
- All config structs use `#[serde(default, deny_unknown_fields)]`
- The `Group` command demonstrates the sub-subcommand pattern with a nested
  `GroupCommand` enum (`src/cli.rs:43-44, 178-195`)
- `Completions` demonstrates the early-return pattern for commands that don't
  need config/registry (`src/main.rs:22-31`)
- No existing pattern for spawning `$EDITOR` — closest is `executor::run_git`
  using `std::process::Command`
- `toml` crate (0.8) supports both `toml::from_str` and `toml::to_string_pretty`
  (used in `registry.rs:54`)

### Key Discoveries:

- `config show` needs to distinguish user-set values from defaults. Since
  `Config` eagerly applies `#[serde(default)]`, the loaded struct doesn't
  preserve this information. The approach: parse the raw file as a
  `toml::Value` table and check which keys are present at each section level.
- Adding `Serialize` to `Config` is needed for `--toml` output. This is safe
  since all field types already implement `Serialize`.
- The `config` command needs `config_dir` but only `show` needs the loaded
  `Config`. The dispatch can happen before or after config loading — there's no
  need for an early-return like `Completions`.

## Desired End State

Three working subcommands:

```
$ gityard config show          # annotated TOML with (default) markers
$ gityard config show --toml   # valid TOML, pipeable
$ gityard config path          # prints config file path
$ gityard config edit          # opens $EDITOR; bootstraps template if needed
```

### Verification:

- `just check` passes (format, lint, build, test, licenses)
- `gityard config show` displays all config fields with `# (default)` on
  unmodified values
- `gityard config show --toml` produces valid TOML parseable by `toml::from_str`
- `gityard config path` prints the config file path
- `gityard config edit` opens the editor and creates a template if no file exists

## What We're NOT Doing

- get/set/unset operations (deferred — goal unclear)
- Config validation subcommand
- JSON output format
- Config init as a standalone subcommand (edit handles bootstrapping)

## Implementation

### Overview

This is a single atomic change: add the `ConfigCommand` enum to cli.rs, the
`cmd_config` dispatch function to main.rs, add `Serialize` to the config
structs, and implement the formatting logic in config.rs. All three subcommands
are simple and interdependent (they share the formatting/template logic).

### Changes Required:

#### 1. Add `Serialize` derive to config structs

**File**: `src/config.rs`

- [x] Add `use serde::{Deserialize, Serialize};` (line 4)
- [x] Add `Serialize` to the derive for `Config`, `StatusConfig`, `ScanConfig`,
      `DefaultsConfig`, and `FetchConfig`

#### 2. Add `config show` formatting functions to `config.rs`

**File**: `src/config.rs`

Following the "colocate code with its consumer" principle, the formatting logic
lives in `config.rs` since it operates directly on `Config` internals and the
raw TOML table.

- [x] Add a `config_path(dir: &Path) -> PathBuf` public helper that returns
      `dir.join("config.toml")` (centralizes the filename)
- [x] Add `Config::format_annotated(&self, raw: &toml::Table) -> Result<String>` that
      produces the TOML-like output with `# (default)` annotations
  > **Deviation:** Changed return type from `String` to `Result<String>` to
  > satisfy the `clippy::expect_used` lint — `expect()` is denied in non-test
  > code.
- [x] Add `Config::format_toml(&self) -> Result<String>` that calls
      `toml::to_string_pretty(self)` for the `--toml` flag
- [x] Add `fn default_template() -> &'static str` that returns the commented
      defaults template string (a `const` or `static` str) for `config edit`
      bootstrapping

The `format_annotated` approach:

```rust
pub fn format_annotated(&self, raw: &toml::Table) -> String {
    // Serialize self to toml::Value::Table to get all effective values
    // For each section, for each key:
    //   - emit "key = value" using TOML formatting
    //   - if the key is NOT in `raw[section]`, append "  # (default)"
    // Handle Option<T> fields: skip None values entirely (they represent
    // "unset" semantics, not a default value)
}
```

For `Option<T>` fields like `fetch.tags`, `fetch.jobs`, `fetch.depth`: when the
effective value is `None`, skip the key in the annotated output (matching the
behavior shown in the idea document where `tags`, `jobs`, and `depth` are
omitted from the example output). In the `--toml` output,
`toml::to_string_pretty` already skips `None` fields when using
`#[serde(skip_serializing_if = "Option::is_none")]`.

- [x] Add `#[serde(skip_serializing_if = "Option::is_none")]` to `FetchConfig`
      fields `tags`, `jobs`, and `depth`

#### 3. Define `ConfigCommand` enum in cli.rs

**File**: `src/cli.rs`

- [x] Add `ConfigCommand` enum following the `GroupCommand` pattern:

```rust
/// Subcommands for configuration management.
#[derive(Debug, Subcommand)]
pub enum ConfigCommand {
    /// Display the effective configuration (defaults + overrides)
    Show {
        /// Output valid TOML instead of annotated human-readable format
        #[arg(long)]
        toml: bool,
    },
    /// Print the path to the configuration file
    Path,
    /// Open the configuration file in $VISUAL or $EDITOR
    Edit,
}
```

- [x] Add `Config` variant to the `Command` enum:

```rust
/// View or edit the configuration
#[command(subcommand)]
Config(ConfigCommand),
```

Note: the variant name `Config` shadows the `config::Config` struct in
`main.rs`, but this is fine — the struct is already imported as
`gityard::config::Config` and can be disambiguated if needed. However, to
avoid any confusion at the import site, consider the clap `name` attribute
if there's a conflict. Check whether `cli::Command::Config` and
`gityard::config::Config` collide in main.rs imports.

#### 4. Add `cmd_config` dispatch and import to main.rs

**File**: `src/main.rs`

- [x] Add `ConfigCommand` to the import from `cli` (line 8-11)
- [x] Add match arm in the `match cli.command` block:
      `Command::Config(cmd) => cmd_config(&config_dir, &config, &cmd),`
- [x] Implement `cmd_config`:

```rust
fn cmd_config(config_dir: &Path, config: &Config, cmd: &ConfigCommand) -> Result<()> {
    match cmd {
        ConfigCommand::Show { toml } => {
            if *toml {
                println!("{}", config.format_toml()?);
            } else {
                let raw = load_raw_config(config_dir)?;
                println!("{}", config.format_annotated(&raw));
            }
            Ok(())
        }
        ConfigCommand::Path => {
            println!("{}", config::config_path(config_dir).display());
            Ok(())
        }
        ConfigCommand::Edit => {
            let path = config::config_path(config_dir);
            if !path.exists() {
                std::fs::create_dir_all(config_dir)
                    .context("failed to create config directory")?;
                std::fs::write(&path, config::default_template())
                    .context("failed to write default config")?;
            }
            let editor = std::env::var("VISUAL")
                .or_else(|_| std::env::var("EDITOR"))
                .map_err(|_| anyhow::anyhow!(
                    "no editor configured: set $VISUAL or $EDITOR"
                ))?;
            let status = std::process::Command::new(&editor)
                .arg(&path)
                .status()
                .with_context(|| format!("failed to launch editor: {editor}"))?;
            if !status.success() {
                anyhow::bail!("editor exited with {status}");
            }
            Ok(())
        }
    }
}
```

- [x] Add `load_raw_config` helper:

```rust
/// Load the raw TOML table from the config file (empty table if file missing).
fn load_raw_config(config_dir: &Path) -> Result<toml::Table> {
    let path = config::config_path(config_dir);
    if !path.exists() {
        return Ok(toml::Table::new());
    }
    let content = std::fs::read_to_string(&path)
        .with_context(|| format!("failed to read {}", path.display()))?;
    content.parse::<toml::Table>()
        .with_context(|| format!("failed to parse {}", path.display()))
}
```

#### Tests

**File**: `src/config.rs` (unit tests)

- [x] Test `format_annotated` with empty raw table — all values should have
      `# (default)` annotations
- [x] Test `format_annotated` with partial raw table — only user-set values
      should lack the annotation
- [x] Test `format_toml` produces valid TOML that round-trips through
      `toml::from_str::<Config>`
- [x] Test `format_toml` with `Option::None` fields — ensure they are omitted
- [x] Test `default_template` is not empty and contains expected section headers
- [x] Test `config_path` returns the expected path

**File**: `tests/cli.rs` (integration tests)

- [x] Test `config path` outputs a path ending in `config.toml`
- [x] Test `config show` with no config file shows all defaults with
      `# (default)` markers
- [x] Test `config show` with a partial config file shows user values without
      markers and defaults with markers
- [x] Test `config show --toml` produces valid TOML (parse the output)
- [x] Test `config show --toml` output round-trips: parse the output as
      `Config` and compare field values against defaults
- [x] Test `config edit` without `$VISUAL`/`$EDITOR` set produces an error
- [x] Test `config edit` creates the template file when no config exists
      (set `EDITOR=true` to make the editor a no-op, then check the file was
      created with expected content)

### Success Criteria:

#### Automated Verification:

- [x] `just check` passes (format + lint + build + test + license check)
- [x] All new unit tests in `config.rs` pass
- [x] All new integration tests in `tests/cli.rs` pass
- [x] `cargo clippy --all-targets` produces no warnings

#### Manual Verification:

- [x] `gityard config show` displays the expected annotated output with a
      real config file that has some user overrides
- [x] `gityard config edit` opens the correct file in `$EDITOR`

#### Post-Review Fix:

- [x] Config subcommand dispatched before `Config::load` so `config edit` and
      `config path` work even with an invalid config file
  > **Deviation:** The plan originally had `cmd_config` receiving `&Config` from
  > `main()`. Changed to lazy loading inside `cmd_config` so that `path` and
  > `edit` bypass config parsing entirely. Added 2 integration tests for
  > invalid-config resilience.

## References

- Idea document: `$THOUGHTS_DIR/ideas/2026-04-16-gityard-config-subcommand.md`
- Config implementation: `src/config.rs`
- CLI definitions: `src/cli.rs`
- Command dispatch: `src/main.rs`
- Sub-subcommand pattern: `GroupCommand` in `src/cli.rs:178-195`
- Registry serialization pattern: `src/registry.rs:40-61`
