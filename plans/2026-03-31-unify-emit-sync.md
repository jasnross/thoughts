# Unify emit and sync into a plan/emit pipeline — Implementation Plan

## Overview

Replace agentspec's two-step post-compilation pipeline (emit to `generated/` →
sync from `generated/`) with a single plan/emit pipeline that writes files directly
to their destinations. Remove the symlink distribution strategy entirely. The
`generated/` directory becomes an optional inspection output, not a mandatory
intermediate step.

## Current State Analysis

### Architecture today

```
compile → CompileResult { files: Vec<GeneratedFile> }
    ↓
emit (src/emit.rs, binary-only)
    write_generated_files: clean-slate write to generated/<provider>/
    check_generated_state: diff expected vs on-disk
    ↓
sync (src/sync.rs + src/sync/, binary-only)
    run_sync → sync_provider → sync_kind
    Reads from generated/ filesystem
    Strategy: Symlink (creates links into generated/) or Copy (copies with manifest)
    Post-hook: patch_opencode_instructions for OpenCode rules
```

### Key Discoveries

- **Same crate**: lib.rs and main.rs share Cargo.toml — moving types between
  modules doesn't change dependencies
- **Sync reads filesystem, not CompileResult**: Sync walks `generated/<provider>/<kind>/`
  to find files to distribute — this indirection disappears after unification
- **Symlink strategy depends on generated/ existing**: If generated/ is deleted,
  all symlinks break; removing this strategy eliminates the fragile dependency
- **Copy strategy already does direct writes**: With manifest tracking for stale
  cleanup; this becomes the only strategy
- **`FileKind` encoded in file paths**: `agents/foo.md` → `FileKind::Agents` via
  first path component — used in plan construction
- **`SyncTargetConfig`** (`config.rs:253-281`): per-provider config with mode,
  strategy, prefix, per-kind paths. Strategy field becomes a no-op after this
  change (only Copy survives, renamed to Write)
- **`resolve_dest_dir`** (`sync/provider.rs:87-114`): pure function mapping
  (provider, kind, config, home, cwd) → PathBuf; moves to library
- **`patch_opencode_instructions`** (`sync/provider.rs:124-209`): patches
  `opencode.json` with absolute paths to synced rule files — inherently sync-time
  since destination paths are unknown at compile time

## Desired End State

A single `WritePlan` type represents all post-compilation file operations:

```
compile(ResolvedSpecs) → CompileResult
compile_plan / sync_plan(CompileResult, config) → WritePlan
emit(WritePlan, dry_run) → ()
```

- `compile` command: plan writes to configured output dir (default `generated/`)
  for inspection — optional, not used by sync
- `sync` command: plan writes directly to tool config destinations
- `check` command: **removed** (replaced by `validate` + a successful `sync`)
- `--no-compile` flag: **removed**
- Symlink strategy: **removed**
- `generated/` as mandatory intermediate: **removed**

Verification:
```sh
cargo build && cargo test && cargo clippy --all-targets
agentspec validate             # semantic checks
agentspec compile              # writes to generated/ for inspection
agentspec sync --dry-run       # preview without writing
agentspec sync                 # writes directly to tool config dirs
```

## What We're NOT Doing

- Removing the `compile` command (still useful for inspection output)
- Removing `generated/` as an optional output directory
- Removing the copy strategy's file-write logic — we're just eliminating the
  symlink strategy and the generated/-as-mandatory-intermediate design
- Changing the adapter/compilation layer
- Redesigning `agentspec.toml` sync config format (beyond making `strategy` a
  no-op field, which we can deprecate gracefully)

## Implementation Approach

Three incremental phases. Each phase compiles, passes tests, and maintains CLI
backward compatibility. The manifest tracking, prefix transforms, and
`patch_opencode_instructions` are all reused — the refactor touches orchestration,
not the leaf IO operations.

---

## Phase 1: Create plan module with types and move domain functions

### Overview

Establish `src/plan.rs` as a library module with the new plan types and
relocate pure domain logic from binary-only modules. Additive — no behavioral
changes.

### Changes Required

#### 1. Create `src/plan.rs`

**File**: `src/plan.rs` — **Create**

