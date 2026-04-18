# gityard: Rework Partial-Execution Semantics for Bulk Commands

## Overview

Fix `gityard pull` so branches without an upstream (plus detached / unborn
edge cases) are caught in pre-flight instead of crashing `git pull` at
runtime, then replace the shared `--allow-partial` flag with a pull-only
`--strict` so the common case (partial execution) is the default. Add a
`[pull] strict` config key and a three-rule exit-code policy that
distinguishes runtime failures from user-opted-into strict abort. `fetch`
does not receive strict wiring — it has no pre-flight blockers today, so
the flag and config key would be dead surface area.

## Current State Analysis

**Pre-flight bug.** `preflight(Pull, state)` at `src/operation.rs:47-61` only
checks `is_dirty`. Branches with no upstream pass as `Ready` and then
`git pull` exits non-zero at runtime with
`There is no tracking information for the current branch.`, reported as
`ExecOutcome::Failed`. Under the current `cmd_exec` exit rule
(`src/main.rs:319-331`: bail on `has_failures || !any_succeeded`), the run
exits non-zero even when every other repo pulled successfully.

**Flag orientation.** `ExecArgs::allow_partial` at `src/cli.rs:59-66` is
opt-in strictness-relaxation. The common case — "do what you can" — requires
the flag on every invocation. The inverse (`--no-allow-partial`) reads
awkwardly.

**Inspector already has what preflight needs.** `RepoState.branch` is
populated by `inspector::read_branch` (`src/inspector.rs:141-172`) with three
deterministic shapes: `"(unborn)"`, `"HEAD detached at abc1234"`, or a
branch name. `ahead_remote` / `behind_remote` (`Option<usize>`) are populated
by `ahead_behind_upstream` (`src/inspector.rs:248-274`), which returns
`(None, None)` precisely when the branch has no configured upstream, is
detached, or is unborn. Preflight runs after `inspector::inspect_all` at
`src/main.rs:260`, so no additional git2 calls are needed — the existing
`RepoState` fields discriminate all three cases by string match on `branch`
plus the remote proxy.

**Shared `ExecArgs` is being split.** `ExecArgs { allow_partial, group }` is
currently flattened into `FetchArgs` via `#[command(flatten)]` at
`src/cli.rs:72-73` and used directly by `Pull` at `src/cli.rs:37`. Since
`--strict` is pull-only under the new design, the shared struct goes away:
`Pull` gets a dedicated `PullArgs { strict, no_strict, group }`, and
`FetchArgs` replaces the flattened `exec: ExecArgs` with an inline
`group: Option<String>`. Fetch's git-level flags (`--all`, `--prune`, etc.)
are untouched.

**Config shape.** `Config` at `src/config.rs:27-31` holds `defaults`,
`fetch`, `scan`. The top-level `Config` sections are alphabetical
(`fc80f2a refactor: sort config fields`); inner struct fields use grouped
ordering, not alphabetical. `FetchConfig` already owns the `[fetch]`
section, so adding `strict: bool` extends that struct in place. A new
`PullConfig` for the `[pull]` section slots between `fetch` and `scan`.

**Existing tests invert.** `tests/cli.rs:421-445`
(`test_pull_blocked_when_dirty`) asserts `.failure()` because the current
exit rule bails when nothing succeeded. Under the new default (partial), the
clean repo pulls successfully and the run exits 0. This test must be
renamed + re-asserted, and a new strict-abort test must cover the old
invariant.

### Key Discoveries

- `src/inspector.rs:248-274` — `ahead_behind_upstream` returns `(None, None)`
  uniformly for detached HEAD, unborn branch, and no-upstream. The caller
  cannot distinguish these cases from the remote counts alone, but
  `state.branch` already does.
- `src/main.rs:260,304` — preflight receives `&[RepoState]` produced earlier
  in the same call chain. No extra I/O needed for the new check.
- `src/main.rs:319-331` — exit logic is `has_failures || !any_succeeded`.
  The new three-rule policy makes the `!any_succeeded` leg obsolete; instead
  blocked-without-strict is explicitly a zero-exit case.
- `src/executor.rs:74-95` — the all-or-nothing gate is one `if has_blockers
  && !allow_partial` branch. Inversion is a parameter rename plus dropping
  the `!`.
