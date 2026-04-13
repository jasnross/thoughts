# Colocate Post-Write Actions With Adapters — Implementation Plan

## Overview

Move `patch_opencode_instructions` from `emit.rs` into the OpenCode adapter
module, replace the `ConfigPatch` enum with a trait-based hook mechanism, and
make emit execute hooks generically without provider-specific knowledge.

## Current State Analysis

`emit.rs` contains ~90 lines of OpenCode-specific logic (`patch_opencode_instructions`,
lines 230-323) that patches `opencode.json` with rule file paths after sync writes.
The `ConfigPatch` enum in `plan.rs` (lines 148-155) has a single variant
(`OpenCodeInstructions`) — it exists solely for this one provider. `sync_plan` in
`sync.rs` (lines 42-47) constructs the patch with resolved paths and attaches it to
`WritePlan.patches`. `emit.rs` matches on the variant (lines 325-336) and calls the
function.

Five tests in `emit.rs` (lines 651-795) exercise `patch_opencode_instructions` directly.

### Key Discoveries:

- `patch_opencode_instructions` takes three parameters: `rules_dest_dir: &Path`,
  `opencode_config_dir: &Path`, `dry_run: bool`
- The first two are resolved by `sync_plan` using `resolve_dest_dir` and
  `opencode_config_dir` — both already computed in the sync planning loop
- `compile_plan` sets `patches: vec![]` — post-write actions are sync-only
- The function uses `walkdir::WalkDir` (already a dependency of the binary crate)
  and `serde_json` (already a dependency)
- `opencode_config_dir` helper (sync.rs:163-178) is only used for constructing
  the `ConfigPatch` — it will need to stay accessible to `sync_plan`

## Desired End State

- `patch_opencode_instructions` and `opencode_config_dir` live in `src/adapters/opencode.rs`
- `ConfigPatch` enum is deleted from `plan.rs`
- `WritePlan.patches` is replaced by `WritePlan.post_write_hooks: Vec<Box<dyn PostWriteHook>>`
- `emit.rs` calls `hook.run(dry_run)?` without knowing what the hooks do
- `sync_plan` dispatches to adapter `post_write_hook` functions — zero
  provider-specific `if` blocks
- All five `patch_opencode_instructions` tests move to `adapters/opencode.rs`
- All existing behavior preserved

### Verification

```sh
cargo fmt --check
cargo clippy --all-targets
cargo test
```

## What We're NOT Doing

- Adding new post-write hooks (this just moves the existing one)
- Changing what `patch_opencode_instructions` does — only where it lives
- Making hooks part of the compile stage or `CompileResult`
- Building a general-purpose event/plugin system

## Implementation Approach

This is an atomic refactor — all changes must land together since removing
`ConfigPatch` and adding the hook mechanism are interdependent.

## Implementation

### Overview

Define a `PostWriteHook` trait in `plan.rs`, move `patch_opencode_instructions`
and `opencode_config_dir` to the OpenCode adapter, add a per-adapter
`post_write_hook` dispatch so `sync_plan` has zero provider-specific logic,
and simplify emit to call hooks generically.

### Changes Required:

#### 1. Define `PostWriteHook` trait in `plan.rs`

**File**: `src/plan.rs`

- [x] Add a `PostWriteHook` trait:

```rust
/// A post-write action that runs after all file writes complete.
///
/// Hooks capture their own context (paths, config) when constructed.
/// Emit calls `run(dry_run)` without knowing what the hook does.
pub trait PostWriteHook: std::fmt::Debug {
    fn run(&self, dry_run: bool) -> anyhow::Result<()>;
}
```

- [x] Replace `patches: Vec<ConfigPatch>` on `WritePlan` with
  `post_write_hooks: Vec<Box<dyn PostWriteHook>>`

- [x] Delete the `ConfigPatch` enum entirely

- [x] Update the `compile_plan` function: set `post_write_hooks: vec![]`

- [x] Update the `test_plan_types_construct` test: remove the
  `ConfigPatch::OpenCodeInstructions` from the patches vec, use
  `post_write_hooks: vec![]` instead

#### 2. Move `patch_opencode_instructions` to the OpenCode adapter

**File**: `src/adapters/opencode.rs`