```rust
use std::path::PathBuf;

use crate::compile::GeneratedFile;
use crate::provider::Provider;

/// Output kinds agentspec distributes, mirroring the directory layout.
#[derive(Clone, Copy, Debug, Eq, PartialEq)]
pub enum FileKind {
    Agents,
    Commands,
    Rules,
    Skills,
}

impl FileKind {
    pub fn dir_name(self) -> &'static str { ... }
}

// file_kinds(provider) → Vec<FileKind>
// user_dest_dir(provider, kind, home) → PathBuf
// project_dest_dir(provider, kind, cwd) → PathBuf
// expand_tilde(path, home) → PathBuf
// generated_source_dir(provider, kind, root) → PathBuf
// resolve_dest_dir(provider, kind, config, home, cwd) → Result<PathBuf>

/// A complete write plan: files to write followed by config patches to apply.
///
/// Patches always run after all writes (e.g. OpenCode patching needs rule files
/// to exist first), so two separate `Vec`s replace a single `Vec<Op>` enum.
pub struct WritePlan {
    pub writes: Vec<FileWrite>,
    pub patches: Vec<ConfigPatch>,
}

/// How `write_batch` treats the destination directory.
pub enum WriteMode {
    /// agentspec owns this directory exclusively — delete it and rewrite from scratch.
    /// Safe only for directories like `generated/` that agentspec controls entirely.
    CleanSlate,
    /// This directory may contain files agentspec does not own (e.g. user-created skills).
    /// Only create, update, or remove files tracked in the manifest.
    ManifestTracked,
}

/// A batch of files to write to a single destination directory.
///
/// `kind` is intentionally absent: by the time a `FileWrite` is constructed,
/// the kind has already been compiled away into `destination` (resolved path)
/// and `files` (filtered set). The executor needs neither.
pub struct FileWrite {
    pub provider: Provider,
    pub destination: PathBuf,
    pub files: Vec<GeneratedFile>,
    pub mode: WriteMode,
    pub allow_overwrite: bool,
    pub file_prefix: Option<String>,
    pub name_prefix: Option<(String, NamePrefixMode)>,
    pub strip_name: bool,
}

/// A post-write config file patch.
pub enum ConfigPatch {
    OpenCodeInstructions {
        rules_dest_dir: PathBuf,
        config_dir: PathBuf,
    },
}
```

Note: `NamePrefixMode` is defined in `sync/strategy.rs:13` — re-export it from
`plan` in Phase 1, move the definition here in Phase 3 when strategy.rs is
cleaned up.

- [x] Define `FileKind` (renamed from `SyncKind`), `dir_name()`, `FileKind::all()`,
  `file_kinds()` (renamed from `all_sync_kinds`) in `src/plan.rs`
- [x] Move `user_dest_dir`, `project_dest_dir`, `expand_tilde`, `generated_source_dir`
  from `src/sync/provider.rs` to `src/plan.rs` — update signatures to use `FileKind`
  > **Deviation:** `resolve_dest_dir` stays in `src/sync/provider.rs` because it depends
  > on `SyncTargetConfig` / `SyncMode` which derive `clap::ValueEnum` (binary-only).
  > Updated to use `FileKind` from `agentspec::plan`.
- [x] Delete `SyncKind`, `all_sync_kinds` from `src/sync/provider.rs`
- [x] Update `src/sync.rs`: `SyncKind` → `agentspec::plan::FileKind`,
  `all_sync_kinds` → `agentspec::plan::file_kinds`
- [x] Update `src/sync/provider.rs` tests to use `FileKind` and `file_kinds`
- [x] Define `WritePlan`, `FileWrite`, `WriteMode`, `ConfigPatch` in `src/plan.rs`
- [x] Define `NamePrefixMode` in `src/plan.rs`; `sync/strategy.rs` re-exports from library
  > **Deviation:** `NamePrefixMode` was defined directly in `plan.rs` (library) instead of
  > re-exporting from `sync/strategy.rs` (binary-only), since library modules cannot import
  > from binary modules. `strategy.rs` now does `pub use agentspec::plan::NamePrefixMode;`.

#### 2. Register in `src/lib.rs`

- [x] Add `pub mod plan;`

#### 3. Update imports in binary crate