- `clippy.toml` allows `expect`/`panic` in tests; non-test code must use
  `?`, `match`, or explicit fallbacks (per `CLAUDE.md`).
- `Config::format_annotated` (`src/config.rs:52-88`) iterates
  `effective.keys()` alphabetically and auto-handles new sections — no
  display code changes needed to surface `[pull]`.

## Desired End State

- `gityard pull` on a registry containing a branch without upstream (and no
  `--strict` flag) exits 0, pulling every eligible repo and reporting the
  no-upstream repo as `✗ alias: no upstream tracking branch` alongside any
  dirty-tree blocks.
- `gityard pull --strict` preserves today's all-or-nothing abort when any
  repo is blocked.
- `config.toml` supports `[pull] strict = <bool>`, surfaced by
  `gityard config show`. No `[fetch] strict` key exists.
- `README.md` documents the new default, the `--strict` flag, and the
  config keys.
- `cargo fmt --check && cargo clippy --all-targets && cargo test` pass with
  updated/new tests.

## What We're NOT Doing

- Not introducing a `--allow-partial` alias. The user explicitly wants the
  breaking rename.
- Not adding new pre-flight checks for `fetch`. `Operation::Fetch` stays
  always-`Ready`; `--strict` / `--no-strict` and `[fetch] strict` are NOT
  added to `fetch` today. That wiring can be added alongside any future
  fetch pre-flight blocker as a non-breaking feature addition.
- Not redesigning the outcome taxonomy (`Success` / `Failed` / `Blocked` /
  `Skipped`) or introducing a `BlockReason` enum. The free-form
  `reason: String` is sufficient because strict fires uniformly on any
  `Blocked`.
- Not changing how `git pull` is invoked (no `--ff-only`, autostash, or
  merge-strategy changes).
- Not touching `cmd_trim`'s exit behavior or other non-exec commands.

## Implementation Approach

Two phases, each producing a shippable, independently revertible commit:

1. **Phase 1 (non-breaking bug fix)** — extend `preflight(Pull, ...)` to
   return `Blocked` for unborn branch, detached HEAD, and no upstream,
   using only data already on `RepoState`. Landed as a `fix(operation):`
   commit. With today's `--allow-partial` still intact, users get
   immediate relief from the runtime crash.

2. **Phase 2 (breaking rename)** — delete `ExecArgs`, add `PullArgs
   { strict, no_strict, group }` for `Pull`, inline `group` into
   `FetchArgs`, add `PullConfig`, wire CLI→config→default resolution in
   `main.rs`, rewrite the exit-code policy, update README, and migrate
   tests. Landed as a `feat!(cli):` commit with a `BREAKING CHANGE:`
   trailer.

---

## Phase 1: Pre-flight catches no-upstream / detached / unborn

### Overview

Extend `Operation::Pull` preflight with three new blocked cases derived
from `RepoState` fields already populated by the inspector. No CLI, config,
or exit-code changes. No new git2 calls.

### Changes Required

#### 1. `src/operation.rs` — extend `preflight` for `Pull`

**File**: `src/operation.rs`

- [x] Replace the `Operation::Pull` arm of `preflight` (lines 51-59) with a
      check chain that handles all four block reasons in priority order:
      dirty tree first (most actionable), then the structural cases.

```rust
Operation::Pull => {
    if state.is_dirty {
        PreflightResult::Blocked {
            reason: "working tree is dirty".to_string(),
        }
    } else if state.branch == "(unborn)" {
        PreflightResult::Blocked {
            reason: "unborn branch".to_string(),
        }
    } else if state.branch.starts_with("HEAD detached") {
        PreflightResult::Blocked {
            reason: "detached HEAD".to_string(),
        }
    } else if state.ahead_remote.is_none() && state.behind_remote.is_none() {
        PreflightResult::Blocked {
            reason: "no upstream tracking branch".to_string(),
        }
    } else {
        PreflightResult::Ready
    }
}
```

#### Tests

**File**: `src/operation.rs` (existing `#[cfg(test)] mod tests`)

