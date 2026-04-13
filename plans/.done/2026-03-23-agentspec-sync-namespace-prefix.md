# Agentspec Sync Namespace Prefix Implementation Plan

## Overview

Add optional namespace prefixing to `agentspec sync` so that synced skills, agents,
and commands avoid name collisions with existing user-owned files. Prefix behavior
is provider-aware: Claude Code uses a colon-separated `name:` frontmatter field
and a dash-separated filesystem directory; OpenCode uses a subdirectory; Cursor and
Codex use a dash-prefixed filename. Simultaneously, change the collision detection
behavior from "silently back up and continue" to "error and abort" by default, with
an opt-out via `--force` CLI flag or `allow_overwrite` config field.

## Current State Analysis

- `SyncTargetConfig` (`config.rs:96-112`) has 7 fields; no prefix or overwrite concept.
- `sync_one_kind` (`sync.rs:37-71`) computes `source_dir` and `dest_dir`, then calls the
  strategy function with no name transformation capability.
- `sync_symlinked_dir` (`strategy.rs:108-180`) places each source entry at
  `dest_dir.join(source_filename)` — no renaming.
- `sync_copied_dir` / `copy_file` (`strategy.rs:191-333`) similarly uses source-relative
  paths verbatim.
- Collisions (regular file at dest not in manifest) silently produce `BackedUp` action
  at `strategy.rs:226-248` and `strategy.rs:74-88`.
- `apply_strip_name` (`strategy.rs:339-391`) is the only post-processor today;
  it walks `SKILL.md` files and strips `name:` from frontmatter.
- `SyncOverrides` (`config.rs:131-139`) has three fields; no `force` concept.
- `SyncArgs` (`cli.rs:55-79`) has `--dry-run`, `--strategy`, `--dest`, `--mode`;
  no `--force`.

## Desired End State

After implementation:

```toml
# agentspec.toml
[sync.claude]
mode = "user"
strategy = "copy"
prefix = "tw"
# skills land at ~/.claude/skills/tw-commit/SKILL.md with name: tw:commit

[sync.opencode]
mode = "user"
prefix = "tw"
# commands land at ~/.config/opencode/commands/tw/commit.md → invoked as /tw/commit

[sync.cursor]
mode = "user"
prefix = "tw"
# skills land at ~/.cursor/skills/tw-commit.md

[sync.codex]
mode = "user"
prefix = "tw"
# skills land at ~/.codex/skills/tw-commit.md
```

Verification:

- `agentspec sync` with prefix configured → prefixed names in all destinations
- `agentspec sync` with a user-owned file at the dest path → error with actionable message
- `agentspec sync --force` with a user-owned file → backs up and continues
- `prefix = "tw"` and `strip_name = true` together → config error at startup
- `prefix = "tw"` and `strategy = "symlink"` for Claude → config error at startup

## What We're NOT Doing

- Alias generation (making both `/commit` and `/tw:commit` work) — up to the tool
- Collision detection for symlink-vs-symlink (two libraries both owning the same name)
- Prefix for rules — rules are auto-applied by glob/alwaysApply, names not user-invoked
- Per-kind prefix control — prefix applies uniformly to agents, skills, and commands
- Configurable separator — separator is always provider-dictated (colon for Claude `name:`,
  dash for filesystem, subdirectory for OpenCode)
- Multi-source aggregation — each independent spec library has its own `agentspec.toml`

## Key Discoveries

- `apply_strip_name` only operates with copy strategy (`sync.rs:196-198`); `apply_prefix_name`
  must follow the same constraint for Claude.
- For OpenCode, prefix is a subdirectory adjustment on `dest_dir` before sync — no
  filename or content modification needed.
- `SyncTargetPartial` (`config.rs:118-128`) must mirror every new field added to
  `SyncTargetConfig`, wrapped in `Option` for partial-override semantics.
- The stale-cleanup pass in `sync_symlinked_dir` (`strategy.rs:137-177`) checks whether
  a symlink target `starts_with(source_dir)` — this is unaffected by prefix since the
  target still points into the same `source_dir`.
- Manifest keys are source-relative paths (`rel_path`); with copy + prefix, the manifest
  key must be the *prefixed* relative path so stale cleanup works correctly.