- [x] Move the `patch_opencode_instructions` function from `emit.rs` to
  `opencode.rs`. Keep it as a module-private function.

- [x] Add the `walkdir` and `serde_json` imports needed by the function
  (check which are already present in the adapter)

- [x] Create an `OpenCodeInstructionsPatch` struct that implements
  `PostWriteHook`:

```rust
/// Post-write hook that patches `opencode.json` instructions with rule file paths.
#[derive(Debug)]
pub struct OpenCodeInstructionsPatch {
    pub rules_dest_dir: PathBuf,
    pub config_dir: PathBuf,
}

impl PostWriteHook for OpenCodeInstructionsPatch {
    fn run(&self, dry_run: bool) -> anyhow::Result<()> {
        patch_opencode_instructions(&self.rules_dest_dir, &self.config_dir, dry_run)
    }
}
```

- [x] Add a `post_write_hook` function that `sync_plan` calls per
  (provider, kind) pair. This replaces the `if provider == OpenCode`
  check that currently lives in `sync_plan`:

```rust
/// Returns a post-write hook for the given file kind, if one is needed.
///
/// Only `OpenCode` rules require a post-write hook (to patch `opencode.json`
/// instructions). All other kinds return `None`.
pub fn post_write_hook(
    kind: FileKind,
    dest: &Path,
    target: &SyncTargetConfig,
    home: &Path,
    cwd: &Path,
) -> Option<Box<dyn PostWriteHook>> {
    if kind != FileKind::Rules {
        return None;
    }
    let config_dir = opencode_config_dir(target, home, cwd);
    Some(Box::new(OpenCodeInstructionsPatch {
        rules_dest_dir: dest.to_path_buf(),
        config_dir,
    }))
}
```

> **Deviation:** The signature was simplified to `(kind, dest, config_dir)`
> instead of `(kind, dest, target, home, cwd)` — the caller resolves paths
> before calling.

- [x] Move `opencode_config_dir` from `sync.rs` to `opencode.rs` (it's
  OpenCode-specific path resolution logic). Make it module-private.

> **Deviation:** `opencode_config_dir` stays in `sync.rs` because it depends on
> binary-crate types (`SyncTargetConfig`, `SyncMode`). The OpenCode adapter's
> `post_write_hook` receives already-resolved paths instead.

- [x] Move all five `patch_opencode_instructions` tests from `emit.rs` to
  `opencode.rs`:
  - `test_patch_no_prior_config_creates_file`
  - `test_patch_preserves_user_entries`
  - `test_patch_replaces_stale_agentspec_entries`
  - `test_patch_empty_rules_dir_removes_agentspec_entries`
  - `test_patch_dry_run_no_file_written`

#### 3. Add no-op `post_write_hook` to Claude and Cursor adapters

**Files**: `src/adapters/claude.rs`, `src/adapters/cursor.rs`

- [x] Add a `post_write_hook` function to each adapter that always returns `None`.
  This establishes a uniform interface across all adapters and prepares for a
  future adapter trait:

```rust
/// Returns a post-write hook for the given file kind, if one is needed.
///
/// Claude does not require any post-write actions.
pub fn post_write_hook(
    _kind: FileKind,
    _dest: &Path,
    _target: &SyncTargetConfig,
    _home: &Path,
    _cwd: &Path,
) -> Option<Box<dyn PostWriteHook>> {
    None
}
```

- [x] Add the necessary imports (`FileKind`, `PostWriteHook`, `SyncTargetConfig`)

> **Deviation:** The Claude/Cursor no-op signatures match the simplified
> `(kind, dest, config_dir)` pattern, not `(kind, dest, target, home, cwd)`.

#### 4. Make `sync_plan` generic via uniform adapter dispatch

**File**: `src/sync.rs`

- [x] Replace the provider-specific `if *provider == OpenCode && kind == Rules`
  block with a uniform dispatch that calls every adapter's `post_write_hook`:

```rust
// Every adapter gets a chance to provide a post-write hook
let hook = match *provider {
    Provider::Claude => claude_post_write_hook(kind, &dest, target, home, cwd),
    Provider::Cursor => cursor_post_write_hook(kind, &dest, target, home, cwd),
    Provider::OpenCode => opencode_post_write_hook(kind, &dest, target, home, cwd),
};
if let Some(h) = hook {
    post_write_hooks.push(h);
}
```