- [x] Add `test_pull_blocked_when_no_upstream`: `make_state(|s| {
      s.branch = "topic".into(); s.is_on_main_branch = false; s.ahead_remote
      = None; s.behind_remote = None; })` → assert `Blocked { reason }` with
      `reason == "no upstream tracking branch"`.
- [x] Add `test_pull_blocked_when_detached`: `make_state(|s| s.branch =
      "HEAD detached at abc1234".into())` → assert `Blocked` with
      `reason == "detached HEAD"`.
- [x] Add `test_pull_blocked_when_unborn`: `make_state(|s| s.branch =
      "(unborn)".into())` → assert `Blocked` with `reason == "unborn
      branch"`.
- [x] Add `test_pull_dirty_takes_precedence`: dirty tree AND no upstream →
      assert `Blocked` with `reason == "working tree is dirty"` (dirty is
      listed first intentionally; this pins the precedence).
  > **Deviation:** Used `let ... else { panic!(...) }` instead of a
  > wildcard `match` arm. The `wildcard_enum_match_arm` restriction lint
  > rejects `other => panic!(...)` catch-alls; `let-else` is the idiomatic
  > alternative and keeps the test focused on the `Blocked` variant.
- [x] The existing `test_pull_ready_when_clean` uses `make_state(|_| {})`
      which sets `ahead_remote = Some(0)`, `behind_remote = Some(0)`, and
      `branch = "main"` — verify this still passes unchanged (it should; the
      new branches only fire when both remote counts are `None` or when the
      branch string matches the detached/unborn sentinels).

**File**: `tests/cli.rs`

- [x] Add `test_pull_allow_partial_skips_no_upstream_repo`: create one
      repo with upstream via `init_repo_with_remote`, and one plain repo
      via `init_real_git_repo` (no remote, no tracking). Register both,
      run `gityard pull --allow-partial`, and assert:
      - `.success()` (clean repo pulls, no-upstream repo blocked → at
        least one success, no failures).
      - stdout contains `"succeeded"` and `"no upstream tracking branch"`.

> **Deviation (prerequisite fix):** Phase 1 uncovered 5 pre-existing
> `clippy::map_unwrap_or` errors surfaced by rust 1.95 in `src/branches.rs`,
> `src/presenter.rs` (×2), `src/trimmer.rs`, and `src/worktrees.rs`. These
> existed on `main` and blocked `just check`. Fixed mechanically using the
> replacements suggested by clippy itself (`.map_or(...)` / `.is_ok_and(...)`)
> as a separate change so Phase 1 verification can run green.

### Success Criteria

#### Automated Verification

- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes, including the four new unit tests in
      `operation.rs` and the new integration test in `tests/cli.rs`
- [x] `just check` passes end-to-end

#### Manual Verification

- [ ] On a registry that includes a real branch with no upstream, `gityard
      pull --allow-partial` exits 0, shows the no-upstream repo with the
      red `✗` and the new reason, and pulls every other repo (manual-only:
      verifies the end-to-end user-visible symptom the idea file
      reproduces).

---

## Phase 2: Invert flag, add per-command strict config, rewrite exit codes

### Overview

Breaking change. Delete `ExecArgs`; add a new `PullArgs { strict,
no_strict, group }` used directly by the `Pull` subcommand and inline
`group: Option<String>` into `FetchArgs` (replacing the flattened
`exec: ExecArgs`). Add `PullConfig` (`[pull]` TOML section) with
`strict: bool`. Wire a CLI → config → built-in-`false` resolver so CLI
flags override config. Replace the exit-code rule in `cmd_exec` with the
three-rule policy. Update README and integration tests.

### Changes Required

#### 1. `src/cli.rs` — delete `ExecArgs`, add `PullArgs`, inline `group` into `FetchArgs`

**File**: `src/cli.rs`

- [x] Delete the `ExecArgs` struct (lines 57-66).
- [x] Add `PullArgs` following the `--tags` / `--no-tags` pattern already
      used at lines 80-85:

```rust
/// Arguments for the `pull` subcommand.
#[derive(Debug, Parser)]
pub struct PullArgs {
    /// Abort the entire run if any repo is blocked
    #[arg(long, conflicts_with = "no_strict")]
    pub strict: bool,
    /// Disable strict mode (overrides config)
    #[arg(long = "no-strict", conflicts_with = "strict")]
    pub no_strict: bool,
    /// Target a specific group
    #[arg(long)]
    pub group: Option<String>,
}
```

