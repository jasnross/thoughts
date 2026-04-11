# Agentspec Sync Explicit Configuration Requirement Implementation Plan

## Overview

Change `agentspec sync` so it does not implicitly sync to user-level defaults when
`[sync.*]` is absent. Sync should operate only on explicitly configured providers
unless the user provides explicit CLI-only sync intent with explicit provider
selection.

## Current State Analysis

_Baseline when this plan was written (pre-implementation)._

- `run_sync` currently chooses providers from `--target` or `config.targets`, not
  from configured sync providers, in `agentspec/src/sync.rs:185`.
- Per-provider sync config currently starts from defaults via
  `self.sync.get(...).cloned().unwrap_or_default()` in
  `agentspec/src/config.rs:249`.
- Default sync mode/strategy is user+symlink from `SyncMode::User` and
  `SyncStrategy::Symlink` in `agentspec/src/types.rs:230` and
  `agentspec/src/types.rs:243`.
- `--dest` is sufficient for CLI-only sync because it fully defines per-kind
  destinations (and implies `mode=path`) in `agentspec/src/config.rs:304`.
- `--mode user` and `--mode project` are also explicit CLI sync intent because
  destination roots are provider-known conventions in `agentspec/src/sync/provider.rs:103`
  and `agentspec/src/sync/provider.rs:104`.
- Existing sync path errors are provider/kind-scoped and actionable, e.g.
  `resolve_dest_dir` error wording in `agentspec/src/sync/provider.rs:112`.

## Desired End State

After implementation:

- Running `agentspec sync` with no `[sync.*]` (and no effective profile sync config)
  exits nonzero with an actionable message explaining that sync config is required.
- Running `agentspec sync` without `--target` syncs only providers with effective
  explicit sync configuration (base `[sync.*]` plus active profile `[profiles.<p>.sync.*]`).
- Running `agentspec sync --target <provider>` errors when that provider is
  unconfigured, unless CLI-only inputs are sufficient to define explicit sync behavior.
- CLI-only scripting workflow is supported when `--target` is provided and one of:
  - `--dest <path>` (implies `mode=path`), or
  - `--mode user`, or
  - `--mode project`.
- `--mode path` without `--dest` is rejected as insufficient in CLI-only mode.
- `agentspec sync --mode user` (or `--mode project`) without `--target` and without
  configured `[sync.*]` providers is rejected to avoid implicit provider fan-out.

### Key Discoveries:

- `resolve_sync_target` currently erases the distinction between configured and
  unconfigured providers by materializing defaults at `agentspec/src/config.rs:249`.
- Provider selection currently happens before any configured/unconfigured filtering
  in `agentspec/src/sync.rs:185`.
- Profile sync overrides live in generic JSON at
  `profiles.<name>.sync.<provider>` and are read during per-provider resolution in
  `agentspec/src/config.rs:252`.

## What We're NOT Doing

- Adding new CLI flags for per-kind sync paths beyond existing `--dest`.
- Changing compile target selection semantics (`config.targets`) for non-sync commands.
- Changing provider destination conventions in `sync/provider.rs`.
- Changing prefix/collision logic implemented in the prior plan.

## Implementation Approach

Introduce explicit sync-target metadata and preflight selection/validation before
sync execution. Keep existing destination resolution and strategy behavior, but
prevent implicit defaults from being treated as configured sync intent.

## Phase 1: Explicit Sync Intent Modeling

### Overview

Add config-level helpers that can tell whether a provider has explicit sync config
from base/profile, and whether CLI overrides provide sufficient explicit intent.

### Changes Required:

#### 1. Sync intent helper types and methods

**File**: `agentspec/src/config.rs`

- [x] Add a helper struct (or equivalent) representing effective sync intent per
      provider, including:
  - [x] whether base/profile config exists for provider
  - [x] whether CLI overrides are sufficient for unconfigured provider
  - [x] final resolved target config (existing behavior)
- [x] Add helper method to list providers explicitly configured for sync under the
      active profile (base `[sync.*]` + profile `profiles.<name>.sync.*`).
- [x] Add helper method to determine whether CLI overrides are sufficient for
      unconfigured providers, with policy:
  - [x] require explicit provider selection via `--target` for CLI-only sync
  - [x] `--dest` is sufficient (implies `mode=path`)
  - [x] `--mode user` is sufficient
  - [x] `--mode project` is sufficient
  - [x] `--mode path` without `--dest` is insufficient
  - [x] `--strategy` alone is insufficient
- [x] Keep current `resolve_sync_target` merge precedence unchanged:
      base → profile → CLI.

