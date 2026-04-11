# Move Frontmatter Transforms Into Adapters — Implementation Plan

## Overview

Move prefix/strip logic from the emit stage into provider adapters so all
provider-specific content decisions (frontmatter fields, output paths, name
transforms) are colocated. This eliminates string-based frontmatter surgery
in emit, removes three plumbing fields from `FileWrite`, and makes `compile`
output mirror what `sync` would deploy.

## Current State Analysis

### How transforms work today

```
compile → plan → emit
  │         │       │
  │         │       ├─ prefix_rel_path: prepends "{prefix}-" to first path component
  │         │       ├─ prefix_frontmatter_name: rewrites "name: X" → "name: {prefix}:X"
  │         │       ├─ strip_frontmatter_name: removes "name:" lines from frontmatter
  │         │       └─ should_prefix_frontmatter_name: decides based on NamePrefixMode
  │         │
  │         └─ sync_plan: constructs FileWrite with file_prefix, name_prefix, strip_name
  │
  └─ adapters: produce GeneratedFile with serialized content (no prefix/strip awareness)
```

### Transform decision matrix (current)

| Provider | Kind     | File path prefix      | Frontmatter name prefix | Strip name eligible |
| -------- | -------- | --------------------- | ----------------------- | ------------------- |
| Claude   | Agents   | `"{prefix}-"`         | `"{prefix}:{name}"`     | No                  |
| Claude   | Skills   | `"{prefix}-"`         | `"{prefix}:{name}"`     | Yes (name optional) |
| Claude   | Rules    | None                  | None                    | No                  |
| Cursor   | Agents   | `"{prefix}-"`         | None (follow-up #7)     | No                  |
| Cursor   | Skills   | `"{prefix}-"`         | None (follow-up #7)     | No (name required)  |
| Cursor   | Rules    | None                  | None                    | No                  |
| OpenCode | Agents   | `"{prefix}-"`         | None (no name field)    | No                  |
| OpenCode | Commands | None (subdir instead) | None (no name field)    | No                  |
| OpenCode | Skills   | `"{prefix}-"`         | None                    | No (name required)  |
| OpenCode | Rules    | None                  | None                    | No                  |

### Key discoveries

- `name_prefix` only applies to Claude agents and skills (`sync.rs:182-195`)
- `file_prefix` applies to all providers' agents + skills, except OpenCode commands
  get a subdirectory (`sync.rs:34-39`)
- `strip_name` only applies to skills (`sync.rs:53`)
- `prefix` and `strip_name` are mutually exclusive (validated in `config.rs:275`)
- No existing test constructs `FileWrite` with non-default prefix/strip values —
  all 9 struct literal sites use `None`/`false`
- `ClaudeSkillFrontmatter.name` is `Option<String>` (`adapters/claude.rs:28`),
  with the struct literal wrapping the value in `Some(...)`

## Desired End State

After this plan is complete:

1. Adapters accept an `Option<&AdapterConfig>` and handle all prefix/strip
   transforms internally — both file paths and frontmatter fields
2. `FileWrite` has no `file_prefix`, `name_prefix`, or `strip_name` fields
3. `NamePrefixMode` is deleted
4. `emit.rs` has no content-transform functions — it purely does file I/O,
   manifest tracking, and conflict resolution
5. `compile` command passes sync config to adapters, so `generated/` output
   mirrors what `sync` would deploy
6. All existing tests pass, with appropriate updates

### Verification

```sh
cargo fmt --check
cargo clippy --all-targets
cargo test
```

All tests pass. No clippy warnings. The integration test
`test_sync_opencode_commands_prefix_subdir` still exercises the OpenCode
subdirectory behavior. The integration test
`test_sync_prefix_strip_name_conflict_errors` still validates the mutual
exclusion constraint.

## What We're NOT Doing

- Defining a formal adapter trait (useful follow-up but not required)
- Adding new frontmatter transforms beyond the existing prefix/strip
- Changing the spec file format or frontmatter schema
- Touching TODO #1 (derive id from path) — confirmed independent
- Changing `SyncTargetConfig` fields — `prefix` and `strip_name` remain in config,
  they're just consumed earlier (at compile time) instead of later (at emit time)

## Implementation Approach

The switchover must be atomic: if adapters apply transforms AND emit still applies
them, we'd get double-application. So Phase 2 adds the transform logic to adapters
and removes it from emit in one step. Phase 1 is pure plumbing that enables Phase 2.
Phase 3 wires the `compile` command.

**Note on Phase 2 atomicity:** Phase 2 has 9 sub-steps that must all land in a
single commit. The code will not compile partway through (e.g., removing
`NamePrefixMode` before removing its references in emit). Do not attempt to
compile or test between sub-steps — apply them all, then verify.

---

## Phase 1: Define `AdapterConfig` and Thread It to Adapters

### Overview

Add a new struct to the library crate that carries per-provider config
relevant to adapters. Thread it through `compile_specs` to each adapter call.
No behavior change — adapters accept the parameter but don't use it yet.

### Changes Required:

#### 1. New struct in lib crate

**File**: `src/compile.rs`

- [x] Define `AdapterConfig` struct:

```rust
/// Per-provider configuration passed to adapters at compile time.
///
/// Adapters use this to apply prefix/strip transforms to output paths and
/// frontmatter fields. When `None` is passed, adapters produce canonical
/// (unprefixed, unstripped) output.
#[derive(Clone, Debug, Default)]
pub struct AdapterConfig {
    /// Namespace prefix for file paths and frontmatter names.
    pub prefix: Option<String>,
    /// Whether to strip `name:` from skill frontmatter.
    pub strip_name: bool,
}
```

- [x] Update `compile_specs` signature to accept per-provider sync config:

```rust
pub(crate) fn compile_specs(
    specs: &[NormalizedSpec],
    presets: &ProviderPresetsMap,
    providers: &[Provider],
    adapter_configs: &HashMap<Provider, AdapterConfig>,
) -> Result<CompileResult> {
```

- [x] Pass the config to each adapter call — look up
      `adapter_configs.get(&provider)` and pass `Option<&AdapterConfig>`:

```rust
let adapter_config = adapter_configs.get(&provider);
let mut adapter_files = match provider {
    Provider::Claude => adapt_claude(spec.clone(), presets, adapter_config)?,
    Provider::Cursor => adapt_cursor(spec.clone(), presets, adapter_config)?,
    Provider::OpenCode => adapt_opencode(spec.clone(), presets, adapter_config)?,
};
```

- [x] Update the public `run()` function to accept and forward sync configs

- [x] Re-export `AdapterConfig` from `lib.rs` (add to `compile` module's public API)
  > **Deviation:** `AdapterConfig` is already public via `pub struct` in `compile.rs`,
  > which is re-exported as `pub mod compile` from `lib.rs`. No additional re-export needed.

#### 2. Update adapter signatures

**Files**: `src/adapters/claude.rs`, `src/adapters/cursor.rs`, `src/adapters/opencode.rs`

- [x] Add `cfg: Option<&AdapterConfig>` parameter to each public adapter
      function (`adapt_claude`, `adapt_cursor`, `adapt_opencode`)
- [x] Thread it to internal `adapt_agent_spec`, `adapt_skill_spec` etc. functions
- [x] No behavior change yet — the parameter is accepted but unused

#### 3. Update callers in main.rs

**File**: `src/main.rs`

- [x] Construct `HashMap<Provider, AdapterConfig>` from `SyncTargetConfig`
      for the sync command. Add a helper method on `AgentspecConfig` or construct
      inline:

```rust
// In the Sync arm, after resolve_sync_targets:
let adapter_configs: HashMap<Provider, AdapterConfig> = targets
    .iter()
    .map(|(p, t)| (*p, AdapterConfig {
        prefix: t.prefix.clone(),
        strip_name: t.strip_name,
    }))
    .collect();
```

- [x] Pass `&adapter_configs` to `run_compile` (update its signature)
- [x] For the `compile` command, pass an empty `HashMap` for now (Phase 3 changes this)

### Tests for This Phase

- [x] Existing tests continue to pass — `run_compile` callers in tests updated
      to pass empty `HashMap`
  > **Deviation:** No test callers of `run_compile` exist — only `main.rs` calls it.
  > Adapter tests call `adapt_claude`/`adapt_cursor`/`adapt_opencode` directly and
  > were updated to pass `None` as the new parameter.
- [x] No new behavioral tests needed (this phase is pure plumbing)

### Success Criteria:

#### Automated Verification:

- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes (all tests)
- [x] Adapters accept `Option<&AdapterConfig>` but produce identical output

---

## Phase 2: Move Transforms Into Adapters and Remove Old Plumbing

### Overview

This is the core change. Adapters apply prefix/strip transforms based on the
`AdapterConfig` they received in Phase 1. Simultaneously, all transform
functions and plumbing fields are removed from emit, plan, and sync. This must
be a single atomic change to avoid double-application.

### Changes Required:

#### 1. Add shared prefix helper to lib crate

**File**: `src/compile.rs` (or a new `src/adapt.rs` if preferred)

- [x] Add a helper function for the common path-prefix pattern, since all three
      adapters need the same file path prefixing logic:

```rust
impl AdapterConfig {
    /// Returns the file path prefix string (e.g., `"tw-"`), if a prefix is configured.
    /// This is shared across all providers — the file path convention is the same.
    pub fn file_prefix(&self) -> Option<String> {
        self.prefix.as_ref().map(|p| format!("{p}-"))
    }
}
```

Note: frontmatter name prefixing is **not** a shared method — the delimiter
differs by provider (Claude uses `:`, Cursor uses `-`). Each adapter applies
its own inline prefix logic.

#### 2. Update Claude adapter

**File**: `src/adapters/claude.rs`

- [x] `adapt_agent_spec`: apply prefix to file path and frontmatter name.
  Note: rename the local from `name` to `id` since the value is the spec id;
  `name` is now derived from `id` with an optional prefix:

```rust
fn adapt_agent_spec(
    spec: NormalizedAgentSpec,
    presets: &ProviderPresetsMap,
    cfg: Option<&AdapterConfig>,
) -> Result<Vec<GeneratedFile>> {
    let id = spec.frontmatter.id;  // renamed from `name` — now distinct from frontmatter name
    // ...

    // File path: agents/{prefix-}{id}.md
    let file_prefix = cfg.and_then(|s| s.file_prefix()).unwrap_or_default();
    let path = Path::new("agents").join(format!("{file_prefix}{id}.md"));

    // Frontmatter name: Claude uses ":" delimiter (e.g., "tw:my-agent")
    let name = match cfg.and_then(|c| c.prefix.as_deref()) {
        Some(prefix) => format!("{prefix}:{id}"),
        None => id,
    };

    let frontmatter = ClaudeAgentFrontmatter { name, description, model, tools };
    // ... serialize and return
}
```

- [x] `adapt_skill_spec`: apply prefix to file path, frontmatter name, and handle
      strip_name. Note: `ClaudeSkillFrontmatter.name` is already `Option<String>`:

```rust
fn adapt_skill_spec(
    spec: NormalizedSkillSpec,
    presets: &ProviderPresetsMap,
    cfg: Option<&AdapterConfig>,
) -> Result<Vec<GeneratedFile>> {
    let id = spec.frontmatter.id;
    // ...

    // File path: skills/{prefix-}{id}/SKILL.md
    let file_prefix = cfg.and_then(|s| s.file_prefix()).unwrap_or_default();
    let skill_dir = Path::new("skills").join(format!("{file_prefix}{id}"));

    // Frontmatter name: Claude uses ":" delimiter (e.g., "tw:my-skill")
    let name = match cfg {
        Some(c) if c.strip_name => None,
        Some(c) => Some(match c.prefix.as_deref() {
            Some(prefix) => format!("{prefix}:{id}"),
            None => id.clone(),
        }),
        None => Some(id.clone()),
    };

    let frontmatter = ClaudeSkillFrontmatter { name, description, model, ... };
    // ... serialize and return
}
```

- [x] `adapt_rule_spec`: no changes (rules are never prefixed)

#### 3. Update Cursor adapter

**File**: `src/adapters/cursor.rs`

- [x] `adapt_agent_spec`: apply prefix to file path only (Cursor frontmatter
      name prefixing is a follow-up — see TODO item #7):

```rust
let file_prefix = cfg.and_then(|s| s.file_prefix()).unwrap_or_default();
let path = Path::new("agents").join(format!("{file_prefix}{name}.md"));
```

- [x] `adapt_skill_spec`: apply prefix to file path. `CursorSkillFrontmatter.name`
      stays as `String` — Cursor requires the `name` field, so `strip_name` is
      ignored for this provider (the adapter simply doesn't act on it):

```rust
let file_prefix = cfg.and_then(|s| s.file_prefix()).unwrap_or_default();
let skill_dir = Path::new("skills").join(format!("{file_prefix}{id}"));

// Cursor requires `name` — strip_name is a no-op for this provider
let frontmatter = CursorSkillFrontmatter {
    name: id.clone(),
    description,
};
```

- [x] `adapt_rule_spec`: no changes

#### 4. Update OpenCode adapter

**File**: `src/adapters/opencode.rs`

- [x] `adapt_agent_spec`: apply prefix to file path (OpenCode agents have no
      `name` field — no frontmatter transform needed):

```rust
let file_prefix = cfg.and_then(|s| s.file_prefix()).unwrap_or_default();
let path = Path::new("agents").join(format!("{file_prefix}{id}.md"));
```

- [x] `adapt_skill_spec`: apply prefix to file path for SKILL.md, and handle
      the special **subdirectory** behavior for commands.
      `OpenCodeSkillFrontmatter.name` stays as `String` — OpenCode requires the
      `name` field, so `strip_name` is ignored for this provider:

```rust
// Commands: prefix is a subdirectory
if user_invocable {
    let cmd_path = match cfg.and_then(|s| s.prefix.as_deref()) {
        Some(prefix) => Path::new("commands").join(prefix).join(format!("{id}.md")),
        None => Path::new("commands").join(format!("{id}.md")),
    };
    // ...
}

// Skills: prefix is a file prefix
if agent_invocable {
    let file_prefix = cfg.and_then(|s| s.file_prefix()).unwrap_or_default();
    let skill_dir = Path::new("skills").join(format!("{file_prefix}{id}"));

    // OpenCode requires `name` — strip_name is a no-op for this provider
    let frontmatter = OpenCodeSkillFrontmatter {
        name: id.clone(),
        description,
        model,
        variant,
        tools,
    };
    // ...
}
```

- [x] `adapt_rule_spec`: no changes

#### 5. Remove transform fields from `FileWrite`

**File**: `src/plan.rs`

- [x] Remove `file_prefix`, `name_prefix`, and `strip_name` fields from
      `FileWrite` (lines 158-160)
- [x] Delete `NamePrefixMode` enum (lines 119-123) and its doc comment
- [x] Update `compile_plan` to not set the removed fields (lines 92-94)
- [x] Remove `NamePrefixMode` from `lib.rs` re-exports if present

#### 6. Simplify `sync_plan`

**File**: `src/sync.rs`

- [x] Remove the `file_prefix` / `name_prefix` / `strip_name` construction
      logic from `sync_plan` (lines 32-53 in current code):
  - Delete the `if let Some(prefix)` block that sets `file_prefix` and the
    OpenCode commands subdirectory join
  - Delete the `resolve_name_prefix` call
  - Delete the `strip_name` assignment

- [x] Remove `resolve_name_prefix` function (lines 182-195)

- [x] The OpenCode commands subdirectory join (`dest = dest.join(prefix)`)
      is now handled by the OpenCode adapter, so `sync_plan` no longer needs
      special-casing

- [x] Remove unused imports: `NamePrefixMode` from the plan import

#### 7. Remove transform functions from emit

**File**: `src/emit.rs`

- [x] Delete `prefix_rel_path` function (lines 35-57)
- [x] Delete `should_prefix_frontmatter_name` function (lines 59-68)
- [x] Delete `prefix_frontmatter_name` function (lines 71-104)
- [x] Delete `strip_frontmatter_name` function (lines 112-140)
- [x] Delete `SyncAction` enum (lines 13-22) — wait, `SyncAction` is still
      used by `write_content_to_dest`. Keep it.
  > **Deviation:** Kept `SyncAction` as planned -- it is still used by `write_content_to_dest`.
- [x] Remove unused imports: `OsString`, `NamePrefixMode`

- [x] Simplify `write_batch` for `ManifestTracked` mode:
  - Remove the `prefix_rel_path` call (line 199) — `rel` is used directly
  - Remove the `strip_frontmatter_name` pre-processing block (lines 207-218)
  - Remove the `name_prefix` extraction (line 220)
  - Simplify `write_content_to_dest` call — remove `name_prefix` argument

- [x] Simplify `write_content_to_dest`:
  - Remove the `name_prefix` parameter
  - Delete the frontmatter name prefix transform block (lines 293-303)
  - The function becomes: check manifest, compare content, write file

#### 8. Update all `FileWrite` struct literals in tests

**Files**: `src/emit.rs`, `src/plan.rs`

- [x] Remove `file_prefix: None`, `name_prefix: None`, `strip_name: false` from
      all `FileWrite` struct literal sites in `emit.rs` (test helpers and inline
      plans) and `plan.rs` (`compile_plan` + test helper). The `sync.rs` literal
      in `sync_plan` is already handled by step 6 above.

#### 9. Update/remove frontmatter transform tests

**File**: `src/emit.rs`

- [x] Delete `test_strip_frontmatter_name_removes_name_lines` (line 965)
- [x] Delete `test_strip_frontmatter_name_preserves_body_name_lines` (line 977)

### Tests for This Phase

- [x] Add adapter test for Claude agent with prefix: verify output path is
      `agents/tw-test-agent.md` and frontmatter contains `name: tw:test-agent`
- [x] Add adapter test for Claude skill with strip_name: verify frontmatter
      has no `name:` line, output path is `skills/test-skill/SKILL.md`
- [x] Add adapter test for OpenCode command with prefix: verify output path
      uses subdirectory `commands/tw/basic-skill.md`
- [x] Add adapter test for Cursor skill with strip_name: verify `name` field
      is still present (Cursor requires it — strip_name is ignored)
- [x] Verify existing `test_adapt_agent_output_format` tests still pass
      (with `sync: None` they produce unchanged output)
- [x] Verify integration test `test_sync_opencode_commands_prefix_subdir` passes
- [x] Verify integration test `test_sync_prefix_strip_name_conflict_errors` passes
  > **Deviation:** `test_sync_opencode_commands_prefix_subdir` was updated to check
  > for `.config/opencode/commands` instead of `.config/opencode/commands/tw` in
  > stderr, since the destination directory no longer includes the prefix
  > subdirectory (the adapter puts it in the file path instead).

### Success Criteria:

#### Automated Verification:

- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes (all tests, including integration tests)
- [x] `grep -r "prefix_frontmatter_name\|strip_frontmatter_name\|prefix_rel_path\|should_prefix_frontmatter_name" src/` returns no matches
- [x] `grep -r "file_prefix\|name_prefix\|strip_name" src/plan.rs` returns no matches
- [x] `grep -r "NamePrefixMode" src/` returns no matches

---

## Phase 3: Wire `compile` Command to Pass Sync Config

### Overview

Make the `compile` command construct and pass adapter config to the compile
stage, so `generated/` output mirrors what `sync` would deploy. If no config
exists for a provider, the adapter produces canonical (unprefixed) output.

### Changes Required:

#### 1. Update compile command in main.rs

**File**: `src/main.rs`

- [x] In the `Command::Compile` arm, construct `adapter_configs` **before**
      calling `run_compile`, then pass them in:

```rust
Command::Compile(_) => {
    let resolved = load_specs(&config, &dirs)?;

    // Build adapter configs before compile — mirrors what sync would use
    let adapter_configs: HashMap<Provider, AdapterConfig> = config
        .configured_sync_providers()
        .into_iter()
        .filter_map(|p| {
            config.sync.get(&p.to_string()).map(|t| {
                (p, AdapterConfig {
                    prefix: t.prefix.clone(),
                    strip_name: t.strip_name,
                })
            })
        })
        .collect();

    let (result, providers) =
        run_compile(resolved, &config.presets, &args.provider,
                    &config.providers, &adapter_configs)?;
    // ...
}
```

#### 2. Update compile integration test expectations

**File**: `tests/pipeline.rs`

- [x] If the test fixture `agent-config/agentspec.toml` has sync config with
      prefixes, the `test_compile_generates_expected_files` test may now produce
      prefixed filenames in `generated/`. Check and update assertions if needed.
- [x] If the fixture has no sync config (likely), no test changes needed
  > **Deviation:** The fixture has `strip_name = true` for Claude and path-based
  > configs for Cursor, but no prefixes. This means `compile` output now strips
  > `name:` from Claude skill frontmatter, matching what `sync` would produce.
  > The integration test only checks file existence, not content, so no test
  > changes were needed.

### Tests for This Phase

- [x] Verify `test_compile_generates_expected_files` still passes
- [x] If the fixture has sync config with prefix, verify `generated/` contains
      prefixed filenames

### Success Criteria:

#### Automated Verification:

- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes
- [x] Running `agentspec compile` with a config that has `[sync.claude] prefix = "tw"`
      produces `generated/claude/agents/tw-agent.md` with `name: tw:agent` in frontmatter

---

## Testing Strategy

### Cross-Phase Testing:

- [ ] Full integration test suite passes after each phase
- [ ] `agentspec sync --dry-run` produces identical output before and after
      (compare stderr logged paths)
- [ ] Manual comparison: run `agentspec compile` before and after on the
      `agent-config/` fixture, diff the `generated/` directories. Output should
      be identical for Phase 1+2 (compile doesn't use sync config yet), and
      correctly prefixed for Phase 3 (if sync config exists)

### Manual Testing Steps:

1. Run `agentspec sync --provider claude --dry-run` on a config with `prefix = "tw"`
   and verify the output paths show `tw-` prefixed filenames
2. Run `agentspec sync --provider open-code --dry-run` on a config with `prefix = "tw"`
   and verify commands show the `tw/` subdirectory path
3. Run `agentspec compile` and verify `generated/` matches the sync preview

## Performance Considerations

None. The transforms are trivial string operations. Moving them from emit to
compile doesn't change the computational cost.

## References

- Idea document: `thoughts/ideas/2026-04-02-agentspec-structured-frontmatter.md`
- `src/compile.rs` — `GeneratedFile`, `compile_specs`, `run`
- `src/adapters/claude.rs` — Claude adapter (name in agents + skills)
- `src/adapters/cursor.rs` — Cursor adapter (name in agents + skills)
- `src/adapters/opencode.rs` — OpenCode adapter (name in skills, subdirectory for commands)
- `src/plan.rs:152-161` — `FileWrite` with transform fields to remove
- `src/plan.rs:119-123` — `NamePrefixMode` to delete
- `src/sync.rs:32-53` — `sync_plan` prefix/strip construction to remove
- `src/sync.rs:182-195` — `resolve_name_prefix` to delete
- `src/emit.rs:35-140` — transform functions to delete
- `src/config.rs:244-270` — `SyncTargetConfig` (stays, fields consumed earlier)
- `src/main.rs:53-81` — compile vs sync command orchestration
- `TODO.md` item #7 — originating TODO