- [x] Remove `opencode_config_dir` from `sync.rs` (moved to opencode adapter
  in step 2)

- [x] Update imports: remove `ConfigPatch`; add `PostWriteHook`;
  import `claude_post_write_hook`, `cursor_post_write_hook`,
  `opencode_post_write_hook` from the adapters module

- [x] Rename `patches` to `post_write_hooks` in the local variable and
  `WritePlan` construction

- [x] Update the `test_sync_plan_produces_correct_writes_and_patches` test:
  assert on `plan.post_write_hooks.len()` instead of `plan.patches.len()`,
  remove the `ConfigPatch::OpenCodeInstructions` pattern match

#### 5. Simplify emit

**File**: `src/emit.rs`

- [x] Delete the `patch_opencode_instructions` function (lines 230-323)

- [x] Delete the `apply_patches` function (lines 325-336)

- [x] Replace `apply_patches(&plan.patches, dry_run)?` in `emit()` with:

```rust
for hook in &plan.post_write_hooks {
    hook.run(dry_run)?;
}
```

- [x] Remove unused imports: `ConfigPatch`, `walkdir::WalkDir`

- [x] Delete all five `patch_opencode_instructions` tests (already moved to
  opencode.rs in step 2)

> **Deviation:** The `patches: vec![]` to `post_write_hooks: vec![]` rename
> was also needed in 7 test helper sites in `emit.rs` (not mentioned in the
> plan but necessary).

#### 6. Re-export `post_write_hook` functions for sync_plan access

**File**: `src/adapters.rs`

- [x] Add `post_write_hook` re-exports for all three adapters, renaming to
  disambiguate at the call site:

```rust
pub use claude::{adapt_claude, post_write_hook as claude_post_write_hook};
pub use cursor::{adapt_cursor, post_write_hook as cursor_post_write_hook};
pub use opencode::{adapt_opencode, post_write_hook as opencode_post_write_hook};
```

#### 7. Update CLAUDE.md pipeline stages

**File**: `CLAUDE.md`

- [x] Update step 7 (Emit) to mention post-write hooks instead of
  "patches `opencode.json`":

```
7. **Emit** — `emit.rs` executes the `WritePlan`: `CleanSlate` mode for `compile`
   (delete-and-rewrite `generated/<provider>/`), `ManifestTracked` mode for `sync`
   (per-file ownership tracking, stale cleanup, direct write to tool config dirs);
   runs post-write hooks (e.g., `OpenCode` `opencode.json` patching). Emit is
   purely file I/O, manifest tracking, and hook execution — no content
   transformation or provider-specific logic.
```

#### Tests

- [x] All five `patch_opencode_instructions` tests pass in their new location
  in `opencode.rs`
- [x] `test_sync_plan_produces_correct_writes_and_patches` updated and passes
- [x] `test_plan_types_construct` updated and passes
- [x] Integration test `test_sync_opencode_commands_prefix_subdir` passes
  (exercises the full sync path including the hook)
- [x] Integration test `test_sync_without_target_only_syncs_configured_providers`
  passes

### Success Criteria:

#### Automated Verification:

- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes (all tests)
- [x] `grep -r "ConfigPatch" src/` returns no matches
- [x] `grep -r "patch_opencode_instructions" src/emit.rs` returns no matches
- [x] `grep -r "patch_opencode_instructions" src/adapters/opencode.rs` returns matches
  (confirming the function moved)

## References

- Idea document: `thoughts/ideas/2026-04-02-agentspec-adapter-post-write-actions.md`
- `src/emit.rs:230-336` — `patch_opencode_instructions` + `apply_patches` (to delete)
- `src/emit.rs:651-795` — five tests to move
- `src/plan.rs:112-155` — `WritePlan`, `ConfigPatch` (to replace)
- `src/sync.rs:42-47` — `ConfigPatch` construction in `sync_plan`
- `src/sync.rs:163-178` — `opencode_config_dir` helper (stays in sync.rs)
- `src/adapters/opencode.rs` — destination for the moved function
- `src/adapters.rs` — re-exports (needs `OpenCodeInstructionsPatch` added)
- `CLAUDE.md` — pipeline stages documentation
- `TODO.md` item #6 — originating TODO