- Claude agents live as flat `.md` files in `~/.claude/agents/`; skills live as
  subdirectories with `SKILL.md`. `apply_prefix_name` must handle both file patterns.

---

## Phase 1: Config & Validation

### Overview

Add `prefix` and `allow_overwrite` to `SyncTargetConfig`/`SyncTargetPartial`, add
`force` to `SyncArgs`/`SyncOverrides`, and implement validation that catches
incompatible combinations before any sync work begins.

### Changes Required

#### 1. `SyncTargetConfig` and `SyncTargetPartial` (`config.rs:96-128`)

**File**: `agentspec/src/config.rs`

- [x] Add `prefix: Option<String>` field to `SyncTargetConfig` after `strip_name`
  (default `None` via `#[serde(default)]` already on the struct)
- [x] Add `allow_overwrite: bool` field to `SyncTargetConfig` after `prefix`
  (default `false`)
- [x] Add `prefix: Option<String>` to `SyncTargetPartial` (semantics: `None` = not
  set in TOML, `Some("")` = override to no-prefix, `Some("tw")` = set prefix)
- [x] Add `allow_overwrite: Option<bool>` to `SyncTargetPartial`
- [x] Update `resolve_sync_target` (`config.rs:205-270`) to merge both new fields
  from partial and apply `SyncOverrides.force` as the highest-precedence override
  for `allow_overwrite`

```rust
// In SyncTargetConfig
/// Optional namespace prefix applied to synced skill/agent/command names.
/// For Claude: filesystem dir uses `{prefix}-{name}`, `name:` frontmatter uses `{prefix}:{name}`.
/// For OpenCode: synced into a `{prefix}/` subdirectory.
/// For Cursor/Codex: filename becomes `{prefix}-{name}`.
/// Rules are never prefixed.
pub prefix: Option<String>,

/// Permit overwriting user-owned files at the destination.
/// When false (default), sync errors on collision. Overridden by `--force`.
#[serde(default)]
pub allow_overwrite: bool,
```

#### 2. `SyncOverrides` (`config.rs:131-139`)

**File**: `agentspec/src/config.rs`

- [x] Add `force: bool` to `SyncOverrides`
- [x] In `resolve_sync_target`, apply: if `cli.force` then `resolved.allow_overwrite = true`

#### 3. `SyncArgs` (`cli.rs:55-79`)

**File**: `agentspec/src/cli.rs`

- [x] Add `--force` boolean flag to `SyncArgs`:

```rust
/// Allow overwriting user-owned files at sync destinations (disables collision errors).
#[arg(long)]
pub force: bool,
```

- [x] Thread `force` into `SyncOverrides` construction at `sync.rs:140-144`

#### 4. Validation function

**File**: `agentspec/src/config.rs`

- [x] Add `pub fn validate_for_sync(&self, provider: Provider) -> Result<()>` on
  `SyncTargetConfig`:

```rust
pub fn validate_for_sync(&self, provider: Provider) -> Result<()> {
    if self.prefix.is_some() && self.strip_name {
        anyhow::bail!(
            "sync config for {provider}: `prefix` and `strip_name` are mutually exclusive"
        );
    }
    // Claude agents and skills have name: frontmatter that must be rewritten;
    // this requires copy strategy since symlinks point at immutable generated sources.
    if self.prefix.is_some()
        && provider == Provider::Claude
        && self.strategy == SyncStrategy::Symlink
    {
        anyhow::bail!(
            "sync config for {provider}: `prefix` requires `strategy = \"copy\"` \
             because Claude skill/agent names come from frontmatter"
        );
    }
    Ok(())
}
```

- [x] Call `target.validate_for_sync(*provider)?` in `run_sync` (`sync.rs:154-163`)
  immediately after resolving each target, before calling `sync_provider`

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds with no errors
- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes

#### Manual Verification

- [x] `agentspec.toml` with `prefix = "tw"` and `strip_name = true` in the same
  provider block → `agentspec sync` prints a clear error and exits nonzero
- [x] `agentspec.toml` with `prefix = "tw"` and `strategy = "symlink"` for Claude →
  `agentspec sync` prints a clear error and exits nonzero

> **Note:** Verified in isolated temp dirs by invoking the built binary with
> `sync --no-compile --target claude` against minimal `agentspec.toml` files; both
> scenarios exited nonzero with the expected error message.

---

## Phase 2: Path-level Prefix Application

### Overview

Transform destination paths during sync: adjust `dest_dir` for OpenCode (prefix
subdirectory), and pass a `file_prefix` string to strategy functions for all other
providers (applied to each file/directory name at the destination).

### Changes Required

#### 1. `sync_one_kind` (`sync.rs:37-71`)

**File**: `agentspec/src/sync.rs`

- [x] After `resolve_dest_dir` (line 44), if `target.prefix.is_some()`:
  - For `Provider::OpenCode` AND `kind == SyncKind::Commands`: append prefix as
    subdirectory to `dest_dir`
  - For all other providers (and kinds other than Rules): compute
    `file_prefix = format!("{}-", prefix)` as a local `String`, pass as
    `Some(file_prefix.as_str())` to strategy functions
  - Rules are excluded — prefix is never applied to rules
- [x] Update calls to `sync_symlinked_dir` and `sync_copied_dir` to pass the new
  `file_prefix` parameter

```rust
// In sync_one_kind, after computing dest_dir:
let (effective_dest_dir, file_prefix_owned);
let (dest_dir, file_prefix): (PathBuf, Option<&str>) =
    if let Some(ref pfx) = target.prefix {
        if provider == Provider::OpenCode && kind == SyncKind::Commands {
            // OpenCode: place commands in a prefix subdirectory
            (dest_dir.join(pfx.as_str()), None)
        } else if kind != SyncKind::Rules {
            // All other prefixable providers/kinds: prepend to filename
            file_prefix_owned = format!("{pfx}-");
            (dest_dir, Some(file_prefix_owned.as_str()))
        } else {
            (dest_dir, None)
        }
    } else {
        (dest_dir, None)
    };