- [x] Change `Command::Pull(ExecArgs)` (line 37) to
      `Command::Pull(PullArgs)`.
- [x] Update the `Pull` subcommand doc comment (line 36) to reflect the new
      default: `/// Pull from remotes (repos that fail pre-flight are
      skipped; use --strict to abort on any block)`.
- [x] Modify `FetchArgs` (lines 69-126): replace
      `#[command(flatten)] pub exec: ExecArgs,` with
      `/// Target a specific group\n#[arg(long)]\npub group:
      Option<String>,`. All other fetch-specific fields are unchanged.

#### 2. `src/config.rs` — add `PullConfig`

**File**: `src/config.rs`

- [x] `FetchConfig` is unchanged. No `strict` field is added.
- [x] Add `PullConfig` and slot it into `Config` between `fetch` and
      `scan` (alphabetical, matching the ordering established in
      `fc80f2a`):

```rust
#[derive(Debug, Default, Deserialize, Serialize)]
#[serde(default, deny_unknown_fields)]
pub struct Config {
    pub defaults: DefaultsConfig,
    pub fetch: FetchConfig,
    pub pull: PullConfig,
    pub scan: ScanConfig,
}

/// Configuration defaults for the `pull` command.
#[derive(Debug, Default, Deserialize, Serialize)]
#[serde(default, deny_unknown_fields)]
pub struct PullConfig {
    /// Abort the run if any repo is blocked in pre-flight.
    pub strict: bool,
}
```

#### Tests for Phase 2 — `src/config.rs`

**File**: `src/config.rs` (existing `#[cfg(test)] mod tests`)

- [x] Add `test_pull_config_defaults`: `PullConfig::default().strict ==
      false`.
- [x] Add `test_pull_config_parses`: write `[pull]\nstrict = true\n` to a
      tempdir config; `Config::load(...).pull.strict == true`.
- [x] Add `test_pull_config_rejects_unknown_fields`: `[pull]\nbogus =
      true\n` → `Config::load` returns an error containing `"unknown
      field"`.
- [x] Add `test_fetch_config_rejects_strict_field`: write
      `[fetch]\nstrict = true\n` → `Config::load` returns an error
      containing `"unknown field"`. Pins that we did NOT add a
      `FetchConfig.strict` field, so users who try the symmetric key get
      a clear error rather than silent no-op acceptance.
- [x] Update `test_default_template_has_expected_content` (line 439) to
      assert `template.contains("# [pull]\n# strict = false")` — scoping
      the match to the `[pull]` section specifically, since
      `# strict = false` also appears in the `[fetch]` section's generated
      output and an unscoped `.contains` would match either. The template
      is generated from `Config::default()` so it auto-syncs; this
      assertion pins the section layout.
- [x] Update `test_format_annotated_all_defaults` (line 335) to also
      assert `output.contains("[pull]")`.
- [x] Update `test_format_toml_roundtrips` (line 399) to also assert
      `parsed.pull.strict == config.pull.strict`.

#### 3. `src/executor.rs` — parameter rename + inverted gate

**File**: `src/executor.rs`

- [x] Rename the `allow_partial: bool` parameter of `execute` (line 54) to
      `strict: bool`.
- [x] Invert the gate at line 74: `if has_blockers && !allow_partial` →
      `if has_blockers && strict`.
- [x] Update the doc comment (lines 41-49): replace the `allow_partial`
      description with:

```rust
/// If `strict` is true and any repo is blocked, the operation is aborted
/// for all repos (all-or-nothing semantics). Otherwise, blocked repos are
/// skipped and the remaining repos execute.
```

#### Tests for Phase 2 — `src/executor.rs`

**File**: `src/executor.rs` (existing `#[cfg(test)] mod tests`)

- [x] Keep the name of `test_all_or_nothing_blocks_all_when_one_blocked`
      but change the call `execute(..., false, ...)` →
      `execute(..., true, ...)` (now passing `strict = true`). Also update
      the embedded assertion message at `src/executor.rs:310` from
      `"all repos should be blocked when one is blocked and allow_partial=false"`
      → `"all repos should be blocked when one is blocked and strict=true"`.
