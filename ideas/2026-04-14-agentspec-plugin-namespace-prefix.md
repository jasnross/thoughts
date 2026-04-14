# Agentspec: Plugin Namespace Prefix (File Name vs. Content Name Divergence)

## Problem Statement

When agentspec specs are deployed as **Claude Code plugins**, there's a naming conflict between agentspec's prefix system and Claude Code's plugin namespace system. Claude Code plugins have a `plugin.json` with a `name` field (e.g., `"tw"`) that automatically prefixes all skills with `{name}:` (colon-separated) at runtime. This means:

- A skill at `skills/resolve-thoughts-dir/` inside a `tw` plugin becomes `tw:resolve-thoughts-dir`
- If agentspec also applies its prefix, the file becomes `skills/tw-resolve-thoughts-dir/`, and Claude Code sees it as `tw:tw-resolve-thoughts-dir` — doubled prefix
- If the agentspec prefix is omitted to avoid doubling, content references via `{{ specs.skill.resolve_thoughts_dir.name }}` resolve to the bare `resolve-thoughts-dir`, but Claude Code requires the colon form `tw:resolve-thoughts-dir` to load the skill

There is currently no way to tell agentspec: "don't prefix the file name, but DO prefix content references with `tw:` (colon-separated)."

## Motivation

- **Correctness in plugin context**: Cross-spec references in generated content must use the form Claude Code actually recognizes. For plugins, that's the colon-namespaced form (`tw:skill-name`), not the hyphenated form (`tw-skill-name`).
- **Single source of truth**: Spec authors shouldn't need to hardcode `tw:resolve-thoughts-dir` in prose — that defeats the purpose of configurable prefixes. Changing the plugin name should propagate automatically.
- **No workarounds should be necessary**: Today the only option is to omit the prefix entirely and accept broken content references, or hardcode them. Both are fragile.
- **Generalizable**: While this is triggered by Claude Code's plugin system, the underlying need — different prefix behavior for file paths vs. content references — is a general concern that could apply to other providers.

## Context

### How Claude Code Plugin Namespacing Works

1. A plugin declares `"name": "tw"` in `plugin.json`
2. All skills within the plugin are automatically namespaced: `{plugin-name}:{skill-directory-name}`
3. The colon form (`tw:resolve-thoughts-dir`) is the **only** form Claude Code recognizes when loading a skill by name
4. The file/directory name on disk must NOT include the plugin prefix — Claude Code adds it

### How Agentspec's Prefix Currently Works

The existing prefix system has a single `prefix: Option<String>` per provider that applies uniformly:

| Concern | Current behavior |
|---------|-----------------|
| File path | `{prefix}-{id}` (e.g., `tw-resolve-thoughts-dir`) |
| Content reference (`model_facing_name`) | `{prefix}-{id}` (e.g., `tw-resolve-thoughts-dir`) |
| Separator | Always `-` (hyphen) |

Both concerns use the same prefix value and the same separator. There is no mechanism to:
- Apply a prefix to content references but not file names
- Use a different separator (`:` vs `-`) for content references vs file paths
- Omit the file prefix while keeping the content prefix

### Prior Art Within Agentspec

- The [prefix-aware spec references idea](2026-04-11-agentspec-prefix-aware-spec-references.md) established the `{{ specs.skill.foo.name }}` template system and moved template resolution into the per-provider compile loop. That work is complete and provides the infrastructure this feature would extend.
- The OpenCode adapter already demonstrates per-provider divergence: its `model_facing_name()` only prefixes agents (not skills/rules), and its file paths use `commands/{prefix}/{id}.md` (prefix as subdirectory). This shows the adapter layer can already express provider-specific naming rules.
- `AdapterConfig::file_prefix()` and per-adapter `model_facing_name()` are the two extension points where this would be implemented.

### The Fundamental Tension

There are two independent naming concerns that are currently conflated:

1. **File path prefix** — whether and how to prefix file/directory names on disk
2. **Content reference prefix** — what name appears when content references a spec (the "model-facing name")

The current system treats both as a single toggle: prefix is either on (hyphen-separated, applied to both) or off (no prefix anywhere).

## Goals / Success Criteria

- [ ] Agentspec can generate specs for Claude Code plugins where file names are unprefixed but content references use the `{prefix}:{id}` colon form
- [ ] The existing non-plugin prefix behavior (`{prefix}-{id}` for both file and content) continues to work unchanged
- [ ] The solution generalizes across providers rather than being Claude-specific config
- [ ] `{{ specs.skill.resolve_thoughts_dir.name }}` resolves to `tw:resolve-thoughts-dir` in the plugin scenario
- [ ] Configuration is minimal and intuitive — the common cases shouldn't require verbose config

## Non-Goals (Out of Scope)

- **Automatic detection of plugin context**: agentspec doesn't need to read `plugin.json` or infer it's targeting a plugin — the user configures this explicitly
- **Plugin manifest generation**: generating or managing `plugin.json` itself
- **Cross-plugin references**: referencing specs from a different plugin package
- **Changing the template syntax**: `{{ specs.skill.foo.name }}` stays the same — only what `.name` resolves to changes
- **Refactoring OpenCode's per-spec-type logic**: OpenCode's adapter already correctly handles its divergent naming (unprefixed skill names, subdirectory commands). That logic stays adapter-internal — this change provides the config fields but doesn't force OpenCode to change.

## Chosen Solution: `prefix` + `content_prefix`

Add a `content_prefix` field alongside the existing `prefix`. The content prefix is a literal string (including its separator) that gets prepended directly to the spec ID in content references. When not specified, it defaults to `"{prefix}-"` (preserving current behavior).

### Config examples

```toml
# Standard prefix (unchanged — backwards compatible)
[sync.claude]
prefix = "tw"
# -> file: tw-{id}, content: tw-{id}

# Claude Code plugin: no file prefix, colon-separated content prefix
[sync.claude]
content_prefix = "tw:"
# -> file: {id}, content: tw:{id}

# Both: file prefix AND different content prefix
[sync.claude]
prefix = "tw"
content_prefix = "tw:"
# -> file: tw-{id}, content: tw:{id}

# No prefix at all (also unchanged)
[sync.claude]
# -> file: {id}, content: {id}
```

### Behavior matrix

| `prefix` | `content_prefix` | File path | Content name |
|----------|------------------|-----------|-------------|
| `"tw"` | (not set) | `tw-{id}` | `tw-{id}` |
| `"tw"` | `"tw:"` | `tw-{id}` | `tw:{id}` |
| (not set) | `"tw:"` | `{id}` | `tw:{id}` |
| (not set) | (not set) | `{id}` | `{id}` |

### Key design decisions

- **`content_prefix` includes its separator**: The value `"tw:"` is prepended directly to the ID — agentspec does not insert a separator. This eliminates the need for a separate `separator` config field and makes the config self-documenting.
- **`content_prefix` defaults to `"{prefix}-"`**: When only `prefix` is set, content names match file names (current behavior). The default is computed, not stored — only an explicit `content_prefix` in TOML overrides it.
- **`prefix` continues to control file paths**: `file_prefix()` remains `"{prefix}-"` with the hyphen added internally. No change to file path behavior.
- **OpenCode stays adapter-internal**: OpenCode's per-spec-type logic (only prefix agents, use subdirectory for commands) remains hardcoded in its adapter. The new `content_prefix` field is available to it via `AdapterConfig` but doesn't force any changes.
- **`AdapterConfig` gets a `content_prefix` field**: Matches the TOML field name for consistency. Adapters read `content_prefix` in `model_facing_name()` instead of (or in addition to) `prefix`.

### Implementation sketch

