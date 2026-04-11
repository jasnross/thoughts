# Move Frontmatter Transforms Into Adapters

## Problem Statement

The agentspec emit stage performs post-hoc string surgery on already-serialized
file content to modify frontmatter fields (`name:` prefixing and stripping).
This is fragile, hard to extend, and placed in the wrong layer — the adapters
already know provider-specific frontmatter structure and construct the `name`
field, yet emit rediscovers that structure via line-by-line string parsing to
modify it.

Today, `prefix_frontmatter_name` and `strip_frontmatter_name` in `emit.rs`
manually walk lines looking for `---` delimiters and `name:` prefixes. This
works for the narrow case of single-line `name:` values but would break on
multi-line YAML values, comments, or non-standard formatting.

## Motivation

- **Colocation**: Adapters already decide which frontmatter fields to set and how
  to serialize them. Adding prefix/strip logic there keeps all provider-specific
  knowledge in one place. A new provider is just one adapter implementation.
- **Elimination of plumbing**: `name_prefix`, `strip_name`, and `file_prefix` are
  threaded through `FileWrite` in the plan stage just to pass them to emit.
  Moving transforms to compile time removes these fields entirely.
- **Correctness**: String-based YAML manipulation is inherently fragile. Adapters
  work on typed structs — prefixing `name` is just conditional logic around a
  field they already set.
- **Simpler emit**: Emit becomes purely about file I/O, manifest tracking, and
  conflict resolution. No content transformation at all.

## Context

### Current data flow

```
compile → plan → emit
  │         │       │
  │         │       └─ applies string transforms (prefix/strip name)
  │         └─ attaches name_prefix, strip_name, file_prefix to FileWrite
  └─ adapters produce GeneratedFile with serialized content
```

The prefix comes from sync config (`agentspec.toml` `[sync.<provider>]`), which
the compile stage currently doesn't see. This is the historical reason transforms
lived in emit — but there's no fundamental barrier to passing sync config to the
compile stage.

### Key facts

- The `name` field is always derived from the spec's `id` — adapters don't
  produce arbitrary name values.
- `strip_name` and `prefix` are mutually exclusive (enforced by config validation).
- OpenCode agents have no `name` field at all. Adapters already know this.
- The `format!("---\n{frontmatter_str}---\n\n{body}")` pattern appears 7 times
  across adapters — already adapter-owned serialization logic.
- File path prefixing (`tw-agent.md`) is also currently applied in emit but is
  conceptually an adapter concern (the adapter constructs the output path).

### What changes

The `compile` command reads sync config and produces `generated/` output that
mirrors what `sync` would deploy. `generated/` becomes "preview of deployment"
rather than "canonical unprefixed output." This eliminates the need for two
separate code paths.

### Affected code

- `src/adapters/claude.rs`, `cursor.rs`, `opencode.rs`: accept optional
  prefix/strip_name config, apply during frontmatter construction and path
  construction
- `src/compile.rs`: `run()` / `compile_specs()` accept sync config per provider
- `src/plan.rs`: remove `name_prefix`, `strip_name`, `file_prefix` from `FileWrite`;
  remove `NamePrefixMode`
- `src/sync.rs`: simplify `sync_plan()` — no longer constructs prefix/strip
  instructions
- `src/emit.rs`: delete `prefix_frontmatter_name`, `strip_frontmatter_name`,
  `should_prefix_frontmatter_name`, `prefix_rel_path`; simplify
  `write_content_to_dest` (no content transforms, just write)
- `src/main.rs`: pass sync config (or relevant subset) to the compile stage for
  both `compile` and `sync` commands

## Goals / Success Criteria

- [ ] No string-level frontmatter manipulation anywhere in the codebase
- [ ] Adapters own all provider-specific content decisions (frontmatter, paths, transforms)
- [ ] `FileWrite` has no content-transform fields (`name_prefix`, `strip_name`, `file_prefix`)
- [ ] `compile` output mirrors what `sync` would deploy
- [ ] Emit is purely file I/O + manifest tracking + conflict resolution
- [ ] Adding a new provider requires implementing one adapter — no emit/plan changes
- [ ] All existing behavior preserved (prefixing, stripping, edge cases)

## Non-Goals (Out of Scope)

- Defining a formal adapter trait — useful follow-up but not required for this change
- Adding new frontmatter transforms — this enables them by putting the logic in the right place
- Changing the spec file format or frontmatter schema
- TODO item #1 (derive id from path) — related but independent

## Proposed Solution

### Adapter signature change