```

#### 2. `sync_symlinked_dir` (`strategy.rs:108-180`)

**File**: `agentspec/src/sync/strategy.rs`

- [x] Add `file_prefix: Option<&str>` parameter
- [x] Add helper `fn prefixed_name(name: &OsStr, prefix: Option<&str>) -> OsString`:
  - If `prefix` is `None`: return `name.to_owned()`
  - Else: return `OsString::from(format!("{}{}", prefix, name.to_string_lossy()))`
- [x] Use `prefixed_name` when computing `link` (line 130):
  ```rust
  let dest_name = prefixed_name(&name, file_prefix);
  let link = dest_dir.join(&dest_name);
  ```
- [x] Stale cleanup handling updated to account for prefix transitions and ownership safety
  while preserving expected source-owned cleanup behavior
  > **Deviation:** This was not left "unaffected". The implementation now normalizes
  > symlink target paths and removes stale managed names during prefix migrations,
  > with `allow_overwrite` guards to avoid deleting user-owned links.

#### 3. `sync_copied_dir` and `copy_file` (`strategy.rs:191-333`)

**File**: `agentspec/src/sync/strategy.rs`

- [x] Add `file_prefix: Option<&str>` parameter to `sync_copied_dir`
- [x] After computing `rel` (line 297-299), compute `prefixed_rel`:
  - If `file_prefix` is `None`: use `rel` unchanged
  - Else: prefix the first path component:

```rust
fn prefix_rel_path(rel: &Path, prefix: Option<&str>) -> PathBuf {
    let Some(pfx) = prefix else { return rel.to_path_buf() };
    let mut components = rel.components();
    let Some(first) = components.next() else { return rel.to_path_buf() };
    let prefixed_first = format!("{pfx}{}", first.as_os_str().to_string_lossy());
    PathBuf::from(prefixed_first).join(components.as_path())
}
```

- [x] Use `prefixed_rel` for both `dest` (line 301) and `rel_str` (the manifest key)
  so that stale cleanup correctly matches the prefixed paths in the manifest
- [x] Stale cleanup uses manifest keys; since keys are now prefixed, it naturally
  tracks only the prefixed files — no further changes needed

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds
- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes (existing tests must still pass)
- [x] New unit tests pass (see Phase 5)

#### Manual Verification

- [x] `agentspec sync --dry-run` with `prefix = "tw"` for Cursor shows destination
  paths like `~/.cursor/skills/tw-review-pr.md` in dry-run output
- [x] `agentspec sync --dry-run` with `prefix = "tw"` for OpenCode shows commands
  landing under `~/.config/opencode/commands/tw/`

> **Deviation:** Current dry-run logging is directory-level (`source_dir -> dest_dir`) and
> does not print per-file destination paths for Cursor. Verified behavior by running both
> dry-run and non-dry-run in isolated temp dirs with `HOME` redirected, then confirming
> the resulting path `/tmp/.../.cursor/skills/tw-review-pr.md` exists and that OpenCode
> commands were synced to `/tmp/.../.config/opencode/commands/tw/commit.md`.

---

## Phase 3: Content-level Prefix Application (Claude)

### Overview

After the path-level sync places files at prefixed destinations, a post-processor
rewrites the `name:` frontmatter field to add the colon-separated prefix. This is
only needed for Claude (agents and skills have `name:` frontmatter that determines
invocation), requires copy strategy (cannot modify symlinked source files), and
runs after all kinds are synced — parallel to `apply_strip_name`.

### Changes Required

#### 1. `apply_prefix_name` (`strategy.rs`)

**File**: `agentspec/src/sync/strategy.rs`

- [x] Add new public function `apply_prefix_name(dest_dir: &Path, prefix: &str, dry_run: bool) -> Result<()>`:

```rust
/// Rewrites `name:` in SKILL.md and agent `.md` frontmatter to add a colon-separated prefix.
///
/// For skills: walks `dest_dir` for files named `SKILL.md`.
/// For agents: walks `dest_dir` for top-level `*.md` files (agents are flat files).
/// In both cases, replaces `name: <value>` with `name: <prefix>:<value>` in the
/// YAML frontmatter block only.
///
/// Only operates on files whose `name:` line does not already start with `<prefix>:`.
/// This makes the operation idempotent on re-sync.
pub fn apply_prefix_name(dest_dir: &Path, prefix: &str, dry_run: bool) -> Result<()> { ... }
```

- [x] The function walks `dest_dir` for `.md` files (both `SKILL.md` at any depth and
  top-level `*.md` files for agents).
- [x] Within the frontmatter block, replaces lines matching `name: ` (that don't already
  start with `name: {prefix}:`) with `name: {prefix}:{original_value}`.
- [x] Only writes back if the content changed (same guard as `apply_strip_name`).
- [x] Dry-run prints `would prefix name: in {path}` without writing.

#### 2. `sync_provider` (`sync.rs:170-204`)

**File**: `agentspec/src/sync.rs`

- [x] Apply Claude `name:` prefixing during copy sync for `SyncKind::Agents` and
  `SyncKind::Skills` so copied content lands with prefixed frontmatter in one pass

```rust
let name_prefix = if provider == Provider::Claude {
    match kind {
        SyncKind::Agents => target.prefix.as_deref().map(|p| (p, NamePrefixMode::Agents)),
        SyncKind::Skills => target.prefix.as_deref().map(|p| (p, NamePrefixMode::Skills)),
        _ => None,
    }
} else {
    None
};
```

- [x] Ensure Claude copy sync remains idempotent with prefix enabled (second sync
  reports `Unchanged` for unchanged inputs)

```rust
let action = copy_file(source, &dest, &rel_str, manifest, name_prefix, dry_run)?;
```

- [x] Thread prefix mode from `sync.rs` to strategy copy functions

> **Deviation:** Kept `apply_prefix_name` implemented, but moved Claude `name:` prefixing
> into the copy pipeline (`copy_file` via `name_prefix` mode) instead of mutating dest files
> in a post-pass. This preserves manifest ownership guarantees and makes repeat syncs
> converge to `Unchanged` rather than re-overwriting on every run.

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds
- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes
- [x] New `apply_prefix_name` unit tests pass (see Phase 5)

#### Manual Verification

- [x] After `agentspec sync` with `prefix = "tw"` for Claude (copy strategy):
  - Skill directory at `~/.claude/skills/tw-commit/SKILL.md` exists
  - `cat ~/.claude/skills/tw-commit/SKILL.md` shows `name: tw:commit` in frontmatter
  - Re-running sync produces `Unchanged` (idempotent)

> **Note:** Verified in isolated temp dirs with `HOME` redirected and
> `sync --no-compile --target claude`: first run created `tw-commit/SKILL.md` with
> `name: tw:commit`; second run reported `1 unchanged`.

---

## Phase 4: Collision Detection

### Overview

Change the default behavior when a user-owned file (not tracked by agentspec) exists
at the sync destination from "silently back up and continue" to "error and abort".
Introduce `--force` CLI flag and `allow_overwrite` config field to restore the old
behavior. Thread `allow_overwrite` through the call stack.

### Changes Required

#### 1. `ensure_symlink` (`strategy.rs:43-100`)

**File**: `agentspec/src/sync/strategy.rs`

- [x] Add `allow_overwrite: bool` parameter
- [x] In the "regular file/dir at dest" branch (currently lines ~74-88 that produce
  `BackedUp`):
  - If `!allow_overwrite`: return an `Err` with message:
    ```
    collision: {} exists and is not managed by agentspec; \
    configure a `prefix` in [sync.<provider>] to avoid conflicts, \
    or pass --force to overwrite
    ```
  - If `allow_overwrite`: keep existing backup-and-symlink behavior, return `BackedUp`

#### 2. `copy_file` (`strategy.rs:191-267`)

**File**: `agentspec/src/sync/strategy.rs`

- [x] Add `allow_overwrite: bool` parameter
- [x] In the "not in manifest but dest exists" branch (line ~226-248 that produces
  `BackedUp`):
  - If `!allow_overwrite`: return an `Err` with same message pattern as above
  - If `allow_overwrite`: keep existing backup-and-copy behavior, return `BackedUp`

#### 3. Thread `allow_overwrite` through call stack

**Files**: `strategy.rs`, `sync.rs`

- [x] Add `allow_overwrite: bool` to `sync_symlinked_dir` — pass to `ensure_symlink`
- [x] Add `allow_overwrite: bool` to `sync_copied_dir` — pass to `copy_file`
- [x] Resolve in `sync.rs`: `let allow_overwrite = target.allow_overwrite;`
  (already includes `--force` override from Phase 1 via `SyncOverrides.force`)
- [x] Pass `allow_overwrite` to strategy calls in `sync_one_kind`

#### 4. Update `print_summary` (`sync.rs:97-122`)

**File**: `agentspec/src/sync.rs`

- [x] The `backed_up` count in the summary is still valid when `allow_overwrite = true`.
  No change needed to the summary format — when default behavior errors, `backed_up`
  will always be 0 in non-force runs.

### Success Criteria

#### Automated Verification

- [x] `cargo build` succeeds
- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes (existing `test_regular_file_in_dest_is_backed_up` must be
  updated to pass `allow_overwrite: true`)
- [x] New collision unit tests pass (see Phase 5)

#### Manual Verification

- [x] Place a regular file at `~/.claude/skills/my-skill/SKILL.md`, run
  `agentspec sync` with a spec named `my-skill` → error with message naming the
  conflicting path and suggesting `prefix` or `--force`
- [x] Same scenario with `--force` → sync completes, backup file exists with
  `.bak.<timestamp>` suffix, summary shows 1 backed_up

> **Note:** Verified in isolated temp dirs with `HOME` redirected. Default run
> exited nonzero with `collision: ... configure a \`prefix\` ... or pass --force`.
> `--force` run succeeded, produced `my-skill.bak.<timestamp>/`, and summary reported
> `1 backed up`.

