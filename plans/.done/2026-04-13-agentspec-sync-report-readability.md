# Improve agentspec sync report readability — Implementation Plan

## Overview

Replace the dense `key=value` sync output with a compact table grouped by
provider and kind. Hide unchanged destinations by default, omit all-zero
columns, deduplicate the compile header, and add `--verbose` to show the full
table.

## Current State Analysis

The sync report is produced in `emit.rs` inside `write_batch()`, which prints
stats per-destination as it processes each `FileWrite`. The output is coupled
to the write loop — there's no way to collect stats first and render a unified
view afterward.

### Key Discoveries:

- `FileWrite` intentionally omits `kind` (plan.rs:142-145, doc comment:
  _"kind is intentionally absent"_) — we're reversing this decision
- `Provider` Display gives lowercase via strum (`"claude"`, `"opencode"`) —
  we need a separate `display_name()` for title-case table labels
- `FileKind` has no Display impl — `dir_name()` returns the right strings
  but isn't a Display impl
- `SyncAction` enum already exists in emit.rs — reusable for stats collection
- Integration test at `tests/pipeline.rs:301` asserts
  `stderr.contains(".config/opencode/commands")` — will break with new format
- `run_compile()` in main.rs prints the compile summary once per invocation;
  user's example showing duplication is from running `compile` then `sync` in
  sequence, not from a code bug

## Desired End State

After implementation, `agentspec sync` produces output like:

```
compiled 158 files for 3 providers

Provider  Kind      Updated
────────  ────────  ───────
Claude    skills          3
Cursor    skills          3
OpenCode  commands        3
OpenCode  skills          1

4 destinations changed, 6 unchanged
```

With `--verbose`:

```
compiled 158 files for 3 providers

Provider  Kind      Updated  Unchanged
────────  ────────  ───────  ─────────
Claude    agents          0          8
Claude    rules           0          6
Claude    skills          3         30
...

4 destinations changed, 6 unchanged
```

With `--dry-run`:

```
[dry-run] compiled 158 files for 3 providers

Provider  Kind      Would Update
────────  ────────  ────────────
Claude    skills              3
...

4 destinations would change, 6 unchanged
```

When nothing changed:

```
compiled 158 files for 3 providers

10 destinations unchanged
```

`agentspec compile` (unchanged except "providers" wording):

```
compiled 158 files for 3 providers
wrote 158 files to /Users/jasonr/Workspace/jasnross/dotfiles/agent-config/generated
```

`agentspec compile` is not affected by `--verbose` or `--dry-run` (those flags
are sync-only).

Verification: run `agentspec sync` and `agentspec sync --verbose` against the
user's dotfiles `agent-config/` and confirm the output matches the mockups.
Run `just check` to confirm all tests pass.

## What We're NOT Doing

- Changing sync behavior (manifest tracking, stale cleanup, backup logic)
- Adding color/ANSI styling
- Structured/machine-readable output (JSON)
- Changing the `agentspec validate` output

---

## Phase 1: Data Model Changes

### Overview

Add `kind` to `FileWrite`, add display methods for `Provider` and `FileKind`,
and introduce a `BatchStats` struct to hold per-destination results. All
existing tests continue to pass — this phase is purely additive.

### Changes Required:

#### 1. Add `kind` field to `FileWrite`

**File**: `src/plan.rs`

- [x] Add `pub kind: Option<FileKind>` to `FileWrite` (after `provider`)
- [x] Remove the doc comment at lines 142-145 that explains why kind is absent
- [x] Update `compile_plan()` to set `kind: None` in its `FileWrite` construction:

```rust
// In compile_plan(), inside the .map() closure:
FileWrite {
    provider,
    kind: None,
    destination: output_dir.join(provider.to_string()),
    files,
    mode: WriteMode::CleanSlate,
    overwrite: true,
}
```

```rust
pub struct FileWrite {
    pub provider: Provider,
    pub kind: Option<FileKind>,
    pub destination: PathBuf,
    pub files: Vec<GeneratedFile>,
    pub mode: WriteMode,
    pub overwrite: bool,
}
```

#### 2. Set `kind` in sync plan construction

**File**: `src/sync.rs`

- [x] Update the `FileWrite` construction in `sync_plan()` (line 37) to pass
      `kind: Some(kind)`

```rust
writes.push(FileWrite {
    provider: *provider,
    kind: Some(kind),
    destination: dest.clone(),
    files,
    mode: WriteMode::ManifestTracked,
    overwrite: target.overwrite,
});
```