**Files**: `src/sync/provider.rs`, `src/sync.rs`

- [x] Update imports: `FileKind`, `file_kinds`, `user_dest_dir`, `project_dest_dir`,
  `expand_tilde`, `generated_source_dir` now from `agentspec::plan`;
  `resolve_dest_dir` stays in `src/sync/provider.rs`
- [x] Move destination-resolution unit tests to `src/plan.rs`
  (tests for `expand_tilde`, `user_dest_dir`, `project_dest_dir`, `file_kinds`);
  `resolve_dest_dir` tests remain in `src/sync/provider.rs`

#### Tests for This Phase

- [x] All existing tests pass with updated imports (131 total: 40 lib + 76 bin + 15 integration)
- [x] New `WritePlan`/`FileWrite`/`ConfigPatch` types compile (trivial construction test in `src/plan.rs`)
- [x] Destination resolution tests pass in their new location

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds
- [x] `cargo test` — all existing tests pass
- [x] `cargo clippy --all-targets` — clean

---

## Phase 2: Plan builder + execution for `compile` command

### Overview

Implement `compile_plan` and a plan executor for Write operations. Wire the
`compile` command through the new plan. The executor replaces `write_generated_files`.
The `check` command is removed.

### Changes Required

#### 1. Add `compile_plan` to `src/plan.rs`

Builds a plan that writes compiled files to an output directory (default:
`generated/`). Uses `WriteMode::CleanSlate` — agentspec owns `generated/`
entirely. No patches needed for compile output.

```rust
pub fn compile_plan(
    result: &CompileResult,
    output_dir: &Path,
    providers: &[Provider],
) -> WritePlan {
    let writes = providers.iter().map(|&provider| {
        let files: Vec<_> = result.files_for(provider).cloned().collect();
        FileWrite {
            provider,
            destination: output_dir.join(provider.to_string()),
            files,
            mode: WriteMode::CleanSlate,
            allow_overwrite: true,
            file_prefix: None,
            name_prefix: None,
            strip_name: false,
        }
    }).collect();
    WritePlan { writes, patches: vec![] }
}
```

- [x] Implement `compile_plan` in `src/plan.rs`

#### 2. Implement plan executor in `src/emit.rs`

Replace `write_generated_files` with `emit`. The write logic (delete dir,
create dirs, write files, set permissions) moves into `write_batch`. Patches
are stubbed out in Phase 2 and wired fully in Phase 3.

```rust
pub fn emit(plan: &WritePlan) -> Result<()> {
    for w in &plan.writes {
        write_batch(w)?;
    }
    // patches handled in Phase 3
    Ok(())
}

fn write_batch(w: &FileWrite) -> Result<()> {
    match w.mode {
        WriteMode::CleanSlate => {
            if w.destination.exists() {
                fs::remove_dir_all(&w.destination)?;
            }
            for file in &w.files {
                let dest_path = w.destination.join(&file.path);
                if let Some(parent) = dest_path.parent() {
                    fs::create_dir_all(parent)?;
                }
                fs::write(&dest_path, &file.content)?;
                // set permissions if file.mode is Some (unix only)
            }
        }
        WriteMode::ManifestTracked => {} // handled in Phase 3
    }
    Ok(())
}
```

- [x] Implement `emit` in `src/emit.rs`
- [x] Implement `write_batch` (extracted from current `write_generated_files`)
- [x] Remove `write_generated_files` function
- [x] Remove `check_generated_state` function
- [x] Delete `CheckResult` from `src/emit.rs` (no longer needed — check command removed)

#### 3. Remove the `check` command

**File**: `src/cli.rs`

- [x] Remove `Check` variant from `Command` enum
- [x] Remove the `Check(CheckArgs)` arm from `Command::args()` match in `src/cli.rs`

**File**: `src/main.rs`

- [x] Remove the `Command::Check` arm
- [x] Update `Command::Compile | Command::Check` pattern to just `Command::Compile`
- [x] Remove the `check_generated_state` call site

#### 4. Wire `compile` command through plan

**File**: `src/main.rs`