```rust
// Sketch
pub struct ResolvedSyncIntent {
    pub target: SyncTargetConfig,
    pub has_explicit_config: bool,
    pub cli_only_allowed: bool,
}
```

### Success Criteria:

#### Automated Verification:

- [x] `cargo test config::tests` passes with new explicit-intent tests
- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes

#### Manual Verification:

- [x] Confirm helper semantics match agreed behavior: unconfigured provider is
      valid only when CLI is sufficient and `--target` is explicit (manual-only: policy confirmation)

**Implementation Note**: After completing this phase and automated checks pass,
pause for confirmation before phase 2 unless instructed to continue.

---

## Phase 2: Sync Target Selection and Fail-Fast Validation

### Overview

Update `run_sync` to select providers from effective explicit config by default,
enforce `--target` behavior, and emit actionable errors when nothing is configured.

### Changes Required:

#### 1. Default provider selection for sync

**File**: `agentspec/src/sync.rs`

- [x] Replace default target selection (`config.targets`) in sync mode with
      effective explicitly configured sync providers.
- [x] Preserve `--target` precedence when user provides explicit provider list.
- [x] If no providers are effectively configured and CLI-only is not sufficient,
      fail early with a clear error message.

#### 2. `--target` validation for unconfigured providers

**File**: `agentspec/src/sync.rs`

- [x] For each targeted provider, require either explicit sync config or
      sufficient CLI-only override.
- [x] When no providers are configured, require `--target` for CLI-only runs
      even when `--mode user`/`--mode project` is provided.
- [x] Emit per-provider actionable error for unconfigured targeted providers.

```rust
// Sketch
if requested_provider_unconfigured && !cli_only_sufficient {
    bail!(
        "sync config for {provider} is not configured; add [sync.{provider}] or pass --target with --mode user|project or --dest"
    );
}
```

#### 3. Error messaging consistency

**Files**: `agentspec/src/sync.rs`, `agentspec/src/config.rs`

- [x] Follow existing actionable error style used in sync/provider validation.
- [x] Ensure top-level no-config error explains how to proceed:
  - [x] add `[sync.<provider>]`
  - [x] or provide CLI-only override with explicit target (`--target ... --mode user|project` or `--target ... --dest ...`)
  - [x] or use `--target` with configured providers

### Success Criteria:

#### Automated Verification:

- [x] `cargo test` passes
- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes

#### Manual Verification:

- [x] Running `agentspec sync` in a repo with no `[sync.*]` exits nonzero and
      prints actionable configuration guidance.
- [x] Running `agentspec sync --target claude` with no `[sync.claude]` and no
      CLI-only mode override exits nonzero with a targeted provider-specific message.
- [x] Running `agentspec sync --target claude --mode user` succeeds in
      selection/preflight and attempts sync with CLI-only config.
- [x] Running `agentspec sync --target claude --mode project` succeeds in
      selection/preflight and attempts sync with CLI-only config.
- [x] Running `agentspec sync --target claude --dest /tmp/out` succeeds in
      selection/preflight and attempts sync with CLI-only config.
- [x] Running `agentspec sync --mode user` with no `[sync.*]` and no `--target`
      exits nonzero with actionable guidance to add `--target`.
  > **Deviation:** Verified via integration tests in `agentspec/tests/pipeline.rs`
  > (`test_sync_no_config_errors_with_guidance`,
  > `test_sync_target_unconfigured_errors_without_dest`,
  > `test_sync_target_unconfigured_with_mode_user_allowed`,
  > `test_sync_target_unconfigured_with_mode_project_allowed`,
  > `test_sync_target_unconfigured_with_dest_allowed`,
  > `test_sync_no_config_mode_user_without_target_errors`) instead of ad-hoc
  > manual CLI runs.

**Implementation Note**: After completing this phase and automated checks pass,
pause for confirmation before phase 3 unless instructed to continue.

---

## Phase 3: Test Coverage Expansion

### Overview

Add focused unit and integration tests to lock in explicit sync behavior,
including profile-driven configuration and CLI-only workflows.

### Changes Required:

#### 1. Config unit tests for explicit sync intent

**File**: `agentspec/src/config.rs`

- [x] Add tests for provider explicit-config detection from `[sync.*]`.
- [x] Add tests for profile-only sync config detection (`profiles.<name>.sync.*`).
- [x] Add tests for CLI-only sufficiency (`--target` required; `--dest` sufficient;
      `--mode user|project` sufficient; `--strategy` alone insufficient).

#### 2. Integration tests for sync command behavior

**File**: `agentspec/tests/pipeline.rs`

