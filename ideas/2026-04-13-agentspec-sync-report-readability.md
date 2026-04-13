# Improve agentspec sync report readability

## Problem Statement

The `agentspec sync` command's terminal output is difficult to scan. The current
format prints one line per sync destination with raw `key=value` stats, full
absolute paths, and zero-heavy counters that bury the signal (what actually
changed) in noise. When most destinations are unchanged — the common case — the
output is a wall of near-identical lines that takes effort to parse.

The compile summary line also prints twice during `sync` (once from `run_compile`
and once from the compile command handler), adding to the noise.

## Motivation

- **Daily workflow friction**: `agentspec sync` runs frequently during
  development. The report is the primary feedback loop for "did my change land
  where I expected?" A quick visual scan should answer that in under a second.
- **Signal-to-noise ratio**: In the common case (a few files updated, everything
  else unchanged), 80%+ of the output is zero-count noise. The user has to
  visually scan every line to find the non-zero values.
- **Provider/kind structure is hidden**: Full absolute paths like
  `/Users/jasonr/.claude/skills` obscure the logical grouping (Claude + skills).
  The user already knows where their config dirs are — they care about *which
  provider/kind* changed, not the filesystem path.

## Context

The sync output is produced in `src/emit.rs` (lines 65–136) via `eprintln!`
calls inside the `ManifestTracked` arm of `write_batch()`. The compile summary
is printed in `src/main.rs` by `run_compile()`.

Current output for a typical sync:

```
compiled 158 files for 3 provider(s)
wrote 158 files to /Users/jasonr/Workspace/jasnross/dotfiles/agent-config/generated
compiled 158 files for 3 provider(s)
  sync → /Users/jasonr/.claude/agents
    created=0 updated=0 removed=0 backed_up=0 unchanged=8
  sync → /Users/jasonr/.claude/rules
    created=0 updated=0 removed=0 backed_up=0 unchanged=6
  sync → /Users/jasonr/.claude/skills
    created=0 updated=3 removed=0 backed_up=0 unchanged=30
  sync → /Users/jasonr/.cursor/agents
    created=0 updated=0 removed=0 backed_up=0 unchanged=8
  ... (10 destinations total)
```

The sync report needs to work for 3 providers × 3–4 kinds = ~10 destinations.
Each destination tracks 5 action types: created, updated, removed, backed_up,
unchanged.

## Goals / Success Criteria

- [ ] A typical sync (few updates, most unchanged) produces a compact, instantly
      scannable report
- [ ] Unchanged-only destinations are hidden from the table, with a summary count
      at the bottom
- [ ] The table uses provider name + kind columns instead of full filesystem paths
- [ ] Columns with all-zero values across all destinations are omitted
- [ ] The compile summary line appears exactly once (not duplicated)
- [ ] Dry-run mode uses a single top-level banner instead of per-line prefixes
- [ ] The table column headers reflect dry-run context (e.g., "Would Update"
      instead of "Updated")
- [ ] `--verbose` shows the table with all destinations (including unchanged)
      and all columns
- [ ] Compile output is cleaned up: summary line appears once, no duplication
      during sync

## Non-Goals (Out of Scope)

- Changing the sync *behavior* (manifest tracking, stale cleanup, backup logic)
- Adding color/ANSI styling (could be a separate follow-up)
- Structured/machine-readable output (JSON, etc.) — this is about human-readable
  terminal output

## Proposed Output Format

### Normal sync (common case — few updates):

```
compiled 158 files for 3 providers

Provider  Kind      Updated
────────  ────────  ───────
Claude    skills          3
Cursor    skills          3
OpenCode  commands        3
OpenCode  skills          1

4 destinations updated, 6 unchanged
```

### Sync with mixed actions:

```
compiled 158 files for 3 providers

Provider  Kind      Created  Updated  Removed
────────  ────────  ───────  ───────  ───────
Claude    agents          2        0        1
Claude    skills          0        3        0
Cursor    agents          2        0        1
Cursor    skills          0        3        0

4 destinations changed, 6 unchanged
```

### Nothing changed:

```
compiled 158 files for 3 providers

10 destinations unchanged
```

### Dry-run:

```
[dry-run] compiled 158 files for 3 providers

Provider  Kind      Would Update
────────  ────────  ────────────
Claude    skills              3
Cursor    skills              3

4 destinations would be updated, 6 unchanged
```

### Verbose (`--verbose`):

```
compiled 158 files for 3 providers

Provider  Kind      Updated  Unchanged
────────  ────────  ───────  ─────────
Claude    agents          0          8
Claude    rules           0          6
Claude    skills          3         30
Cursor    agents          0          8
Cursor    rules           0          6
Cursor    skills          3         30
OpenCode  agents          0          8
OpenCode  commands        3         23
OpenCode  rules           0          6
OpenCode  skills          1         23

4 updated, 6 unchanged
```

## Constraints

- Output goes to stderr (existing convention via `eprintln!`)
- No external crate dependencies for table formatting — the table is simple
  enough to format with padding/alignment in std Rust
- `FileWrite` needs a new `kind: FileKind` field — `sync_plan()` already knows
  the kind when constructing each `FileWrite` but currently discards it
- Each `write_batch` call currently prints its own stats independently; the new
  format requires collecting all batch results first, then printing a single
  table — this is an architectural change in how `emit()` orchestrates output

## Resolved Questions

- **Kind derivation**: Add `kind: FileKind` to `FileWrite`. `sync_plan()`
  already knows the kind — one-line struct change + one-line assignment. No
  path-parsing fragility. `CleanSlate` writes (compile) can use `Option<FileKind>`
  since they don't have a meaningful kind.
- **Verbose flag**: `--verbose` shows the same table format but includes all
  destinations (including unchanged) and all columns. Consistent look, just more
  rows.
- **Compile-only output**: Included in this effort. Deduplicate the compile
  summary line that currently prints twice during sync, and tighten wording.

## Affected Systems/Services

- `src/emit.rs` — primary change site; `emit()` and `write_batch()` output logic
- `src/main.rs` — deduplicate the compile summary line for the sync path
- `src/plan.rs` / `src/sync.rs` — may need to carry "kind" metadata in
  `FileWrite` if we don't derive it from the path
- `src/cli.rs` — add `--verbose` flag to sync subcommand