```rust
// compile.rs
pub struct AdapterConfig {
    /// Namespace prefix for file paths (e.g., "tw" → files prefixed "tw-").
    pub prefix: Option<String>,
    /// Literal prefix for content/model-facing names (e.g., "tw:" → "tw:{id}").
    /// Defaults to "{prefix}-" when not explicitly set.
    pub content_prefix: Option<String>,
}

impl AdapterConfig {
    /// File path prefix — always hyphen-separated.
    pub fn file_prefix(&self) -> Option<String> {
        self.prefix.as_ref().map(|p| format!("{p}-"))
    }

    /// Content/model-facing name prefix — literal prepend, no separator added.
    /// Falls back to "{prefix}-" when content_prefix is not explicitly set.
    pub fn content_prefix(&self) -> Option<String> {
        self.content_prefix
            .clone()
            .or_else(|| self.prefix.as_ref().map(|p| format!("{p}-")))
    }
}

// adapters/claude.rs
pub fn model_facing_name(spec: &NormalizedSpec, cfg: Option<&AdapterConfig>) -> String {
    let id = spec.id();
    match cfg.and_then(AdapterConfig::content_prefix) {
        Some(prefix) => format!("{prefix}{id}"),
        None => id.to_owned(),
    }
}
```

## Constraints

- **Backwards compatibility**: Existing configs with `prefix = "tw"` must continue to produce `tw-{id}` for both files and content. The `content_prefix` default-to-`prefix` behavior ensures this.
- **Template context reuse**: The `{{ specs.skill.foo.name }}` system already works. The solution only changes what `.name` resolves to — it doesn't require new template syntax or a new resolution stage.
- **Per-provider independence**: Each provider's prefix config is already independent. The solution maintains this — Claude can use `content_prefix = "tw:"` while Cursor uses the standard `prefix = "tw"`.

## Resolved Questions

- **Should `file_prefix` and `content_prefix` be fully independent, or should `content_prefix` default to `file_prefix`?** — `content_prefix` defaults to `"{prefix}-"` when not set. Both fields are explicit in config but the common case (standard prefix) requires only `prefix`.
- **How should this interact with OpenCode's existing divergence?** — OpenCode's per-spec-type prefix logic stays adapter-internal. It already works correctly. The new `content_prefix` is available via `AdapterConfig` if OpenCode ever needs it, but no code changes are required.
- **Is `separator` the right abstraction?** — No. Embedding the separator in the `content_prefix` value itself (e.g., `"tw:"`) is simpler and more flexible. No separate `separator` config field is needed.
- **What should the `AdapterConfig` field be named?** — `content_prefix`, matching the TOML config field name for consistency.

## References

- Prior idea: [Prefix-Aware Spec References](2026-04-11-agentspec-prefix-aware-spec-references.md) — the infrastructure this builds on
- Prior idea: [Built-in Template Variables](2026-04-10-agentspec-built-in-template-variables.md) — the `{{ specs }}` context system
- `AdapterConfig::file_prefix()`: `agentspec/src/compile.rs:27-29`
- Claude `model_facing_name()`: `agentspec/src/adapters/claude.rs:252-258`
- OpenCode `model_facing_name()`: `agentspec/src/adapters/opencode.rs:371-380`
- Template context construction: `agentspec/src/templating/context.rs:63-117`
- Config resolution: `agentspec/src/config.rs:78-105`

## Affected Systems/Services

- `agentspec` compiler — config parsing (`SyncTargetConfig`), `AdapterConfig`, `file_prefix()`, `content_prefix()`
- Claude and Cursor adapters — `model_facing_name()` reads `content_prefix` instead of `prefix`
- OpenCode adapter — no changes required (adapter-internal logic continues to work)
- Template context (`templating/context.rs`) — `name_fn` already delegates to adapter `model_facing_name()`, so it picks up the new behavior automatically
- Existing specs — no changes required; specs using `{{ specs.skill.foo.name }}` get the correct resolved name