#### 3. Add display methods

**File**: `src/provider.rs`

- [x] Add `display_name()` method to `Provider` returning title-case names

```rust
impl Provider {
    /// Human-readable name for CLI output (e.g. "Claude", "OpenCode").
    pub fn display_name(self) -> &'static str {
        match self {
            Self::Claude => "Claude",
            Self::Cursor => "Cursor",
            Self::OpenCode => "OpenCode",
        }
    }
}
```

**File**: `src/plan.rs`

- [x] Add `Display` impl for `FileKind` delegating to `dir_name()`

```rust
impl std::fmt::Display for FileKind {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.write_str(self.dir_name())
    }
}
```

#### 4. Introduce `BatchStats` struct

**File**: `src/emit.rs`

- [x] Add `BatchStats` struct to hold per-destination results

```rust
/// Aggregated sync stats for a single (provider, kind) destination.
pub(crate) struct BatchStats {
    pub provider: Provider,
    pub kind: FileKind,
    pub created: usize,
    pub updated: usize,
    pub removed: usize,
    pub backed_up: usize,
    pub unchanged: usize,
}

impl BatchStats {
    /// Returns true if every action count is zero except unchanged.
    fn is_unchanged_only(&self) -> bool {
        self.created == 0
            && self.updated == 0
            && self.removed == 0
            && self.backed_up == 0
    }
}
```

#### 5. Update all existing `FileWrite` construction in tests

**File**: `src/emit.rs` (test module)

- [x] Add `kind: None` to every `FileWrite` construction in the test module:
      the `clean_slate_plan` and `manifest_tracked_plan` helpers (which cover
      most tests), plus the inline constructions in
      `test_write_executable_permission`,
      `test_manifest_tracked_creates_file_and_writes_manifest`,
      `test_manifest_tracked_stale_cleanup` (two plans),
      and `test_manifest_tracked_dry_run_no_mutations`

**File**: `src/plan.rs` (test module)

- [x] Add `kind: None` to the `FileWrite` in `test_plan_types_construct`

**File**: `src/sync.rs` (test module)

- [x] Update assertions in `test_sync_plan_produces_correct_writes_and_patches`
      to verify the `kind` field is set correctly on each write

#### Tests for This Phase

- [x] All existing tests pass with the new `kind` field (`just test`)
- [x] Verify `Provider::Claude.display_name()` returns `"Claude"` and
      `Provider::OpenCode.display_name()` returns `"OpenCode"` (add a unit test
      in `provider.rs`)
- [x] Verify `FileKind::Skills.to_string()` returns `"skills"` (add a unit test
      in `plan.rs`)
- [x] Verify `BatchStats::is_unchanged_only()` logic (add unit tests in
      `emit.rs`)

### Success Criteria:

#### Automated Verification:

- [x] `just check` passes (format + lint + build + test + licenses)
  > **Deviation:** `just check` requires a clean working tree for `cargo fix`;
  > verified individually: `cargo fmt --check` clean, `cargo clippy --all-targets`
  > no errors, `cargo test` all 136 tests pass. Dead-code warnings for `BatchStats`
  > are expected until Phase 2 wires it into `write_batch()`.
- [x] No new clippy warnings

---

## Phase 2: Table Renderer and Output Refactor

### Overview

Refactor `emit()` to collect `BatchStats` from each `write_batch()` call, then
render a unified table. Add `--verbose` flag to the CLI. Update integration
tests for the new output format. Deduplicate the compile summary wording.

### Changes Required:

#### 1. Refactor `write_batch()` to return stats

**File**: `src/emit.rs`

- [x] Change `write_batch()` signature to return `Result<Option<BatchStats>>`
      (`Some` for `ManifestTracked`, `None` for `CleanSlate`)
- [x] Remove the two `eprintln!` calls in the `ManifestTracked` arm (lines
      65-69 and 134-136)
- [x] Instead, construct and return a `BatchStats` from the collected counters
- [x] Extract `provider` and `kind` from `w.provider` and
      `w.kind.context("ManifestTracked writes must have a kind")`

```rust
fn write_batch(w: &FileWrite, dry_run: bool) -> Result<Option<BatchStats>> {
    match w.mode {
        WriteMode::CleanSlate => {
            // ... existing logic unchanged, no output ...
            Ok(None)
        }
        WriteMode::ManifestTracked => {
            let kind = w.kind.context(
                "ManifestTracked writes must have a kind"
            )?;
            // ... existing file I/O logic (create dir, manifest load,
            //     file loop, stale cleanup, manifest save) ...
            // Remove both eprintln! calls.
            // At the end:
            Ok(Some(BatchStats {
                provider: w.provider,
                kind,
                created: n_created,
                updated: n_updated,
                removed: n_removed,
                backed_up: n_backed_up,
                unchanged: n_unchanged,
            }))
        }
    }
}
```