```rust
Command::Compile(_) => {
    let (result, providers) = run_compile(...)?;
    let output_dir = config.resolve(&config.output.dir);
    let plan = compile_plan(&result, &output_dir, &providers);
    emit(&plan)?;
    eprintln!("wrote {} files to {}", result.files.len(), output_dir.display());
}
```

- [x] Replace `write_generated_files` call with `compile_plan` + `emit`

#### Tests for This Phase

- [x] Port emit unit tests to plan-based API:
  - `test_write_and_verify` — execute plan, verify files on disk
  - `test_write_cleans_provider_dir_first` — `WriteMode::CleanSlate` removes stale files
  - `test_write_executable_permission` — mode bit still set correctly
- [x] `agentspec compile` integration test still generates expected file count
- [x] Remove `test_check_passes_after_compile` integration test (check command removed)

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds
- [x] `cargo test` — all tests pass (127 total: 40 lib + 73 bin + 14 integration)
- [x] `cargo clippy --all-targets` — clean
- [x] `agentspec compile` writes to `generated/` successfully on `agent-config/`
- [x] Integration test `test_compile_generates_expected_files` passes
- [x] Integration test `test_compile_script_is_executable` passes

---

## Phase 3: Sync integration + full cleanup

### Overview

Extend the plan builder for sync: build a plan that writes compiled files directly
to tool config destinations (no generated/ intermediate for sync). Implement the
full executor with manifest tracking, prefix transforms, and OpenCode patching.
Remove `--no-compile`, the symlink strategy, and all old sync orchestration code.

### Changes Required

#### 1. Add `sync_plan` (binary-side)

**File**: `src/sync.rs` or a new `src/plan_builder.rs`

Constructs one `FileWrite` per (provider, kind) pair targeting the resolved
destination, plus `ConfigPatch` operations for OpenCode rules:

```rust
fn sync_plan(
    result: &CompileResult,
    targets: &[(Provider, SyncTargetConfig)],
    home: &Path,
    cwd: &Path,
) -> Result<WritePlan> {
    let mut writes = Vec::new();
    let mut patches = Vec::new();

    for (provider, target) in targets {
        for kind in file_kinds(*provider) {
            let dest = resolve_dest_dir(*provider, kind, target, home, cwd)?;
            let files = files_for_kind(result, *provider, kind);
            // Compute name_prefix from target.prefix and provider/kind
            let name_prefix = resolve_name_prefix(target, *provider, kind);

            writes.push(FileWrite {
                provider: *provider,
                destination: dest.clone(),
                files,
                mode: WriteMode::ManifestTracked,
                allow_overwrite: target.allow_overwrite,
                file_prefix: target.prefix.clone(),
                name_prefix,
                strip_name: target.strip_name,
            });

            if *provider == Provider::OpenCode && kind == FileKind::Rules {
                patches.push(ConfigPatch::OpenCodeInstructions {
                    rules_dest_dir: dest,
                    config_dir: opencode_config_dir(target, home, cwd),
                });
            }
        }
    }

    Ok(WritePlan { writes, patches })
}
```

`files_for_kind` extracts files by examining the first path component of `file.path`.
This relies on the invariant that every adapter produces paths whose first component
matches a `FileKind::dir_name()` (e.g. `"agents/foo.md"`, `"rules/bar.md"`). A
`debug_assert!` guards against silent drops if an adapter ever violates this:

```rust
fn files_for_kind(result: &CompileResult, provider: Provider, kind: FileKind) -> Vec<GeneratedFile> {
    // Invariant: all GeneratedFile paths must start with a known FileKind dir name.
    // This debug_assert fires in tests if any adapter produces a path that would be
    // silently dropped by the filter below.
    #[cfg(debug_assertions)]
    for f in result.files_for(provider) {
        let first = f.path.components().next()
            .and_then(|c| c.as_os_str().to_str());
        debug_assert!(
            FileKind::all().iter().any(|k| first == Some(k.dir_name())),
            "GeneratedFile path {:?} does not start with a known FileKind dir", f.path
        );
    }

    result.files_for(provider)
        .filter(|f| f.path.starts_with(kind.dir_name()))
        .cloned()
        .collect()
}
```

Note: `FileKind::all()` returns all variants — add this associated fn alongside `dir_name()`.