- [x] Add `test_sync_no_config_errors_with_guidance`.
- [x] Add `test_sync_without_target_only_syncs_configured_providers`.
- [x] Add `test_sync_target_unconfigured_errors_without_dest`.
- [x] Add `test_sync_target_unconfigured_with_dest_allowed`.
- [x] Add `test_sync_target_unconfigured_with_mode_user_allowed`.
- [x] Add `test_sync_target_unconfigured_with_mode_project_allowed`.
- [x] Add `test_sync_no_config_mode_user_without_target_errors`.
- [x] Add `test_sync_profile_config_only_is_respected` (configured only via profile).

### Success Criteria:

#### Automated Verification:

- [x] `cargo test` passes with all new sync-selection tests
- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] No regressions in existing sync/prefix/collision tests

#### Manual Verification:

- [x] Confirm new integration test messages are readable and actionable
      (manual-only: wording quality review)

---

## Phase 4: Documentation Updates

### Overview

Update sync docs so users understand that sync requires explicit configuration
or sufficient CLI overrides and no longer defaults to implicit user-level sync.

### Changes Required:

#### 1. README sync behavior section

**File**: `agentspec/README.md`

- [x] Add a concise “Sync target selection” subsection explaining:
  - [x] default sync scope is explicitly configured providers only
  - [x] no-config behavior is a fail-fast error with guidance
  - [x] `--target` requires configured provider or sufficient CLI overrides
  - [x] CLI-only workflow with `--target` using `--mode user|project` or `--dest`
- [x] Add examples for:
  - [x] configured-only default sync
  - [x] failing unconfigured targeted sync
  - [x] successful CLI-only targeted sync via `--mode user|project`
  - [x] successful CLI-only targeted sync via `--dest`

### Success Criteria:

#### Automated Verification:

- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes

#### Manual Verification:

- [x] README accurately describes explicit-config and CLI-only behavior.
- [x] Examples match tested command behavior.

---

## Testing Strategy

### Unit Tests:

- [x] Explicit sync provider detection from base config
- [x] Profile-only sync provider detection
- [x] CLI-only sufficiency checks (`--target` requirement, `--mode user|project`, `--dest`, and insufficient combinations)
- [x] Targeted provider validation behavior

### Integration Tests:

- [x] No `[sync.*]` + no sufficient CLI overrides -> fail fast
- [x] Configured subset sync without `--target` -> only configured providers run
- [x] `--target` unconfigured provider -> fail unless explicit CLI-only mode is sufficient (`--mode user|project` or `--dest`)
- [x] no-config + `--mode user`/`--mode project` without `--target` -> fail with guidance
- [x] Profile-only configured sync behavior

### Manual Testing Steps:

- [x] Create temp repo with generated outputs and no `[sync.*]`; run `agentspec sync`
      and verify fail-fast message.
- [x] Add only `[sync.cursor]`; run `agentspec sync` and verify only cursor is synced.
- [x] Run `agentspec sync --target claude --mode user` with no `[sync.claude]`
      and verify CLI-only flow works.
- [x] Run `agentspec sync --target claude --mode project` with no `[sync.claude]`
      and verify CLI-only flow works.
- [x] Run `agentspec sync --target claude --dest /tmp/sync-out` with no
      `[sync.claude]` and verify CLI-only flow works.
  > **Deviation:** These were executed as deterministic integration tests in
  > `agentspec/tests/pipeline.rs` under the named test cases rather than
  > one-off shell commands.

## Performance Considerations

- [x] Keep provider selection checks O(number_of_providers) with no filesystem I/O.
- [x] Avoid additional sync passes; perform preflight once before provider loop.

## Migration Notes

- [x] This is a behavioral breaking change for users relying on implicit default
      user-level sync without `[sync.*]`.
- [x] Mitigation paths:
  - [x] add explicit `[sync.<provider>]` blocks, or
  - [x] script with `--target ... --mode user|project` or `--target ... --dest ...`.

## References

- Related idea: `/Users/jasonr/dotfiles/thoughts/ideas/2026-03-22-agentspec-sync-command.md`
- Related plan: `/Users/jasonr/dotfiles/thoughts/plans/2026-03-23-agentspec-sync-namespace-prefix.md`
- Sync orchestrator/provider loop: `agentspec/src/sync.rs:166`
- Current default target selection: `agentspec/src/sync.rs:185`
- Current default config materialization: `agentspec/src/config.rs:249`
- CLI `--dest` override semantics: `agentspec/src/config.rs:304`
- Path-mode missing-path error style: `agentspec/src/sync/provider.rs:112`
