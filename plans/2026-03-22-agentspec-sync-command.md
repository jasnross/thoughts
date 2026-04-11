# `agentspec sync` Command — Implementation Plan

## Overview

Add an `agentspec sync` subcommand that compiles specs and distributes the
generated output to each tool's expected config directory in a single
invocation. Sync targets (destination, strategy, spec kinds) are configurable
in `agentspec.toml`, with profile overrides and CLI flag escapes. This replaces
the generated-file distribution logic currently in `agent-config/setup.sh`.

## Current State Analysis

- `agentspec compile` runs the full spec pipeline and writes
  `generated/<provider>/<kind>/` files.
- `agent-config/setup.sh` distributes those files via:
  - `sync_symlinked_children` — creates symlinks from each tool's config dir
    into `generated/`; cleans stale symlinks.
  - `copy_to_plugin` — copies Claude/Cursor generated files into
    `~/Workspace/thoughts/plugin/` when `AGENTSPEC_PROFILE=work`; strips
    `name:` lines from skill frontmatter for plugin namespace prefixing.
- Rules (`generated/*/rules/`) are compiled but **not yet distributed** by
  `setup.sh` — a gap the sync command will fill.
- `generated/opencode/instructions.json` is currently produced by
  `build_opencode_instructions` in the compile pipeline. It lists rule paths
  as relative strings (e.g. `"generated/opencode/rules/foo/AGENTS.md"`) and
  is never distributed. This file is not an OpenCode concept — OpenCode reads
  rules via the `instructions` array in `opencode.json`. The compile step that
  produces it will be removed; the sync step will patch `opencode.json` instead.
- **Codex rules**: `~/.codex/rules/` is a Starlark execution-policy directory
  (not markdown). Codex reads markdown instructions from `~/.codex/AGENTS.md`
  (a single file). The Codex adapter currently emits individual `.md` files to
  `generated/codex/rules/` — there is no valid per-file sync destination.
  Codex rules are **out of scope** for this sync command; a separate fix to the
  Codex adapter is tracked in `TODO.md`.

## Desired End State

Running `agentspec sync` (or `agentspec sync --profile work`) from
`agent-config/` compiles and distributes all generated files to each
configured target. `setup.sh` retains only profile prompting, settings merges,
and static symlinks.

### Verification

- `agentspec sync --dry-run` prints all planned operations without touching
  the filesystem.
- `agentspec sync` runs end-to-end and the expected files appear in each
  tool's config dir.
- `agentspec sync --profile work` distributes to plugin dirs instead of
  user-level dirs.
- Re-running `agentspec sync` is idempotent (no spurious changes).

### Key Discoveries

- `config.rs:AgentspecConfig` holds `presets` and `profiles`; the latter uses
  `HashMap<String, HashMap<String, HashMap<String, serde_json::Value>>>`. The
  third-level key "sync" can carry sync overrides without any schema change —
  we add `resolve_sync_target()` that reads `profiles[name]["sync"][provider]`
  and deserializes it as a `SyncTargetConfig`.
- `cli.rs` `CommonArgs` already has `--target`, `--profile`, `--strict`;
  `SyncArgs` embeds `CommonArgs` and adds sync-specific flags.
- The `too_many_lines = "deny"` clippy rule requires the sync logic be split
  across sub-modules from the start.
- OpenCode skill: flat `commands/<id>.md` + nested `skills/<id>/SKILL.md`
  (dual output per skill invocability flags). Both subdirs are synced
  independently.
- OpenCode rules have no path-scoping equivalent — `paths:` is already dropped
  by the adapter (`adapters/opencode.rs:62`). All OpenCode rules are global.
- `strip_name` post-processing: the work-profile plugin copy strips `name:`
  from `*/SKILL.md` files so the plugin's namespace prefix (e.g. `tw:`) is
  applied instead of the explicit name.

## What We're NOT Doing

- Replacing `setup.sh`'s settings merge logic (`yq`-based JSON merging for
  `claude/settings.json` and `opencode.json` init).
- Replacing `setup.sh`'s static file symlinking (`AGENTS_GLOBAL.md`,
  `keybindings.json`).
