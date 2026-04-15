# Agentspec: `content_prefix` Implementation Plan

## Overview

Add a `content_prefix` config field to agentspec that controls the prefix used in model-facing content references (via `model_facing_name()`), independent of the `prefix` field that controls file paths. This enables Claude Code plugin deployments where files are unprefixed but body text references use the colon-namespaced form (e.g., `tw:resolve-thoughts-dir`).

## Current State Analysis

The prefix system has a single `prefix: Option<String>` that flows through:

```
agentspec.toml [sync.claude] prefix = "tw"
  → SyncTargetConfig.prefix          (config.rs:254)
  → resolve_sync_target() merging    (config.rs:78-105)
  → adapter_configs() construction   (config.rs:135-149)
  → AdapterConfig.prefix             (compile.rs:19-22)
  → file_prefix() → "{prefix}-"     (compile.rs:27-29)
  → model_facing_name() per adapter  (claude.rs:252, cursor.rs:162, opencode.rs:371)
  → adapt_*_spec() inline name logic (claude.rs:120, cursor.rs:69/94)
```

Both file paths and content references read from the same `prefix` field. There is no way to prefix content references differently from file paths.

### Key Discoveries

- `model_facing_name()` is the sole extension point for content references — it feeds into `TemplateContext::from_specs_for_provider()` at `context.rs:135-149`, which populates `{{ specs.*.name }}` template variables. No other callers exist.
- The inline prefix logic in `adapt_agent_spec` (claude.rs:120-123, cursor.rs:69-72) and `adapt_skill_spec` (cursor.rs:94-97) sets the **frontmatter `name`** field. This is provider-assigned identity and should continue using `prefix`, not `content_prefix`. In a plugin context, the frontmatter name is bare (no prefix) because Claude Code adds the plugin namespace.
- OpenCode's `model_facing_name` (opencode.rs:371-380) already has per-spec-type branching — only agents get prefixed. This logic stays the same; it just reads `content_prefix()` instead of `prefix` for the agent branch.
- Empty-string normalization already exists for `prefix` at config.rs:100-102. The same pattern applies to `content_prefix`.

## Desired End State

### Behavior Matrix

All four combinations of `prefix` and `content-prefix`:

| `prefix` | `content-prefix` | File path | Frontmatter `name` | Content reference (`model_facing_name`) |
|----------|-------------------|-----------|---------------------|-----------------------------------------|
| `"tw"` | (not set) | `tw-{id}` | `tw-{id}` | `tw-{id}` (defaults to `"{prefix}-"`) |
| `"tw"` | `"tw:"` | `tw-{id}` | `tw-{id}` | `tw:{id}` |
| (not set) | `"tw:"` | `{id}` | `{id}` | `tw:{id}` |
| (not set) | (not set) | `{id}` | `{id}` | `{id}` |

Key observations:
- **Row 1** (prefix only): Current behavior, unchanged. `content_prefix()` falls back to `"{prefix}-"`.
- **Row 2** (both set): File paths and frontmatter use `prefix` with hyphen. Content references use the explicit `content-prefix` with colon. Useful if you want prefixed file names but a different content format.
- **Row 3** (content-prefix only): The plugin case. Files and frontmatter are bare. Content references use the colon form. This is the primary use case motivating this change.
- **Row 4** (neither set): No prefixing at all. Also unchanged.

The rule: `prefix` controls **file paths** and **frontmatter names** (via `file_prefix()` and inline adapter logic). `content-prefix` controls **content references** (via `content_prefix()` → `model_facing_name()`). When `content-prefix` is not set, it inherits from `prefix` with a hyphen separator.

### Config Examples

```toml
# Standard prefix (unchanged — backwards compatible)
[sync.claude]
prefix = "tw"
# → file: tw-{id}, frontmatter: tw-{id}, content: tw-{id}

# Claude Code plugin: no file prefix, colon-separated content prefix
[sync.claude]
content-prefix = "tw:"
# → file: {id}, frontmatter: {id}, content: tw:{id}

# Both: prefixed files with different content format
[sync.claude]
prefix = "tw"
content-prefix = "tw:"
# → file: tw-{id}, frontmatter: tw-{id}, content: tw:{id}
```

### Verification

```sh
just check    # format + lint + build + test + license check
```

## What We're NOT Doing