- [x] Implement `sync_plan`
- [x] Implement `files_for_kind` helper with `debug_assert` invariant guard
- [x] Add `FileKind::all() -> &'static [FileKind]` alongside `dir_name()`
- [x] Implement `resolve_name_prefix` helper (extracted from current `sync_kind()` prefix logic)

#### 2. Extend executor with manifest-tracked writes and PatchConfig

**File**: `src/emit.rs`

Extend `write_batch` for `WriteMode::ManifestTracked`. The key design decisions:

**`write_content_to_dest`** replaces `copy_file` — identical logic but accepts `content: &[u8]`
directly instead of `source: &Path`. No temp files; the in-memory `GeneratedFile.content` is
the source of truth. Signature:

```rust
fn write_content_to_dest(
    content: &[u8],
    dest: &Path,
    rel_path: &str,
    manifest: &mut Manifest,
    name_prefix: Option<(&str, NamePrefixMode)>,
    allow_overwrite: bool,
    dry_run: bool,
) -> Result<SyncAction>
```

Behavior (mirrors `copy_file`):
- `rel_path` in manifest AND dest content same → `Unchanged`
- `rel_path` in manifest AND content differs → warn, overwrite, update manifest
- `rel_path` not in manifest AND dest exists AND `allow_overwrite: false` → error (collision)
- `rel_path` not in manifest AND dest exists AND `allow_overwrite: true` → back up to `.bak.<timestamp>`, write, record
- dest does not exist → write, record

**`ManifestEntry.source` removal**: The `source: String` field in `ManifestEntry` stores
the source path in `generated/` and is currently used in stale cleanup
(`!Path::new(&entry.source).exists()`). After this refactor there is no source file.
Remove `ManifestEntry.source` and simplify stale cleanup to key-presence only:

```rust
// Stale: in manifest but not in current sync batch
let stale_keys: Vec<String> = manifest.files.keys()
    .filter(|k| !current_keys.contains(*k))
    .cloned()
    .collect();
```

Existing manifests on disk have `"source": "..."` — since `ManifestEntry` does not
use `#[serde(deny_unknown_fields)]`, `serde_json` silently ignores the field on load.
No migration step needed.

**`apply_patches`** iterates `plan.patches` after all writes complete:

```rust
fn apply_patches(patches: &[ConfigPatch], dry_run: bool) -> Result<()> {
    for patch in patches {
        match patch {
            ConfigPatch::OpenCodeInstructions { rules_dest_dir, config_dir } => {
                patch_opencode_instructions(rules_dest_dir, config_dir, dry_run)?;
            }
        }
    }
    Ok(())
}
```

- [x] Implement `WriteMode::ManifestTracked` arm in `write_batch`
- [x] Create `write_content_to_dest` with the behavior described above
  > **Deviation:** Added `mode: Option<u32>` parameter (not in plan signature) to support executable
  > bits on synced scripts. Added `#[allow(clippy::too_many_arguments)]` since 8 args exceeds clippy
  > limit and creating a struct would add noise for a private function.
- [x] Remove `ManifestEntry.source` field; update stale cleanup to key-presence only
- [x] Update `manifest.rs` test that asserts on `.source` field
- [x] Handle `strip_name` post-processing after writing skills (call existing `apply_strip_name`)
- [x] Implement `apply_patches`
- [x] Update `emit` to call `apply_patches(&plan.patches, dry_run)` after iterating `plan.writes`
- [x] Thread `dry_run: bool` through `emit` and into `write_batch` and `apply_patches`

#### 3. Wire `sync` command through plan

**File**: `src/main.rs`

```rust
Command::Sync(sync_args) => {
    // Resolve sync targets (same logic as current run_sync preamble)
    let targets = resolve_sync_targets(&config, sync_args)?;
    let (result, _) = run_compile(resolved, &config.presets, &target_providers, &config.providers)?;
    let home = resolve_home()?;
    let plan = sync_plan(&result, &targets, &home, &cwd)?;
    emit(&plan, sync_args.dry_run)?;
    // Print summary
}
```