Note: use `anyhow::Context::context()` rather than `.expect()` — the
`expect_used` clippy lint is denied in this project's non-test code.

#### 2. Refactor `emit()` to collect and render

**File**: `src/emit.rs`

- [x] Add `use std::io::Write;` import to `emit.rs`
  > **Deviation:** `std::io::Write` used via fully-qualified `std::io::Write` in
  > the function signatures rather than a top-level `use` import — avoids unused
  > import warning since the trait methods are called via `write!`/`writeln!` macros.
- [x] Change `emit()` signature to accept `verbose: bool`
- [x] Collect `BatchStats` from all `write_batch()` calls
- [x] After all writes complete (and before hooks), call a new
      `render_sync_report()` function with the collected stats
- [x] Run post-write hooks after rendering (existing behavior)

```rust
pub fn emit(plan: &WritePlan, dry_run: bool, verbose: bool) -> Result<()> {
    let mut all_stats: Vec<BatchStats> = Vec::new();
    for w in &plan.writes {
        if let Some(stats) = write_batch(w, dry_run)? {
            all_stats.push(stats);
        }
    }
    if !all_stats.is_empty() {
        let mut stderr = std::io::stderr();
        let _ = render_sync_report(&mut stderr, &all_stats, dry_run, verbose);
    }
    for hook in &plan.post_write_hooks {
        hook.run(dry_run)?;
    }
    Ok(())
}
```

#### 3. Build `render_sync_report()`

**File**: `src/emit.rs`

- [x] Implement `render_sync_report(out: &mut dyn Write, stats: &[BatchStats], dry_run: bool, verbose: bool) -> std::io::Result<()>`
  > **Deviation:** Extracted into three functions (`render_sync_report`,
  > `render_table`, `render_footer`) and a module-level `ReportColumn` struct +
  > `REPORT_COLUMNS` const to satisfy clippy's `too_many_lines` and
  > `items_after_statements` pedantic lints.

Accept a `&mut dyn std::io::Write` so unit tests can pass a `Vec<u8>` buffer
instead of writing to stderr. Returns `std::io::Result<()>` since `writeln!`
can fail. The `emit()` caller ignores the result with `let _ =` — stderr write
failures are not actionable.

The function:

1. **Partition** rows into changed vs unchanged based on
   `BatchStats::is_unchanged_only()`
2. **Determine visible rows**: if verbose, show all rows (including zero-total
   rows where all counters are 0); otherwise show only changed rows
3. **If no visible rows** (nothing changed): print
   `"\n{N} destinations unchanged"` and return
4. **Determine visible columns**: for each action type (created, updated,
   removed, backed_up), check if any visible row has a non-zero count. If
   verbose, also include the unchanged column. Omit all-zero columns.
5. **Compute column widths**: max of header length and formatted number width
   for each column, plus provider/kind column widths from the data
6. **Render header row**: left-aligned "Provider" and "Kind", right-aligned
   action columns. For dry-run, prefix action names with "Would " (e.g.,
   "Would Update")
7. **Render separator row**: repeat `─` (`\u{2500}`) under each column, matching
   width. Use character count for column widths, not byte length (the `─`
   character is multi-byte in UTF-8 but 1 display column wide)
8. **Render data rows**: left-aligned provider display name and kind, right-aligned
   numbers
9. **Render summary footer**: blank line, then a summary like
   `"4 destinations changed, 6 unchanged"` (adapt verb for dry-run:
   "would change")

Column header mapping (normal / dry-run):
| Field      | Normal Header | Dry-Run Header     |
|------------|---------------|--------------------|
| created    | Created       | Would Create       |
| updated    | Updated       | Would Update       |
| removed    | Removed       | Would Remove       |
| backed_up  | Backed Up     | Would Back Up      |
| unchanged  | Unchanged     | Unchanged          |

Summary footer logic:
- Count how many destinations have any non-zero action (excluding unchanged)
  → `n_changed`
- Count unchanged-only destinations → `n_unchanged`
- If dry-run: `"{n_changed} destinations would change, {n_unchanged} unchanged"`
- If not dry-run: `"{n_changed} destinations changed, {n_unchanged} unchanged"`