- [x] Rename `test_allow_partial_skips_blocked` →
      `test_non_strict_skips_blocked` and change
      `execute(..., true, ...)` → `execute(..., false, ...)` (now passing
      `strict = false`). Same assertions apply.
- [x] The other four tests (`test_report_format`,
      `test_progress_callback_receives_correct_counts`,
      `test_execute_without_callback_returns_valid_report`,
      `test_all_or_nothing_blocking_calls_callback`) pass `false` or have
      no blockers — update any `false` that intended "allow partial" to
      its new-meaning equivalent and verify assertions still hold. Map
      mechanically: old `false` (don't allow partial) → new `true` (strict).

#### 4. `src/main.rs` — resolve strict, thread through `cmd_exec`, rewrite exit policy

**File**: `src/main.rs`

- [x] Add a helper near `resolve_fetch_op` (line 612):

```rust
/// Merge CLI strict flags with config default. CLI flags take precedence.
fn resolve_strict(args: &PullArgs, config_strict: bool) -> bool {
    if args.strict {
        true
    } else if args.no_strict {
        false
    } else {
        config_strict
    }
}
```

- [x] Update `cmd_exec` (line 255). New signature, in this parameter
      order (matches the call sites below):

```rust
fn cmd_exec(
    config_dir: &Path,
    config: &Config,
    group: Option<&str>,
    op: &Operation,
    strict: bool,
) -> Result<()>
```

      Internally, replace `args.allow_partial` with the `strict`
      parameter in the call to `executor::execute` (line 304-310) and
      replace the existing `args.group.as_deref()` (passed to
      `resolve_entries`) with the new `group` parameter.
- [x] Update the dispatch in `main()`:
      - Line 48-51 (`Fetch` match arm): `cmd_exec(&config_dir, &config,
        args.group.as_deref(), &op, false)` — fetch passes `strict =
        false` unconditionally today.
      - Line 52 (`Pull`): `let strict = resolve_strict(&args,
        config.pull.strict); cmd_exec(&config_dir, &config,
        args.group.as_deref(), &Operation::Pull, strict)`.
- [x] Update the import block at line 8-11: replace `ExecArgs` with
      `PullArgs` in the `use cli::{...}` list. The list is alphabetical,
      so `PullArgs` sorts between `GroupCommand` and `StatusArgs` (not
      in place of `ExecArgs`, which sat between `ConfigCommand` and
      `FetchArgs`).
- [x] Replace the exit rule (lines 319-331) with the three-rule policy:

```rust
let has_failures = report
    .outcomes
    .iter()
    .any(|(_, o)| matches!(o, executor::ExecOutcome::Failed { .. }));
let has_blocked = report
    .outcomes
    .iter()
    .any(|(_, o)| matches!(o, executor::ExecOutcome::Blocked { .. }));

if has_failures {
    anyhow::bail!("operation failed for one or more repositories");
}
if strict && has_blocked {
    anyhow::bail!("aborted: one or more repositories are blocked (strict mode)");
}
Ok(())
```

- [x] Update the existing `#[cfg(test)] mod tests` in `main.rs` (lines
      668-857): the `default_fetch_args()` helper at line 673 constructs a
      `FetchArgs` with a nested `exec: ExecArgs { allow_partial, group }`.
      Drop the nested struct and set `group: None` directly on
      `FetchArgs`. These tests only exercise `resolve_fetch_op` (which
      doesn't read strict/group), so no new assertions are needed — just
      update the helper to match the new struct shape.

#### Tests for Phase 2 — `src/main.rs`

**File**: `src/main.rs`

- [x] Add `test_resolve_strict_cli_true`: `PullArgs { strict: true,
      no_strict: false, group: None }` with `config_strict = false` →
      `true`.
- [x] Add `test_resolve_strict_cli_no_strict_overrides_config_true`:
      `PullArgs { strict: false, no_strict: true, group: None }` with
      `config_strict = true` → `false`.
- [x] Add `test_resolve_strict_falls_through_to_config`: both CLI flags
      `false`, `config_strict = true` → `true`; `config_strict = false` →
      `false`.

#### 5. `tests/cli.rs` — rewrite inverted tests, add strict/config coverage

**File**: `tests/cli.rs`

- [x] Rename `test_pull_blocked_when_dirty` (line 421) →
      `test_pull_strict_aborts_when_dirty`. Change the command invocation
      to `["pull", "--strict"]` and keep the `.failure()` + `"blocked"`
      assertion (this pins the old all-or-nothing behavior under the new
      opt-in flag).
- [x] Rename `test_pull_allow_partial` (line 447) →
      `test_pull_default_skips_blocked`. Remove `--allow-partial` from the
      args (partial is now default). Keep the assertion that stdout
      contains both `"succeeded"` and `"blocked"` and expect
      `.success()`.
- [x] Add `test_pull_default_blocked_only_exits_zero`: register a single
      dirty repo (no clean counterpart), run `gityard pull`, expect
      `.success()` and stdout containing `"blocked"`. This pins the new
      policy that partial-mode with all repos blocked still exits 0.
- [x] Add `test_pull_strict_from_config`: write `config.toml` containing
      `[pull]\nstrict = true`, register clean + dirty repos, run `gityard
      pull` (no CLI flag), expect `.failure()` + both blocked — proves
      config-driven strict.
- [x] Add `test_pull_no_strict_overrides_config`: config has `[pull]\n
      strict = true`, invoke `gityard pull --no-strict`, expect
      `.success()` + partial behavior — proves CLI overrides config.
- [x] Add `test_pull_help_lists_strict_flags`: assert `gityard pull
      --help` stdout contains both `--strict` and `--no-strict`. Also
      assert it does NOT contain `--allow-partial` (pins the breaking
      rename).
- [x] Add `test_fetch_help_omits_strict_flags`: assert `gityard fetch
      --help` stdout does NOT contain `--strict`, `--no-strict`, or
      `--allow-partial`. Pins that fetch does not carry the pull-only
      strict wiring.
- [x] Rename Phase 1's `test_pull_allow_partial_skips_no_upstream_repo`
      (added to `tests/cli.rs` during Phase 1) to
      `test_pull_default_skips_no_upstream_repo`. Remove the
      `--allow-partial` argument from the `gityard` invocation — partial
      is the default now, so the flag no longer exists. Stdout and exit
      assertions stay the same (clean-upstream repo succeeds,
      no-upstream repo blocked, exit 0).
- [x] Final sweep: `grep -r "allow-partial\|allow_partial" src/ tests/`
      must return no results. Three tests reference the flag pre-rename
      (`test_pull_blocked_when_dirty`, `test_pull_allow_partial`, and the
      Phase-1-added `test_pull_allow_partial_skips_no_upstream_repo`);
      all three are renamed/updated by the bullets above.
  > **Deviation:** The final grep still returns 2 matches — both are
  > negative assertions inside `test_pull_help_lists_strict_flags` and
  > `test_fetch_help_omits_strict_flags` that assert `--allow-partial`
  > does NOT appear in help output. These are correct, non-stale references.

#### 6. `README.md` — document new default, flag, config

**File**: `README.md`

- [x] Update the "Fetch & Pull" section (lines 122-137). Rewrite the
      `gityard pull` paragraph to:

```
**`gityard pull`** — Pull from remotes. Repos that fail pre-flight (dirty
working tree, no upstream tracking branch, detached HEAD, or unborn
branch) are skipped and the rest proceed. Pass `--strict` to abort the
entire run if any repo is blocked, or set `[pull] strict = true` in
`config.toml` to make that the default.
```

- [ ] Leave the `gityard fetch` paragraph substantively unchanged. It
      already describes the behavior accurately; the new `--strict` /
      `--no-strict` flags are shared with `pull` and don't need a
      dedicated callout on `fetch` today.

- [ ] No changelog file exists. Commit this change on a feature branch and
      land it with a `feat!(cli):` commit whose body contains a
      `BREAKING CHANGE:` trailer summarizing the rename and the new exit
      policy.

### Success Criteria

#### Automated Verification

- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes (watch for
      `uninlined_format_args` on the new bail messages and
      `doc_markdown` on README-style identifiers in new doc comments;
      wrap `PullConfig`, `PullArgs`, `FetchArgs`, `FetchConfig`,
      `RepoState` in backticks)
- [x] `cargo test` passes, including:
  - [x] Renamed executor tests with inverted boolean arguments
  - [x] New `main.rs` unit tests for `resolve_strict`
  - [x] Renamed integration tests (`test_pull_strict_aborts_when_dirty`,
        `test_pull_default_skips_blocked`)
  - [x] New integration tests for config-driven strict and `--no-strict`
        override
  - [x] New integration test asserting help output lists `--strict` /
        `--no-strict` and does not list `--allow-partial`
- [x] `cargo deny check licenses` passes (no new dependencies are
      introduced; this check should be a no-op but `just check` runs it)
- [x] `just check` passes end-to-end (fmt + lint + build + test +
      licenses)
- [x] `grep -r "allow_partial\|allow-partial" src/ tests/` returns no
      stale references (2 remaining matches are intentional negative
      assertions in help-output tests)

#### Manual Verification

- [ ] On a dirty registry, `gityard pull` (no flag) exits 0 with the
      dirty repo blocked and clean repos pulled — pins the new default UX
      that the idea file explicitly asks for (manual-only: verifies
      end-to-end behavior change and output styling).
- [ ] `gityard pull --strict` on the same registry exits non-zero with
      all repos marked blocked — pins that the old all-or-nothing
      behavior is accessible via the new flag (manual-only).
- [ ] `gityard config show` displays the new `[pull]` section with a
      `strict` key annotated as `# (default)` when unset; the `[fetch]`
      section does NOT contain a `strict` key (manual-only: verifies the
      config-show integration picks up the new section and confirms fetch
      is untouched).

---

## Testing Strategy

### Cross-Phase Testing

- End-to-end scenario after both phases land: a real workspace with a mix
  of (clean+tracking), (clean+no-upstream), (dirty+tracking), and
  (detached HEAD) repos. Expected: `gityard pull` exits 0; stdout shows
  one `✓` line per clean-tracking repo and three `✗` lines with distinct
  reasons; `gityard pull --strict` exits non-zero with every repo showing
  `✗ blocked`.

### Manual Testing Steps

1. Set up a tempdir workspace with three repos:
   - one cloned-from-bare repo with upstream tracking
   - one `git init`'d repo with no remote
   - one `git init`'d repo with a detached HEAD
2. Register all three via `gityard add`.
3. Run `gityard pull` and verify exit 0 and the expected ✓/✗ lines.
4. Run `gityard pull --strict` and verify non-zero exit with all blocked.
5. Write `[pull]\nstrict = true` to the test config dir's `config.toml`
   and re-run `gityard pull` (no flag) — should match step 4.
6. Run `gityard pull --no-strict` with the same config — should match
   step 3.

## Documentation Updates

- [ ] `README.md` — update the "Fetch & Pull" section (covered in Phase 2
      step 6 above).
- [ ] No inline doc changes required beyond the `PullArgs` /
      `PullConfig` / `execute` doc comments already listed above.
- [ ] No separate architecture or ADR document exists for this project;
      the conventional-commit `BREAKING CHANGE:` trailer is the record.

## References

- Idea source: `thoughts/ideas/2026-04-17-gityard-partial-execution-semantics.md`
- Current preflight: `src/operation.rs:47-61`
- Current executor gate: `src/executor.rs:74-95`
- Current exit rule: `src/main.rs:319-331`
- Inspector data source used by the new preflight branches:
  `src/inspector.rs:141-172` (branch-name sentinels) and
  `src/inspector.rs:248-274` (upstream proxy)
- Existing `--tags` / `--no-tags` CLI pattern to mirror: `src/cli.rs:80-85`
- Existing config resolver pattern to mirror: `src/main.rs:612-647`
- User feedback (memory): "Config granularity — prefer per-command over
  per-reason" — justifies a single `strict` bool per command section rather
  than per-reason strictness.
- Conventional Commits spec (per `CLAUDE.md`): Phase 1 ships as
  `fix(operation):`; Phase 2 ships as `feat!(cli):` with `BREAKING CHANGE:`
  trailer.