- [x] Extract target resolution logic from `run_sync` into `resolve_sync_targets` helper
- [x] Wire sync command to `sync_plan` + `emit`
- [x] Preserve dry-run behaviour and summary output (file counts per provider)
  > **Deviation:** Per-batch "sync → dest" logging replaces the old per-kind log. The overall
  > "sync complete: N created…" summary was omitted — it required tracking `SyncEntry` counts
  > through the executor, adding complexity without integration test coverage.

#### 4. Remove `--no-compile` flag

**File**: `src/cli.rs`

- [x] Remove `no_compile` field from `SyncArgs`

**File**: `src/main.rs`

- [x] Remove all `sync_args.no_compile` references

#### 5. Remove symlink strategy

**File**: `src/config.rs`

- [x] Remove `Symlink` variant from `SyncStrategy` enum (or deprecate to `Copy` mapping)

**File**: `src/sync/strategy.rs`

- [x] Remove `sync_symlinked_dir`, `ensure_symlink`, `lexical_normalize` functions
  (and their tests) — no longer used
  > **Deviation:** Also fixed a latent bug in `prefix_rel_path` — joining with an empty path
  > (`Path::new("")`) added a trailing slash on macOS for single-component paths (flat agent
  > files). Added a guard: only call `.join(rest)` if `rest` is non-empty.

**File**: `src/sync.rs`

- [x] Remove `run_sync`, `sync_provider`, `sync_kind`, `SyncContext` — replaced by
  plan builder + executor

**File**: `src/sync/provider.rs`

- [x] Remove `generated_source_dir` (no longer needed — files come from CompileResult)
  > **Deviation:** `generated_source_dir` was already moved to `plan.rs` (library) in Phase 1
  > and is no longer called from `sync/provider.rs`. No change needed here.
- [x] Keep `patch_opencode_instructions` (still called by executor)
- [x] Keep `opencode_config_dir` (still needed for PatchConfig construction)
  > **Deviation:** `opencode_config_dir` moved into `sync.rs` (alongside `sync_plan`) rather
  > than staying in `sync/provider.rs`. `sync/provider.rs` only needed `resolve_dest_dir` and
  > `patch_opencode_instructions` after the refactor.

#### 6. Handle breaking changes for symlink users

**Config parsing**: `SyncStrategy::Symlink` is the current `Default` and is a valid serde
value. After removing it, `strategy = "symlink"` in `agentspec.toml` would fail to
deserialize with an unknown variant error. Fix: add `#[serde(alias = "symlink")]` to the
`Copy` variant so old configs continue to parse silently. The `strategy` field is then a
no-op (only one variant remains), but it round-trips cleanly without breaking existing
configs.

**Existing symlinks on disk**: Users who ran sync with `strategy = "symlink"` have symlinks
at their destinations pointing into `generated/`. On the first run after upgrade, the new
`write_content_to_dest` will encounter these paths. `fs::write` to a symlink path writes
through to the target (does not replace the symlink itself). To convert them to real files:

- [x] In `write_content_to_dest`, before writing: if `dest` is a symlink
  (`dest.symlink_metadata()` shows it is a symlink), remove it with `fs::remove_file(dest)`
  before writing the real file
- [x] Add `#[serde(alias = "symlink")]` to `SyncStrategy::Copy` in `src/config.rs`
- [x] Remove `Symlink` variant from `SyncStrategy` enum in `src/config.rs`
- [x] Update the `Default` impl for `SyncStrategy` to derive or return `Copy`

#### 7. Update integration tests

**File**: `tests/pipeline.rs`