---

## Phase 5: Tests

### Overview

Add unit tests for all new behavior. No integration tests for sync exist yet;
add one end-to-end test using the pipeline.rs fixture pattern.

### Changes Required

#### 1. Strategy unit tests (`strategy.rs` test module)

**File**: `agentspec/src/sync/strategy.rs`

- [x] Update existing `test_regular_file_in_dest_is_backed_up` (line ~468) to pass
  `allow_overwrite: true` (or update its call to `ensure_symlink`)
- [x] `test_ensure_symlink_collision_errors_by_default`: regular file at dest,
  `allow_overwrite = false` → `Err` result, file unchanged, no backup created
- [x] `test_ensure_symlink_collision_allowed_with_force`: regular file at dest,
  `allow_overwrite = true` → `BackedUp`, backup exists, symlink created
- [x] `test_copy_file_collision_errors_by_default`: dest file not in manifest,
  `allow_overwrite = false` → `Err`, original file unchanged
- [x] `test_copy_file_collision_allowed_with_force`: dest file not in manifest,
  `allow_overwrite = true` → `BackedUp`, backup exists, manifest entry added
- [x] `test_sync_symlinked_dir_applies_file_prefix`: source has entry `commit/`,
  `file_prefix = Some("tw-")` → link created at `dest/tw-commit`, target is
  `source/commit`