Row ordering: rows appear in `sync_plan()` iteration order (providers in config
order, kinds in `file_kinds()` order per provider). No explicit sort needed.

#### 4. Add `--verbose` flag to CLI

**File**: `src/cli.rs`

- [x] Add `verbose` flag to `SyncArgs`

```rust
/// Show all sync destinations including unchanged ones
#[arg(long, short = 'v')]
pub verbose: bool,
```

#### 5. Update call sites in `main.rs`

**File**: `src/main.rs`

- [x] Update sync handler to pass `verbose` to `emit()`:
      `emit(&plan, sync_args.dry_run, sync_args.verbose)?;`
- [x] Update compile handler to pass `verbose: false` to `emit()`:
      `emit(&plan, false, false)?;`
- [x] Fix pluralization in `run_compile()`: use `"provider"` / `"providers"`
      based on count (replace `"provider(s)"` with a conditional)

```rust
fn run_compile(
    // ... existing params unchanged ...
) -> Result<(CompileResult, Vec<Provider>)> {
    let providers = providers.to_vec();
    let result = compile::run(validated, templating, presets, &providers, adapter_configs)?;
    let n = providers.len();
    eprintln!(
        "compiled {} files for {n} {}",
        result.files.len(),
        if n == 1 { "provider" } else { "providers" }
    );
    Ok((result, providers))
}
```

- [x] Print `[dry-run]` prefix in the sync handler, before `run_compile()`:

```rust
Command::Sync(sync_args) => {
    // ... validation, targets, adapter_configs ...
    if sync_args.dry_run {
        eprint!("[dry-run] ");
    }
    let (result, _) = run_compile(...)?;
    // ...
}
```

This keeps `run_compile()` unaware of dry-run — the prefix is the sync
handler's responsibility.

#### 6. Update integration tests

**File**: `tests/pipeline.rs`

- [x] Update `test_sync_opencode_commands_prefix_subdir` (line 301): change
      the stderr assertion from `stderr.contains(".config/opencode/commands")`
      to assert on the new table format — check for both `"OpenCode"` and
      `"commands"` appearing in the output
- [x] Review all other sync integration tests for stderr assertions that
      depend on the old format and update them. Most sync tests only assert on
      `output.status.success()` or file existence and won't need changes.

#### Tests for This Phase

- [x] Add unit tests for `render_sync_report()` in `emit.rs`:
  - All unchanged (non-verbose): outputs `"\nN destinations unchanged"` with
    no table headers, no separator row, no data rows
  - Single action type (e.g., only updated): table shows only "Updated" column
  - Separator row width matches header width (catches byte-length vs
    char-count bugs with the `─` character)
  - Mixed actions: table shows multiple columns
  - Verbose mode: table includes unchanged rows and "Unchanged" column
  - Dry-run: column headers use "Would ..." prefix, summary uses conditional
    verb
  - Column width adapts to data (e.g., "OpenCode" is wider than "Claude")
- [x] Add unit test verifying `write_batch()` returns `None` for `CleanSlate`
      and `Some(BatchStats)` for `ManifestTracked`

### Success Criteria:

#### Automated Verification:

- [x] `just check` passes
  > **Deviation:** Same as Phase 1 — `just check` requires clean working tree
  > for `cargo fix`. Verified individually: fmt clean, clippy zero errors/warnings,
  > all 145 tests pass.
- [x] No new clippy warnings
- [x] Integration tests pass with updated assertions

#### Manual Verification:

- [x] Run `agentspec sync` from `agent-config/` and confirm output matches the
      mockup (table with only changed destinations, dynamic columns, summary
      footer)
- [x] Run `agentspec sync --verbose` and confirm all destinations appear
- [x] Run `agentspec sync --dry-run` and confirm `[dry-run]` banner and
      "Would ..." column headers
- [x] Run `agentspec compile` and confirm output uses proper pluralization
      ("3 providers" for multiple, "1 provider" for single) instead of
      "provider(s)"

---

## References

- Idea document: `thoughts/ideas/2026-04-13-agentspec-sync-report-readability.md`
- Current emit logic: `src/emit.rs:35-139`
- FileWrite struct: `src/plan.rs:147-153`
- sync_plan construction: `src/sync.rs:23-65`
- compile_plan construction: `src/plan.rs:76-99`
- CLI definitions: `src/cli.rs:44-68`
- Integration tests: `tests/pipeline.rs`
