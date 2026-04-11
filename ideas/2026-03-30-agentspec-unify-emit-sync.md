# Unify emit and sync into a placement/strategy layer

## Problem Statement

agentspec's post-compilation pipeline has two independent subsystems — emit and
sync — that both solve the same problem: placing generated files at a
destination. They share no code and use different abstractions, even though the
core operation is identical: "given a set of files, write them somewhere with a
specific strategy."

This structural duplication makes the codebase harder to reason about. Today,
emit writes to `generated/` using direct `fs::write`, while sync reads back from
`generated/` and distributes to tool config directories using either symlinks or
copies. The `generated/` directory sits between them as a mandatory intermediate
step, coupling both sides to the same filesystem layout.

Additionally, emit and sync are binary-only modules (`src/emit.rs`,
`src/sync.rs` in the binary crate). The library (`lib.rs`) exposes everything up
through compilation but nothing about where files go. Consumers who want to use
agentspec as a library — or even just reason about destinations — can't access
that knowledge without the CLI.

## Motivation

- **Architectural clarity**: emit and sync are the same operation with different
  parameters (target directory, write strategy). Unifying them reduces conceptual
  surface area.
- **Library-first destinations**: Knowing *where* files should go for each
  provider (e.g., `~/.claude/skills/` for Claude user-mode sync) is valuable
  domain knowledge that belongs in the library, not buried in CLI config
  resolution.
- **Separation of data from IO**: The set of files to place and their
  destinations should be expressible as data. The actual filesystem IO (write,
  symlink, copy) can be a separate concern — potentially pluggable or testable
  without touching disk.
- **Reduced coupling**: emit currently depends on knowing the output directory
  name (`generated/`) to reconstruct paths. Sync depends on the filesystem state
  left by emit. Unifying them lets the pipeline express placement as a single
  plan.

## Context

### Current architecture

```
compile → CompileResult (Vec<GeneratedFile>)
    ↓
emit (binary-only) → writes to generated/<provider>/<kind>/...
    ↓
sync (binary-only) → reads generated/ → writes to tool config dirs
                      (symlink or copy strategy per provider)
```

### Recent improvements

- `GeneratedFile.path` was just decoupled from the output directory structure —
  paths are now relative to the provider root (e.g., `agents/foo.md`), not
  `generated/claude/agents/foo.md`. This makes `GeneratedFile` destination-agnostic.
- The `check` command already computes what *should* be on disk vs. what *is* on
  disk — a precursor to a placement plan concept.

### Key files today

| Module | Location | Role |
|--------|----------|------|
| emit | `src/emit.rs` (binary) | `write_generated_files`, `check_generated_state` |
| sync | `src/sync.rs` + `src/sync/` (binary) | Provider config resolution, symlink/copy strategies, manifest tracking |
| compile | `src/compile.rs` (library) | `GeneratedFile`, `CompileResult` |

## Goals / Success Criteria

- [ ] A single abstraction for "place these files at this destination with this
      strategy" that subsumes both emit and sync
- [ ] Destination resolution (where files go per provider, per mode) is available
      in the library crate, not just the binary
- [ ] The concept of a placement plan (set of files + destinations) is
      expressible as data, separate from the IO of executing it
- [ ] `generated/` remains a supported destination (it's actively inspected for
      review/debugging) but is not architecturally privileged over tool config dirs
- [ ] The `check` command works against the unified model
- [ ] Existing CLI commands (`compile`, `check`, `sync`) continue to work with
      equivalent behavior

## Non-Goals (Out of Scope)

- Removing the `generated/` directory or making it optional (it's used for
  inspection and review)
- Changing the adapter/compilation layer — this is purely about post-compilation
  placement
- Supporting non-filesystem destinations (e.g., API uploads)
- Redesigning the config file format for sync targets

## Constraints

- Must maintain backward compatibility with existing `agentspec.toml` sync
  configuration
- Symlink vs. copy distinction is real and must be preserved — some tools
  (Claude) work with symlinks, others may not
- Sync manifest tracking (knowing what was previously synced for cleanup) must
  continue to work
- The refactor should be incremental — not a big-bang rewrite

## Proposed Design Direction

### Pipeline shape

Destinations are resolved separately from compilation. The target pipeline is:

```
compile(ResolvedSpecs) → CompileResult
plan(CompileResult, config) → PlacementPlan   ← new step
execute(PlacementPlan) → ()
```

`compile::run()` stays destination-agnostic. A new `plan()` step maps
`CompileResult + config` into a `PlacementPlan` — a data structure describing
where each file goes and with what strategy. Execution is a separate IO leaf.

### lib.rs boundary

Pure domain logic worth moving into the library (no IO):
- `SyncMode`, `SyncStrategy` — policy types
- `SyncTargetConfig` — the configuration type for a sync target
- `resolve_dest_dir` / `all_sync_kinds` — destination resolution
- `CheckResult` + `check_generated_state` — state comparison

Strategy execution (symlink/copy), manifest tracking, and the OpenCode JSON
patching can remain closer to the binary or be gated behind a feature flag.

### `--no-compile`

Remove. After unification, `agentspec sync` compiles and places in one step.
The CI/CD separation use case can simply re-compile — it's fast enough.

## Open Questions

- **OpenCode JSON patching**: The OpenCode provider patches `opencode.json` with
  absolute paths to the synced rules destination — this is inherently a
  sync-time operation since destination paths are unknown at compile time. The
  design question is whether to model this as a `PostPlacementOp` enum variant
  (first-class alongside file placement) or keep it as a provider-specific hook
  after file distribution. To be resolved during planning.
- **lib.rs boundary (fine-grained)**: The coarse boundary is clear (destination
  resolution → library, IO → binary/feature). The open detail is manifest
  tracking — it's stateful and tied to cleanup semantics, which may or may not
  belong in the library depending on whether library consumers need ownership
  tracking.

## References

- `src/emit.rs` — current emit implementation (binary crate)
- `src/sync.rs`, `src/sync/` — current sync implementation (binary crate)
- `src/compile.rs` — `GeneratedFile` and `CompileResult` (library crate)
- TODO.md item #3 — original notes