- [x] `test_sync_symlinked_dir_no_prefix_unchanged`: `file_prefix = None` → existing
  behavior unchanged (regression guard)
- [x] `test_sync_copied_dir_applies_file_prefix`: source has `commit/SKILL.md`,
  `file_prefix = Some("tw-")` → dest has `tw-commit/SKILL.md`, manifest key is
  `tw-commit/SKILL.md`
- [x] `test_sync_copied_dir_prefix_stale_cleanup`: sync with prefix, remove source
  `commit/`, sync again → `tw-commit/` removed from dest and manifest

#### 2. `apply_prefix_name` unit tests (`strategy.rs` test module)

**File**: `agentspec/src/sync/strategy.rs`

- [x] `test_apply_prefix_name_adds_prefix_to_skill`: dir contains
  `skill-foo/SKILL.md` with `name: commit` in frontmatter → after call,
  content has `name: tw:commit`
- [x] `test_apply_prefix_name_does_not_modify_body`: `name:` appears in both
  frontmatter and body → only frontmatter occurrence is modified
- [x] `test_apply_prefix_name_idempotent`: already-prefixed `name: tw:commit` →
  content unchanged on second call
- [x] `test_apply_prefix_name_dry_run`: dry run → content unchanged, message printed
- [x] `test_apply_prefix_name_flat_agent`: dir contains top-level `review-pr.md`
  with `name: review-pr` → after call, `name: tw:review-pr`

#### 3. Config validation unit tests (`config.rs` test module or `tests/`)

**File**: `agentspec/src/config.rs`

- [x] `test_validate_prefix_strip_name_conflict`: `prefix = Some("tw")`,
  `strip_name = true`, any provider → `validate_for_sync` returns `Err`
- [x] `test_validate_prefix_requires_copy_for_claude`: `prefix = Some("tw")`,
  `strategy = Symlink`, `provider = Claude` → `validate_for_sync` returns `Err`
- [x] `test_validate_prefix_symlink_ok_for_opencode`: `prefix = Some("tw")`,
  `strategy = Symlink`, `provider = OpenCode` → `validate_for_sync` returns `Ok`
- [x] `test_validate_prefix_none_no_error`: `prefix = None` → `validate_for_sync`
  returns `Ok` for all providers

#### 4. Provider destination tests (if needed)

**File**: `agentspec/src/sync/provider.rs`

- [x] Verify OpenCode prefix subdirectory behavior is covered by existing tests or
  add `test_opencode_commands_prefix_subdir`: after applying prefix in
  `sync_one_kind`, commands land in `~/.config/opencode/commands/tw/`

> **Deviation:** Added integration coverage in `tests/pipeline.rs` as
> `test_sync_opencode_commands_prefix_subdir` instead of a `provider.rs` unit test,
> because prefix behavior is applied in `sync_one_kind` (orchestrator) rather than
> provider destination helpers.

#### 5. Sync integration tests (`tests/pipeline.rs`)

**File**: `agentspec/tests/pipeline.rs`

- [x] Add `test_sync_prefix_strip_name_conflict_errors`: fixture `agentspec.toml`
  with `[sync.claude] prefix = "tw"` and `strip_name = true`, run
  `agentspec sync --no-compile --target claude`, assert nonzero exit and stderr
  contains the mutual-exclusion message
- [x] Add `test_sync_prefix_symlink_conflict_errors`: fixture `agentspec.toml`
  with `[sync.claude] prefix = "tw"` and `strategy = "symlink"`, run
  `agentspec sync --no-compile --target claude`, assert nonzero exit and stderr
  contains the Claude copy-strategy requirement message

### Success Criteria

#### Automated Verification

- [x] `cargo test` passes with all new tests
- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] No regressions in existing tests

---

## Phase 6: Documentation

### Overview