Adapters currently:
```rust
pub fn adapt_claude(spec: NormalizedSpec, presets: &ProviderPresetsMap) -> Result<Vec<GeneratedFile>>
```

Proposed:
```rust
pub fn adapt_claude(spec: NormalizedSpec, presets: &ProviderPresetsMap, sync: Option<&AdapterSyncConfig>) -> Result<Vec<GeneratedFile>>
```

Where `AdapterSyncConfig` carries the subset of sync config relevant to adapters:
```rust
pub struct AdapterSyncConfig {
    pub prefix: Option<String>,
    pub strip_name: bool,
}
```

When `sync` is `None` (no sync config), adapters produce canonical output (same as
today). When `Some`, they apply prefix to `name` field and file paths, or strip `name`
if configured.

### Adapter logic (Claude example)

```rust
// Before (current)
let frontmatter = ClaudeAgentFrontmatter {
    name,  // always the spec id
    ...
};
let path = Path::new("agents").join(format!("{name}.md"));

// After
let display_name = match sync {
    Some(cfg) if cfg.strip_name => None,  // or skip the field
    Some(cfg) => Some(apply_prefix(&name, cfg.prefix.as_deref())),
    None => Some(name.clone()),
};
let path_name = match sync {
    Some(cfg) => apply_path_prefix(&name, cfg.prefix.as_deref()),
    None => name.clone(),
};
let path = Path::new("agents").join(format!("{path_name}.md"));
```

### compile and sync commands

Both commands load sync config. `compile` and `sync` pass it to the compile stage:

```rust
// main.rs — compile command
let sync_config = config.adapter_sync_config(&providers);
let result = compile::run(resolved, &presets, &providers, &sync_config)?;
let plan = compile_plan(&result, &output_dir, &providers);  // no prefix/strip fields
emit(&plan, false)?;

// main.rs — sync command (same compile call, different plan)
let result = compile::run(resolved, &presets, &providers, &sync_config)?;
let plan = sync_plan(&result, &targets, &home, &cwd)?;  // simpler — no prefix/strip
emit(&plan, dry_run)?;
```

### FileWrite simplification

```rust
// Before
pub struct FileWrite {
    pub provider: Provider,
    pub destination: PathBuf,
    pub files: Vec<GeneratedFile>,
    pub mode: WriteMode,
    pub allow_overwrite: bool,
    pub file_prefix: Option<String>,      // removed
    pub name_prefix: Option<(String, NamePrefixMode)>,  // removed
    pub strip_name: bool,                 // removed
}

// After
pub struct FileWrite {
    pub provider: Provider,
    pub destination: PathBuf,
    pub files: Vec<GeneratedFile>,
    pub mode: WriteMode,
    pub allow_overwrite: bool,
}
```

## Constraints

- The compile stage receives an `AdapterSyncConfig` struct — a plain data bag with
  `prefix` and `strip_name`. It does NOT read `agentspec.toml` or depend on the
  config layer. The caller (`main.rs`) constructs this from `SyncTargetConfig`,
  following the existing pattern where compile receives `&ProviderPresetsMap` and
  `&[Provider]` rather than raw config.
- Adapters become slightly more complex (conditional prefix/strip logic), but this
  is provider-specific logic that belongs with them.
- The OpenCode commands prefix behavior (subdirectory instead of file prefix) must
  move into the OpenCode adapter.

## Resolved Questions

1. **How does `compile` handle multiple sync targets for the same provider?** Not
   applicable — the config model allows exactly one sync target per provider
   (`HashMap<provider, SyncTargetConfig>`), and there's no foreseeable need for
   multiple targets per provider. One provider = one `AdapterSyncConfig`.

2. **Should `AdapterSyncConfig` be per-provider or shared?** Per-provider. Different
   providers can have different prefix/strip_name settings. The natural representation
   is `HashMap<Provider, AdapterSyncConfig>`, mirroring the existing
   `ProviderPresetsMap` pattern.

3. **Does this interact with TODO item #1 (derive id from path)?** No. If id is
   derived from path, that change happens in the load stage (`Specs::load`). The
   spec structs still carry `id: String` — adapters and everything downstream see
   the same interface regardless of where id was populated from.

## References

- `src/adapters/` — Claude, Cursor, OpenCode adapter implementations
- `src/emit.rs` — current string-based transforms (to be deleted)
- `src/plan.rs` — `FileWrite`, `NamePrefixMode` definitions (to be simplified)
- `src/sync.rs` — `sync_plan`, `resolve_name_prefix` (to be simplified)
- `src/main.rs` — command orchestration
- `TODO.md` item #7 — the originating TODO
