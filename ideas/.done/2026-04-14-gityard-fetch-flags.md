# gityard fetch: Comprehensive Flag Support

## Problem Statement

`gityard fetch` currently hardcodes `git fetch --prune` with no user-controllable
flags. Users with established git fetch workflows (e.g.,
`git fetch --all --tags --force --prune --jobs=10`) lose control over fetch
behavior when moving to gityard's bulk operations. The command should expose the
most useful `git fetch` flags both as CLI arguments and as configurable defaults.

## Motivation

- **Feature parity with git**: Users expect to control fetch behavior the same
  way they do with raw git commands. A bulk tool that removes control is less
  useful than a shell loop.
- **Multi-repo context amplifies flag value**: Flags like `--dry-run` become
  *more* valuable when operating across many repos — previewing what 30 repos
  would fetch is something you can't easily do with a shell alias.
- **Configurable defaults reduce repetition**: Users who always want `--tags` or
  `--all` shouldn't need to pass them on every invocation.

## Context

Current implementation (`operation.rs:91`):
```rust
Operation::Fetch => vec!["fetch".to_string(), "--prune".to_string()],
```

The `Push` command already demonstrates the pattern for command-specific flags:
a wrapper args struct with `#[command(flatten)]` over `ExecArgs`, fields flowing
into the `Operation` enum variant, and conditional `git_args()` construction.

Repo-level parallelism is already handled by `executor.rs` via `thread::scope` —
all repos fetch concurrently regardless of these flags.

## Goals / Success Criteria

- [ ] All flags listed below are available as CLI arguments on `gityard fetch`
- [ ] `--tags` uses three-state (auto/on/off); `--prune` is on by default; all
  other flags are opt-in
- [ ] Global default fetch flags can be set in gityard config
- [ ] CLI flags override config defaults
- [ ] `--dry-run` output clearly shows what *would* happen per-repo
- [ ] Existing behavior (fetch with `--prune`) is preserved when no flags are passed

## Proposed Flags

### From `gfa` alias (high priority)

| Flag | Default | Notes |
|------|---------|-------|
| `--all` | off | Fetch all remotes. Useful for repos with multiple remotes (e.g., origin + upstream). |
| `--tags` / `--no-tags` | **auto** | By default, git auto-follows tags on fetched commits (no flag passed). `--tags` fetches ALL tags; `--no-tags` disables tag fetching entirely. |
| `--force` | off | Force update of local refs (including tags). |
| `--jobs=N` | unset | Controls git's internal parallelism (submodule fetches, multiple remotes). Distinct from gityard's repo-level parallelism which is always on. |

### Additional flags

| Flag | Default | Notes |
|------|---------|-------|
| `--dry-run` | off | Show what would be fetched without actually fetching. Especially valuable for bulk operations. |
| `--recurse-submodules` | off | Also fetch updates for submodules within each repo. |
| `--prune-tags` | off | Prune local tags that no longer exist on the remote. Complements `--prune`. |
| `--depth=N` | unset | Control shallow clone fetch depth. |
| `--unshallow` | off | Convert a shallow clone to a full clone. |

### Already hardcoded

| Flag | Default | Notes |
|------|---------|-------|
| `--prune` | **on** | Already hardcoded. Should remain default-on but could add `--no-prune` for opt-out. |

## Non-Goals (Out of Scope)

- **Per-group config defaults**: Config defaults are global only. Per-group
  overrides can be added later if needed.
- **Repo-level parallelism control**: gityard already handles this via its
  thread pool. Not exposing a flag for "how many repos fetch concurrently."
- **Refspec support**: Specifying custom refspecs is too advanced for the
  initial implementation.
- **Output verbosity flags** (`--quiet`, `--verbose`): gityard manages its own
  output formatting; passing these through to git could conflict.

## Config Defaults

Global fetch defaults in gityard config, e.g.:

```toml
[fetch]
tags = true       # already proposed as default
all = false
force = false
prune-tags = false
recurse-submodules = false
```

CLI flags override config values. The merge semantics should be straightforward:
CLI flag present = use it; absent = fall back to config; absent from config =
use hardcoded default.

## Decisions

- **`--force` is a pure git passthrough.** Unlike `push --force` which overrides
  a pre-flight safety check, fetch has no pre-flight blocks — it's always
  `Ready`. No dual role needed.
- **`--jobs` defaults to unset (let git decide).** Since gityard already runs
  all repos in parallel via `thread::scope`, adding a high `--jobs` default
  would multiply concurrent network operations aggressively. Users who want it
  can set it in config.
- **`--no-prune` is supported.** Any default-on flag should be overridable for
  consistency (`--tags`/`--no-tags`, `--prune`/`--no-prune`). This also enables
  the config layer to set `prune = false`.
- **`--dry-run` is a git passthrough.** Pass `--dry-run` to git and show its
  native per-ref output, wrapped in gityard's usual per-repo header formatting.
  Familiar to git users and shows exactly what would change.

## Constraints

- Must follow the existing pattern established by `PushArgs` / `CheckoutArgs`
  for adding command-specific flags.
- Default-on flags (`--tags`, `--prune`) need negation variants (`--no-tags`,
  `--no-prune`) for opt-out.

## References

- `git fetch` documentation: flags and behavior
- Existing pattern: `PushArgs` in `cli.rs:69-77`, `Operation::Push` in
  `operation.rs:93-98`
- Current fetch implementation: `operation.rs:91`