Update `README.md` to document the new `prefix` and `allow_overwrite` config fields
and the `--force` CLI flag. The README's Sync section is the primary reference for
users configuring `agentspec.toml`; it already documents `strip_name` and needs to
cover the parallel prefix feature alongside collision behavior.

### Changes Required

#### 1. Sync section — namespace prefix (`README.md`)

**File**: `agentspec/README.md`

- [x] Add a `**Namespace prefix:**` paragraph (or subsection) after the Strategies
  block (after line 95) explaining:
  - What `prefix` does and why to use it (collision avoidance with user-owned or
    cross-library names)
  - Provider-specific behavior table: colon in `name:` frontmatter for Claude,
    subdirectory for OpenCode, dash-prefixed filename for Cursor/Codex
  - That rules are excluded from prefixing
  - That Claude requires `strategy = "copy"` when prefix is set
  - That `prefix` and `strip_name` are mutually exclusive

- [x] Add annotated TOML examples for each provider:

```toml
# Namespace prefix: avoids collisions with user-owned or cross-library names.
# Claude: directory becomes tw-commit/, name: frontmatter becomes tw:commit
# Requires strategy = "copy" for Claude (name: is in frontmatter, not filename).
[sync.claude]
strategy = "copy"
prefix = "tw"

# OpenCode: commands land in a tw/ subdirectory → invoked as /tw/commit
[sync.opencode]
prefix = "tw"

# Cursor/Codex: filename becomes tw-commit.md
[sync.cursor]
prefix = "tw"

[sync.codex]
prefix = "tw"
```

#### 2. Sync section — collision detection (`README.md`)

**File**: `agentspec/README.md`

- [x] Add a `**Collision detection:**` paragraph explaining:
  - Sync now errors when a user-owned file exists at the destination (changed from
    silent backup)
  - How to resolve: configure `prefix`, or remove the conflicting file manually
  - `allow_overwrite = true` in config restores the old backup-and-continue behavior
  - `--force` flag as the one-time CLI equivalent
  - Add `agentspec sync --force` to the Usage command table at the top of the README

#### 3. Usage table (`README.md:14-23`)

**File**: `agentspec/README.md`

- [x] Add `agentspec sync --force` entry with a brief description

### Success Criteria

#### Automated Verification

- [x] `cargo fmt --check` passes (Rust source unchanged in this phase)
- [x] `cargo clippy --all-targets` passes

#### Manual Verification

- [x] README Sync section accurately describes `prefix` behavior per provider
- [x] README documents `allow_overwrite` and `--force` with the correct defaults
- [x] README example TOML is consistent with the actual field names in `SyncTargetConfig`

---

## Testing Strategy

### Unit Tests

- Strategy-level tests cover the full behavior matrix: with/without prefix,
  with/without `allow_overwrite`, dry-run variants
- `apply_prefix_name` tests mirror the existing `apply_strip_name` test structure
- Config validation tests use `SyncTargetConfig { ... }` struct literals directly

### Manual Testing Steps

1. Build and install: `cargo install --path .` in `agentspec/`
2. In `agent-config/`, add `[sync.claude]` with `strategy = "copy"`, `prefix = "tw"`
3. Run `agentspec sync --dry-run` → verify output shows `tw-commit`, `tw-describe-pr`, etc.
4. Run `agentspec sync` → verify `~/.claude/skills/tw-commit/SKILL.md` has `name: tw:commit`
5. Place a regular file at a would-be destination, run `agentspec sync` → verify error
6. Same with `--force` → verify backup created, sync completes
7. Remove the `prefix` config, restore original spec library state

## References

- Idea document: `thoughts/ideas/2026-03-23-agentspec-sync-namespace-prefix.md`
- `apply_strip_name` pattern: `agentspec/src/sync/strategy.rs:339-391`
- `SyncTargetConfig`: `agentspec/src/config.rs:94-112`
- `SyncTargetPartial`: `agentspec/src/config.rs:118-128`
- `sync_one_kind` dest computation: `agentspec/src/sync.rs:43-44`
- Strategy collision paths: `strategy.rs:74-88` (symlink), `strategy.rs:226-248` (copy)
- `sync_provider` post-processing pattern: `agentspec/src/sync.rs:195-201`
- `SyncArgs` / `SyncOverrides`: `agentspec/src/cli.rs:55-79`, `agentspec/src/config.rs:131-139`