- Interactive profile prompting inside `agentspec` — profile comes from
  `--profile` flag or `AGENTSPEC_PROFILE` env var.
- Tool detection guards — `agentspec sync` is unconditional; detection belongs
  in the caller (`setup.sh` or user's shell).
- Syncing Codex rules — the Codex adapter emits individual `.md` files but
  Codex expects a single `~/.codex/AGENTS.md`; fixing the adapter is a
  separate task.
- Windows support — macOS and Linux only; `#[cfg(unix)]` for symlink and
  permission APIs.

## Provider Conventions (hardcoded, `mode = "user"`)

| Provider  | Spec Kind | User-level destination                  | Notes                                      |
|-----------|-----------|-----------------------------------------|--------------------------------------------|
| Claude    | agents    | `~/.claude/agents/`                     |                                            |
| Claude    | skills    | `~/.claude/skills/`                     |                                            |
| Claude    | rules     | `~/.claude/rules/`                      |                                            |
| Codex     | skills    | `~/.codex/skills/`                      |                                            |
| Codex     | rules     | *(skipped — adapter fix required)*      | See TODO.md                                |
| Cursor    | skills    | `~/.cursor/skills/`                     |                                            |
| Cursor    | rules     | `~/.cursor/rules/`                      |                                            |
| OpenCode  | agents    | `~/.config/opencode/agents/`            |                                            |
| OpenCode  | commands  | `~/.config/opencode/commands/`          |                                            |
| OpenCode  | skills    | `~/.config/opencode/skills/`            |                                            |
| OpenCode  | rules     | `~/.config/opencode/rules/`             | Also patches `opencode.json` `instructions`|

For `mode = "project"`, replace `~/.<tool>/` with `./<tool_dir>/` from the
current working directory (`.claude/`, `.cursor/`, `.codex/`, `.opencode/`).

## Implementation Approach

Add a `sync` module (`src/sync.rs` + `src/sync/*.rs` sub-modules). Wire it
into `main.rs` after the existing compile pipeline. Extend `cli.rs` with
`SyncArgs` and `config.rs` with `SyncTargetConfig`. Shrink `setup.sh` after
the new command is verified.

---

## Phase 1: Config Schema Extension

### Overview

Add `SyncTargetConfig` and related types to `config.rs`. Extend
`AgentspecConfig` with a `sync` field and a `resolve_sync_target()` method
that merges base config, profile overrides, and CLI flags.

### Changes Required

#### 1. New types in `config.rs`

**File**: `agentspec/src/config.rs`

- [x] Add `SyncMode` enum (`User`, `Project`, `Path`); derive
  `Debug, Clone, Default, Deserialize` with `#[serde(rename_all = "lowercase")]`
  > **Deviation:** Placed in `types.rs` instead of `config.rs` to avoid circular
  > imports (`cli.rs` needs these for Phase 2 and already imports from `types.rs`).
  > Also derived `ValueEnum` (needed for Phase 2 CLI args).
- [x] Add `SyncStrategy` enum (`Symlink`, `Copy`); same derives
  > **Deviation:** Same as `SyncMode` — placed in `types.rs` with `ValueEnum`.
- [x] Add `SyncTargetConfig` struct:
  ```rust
  #[derive(Debug, Clone, Deserialize)]
  #[serde(default)]
  pub struct SyncTargetConfig {
      pub mode: SyncMode,           // default: User
      pub strategy: SyncStrategy,   // default: Symlink
      pub strip_name: bool,         // default: false
      // For mode = "path": per-kind explicit dirs (tilde-expanded at use site)
      pub agents: Option<String>,
      pub skills: Option<String>,
      pub rules: Option<String>,
      pub commands: Option<String>,
  }
  ```
- [x] Add `pub sync: HashMap<String, SyncTargetConfig>` field to
  `AgentspecConfig` (keyed by provider name string)
- [x] Update `AgentspecConfig::default()` to include `sync: HashMap::new()`

#### 2. Profile overlay support

**File**: `agentspec/src/config.rs`

- [x] Add `resolve_sync_target(provider: Provider, active_profile: Option<&str>, cli: &SyncOverrides) -> SyncTargetConfig`:
  - Start with `self.sync.get(provider_str).cloned().unwrap_or_default()`
  - If profile active, look up `self.profiles[profile_name]["sync"][provider_name]`
    and deserialize via `serde_json::from_value` into a partial
    `SyncTargetConfig`; apply field-by-field (only `Some`/non-default values
    override)
  - Apply `SyncOverrides` from CLI flags last (highest precedence)

#### 3. `SyncOverrides` struct for CLI flag passthrough

**File**: `agentspec/src/config.rs` (or `cli.rs` — whichever avoids circular deps)

- [x] Add `SyncOverrides` struct holding `Option<SyncMode>`, `Option<SyncStrategy>`, `Option<String>` (dest root), for CLI flag overrides

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds with no errors
- [x] `cargo clippy --all-targets` passes (no warnings, no lint violations)
- [x] `cargo test` passes
- [x] `cargo fmt --check` passes
- [x] Unit tests for `resolve_sync_target`:
  - [x] Returns default when no `[sync.*]` configured
  - [x] Applies base `[sync.claude]` config correctly
  - [x] Profile override merges over base (only specified fields)
  - [x] Unknown profile is a no-op (mirrors `resolve_presets` behavior)
  - [x] CLI `SyncOverrides` wins over profile

---

## Phase 2: CLI Extension

### Overview

Add `Sync(SyncArgs)` to the `Command` enum. `SyncArgs` embeds `CommonArgs`
and adds sync-specific flags.

### Changes Required

#### 1. `SyncArgs` and new command variant

**File**: `agentspec/src/cli.rs`

- [x] Add `SyncArgs` struct:
  ```rust
  #[derive(Parser, Debug)]
  pub struct SyncArgs {
      #[command(flatten)]
      pub common: CommonArgs,
      /// Skip compilation; sync from existing generated output
      #[arg(long)]
      pub no_compile: bool,
      /// Show what would be synced without making changes
      #[arg(long)]
      pub dry_run: bool,
      /// Override sync strategy for all providers
      #[arg(long, value_enum)]
      pub strategy: Option<SyncStrategy>,
      /// Override destination root (implies mode=path)
      #[arg(long)]
      pub dest: Option<String>,
      /// Override target mode for all providers
      #[arg(long, value_enum)]
      pub mode: Option<SyncMode>,
  }
  ```
- [x] Add `Sync(SyncArgs)` variant to `Command` enum
- [x] Update `Command::args()` to return `Some(args.common)` for `Sync`
- [x] `SyncMode` and `SyncStrategy` must implement `clap::ValueEnum` — add
  derive in `config.rs` or move enums to `types.rs` if needed to avoid
  circular imports
  > **Deviation:** Done in Phase 1; placed in `types.rs` with `ValueEnum` already derived.

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds
- [x] `agentspec sync --help` prints usage with all flags documented
- [x] `agentspec sync --dry-run --help` does not error
- [x] `cargo clippy --all-targets` passes
- [x] `cargo fmt --check` passes

---

## Phase 3: Provider Convention Tables + Path Resolution

### Overview

New `sync` module root (`src/sync.rs`) containing the hardcoded provider
convention tables and the path-resolution logic that maps
`(provider, spec_kind, SyncTargetConfig)` to an absolute destination directory.

### Changes Required

#### 1. Module root and provider tables

**File**: `agentspec/src/sync.rs`

- [x] Declare sub-modules: `pub mod config; mod strategy; mod provider; mod manifest;`
- [x] Re-export the public surface: `pub use config::...; pub use strategy::run_sync;`

**File**: `agentspec/src/sync/provider.rs`

- [x] Add `user_dest_dir(provider: Provider, kind: SyncKind, home: &Path) -> PathBuf`
  implementing the convention table (see table above)
- [x] Add `project_dest_dir(provider: Provider, kind: SyncKind, cwd: &Path) -> PathBuf`
  for `mode = "project"` (`.claude/`, `.cursor/`, etc.)
- [x] Add `SyncKind` enum: `Agents`, `Skills`, `Rules`, `Commands` (distinct
  from `SpecKind` since OpenCode emits `commands` as a separate dir even though
  specs use `Skill` kind)
- [x] Add `generated_source_dir(provider: Provider, kind: SyncKind, generated_root: &Path) -> PathBuf`
  — maps to `generated/<provider>/<kind_str>/`
- [x] Add `all_sync_kinds(provider: Provider) -> Vec<SyncKind>` — returns the
  spec kinds generated for a provider, skipping Codex rules (returns
  `[Skills]` for Codex until the adapter is fixed)
- [x] Add `expand_tilde(path: &str, home: &Path) -> PathBuf` — replaces leading
  `~/` with `home`; returns `PathBuf::from(path)` otherwise

#### 2. Destination resolution

**File**: `agentspec/src/sync/provider.rs`

- [x] Add `resolve_dest_dir(provider: Provider, kind: SyncKind, config: &SyncTargetConfig, home: &Path, cwd: &Path) -> Result<PathBuf>`:
  - `SyncMode::User` → `user_dest_dir(...)`
  - `SyncMode::Project` → `project_dest_dir(...)`
  - `SyncMode::Path` → look up the per-kind field on `SyncTargetConfig`
    (`agents`, `skills`, `rules`, `commands`); error if the kind is needed
    but the field is `None`; tilde-expand the value

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds
- [x] `cargo clippy --all-targets` passes
- [x] `cargo fmt --check` passes
- [x] Unit tests for `resolve_dest_dir`:
  - [x] `mode = user`, Claude agents → `~/.claude/agents/`
  - [x] `mode = project`, Cursor skills → `./.cursor/skills/`
  - [x] `mode = path` with explicit `skills = "~/foo"` → `$HOME/foo`
  - [x] `mode = path` with missing `agents` for an agents sync → error
- [x] Unit tests for `all_sync_kinds`:
  - [x] Codex returns `[Skills]` (rules skipped)
  - [x] OpenCode returns `[Agents, Commands, Skills, Rules]`
- [x] Unit tests for `expand_tilde`:
  - [x] `~/foo/bar` → `$HOME/foo/bar`
  - [x] `/absolute/path` → `/absolute/path` unchanged

---

## Phase 4: Symlink Strategy + Stale Cleanup

### Overview

Implement the symlink distribution strategy: create per-entry symlinks from
the destination dir into `generated/`, and clean up stale symlinks that point
into the source dir but whose targets no longer exist.

### Changes Required

**File**: `agentspec/src/sync/strategy.rs`

- [x] Add `ensure_symlink(target: &Path, link: &Path, dry_run: bool) -> Result<SyncAction>`:
  - If `link` is already a symlink pointing to `target`: return `SyncAction::Unchanged`
  - If `link` is a symlink pointing elsewhere: remove it, create new symlink; return `SyncAction::Updated`
  - If `link` is a regular file or dir: backup with timestamp suffix (`.bak.YYYYMMDDHHMMSS`), create symlink; return `SyncAction::BackedUp`
  - If `link` does not exist: create symlink; return `SyncAction::Created`
  - When `dry_run`: print planned action, return appropriate `SyncAction` without mutating filesystem
- [x] Add `SyncAction` enum: `Created`, `Updated`, `Unchanged`, `Removed`, `BackedUp`
- [x] Add `sync_symlinked_dir(source_dir: &Path, dest_dir: &Path, dry_run: bool) -> Result<Vec<SyncEntry>>`:
  - `mkdir -p dest_dir`
  - For each entry in `source_dir/*`: call `ensure_symlink(entry, dest_dir/name, dry_run)`
  - Stale cleanup pass: for each symlink in `dest_dir/*`, resolve to absolute
    target; if target starts with `source_dir` AND target no longer exists:
    remove (or print if dry_run)
  - Return `Vec<SyncEntry>` (path + action) for reporting
- [x] Add `SyncEntry` struct: `path: PathBuf`, `action: SyncAction`

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds
- [x] `cargo clippy --all-targets` passes
- [x] `cargo fmt --check` passes
- [x] Unit tests using `tempfile::TempDir`:
  - [x] New entry → symlink created pointing to source
  - [x] Existing correct symlink → no-op (`Unchanged`)
  - [x] Symlink pointing elsewhere → removed and recreated (`Updated`)
  - [x] Regular file in dest → backed up, symlink created
  - [x] Stale symlink (target deleted from source) → removed
  - [x] Non-stale symlink pointing outside source dir → preserved
  - [x] `dry_run = true` → no filesystem mutations, correct actions reported

---

## Phase 5: Copy Strategy + Manifest

### Overview

Implement the copy strategy with `.agentspec-manifest.json` for ownership
tracking, backup-before-overwrite on first sync, and `strip_name`
post-processing.

### Changes Required

#### 1. Manifest read/write

**File**: `agentspec/src/sync/manifest.rs`

- [x] Add `Manifest` struct:
  ```rust
  pub struct Manifest {
      pub version: u32,         // always 1
      pub files: HashMap<String, ManifestEntry>,  // rel path → entry
  }
  pub struct ManifestEntry {
      pub source: String,       // absolute path to generated file
  }
  ```
- [x] Add `Manifest::load(dir: &Path) -> Result<Self>` — reads
  `.agentspec-manifest.json` from `dir`; returns empty manifest if absent
- [x] Add `Manifest::save(&self, dir: &Path) -> Result<()>` — writes
  `.agentspec-manifest.json` to `dir`

#### 2. Copy strategy engine

**File**: `agentspec/src/sync/strategy.rs`

- [x] Add `copy_file(source: &Path, dest: &Path, manifest: &mut Manifest, dry_run: bool) -> Result<SyncAction>`:
  - Rel path = `dest` filename (for single-file entries) or subpath
  - If rel path NOT in manifest AND dest exists: backup, copy, record in manifest
  - If rel path in manifest AND content differs: warn, overwrite, update manifest
  - If rel path in manifest AND content same: `Unchanged`
  - If dest does not exist: copy, record in manifest
  - `dry_run`: print, skip mutations
- [x] Add `sync_copied_dir(source_dir: &Path, dest_dir: &Path, manifest: &mut Manifest, dry_run: bool) -> Result<Vec<SyncEntry>>`:
  - `mkdir -p dest_dir`
  - Walk `source_dir` recursively; for each file call `copy_file`
  - Stale cleanup: entries in `manifest.files` whose source paths are no
    longer present in the current generated output → remove dest file, remove
    from manifest (or print if dry_run)
- [x] Add `apply_strip_name(dest_dir: &Path, dry_run: bool) -> Result<()>`:
  - Walk `dest_dir/**/SKILL.md`; remove lines matching `^name: .*` from each
  - `dry_run`: print files that would be modified

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds
- [x] `cargo clippy --all-targets` passes
- [x] `cargo fmt --check` passes
- [x] Unit tests using `tempfile::TempDir`:
  - [x] No manifest, no dest file → file copied, manifest created
  - [x] No manifest, dest file exists → backup created, file copied
  - [x] Manifest entry, content same → `Unchanged`, no backup
  - [x] Manifest entry, content differs → overwritten with warning
  - [x] Stale manifest entry (source deleted) → dest file removed, entry removed
  - [x] `strip_name = true` → `name:` lines removed from `SKILL.md` files
  - [x] `dry_run = true` → no filesystem mutations

---

## Phase 6: OpenCode `opencode.json` `instructions` Patch

### Overview

After syncing OpenCode rules, patch the `instructions` array in
`~/.config/opencode/opencode.json` (or the equivalent `mode = "path"` config
dest) with absolute paths to the synced rule locations.

Ownership contract: agentspec owns any `instructions` entry whose path falls
under the rules destination directory. On each sync, those entries are
replaced wholesale; all other entries (user-added) are preserved.

OpenCode has no per-rule path scoping — all rules are global regardless of
`paths:` in the canonical spec (already dropped by the adapter). This is a
known limitation.

### Changes Required

**File**: `agentspec/src/sync/provider.rs`

- [x] Add `patch_opencode_instructions(rules_dest_dir: &Path, opencode_config_dir: &Path, dry_run: bool) -> Result<()>`:
  - Read `opencode_config_dir/opencode.json`; if absent, start with `{}`
  - Parse `instructions` array (default to `[]` if absent)
  - Remove entries whose path starts with `rules_dest_dir` (agentspec-owned)
  - Walk `rules_dest_dir/*/AGENTS.md` to enumerate current rule files; add
    their absolute paths to `instructions`
  - Write updated `opencode.json` back (pretty-printed)
  - `dry_run`: print the diff of entries added/removed without writing

**File**: `agentspec/src/compile.rs` + `agentspec/src/adapters/opencode.rs`

- [x] Remove the `build_opencode_instructions(&files)` call from `compile_specs`
  in `compile.rs`
- [x] Remove the `build_opencode_instructions` function from
  `adapters/opencode.rs` and its tests
- [x] Remove the corresponding `use` import in `compile.rs`

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds
- [x] `cargo clippy --all-targets` passes
- [x] `cargo fmt --check` passes
- [x] `cargo test` passes (adapter tests updated to not assert on
  `instructions.json`)
- [x] Unit tests for `patch_opencode_instructions`:
  - [x] No prior `opencode.json` → file created with rule paths
  - [x] Existing user entries preserved; agentspec entries replaced
  - [x] Stale agentspec entry (rule removed) → removed from array
  - [x] Empty rules dir → `instructions` array has agentspec entries removed
  - [x] `dry_run = true` → no file written, diff printed
- [x] `agentspec compile` no longer writes `generated/opencode/instructions.json`

---

## Phase 7: Wire into `main.rs`

### Overview

Dispatch the `sync` command in `main.rs`. The sync orchestrator runs compile
(unless `--no-compile`) then iterates over target providers, resolves each
sync target config, and calls the appropriate strategy for each spec kind.
Emit a summary report.

### Changes Required

#### 1. Sync orchestrator

**File**: `agentspec/src/sync.rs`

- [x] Add `pub fn run_sync(config: &AgentspecConfig, args: &SyncArgs, compile_result: Option<&CompileResult>) -> Result<()>`:
  - Determine `home_dir` via `std::env::var("HOME")`
  - Determine `cwd` from `std::env::current_dir()`
  - For each targeted provider:
    - Call `config.resolve_sync_target(provider, profile, cli_overrides)`
    - For each `SyncKind` in `all_sync_kinds(provider)`:
      - Resolve `source_dir = generated_source_dir(provider, kind, &output_dir)`
      - Resolve `dest_dir = resolve_dest_dir(provider, kind, &target_config, &home_dir, &cwd)?`
      - If source dir doesn't exist: skip with info message
      - Match on `target_config.strategy`:
        - `Symlink` → `sync_symlinked_dir(source_dir, dest_dir, dry_run)`
        - `Copy` → load manifest, `sync_copied_dir(...)`, save manifest
      - If provider is OpenCode and kind is Rules: call
        `patch_opencode_instructions(dest_dir, opencode_config_dest, dry_run)`
    - If `target_config.strip_name && strategy == Copy`: call
      `apply_strip_name(skills_dest_dir, dry_run)`
  - Collect all `SyncEntry` results and print summary
  > **Deviation:** Introduced `SyncContext<'a>` struct to pass shared arguments without exceeding the `too_many_arguments` clippy limit. Compile result is not passed to `run_sync` — compile is handled in `main.rs` before calling `run_sync`. Also fixed several code-review findings: `--dest` flag now appends kind subdirs; `apply_strip_name` scoped to frontmatter only; `patch_opencode_instructions` uses `follow_links(true)` to traverse directory-level symlinks; skips creating `opencode.json` when file is absent and no instructions to write.

#### 2. `main.rs` dispatch

**File**: `agentspec/src/main.rs`

- [x] Declare `mod sync;` alongside existing module declarations
- [x] In the `Command::Sync(sync_args)` match arm:
  - If `!sync_args.no_compile`: run the full compile pipeline (reuse existing
    code path) to produce `CompileResult`
  - Call `sync::run_sync(&config, &sync_args, compile_result_opt)?`
- [x] Ensure `CommonArgs` extraction works for `Sync` (already handled by
  `Command::args()` update in Phase 2)

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds
- [x] `cargo clippy --all-targets` passes
- [x] `cargo fmt --check` passes
- [x] `cargo test` passes (existing tests unbroken)
- [x] `agentspec sync --dry-run` runs without error from `agent-config/`
  and prints planned operations
- [x] `agentspec sync --no-compile --dry-run` skips compilation output and
  prints planned sync operations only
- [x] `agentspec sync --target claude --dry-run` only reports Claude operations
- [x] Integration test: `agentspec sync --dry-run` against the fixture
  `agent-config/` asserts planned operations match expected provider/kind set
  (see Testing Strategy)
- [x] Integration test: `agentspec sync --mode path --dest <tempdir>` syncs
  into a temp dir; assert files exist at expected paths and
  `opencode.json` `instructions` array contains absolute paths to synced rules


---

## Phase 8: Shrink `setup.sh`

### Overview

Remove the generated-file distribution logic from `setup.sh` now that
`agentspec sync` handles it. Replace `generate_agent_configs` +
`setup_<tool>` + `copy_to_plugin` calls with a single `agentspec sync` call.

### Changes Required

**File**: `agent-config/setup.sh`

- [x] Replace the `generate_agent_configs` call and the four `setup_*` +
  `copy_to_plugin` calls with:
  ```bash
  agentspec sync
  ```
  (Profile is already exported as `AGENTSPEC_PROFILE` by `resolve_mapping_profile`)
- [x] Remove `generate_agent_configs` function (compile is now internal to sync)
- [x] Remove `sync_symlinked_children` function
- [x] Remove `ensure_symlink` calls for generated files from `setup_claude`,
  `setup_codex`, `setup_cursor`, `setup_opencode` (keep static-file symlinks:
  `AGENTS_GLOBAL.md`, `keybindings.json`)
- [x] Remove `copy_to_plugin` function
- [x] Keep `setup_claude_default_settings` (settings merge via `yq`)
- [x] Keep `setup_opencode_default_settings` (opencode.json de-symlink guard)
- [x] Keep `setup_*` function shells (just settings + dir creation) or inline
  those into a simpler structure
- [x] Keep `resolve_mapping_profile` (interactive profile prompt)
- [x] Keep legacy stale-dir removals for `~/.claude/commands` and
  `~/.cursor/commands` until they're no longer needed
  > **Deviation:** `setup_opencode_default_settings` no longer creates `{}` since `agentspec sync` handles opencode.json creation. The `setup_*` per-tool functions were inlined into the main body rather than kept as named functions. `setup_opencode_default_settings` runs before `agentspec sync` (not after) to ensure opencode.json is not a symlink before patch runs.

### Success Criteria

#### Automated Verification

- [x] `bash -n agent-config/setup.sh` (syntax check) passes
- [x] `agentspec sync --dry-run` from `agent-config/` still works
- [x] `agentspec check` still passes after running `agentspec sync`
- [x] `wc -l agent-config/setup.sh` significantly reduced (376 → 210 lines)

#### Manual Verification

- [x] Run `agent-config/setup.sh` end-to-end and confirm generated files
  land in the correct locations (manual-only: touches real home directories)

**Implementation Note**: After manual verification passes, pause for user
confirmation before proceeding to Phase 9.

---

## Phase 9: Documentation

### Overview

Update all documentation to reflect the new `sync` command, the `[sync.*]`
config schema, the removed `instructions.json`, and the shrunk `setup.sh`
workflow.

### Changes Required

#### 1. `agentspec/README.md`

- [x] Add `agentspec sync` to the Usage section with common invocations:
  ```sh
  agentspec sync                   # compile and sync to all configured targets
  agentspec sync --profile work    # apply a machine profile
  agentspec sync --dry-run         # preview without making changes
  agentspec sync --no-compile      # sync from existing generated output
  agentspec sync --target claude   # sync a specific provider only
  ```
- [x] Add a `## Sync` section documenting the `[sync.*]` TOML config, the
  three modes (`user`, `project`, `path`), the two strategies (`symlink`,
  `copy`), `strip_name`, and profile overrides; include the example from the
  idea doc (`[profiles.work.sync.claude]`)
- [x] Note that Codex rules are not yet synced (adapter fix pending)

#### 2. `agentspec/CLAUDE.md`

- [x] Add `agentspec sync` to the Commands section
- [x] Add Phase 9 (Sync) to the Pipeline Stages list
- [x] Add `sync/` module family to the Module Map table
- [x] Remove the mention of `instructions.json` from any adapter documentation
  (it is no longer produced by the compile step)

#### 3. `agent-config/CLAUDE.md`

- [x] Update the key commands section to show `agentspec sync` as the primary
  workflow, with `agentspec compile` noted as still available for compile-only
  use

### Success Criteria

#### Automated Verification

- [x] `cargo fmt --check` passes (no Rust files changed in this phase)
- [x] `bash -n agent-config/setup.sh` passes (no shell files changed)

#### Manual Verification

- [x] `agentspec/README.md` accurately reflects the full command set and
  `[sync.*]` config schema
- [x] `agentspec/CLAUDE.md` module map and pipeline stages match the
  implemented module structure

**Implementation Note**: After manual verification passes, pause for user
confirmation that the plan is complete.

---

## Testing Strategy

### Unit Tests

Each phase's unit tests are specified inline above. Key coverage areas:

- `resolve_sync_target`: base, profile overlay, CLI override precedence
- `resolve_dest_dir`: all three modes, per-provider/per-kind combos
- `ensure_symlink`: all filesystem states (missing, correct, wrong target, regular file)
- Stale symlink cleanup: symlinks inside source dir vs outside
- `copy_file` + manifest lifecycle: first-time, idempotent, content-changed, stale
- `apply_strip_name`: strips `name:` lines from `SKILL.md` files only
- `patch_opencode_instructions`: ownership contract, user entries preserved,
  stale removal, empty rules

### Integration Tests

- [x] Add to `agentspec/tests/` (alongside `pipeline.rs`): end-to-end sync
  test that runs `agentspec sync --dry-run` against the fixture
  `agent-config/` directory and asserts on the planned operations output
  > **Deferred:** tracked in `dotfiles/TODO.md`

### Manual Testing Steps

1. Run `agentspec sync --dry-run` and verify planned operations match current
   `setup.sh` behavior
2. Run `agentspec sync` and verify files appear in tool config dirs
3. Delete one file from a synced dir; re-run `agentspec sync` and verify it's
   restored
4. Add a stale file manually to a synced dir (symlink strategy); re-run and
   verify stale symlink is cleaned up
5. Run with `--profile work` and verify copy-strategy targets, including
   `strip_name` post-processing on `SKILL.md` files
6. Verify `~/.config/opencode/opencode.json` `instructions` array is updated
   correctly, with user-added entries preserved

## Performance Considerations

The compile pipeline already runs in O(specs × providers) time. The sync step
adds O(files) filesystem operations. No performance concerns at current scale.

## Migration Notes

After Phase 8, `setup.sh` will no longer call `agentspec compile` internally.
Users who call `agentspec compile` directly (for testing, CI) are unaffected —
`compile` remains a standalone command. The `--no-compile` flag on `sync`
supports the case where someone wants to sync from an already-compiled output
without recompiling.

## References

- Idea doc: `thoughts/ideas/2026-03-22-agentspec-sync-command.md`
- Current distribution script: `agent-config/setup.sh`
- Compiler config: `agentspec/src/config.rs`
- CLI definitions: `agentspec/src/cli.rs`
- Emit logic (for structural reference): `agentspec/src/emit.rs`
- OpenCode adapter: `agentspec/src/adapters/opencode.rs`
- Codex adapter fix tracked in: `TODO.md`