- Changing the inline prefix logic in `adapt_agent_spec` / `adapt_skill_spec` — frontmatter `name` stays controlled by `prefix`
- Refactoring OpenCode's per-spec-type branching
- Adding validation that `content_prefix` has a valid format (it's a literal string — any value is valid)
- Adding separate `content_prefix` handling for the `compile` command — it shares the same `adapter_configs()` plumbing updated in Phase 1, so no additional code path is needed

## Implementation Approach

Two phases: code changes first (the `content_prefix` plumbing + tests), then documentation updates. The code change is atomic — `content_prefix` flows through the same pipeline as `prefix` but is only consumed by `model_facing_name()` and the `content_prefix()` accessor, not by `file_prefix()` or the inline frontmatter logic.

---

## Phase 1: Code Changes

### Overview

Add `content_prefix: Option<String>` at each layer of the pipeline, with a `content_prefix()` method on `AdapterConfig` that falls back to `"{prefix}-"` when not explicitly set.

### Changes Required

#### 1. Config: `SyncTargetConfig` and `SyncFlags`

**File**: `src/config.rs`

- [x] Add `content_prefix: Option<String>` to `SyncTargetConfig` (after line 254)

Add `rename_all = "kebab-case"` to the struct-level serde attributes (per CLAUDE.md convention: prefer struct-level attributes over per-field). All existing fields are single words, so kebab-case normalization changes nothing for them. This handles `content_prefix` → `content-prefix` automatically:

```rust
#[derive(Clone, Debug, Default, Deserialize)]
#[serde(default, deny_unknown_fields, rename_all = "kebab-case")]
pub struct SyncTargetConfig {
    // ... existing fields unchanged ...
    /// Optional content-reference prefix. When set, `model_facing_name()` uses
    /// this literal string (including separator) instead of deriving from `prefix`.
    /// For example, `"tw:"` produces content references like `tw:skill-name`.
    pub content_prefix: Option<String>,
}
```

- [x] In `resolve_sync_target()`, add CLI override and empty-string normalization for `content_prefix` (after line 98, before the existing empty-prefix normalization):

```rust
if let Some(content_prefix) = cli.content_prefix.as_deref() {
    resolved.content_prefix = Some(content_prefix.to_string());
}

// Normalize empty strings to None (existing prefix + new content_prefix)
if resolved.prefix.as_deref() == Some("") {
    resolved.prefix = None;
}
if resolved.content_prefix.as_deref() == Some("") {
    resolved.content_prefix = None;
}
```

Note: the existing empty-prefix normalization at lines 100-102 is replaced by the block above that handles both fields.

- [x] Update `adapter_configs()` (lines 135-149) to pass `content_prefix` through:

```rust
AdapterConfig {
    prefix: t.prefix.clone(),
    content_prefix: t.content_prefix.clone(),
}
```

**File**: `src/config.rs` (SyncFlags)

- [x] Add `content_prefix: Option<String>` to `SyncFlags` (after line 287):

```rust
/// Override `content_prefix` setting.
pub content_prefix: Option<String>,
```

#### 2. CLI: `--content-prefix` flag

**File**: `src/cli.rs`

- [x] Add `--content-prefix` argument to `SyncArgs` (after line 71):

```rust
/// Override the content-reference prefix (e.g., "tw:" for plugin namespaces)
#[arg(long)]
pub content_prefix: Option<String>,
```

**File**: `src/sync.rs`

- [x] Add `content_prefix` to the `SyncFlags` construction at `sync.rs:76-81` in `resolve_sync_targets()`:

```rust
let sync_flags = SyncFlags {
    force: args.force,
    dest: args.dest.clone(),
    mode: args.mode,
    prefix: args.prefix.clone(),
    content_prefix: args.content_prefix.clone(),
};
```

#### 3. `AdapterConfig`: add field and accessor

**File**: `src/compile.rs`

- [x] Add `content_prefix: Option<String>` field to `AdapterConfig` (after line 21):

```rust
pub struct AdapterConfig {
    /// Namespace prefix for file paths and frontmatter names.
    pub prefix: Option<String>,
    /// Literal prefix for content/model-facing names (e.g., `"tw:"` → `"tw:{id}"`).
    /// When `None`, `content_prefix()` falls back to `"{prefix}-"`.
    pub content_prefix: Option<String>,
}
```

- [x] Add `content_prefix()` method (after `file_prefix()` at line 29):

```rust
/// Returns the content-reference prefix string, if any prefix is configured.
///
/// When an explicit `content_prefix` is set (e.g., `"tw:"`), returns it directly.
/// Otherwise falls back to `"{prefix}-"` (matching the file prefix format).
/// Returns `None` when neither field is set.
pub fn content_prefix(&self) -> Option<String> {
    self.content_prefix
        .clone()
        .or_else(|| self.prefix.as_ref().map(|p| format!("{p}-")))
}
```

Returns:
- `Some("tw:")` when `content_prefix = Some("tw:")`
- `Some("tw-")` when `content_prefix = None, prefix = Some("tw")` (fallback preserves current behavior)
- `None` when both are `None`

- [x] Update `file_prefix()` doc comment to clarify it only controls file paths (lines 25-26):

```rust
/// Returns the file path prefix string (e.g., `"tw-"`), if a prefix is configured.
/// Only used for filesystem paths — content references use `content_prefix()`.
```

#### 4. Update existing `AdapterConfig` struct literals in tests

- [x] Add `content_prefix: None` (or use `..AdapterConfig::default()`) to all existing `AdapterConfig` struct literals in test code. `AdapterConfig` derives `Default`, but struct literal syntax requires all fields. There are 9 test sites: `claude.rs` (2), `cursor.rs` (3), `opencode.rs` (1), `context.rs` (2), `fragments.rs` (1). Search for `AdapterConfig { prefix:`. (The production site in `config.rs:143` is already covered in section 1 step 3.)

#### 5. Claude adapter: update `model_facing_name`

**File**: `src/adapters/claude.rs`

- [x] Update `model_facing_name` (lines 252-258) to use `content_prefix()`:

```rust
pub fn model_facing_name(spec: &NormalizedSpec, cfg: Option<&AdapterConfig>) -> String {
    let id = spec.id();
    match cfg.and_then(|c| c.content_prefix()) {
        Some(prefix) => format!("{prefix}{id}"),
        None => id.to_owned(),
    }
}
```

Note the change from `format!("{prefix}-{id}")` to `format!("{prefix}{id}")` — the separator is now embedded in the prefix value itself.

#### 6. Cursor adapter: update `model_facing_name`

**File**: `src/adapters/cursor.rs`

- [x] Update `model_facing_name` (lines 162-168) — same change as Claude:

```rust
pub fn model_facing_name(spec: &NormalizedSpec, cfg: Option<&AdapterConfig>) -> String {
    let id = spec.id();
    match cfg.and_then(|c| c.content_prefix()) {
        Some(prefix) => format!("{prefix}{id}"),
        None => id.to_owned(),
    }
}
```

#### 7. OpenCode adapter: update `model_facing_name`

**File**: `src/adapters/opencode.rs`

- [x] Update `model_facing_name` (lines 371-380) — same change but only in the agent branch:

```rust
pub fn model_facing_name(spec: &NormalizedSpec, cfg: Option<&AdapterConfig>) -> String {
    let id = spec.id();
    match spec {
        NormalizedSpec::Agent(_) => match cfg.and_then(|c| c.content_prefix()) {
            Some(prefix) => format!("{prefix}{id}"),
            None => id.to_owned(),
        },
        NormalizedSpec::Skill(_) | NormalizedSpec::Rule(_) => id.to_owned(),
    }
}
```

#### Tests

**File**: `src/config.rs` (config tests)

- [x] Add test: `content_prefix` parsed from TOML

```rust
#[test]
fn test_resolve_sync_target_content_prefix_from_config() {
    // [sync.claude] content-prefix = "tw:"
    // → resolved.content_prefix == Some("tw:")
}
```

- [x] Add test: `content_prefix` CLI override

```rust
#[test]
fn test_resolve_sync_target_cli_content_prefix_overrides_config() {
    // Config: content-prefix = "original:", CLI: --content-prefix "cli:"
    // → resolved.content_prefix == Some("cli:")
}
```

- [x] Add test: empty `content_prefix` normalized to `None`

```rust
#[test]
fn test_resolve_sync_target_empty_content_prefix_normalized_to_none() {
    // content_prefix = "" → None
}
```

- [x] Add test: `content_prefix` defaults to `None` when not set (existing tests already cover this implicitly but verify explicitly)

**File**: `src/compile.rs` (or add inline tests)

- [x] Add test: `AdapterConfig::content_prefix()` returns explicit value when set

```rust
let cfg = AdapterConfig { prefix: Some("tw".to_owned()), content_prefix: Some("tw:".to_owned()) };
assert_eq!(cfg.content_prefix(), Some("tw:".to_owned()));
```

- [x] Add test: `AdapterConfig::content_prefix()` falls back to `"{prefix}-"` when not set

```rust
let cfg = AdapterConfig { prefix: Some("tw".to_owned()), content_prefix: None };
assert_eq!(cfg.content_prefix(), Some("tw-".to_owned()));
```

- [x] Add test: `AdapterConfig::content_prefix()` returns `None` when both are `None`

```rust
let cfg = AdapterConfig { prefix: None, content_prefix: None };
assert_eq!(cfg.content_prefix(), None);
```

- [x] Add test: `AdapterConfig::file_prefix()` is unaffected by `content_prefix`

```rust
let cfg = AdapterConfig { prefix: Some("tw".to_owned()), content_prefix: Some("tw:".to_owned()) };
assert_eq!(cfg.file_prefix(), Some("tw-".to_owned()));
```

**File**: `src/adapters/claude.rs` (adapter tests)

- [x] Add test: agent with `content_prefix` — file path uses `prefix`, frontmatter name uses `prefix`, but `model_facing_name` uses `content_prefix`

```rust
#[test]
fn test_adapt_agent_content_prefix_does_not_affect_frontmatter() {
    // AdapterConfig { prefix: None, content_prefix: Some("tw:") }
    // → file path: agents/test-agent.md (no file prefix)
    // → frontmatter name: test-agent (no prefix)
}
```

- [x] Add test: `model_facing_name` with explicit `content_prefix`

```rust
#[test]
fn test_model_facing_name_uses_content_prefix() {
    // AdapterConfig { prefix: None, content_prefix: Some("tw:") }
    // → model_facing_name returns "tw:test-agent"
}
```

- [x] Add test: `model_facing_name` with prefix only (backwards compat)

```rust
#[test]
fn test_model_facing_name_falls_back_to_prefix() {
    // AdapterConfig { prefix: Some("tw"), content_prefix: None }
    // → model_facing_name returns "tw-test-agent"
}
```

**File**: `src/templating/context.rs` (template context tests)

- [x] Add test: `from_specs_for_provider` with `content_prefix` — keyed access returns colon-prefixed name

```rust
#[test]
fn test_from_specs_for_provider_claude_with_content_prefix() {
    // AdapterConfig { prefix: None, content_prefix: Some("tw:") }
    // → ctx.specs.agent.get("my_agent").name == "tw:my-agent"
    // → ctx.specs.skill.get("gh_safe").name == "tw:gh-safe"
}
```

- [x] Add test: OpenCode with `content_prefix` — agents use it, skills don't

```rust
#[test]
fn test_from_specs_for_provider_opencode_with_content_prefix() {
    // AdapterConfig { prefix: None, content_prefix: Some("tw:") }
    // → agent name: "tw:my-agent"
    // → skill name: "gh-safe" (unchanged — OpenCode ignores prefix for skills)
}
```

**File**: `tests/pipeline.rs` (integration tests)

- [x] Add test: sync with `content_prefix` only — files unprefixed, content references colon-prefixed

```rust
#[test]
fn test_sync_content_prefix_without_file_prefix() {
    // Config: [sync.claude] content-prefix = "tw:"
    // → file: .claude/skills/basic-skill/SKILL.md (no prefix in path)
    // → body contains "Agent: tw:test-agent" (colon-prefixed reference)
}
```

- [x] Add test: sync with both `prefix` and `content_prefix` — files use prefix, content uses content_prefix

```rust
#[test]
fn test_sync_prefix_and_content_prefix() {
    // Config: [sync.claude] prefix = "tw", content-prefix = "tw:"
    // → file: .claude/skills/tw-basic-skill/SKILL.md
    // → body contains "Agent: tw:test-agent"
}
```

- [x] Add test: CLI `--content-prefix` override

```rust
#[test]
fn test_sync_cli_content_prefix_overrides_config() {
    // Config: content-prefix = "original:", CLI: --content-prefix "cli:"
    // → body contains "Agent: cli:test-agent"
}
```

### Success Criteria

#### Automated Verification

- [x] All existing tests pass unchanged: `cargo test`
- [x] New unit tests pass for `AdapterConfig::content_prefix()` accessor
- [x] New unit tests pass for config parsing/resolution of `content_prefix`
- [x] New unit tests pass for `model_facing_name` with `content_prefix`
- [x] New template context tests pass for `content_prefix`
- [x] New integration tests pass for end-to-end `content_prefix` behavior
- [x] Clippy passes: `cargo clippy --all-targets`
- [x] Formatting passes: `cargo fmt --check`
- [x] Full suite: `just check`

---

## Phase 2: Documentation

### Overview

Update user-facing and developer-facing documentation to describe the new `content-prefix` field, how it interacts with `prefix`, and when to use it.

### Changes Required

#### 1. README.md — Config field reference table

**File**: `README.md`

- [x] Add `content-prefix` row to the sync config table (after the `prefix` row at line 395):

```markdown
| `content-prefix` | `null` | Literal prefix for content references (model-facing names). Includes its separator (e.g., `"tw:"` → `tw:skill-name`). When unset, defaults to `"{prefix}-"`. See [Content-reference prefix](#content-reference-prefix). |
```

#### 2. README.md — Prefix behavior section

**File**: `README.md`

- [x] Update the "Prefix behavior" section (lines 399-407) to explain `content-prefix` and how it differs from `prefix`. Add a paragraph after the existing provider table:

```markdown
#### Content-reference prefix

By default, content references (via `{{ specs.skill.foo.name }}`) use the same
prefix format as file paths: `{prefix}-{id}`. To use a different format —
for example, Claude Code plugins require the colon-namespaced form
`tw:skill-name` — set `content-prefix` explicitly:

| Config                             | File path         | Content reference |
| ---------------------------------- | ----------------- | ----------------- |
| `prefix = "tw"`                    | `tw-commit/`      | `tw-commit`       |
| `content-prefix = "tw:"`           | `commit/`          | `tw:commit`       |
| `prefix = "tw"`, `content-prefix = "tw:"` | `tw-commit/` | `tw:commit`       |

`content-prefix` is a literal string prepended directly to the spec ID — the
separator (`:`, `-`, etc.) is part of the value itself.
```

#### 3. README.md — Config example snippet

**File**: `README.md`

- [x] Add commented `content-prefix` to the config example at line 339:

```toml
# prefix = "tw"             # namespace prefix for synced file names
# content-prefix = "tw:"    # content-reference prefix (defaults to "{prefix}-")
```

- [x] Add commented `content-prefix` to the sync config example at line 387:

```toml
# prefix = "tw"
# content-prefix = "tw:"
```

#### 4. README.md — `specs` template variable description

**File**: `README.md`

- [x] Update the `name` field description in the specs variable table (line 263) to mention `content-prefix`:

```markdown
| `name` | The spec's name as the model sees it (uses `content-prefix` if set, else `prefix`) |
```

- [x] Update the prose below the table (lines 268-270) to mention `content-prefix`:

```markdown
When compiled with a sync prefix, the `name` field resolves to the
prefix-aware model-facing name. By default this is `{prefix}-{id}`
(e.g., `tw-gh-safe`), but when `content-prefix` is set explicitly
(e.g., `"tw:"`), the `name` uses that format instead (e.g., `tw:gh-safe`).
Without any prefix, `name` is the canonical ID.
```

#### 5. CLAUDE.md — Pipeline description

**File**: `CLAUDE.md`

- [x] Update the `AdapterConfig` reference in the pipeline stage 4 description (line 216) to mention `content_prefix`:

```markdown
`Option<&AdapterConfig>` for prefix/strip transforms (`prefix` controls file
paths and frontmatter, `content_prefix` controls model-facing names)
```

### Success Criteria

#### Automated Verification

- [x] No broken markdown links: verify both `#prefix-behavior` and `#content-reference-prefix` anchors resolve from the config table
- [x] `cargo fmt --check` still passes (no source changes in this phase)
- [x] `agentspec sync --help` renders the new `--content-prefix` flag with its description

#### Manual Verification

- [ ] README accurately describes the behavior matrix from the Desired End State section
- [ ] Config examples are consistent between the quick-start snippet and the sync config reference
- [ ] The `content-prefix` description is clear to someone who hasn't read the implementation

## References

- Idea document: `thoughts/ideas/2026-04-14-agentspec-plugin-namespace-prefix.md`
- Prior idea (infrastructure): `thoughts/ideas/2026-04-11-agentspec-prefix-aware-spec-references.md`
- `SyncTargetConfig`: `src/config.rs:245-262`
- `SyncFlags`: `src/config.rs:278-288`
- `resolve_sync_target()`: `src/config.rs:78-105`
- `adapter_configs()`: `src/config.rs:135-149`
- `AdapterConfig`: `src/compile.rs:19-30`
- Claude `model_facing_name`: `src/adapters/claude.rs:252-258`
- Claude `adapt_agent_spec` inline prefix: `src/adapters/claude.rs:120-123`
- Cursor `model_facing_name`: `src/adapters/cursor.rs:162-168`
- Cursor `adapt_agent_spec` inline prefix: `src/adapters/cursor.rs:69-72`
- Cursor `adapt_skill_spec` inline prefix: `src/adapters/cursor.rs:94-97`
- OpenCode `model_facing_name`: `src/adapters/opencode.rs:371-380`
- Template context: `src/templating/context.rs:135-149`
- CLI args: `src/cli.rs:69-71`
- Integration test fixture: `tests/fixtures/agent-config/spec/skills/basic-skill/SKILL.md`