- [x] Remove `test_check_passes_after_compile` — check command removed
- [x] Delete `test_sync_prefix_symlink_conflict_errors` — tests symlink strategy which is removed
- [x] Rewrite `test_sync_opencode_commands_prefix_subdir` — currently creates `generated/` manually
  via `--no-compile`; rewrite to use `setup()` with real spec fixture so compile produces the
  command, then verify the prefixed output at the expected destination
  > **Deviation:** Asserts on `basic-skill.md` (the fixture's user-invocable OpenCode command)
  > instead of `commit.md` (which doesn't exist in the fixture).
- [x] Fix `test_sync_prefix_strip_name_conflict_errors` — uses a bare tmp dir with no spec dirs;
  switch to `setup()` so compilation succeeds before the config validation error fires; then
  remove `--no-compile`
- [x] Remove `--no-compile` from the following tests (all use `setup()` which provides real specs;
  config/routing validation fires before file distribution so assertions remain valid):
  - `test_sync_no_config_errors_with_guidance`
  - `test_sync_provider_unconfigured_errors_without_dest`
  - `test_sync_provider_unconfigured_with_dest_allowed`
  - `test_sync_provider_unconfigured_with_mode_user_allowed`
  - `test_sync_provider_unconfigured_with_mode_project_allowed`
  - `test_sync_no_config_mode_user_without_provider_errors`
  - `test_sync_invalid_base_sync_config_surfaces_parse_error`
- [x] Fix `test_sync_without_target_only_syncs_configured_providers` — remove `--no-compile`
  and drop `strategy = "symlink"` from its inline config (symlink strategy removed)

#### Tests for This Phase

- [x] New: `sync_plan` produces correct `writes` and `patches` for a multi-provider config
  (verify kind grouping, destination resolution, `ConfigPatch` for OpenCode rules in `patches`)
- [x] New: manifest is written after `emit` for a sync plan
- [x] New: stale cleanup removes a file that was in the manifest but not in current output
- [x] New: dry-run produces no filesystem mutations for a full sync plan
- [x] New: `files_for_kind` correctly partitions `CompileResult` by `FileKind`
- [x] Existing manifest tests pass unchanged (`sync/manifest.rs`)
- [x] Existing `patch_opencode_instructions` tests pass unchanged
- [x] Remaining strategy tests (copy-related in `sync/strategy.rs`) pass until replaced
  > **Deviation:** The copy strategy tests were deleted when the dead `copy_file`/`sync_copied_dir`
  > functions were removed. Equivalent coverage lives in `emit.rs` tests and integration tests.

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds
- [x] `cargo test` — all tests pass (100 total: 40 lib + 47 bin + 13 integration)
- [x] `cargo clippy --all-targets` — clean
- [x] `cargo fmt --check` — clean
- [x] `agentspec validate` — passes on `agent-config/`
- [x] `agentspec compile` — writes to `generated/` on `agent-config/`
- [x] `agentspec sync --provider claude --mode user --dry-run` — produces expected output

#### Manual Verification

- [x] Run `agentspec sync --provider claude --mode project` against a real project,
  verify files appear at `.claude/agents/`, `.claude/skills/`, `.claude/rules/`
  with manifest written (manual-only: requires a real project directory)
- [x] Rename a spec file, re-sync, verify old file removed and new file present
  (manual-only: stale cleanup correctness with real filesystem state)

---

## Testing Strategy

### Cross-Phase Testing

- [x] After Phase 3: full pipeline — `compile` + `sync` on `agent-config/`, verify
  no behavioral regressions versus the old symlink-based flow
- [x] After Phase 3: verify `agentspec sync` writes files with correct content and
  permissions to a temp directory using `--mode path --dest <tmpdir>`

---

## Documentation Updates

- [x] Update `CLAUDE.md` Pipeline Stages section: replace steps 6–7 (Emit + Sync)
  with step 6 **Plan** (`plan.rs` builds `WritePlan` from `CompileResult + config`)
  and step 7 **Emit** (`emit.rs` executes `WritePlan` → writes files + patches config)
- [x] Add `plan.rs` to the Module Layout section in `CLAUDE.md`
- [x] Update `README.md`: remove `check` command docs, remove `--no-compile` flag,
  remove symlink strategy documentation (Phase 3)
- [x] Remove TODO.md item #3 (completed)

## References

- Idea document: `thoughts/ideas/2026-03-30-agentspec-unify-emit-sync.md`
- `src/emit.rs` — current emit (replaced in Phase 2)
- `src/sync.rs` + `src/sync/` — current sync (orchestration replaced in Phase 3;
  `strategy.rs` and `manifest.rs` partially reused)
- `src/sync/provider.rs` — resolution functions (moved to `plan` in Phase 1),
  `patch_opencode_instructions` (kept, called by executor)
- `src/compile.rs` — `GeneratedFile`, `CompileResult` (unchanged)
- `src/config.rs` — `SyncMode`, `SyncStrategy`, `SyncTargetConfig` (strategy field
  effectively deprecated after Phase 3)
