# Prefix-Aware Spec References Implementation Plan

## Overview

Enable spec authors to reference other specs using MiniJinja template syntax (`{{ specs.skill.gh_safe.name }}`) with automatic prefix resolution per provider. This requires moving template resolution into the per-provider compile loop (currently it runs once, before provider context is available) and extending the template context with keyed access maps that expose model-facing names.

## Current State Analysis

The pipeline runs as a linear typestate progression:

```
Specs → NormalizedSpecs → ValidatedSpecs → ResolvedSpecs → CompileResult
```

Template resolution (`templating::resolve()` at `src/templating.rs:35-40`) consumes `ValidatedSpecs` and produces `ResolvedSpecs` **before** compilation begins. The `TemplateContext` is built once from all specs (`main.rs:109`) with **unprefixed** canonical IDs as the `name` field (`context.rs:44`). The compile stage (`compile.rs:97-125`) then iterates `(spec, provider)` pairs, passing `AdapterConfig` (which holds the prefix) to each adapter — but by this point, template expressions in spec bodies have already been resolved without prefix awareness.

### Key Discoveries:

- `TemplateContext::from_specs()` at `src/templating/context.rs:37-76` builds `SpecEntry` values using `spec.id()` (canonical, unprefixed) as the `name`
- `SpecsContext` at `context.rs:18-23` exposes `agents`, `skills`, `rules` as `Vec<SpecEntry>` (list-only, no keyed access)
- `resolve_fragments()` at `src/templating/fragments.rs:17-44` takes ownership of specs and renders each body against a single shared context
- `compile_specs()` at `compile.rs:97-125` receives already-resolved specs and has no opportunity to influence template output
- MiniJinja `Environment` at `fragments.rs:82-94` is set to `Lenient` undefined behavior. Under `Lenient`, accessing a missing map key returns `undefined` silently, but **attribute access on that undefined value errors**. So `{{ specs.skill.nonexistent.name }}` errors (`.name` on undefined), while `{{ specs.skill.nonexistent }}` alone would silently render as empty string. The chained access pattern spec authors will use (`.name` on a map lookup result) provides free validation without changing undefined behavior. Note: this relies on MiniJinja 2.x `Lenient` semantics — if a future MiniJinja version changes how `Lenient` handles chained access, the validation would need to be revisited.
- Model-facing names differ by provider: Claude/Cursor use `{prefix}-{id}`, OpenCode skills/commands use unprefixed `{id}` (prefix is structural, in directory path)

## Desired End State

After this plan is complete:

1. Template resolution runs **per-provider** inside the compile loop, with access to the provider's `AdapterConfig` (prefix)
2. `SpecsContext` exposes both list access (`specs.agents`, `specs.skills`, `specs.rules` — unchanged) and keyed access (`specs.agent`, `specs.skill`, `specs.rule` — new, underscore-normalized keys)
3. `SpecEntry.name` in the keyed maps contains the **model-facing name** for the target provider (e.g., `tw-gh-safe` for Claude, `gh-safe` for OpenCode skills)
4. Referencing a nonexistent spec via template syntax produces a clear compile-time error (MiniJinja's existing undefined behavior)
5. Underscore-normalized ID collisions are caught during validation
6. Existing specs using list iteration (`{% for agent in specs.agents %}`) continue to work identically
7. The `Validate` command continues to check template syntax (with unprefixed context)

### Verification:

- `cargo test` passes (all existing + new tests)
- `cargo clippy --all-targets` clean
- `cargo fmt --check` passes
- A fixture spec using `{{ specs.skill.basic_skill.name }}` compiles to the correct prefixed name per provider
- A fixture spec referencing a nonexistent spec produces a compile error
- Two specs with IDs that collide under underscore normalization produce a validation error

## What We're NOT Doing

- Custom MiniJinja macros or functions (e.g., `{{ skill_name("gh-safe") }}`) — deferred for future ergonomics
- Validating plain-text references in spec bodies
- Provider-specific tool name resolution (`{{ tool("Question") }}`) — separate idea, but benefits from the same pipeline refactor
- Cross-library spec references
- Migrating existing dotfiles specs to use the new syntax (separate follow-up)

## Implementation Approach

The work is split into four phases. Phase 1 is a pure refactor (no behavior change) that moves template resolution into the compile loop. Phases 2-3 layer the new functionality. Phase 4 adds comprehensive tests.

The key architectural decision: `compile::run()` will accept `ValidatedSpecs` (instead of `ResolvedSpecs`) plus a `TemplatingResources` struct that owns the fragment map. Inside `compile_specs()`, for each provider, we build a provider-specific `TemplateContext`, clone the specs, resolve templates, then pass to the adapter. The `ResolvedSpecs` public type is eliminated — template resolution becomes an internal step of compilation.

---

## Phase 1: Pipeline Refactor — Per-Provider Template Resolution

### Overview

Move template resolution from a standalone pipeline stage into the per-provider compile loop. This is a pure refactor — no behavior change, no new features. All existing tests must continue to pass.

### Changes Required:

#### 1. New `TemplatingResources` struct

**File**: `src/templating.rs`

- [x] Add a `TemplatingResources` struct that owns the fragment map and exposes a method to build a `MiniJinja Environment`:

```rust
pub struct TemplatingResources {
    fragment_map: HashMap<String, String>,
}

impl TemplatingResources {
    pub fn load(fragments_dir: &Path) -> Result<Self> {
        let fragment_map = load_fragments(fragments_dir)?;
        Ok(Self { fragment_map })
    }

    pub fn build_environment(&self) -> Result<Environment<'_>> {
        build_environment(&self.fragment_map)
    }
}
```

- [x] Ensure `load_fragments` and `build_environment` in `fragments.rs` are accessible to `TemplatingResources` (they are `pub` within `fragments.rs` and accessible to sibling modules in `templating/`, but not re-exported to the rest of the crate — `TemplatingResources` mediates access)
- [x] Keep `resolve_fragments()` as a `pub(crate)` function — it will be called from `compile.rs` instead of `templating.rs`
- [x] Re-export `resolve_fragments` from the `templating` module so `compile.rs` can call it

#### 2. Remove `ResolvedSpecs` as a public pipeline type

**File**: `src/templating.rs`

- [x] Remove the `ResolvedSpecs` struct and its `impl` block (lines 26-54)
- [x] Remove the `resolve()` function (lines 35-40) — its logic moves into `compile.rs`
- [x] Remove the `TemplatingConfig` struct (lines 17-20) — replaced by `TemplatingResources`
- [x] Update `pub use` exports: remove `ResolvedSpecs`, add `TemplatingResources`
- [x] Update `CLAUDE.md` — two sections need changes:
  - (a) **Pipeline Stages** (lines 204-207): remove `ResolvedSpecs` as a separate stage; note that template resolution is now an internal step of compilation
  - (b) **Use config structs at module boundaries** (line 185): replace `TemplatingConfig` with `TemplatingResources` in the list of established examples

#### 3. Update `compile::run()` to handle template resolution

**File**: `src/compile.rs`

- [x] Change `compile::run()` signature: accept `&ValidatedSpecs` and `&TemplatingResources` instead of `ResolvedSpecs`:
  > **Deviation:** Uses `&ValidatedSpecs` (borrowed) instead of owned `ValidatedSpecs`. Clippy flagged pass-by-value as unused consumption since we only call `.specs()`. The borrow is cleaner and lets `main.rs` hold onto the validated specs for the `Validate` command's `into_specs()` call.

```rust
pub fn run(
    validated: &ValidatedSpecs,
    templating: &TemplatingResources,
    presets: &ProviderPresetsMap,
    providers: &[Provider],
    adapter_configs: &HashMap<Provider, AdapterConfig>,
) -> Result<CompileResult>
```

- [x] Inside `compile_specs()`, build the MiniJinja environment once, then for each provider: build `TemplateContext` from specs (unprefixed for now — Phase 2 adds prefix awareness), clone specs, resolve templates, then compile. **Note**: this inverts the existing loop order from `(spec, provider)` to `(provider, spec)`. The final `files.sort_by` ensures deterministic output ordering is preserved regardless of iteration order:

```rust
// Takes `&[NormalizedSpec]` (borrowed) even though `resolve_fragments` needs
// ownership. Phase 2 clones from this slice once per provider; Phase 1 clones
// once. The borrow avoids consuming ValidatedSpecs prematurely.
pub(crate) fn compile_specs(
    specs: &[NormalizedSpec],
    templating: &TemplatingResources,
    presets: &ProviderPresetsMap,
    providers: &[Provider],
    adapter_configs: &HashMap<Provider, AdapterConfig>,
) -> Result<CompileResult> {
    let env = templating.build_environment()?;
    let mut files: Vec<GeneratedFile> = Vec::new();

    let mut sorted_providers: Vec<Provider> = providers.to_vec();
    sorted_providers.sort_by_key(ToString::to_string);

    // Build context once and resolve templates once (Phase 2 will make this per-provider)
    let context = TemplateContext::from_specs(specs);
    let resolved = resolve_fragments(specs.to_vec(), &env, &context)?;

    for &provider in &sorted_providers {
        let adapter_config = adapter_configs.get(&provider);

        for spec in &resolved {
            let mut adapter_files = match provider {
                Provider::Claude => adapt_claude(spec.clone(), presets, adapter_config)?,
                Provider::Cursor => adapt_cursor(spec.clone(), presets, adapter_config)?,
                Provider::OpenCode => adapt_opencode(spec.clone(), presets, adapter_config)?,
            };
            files.append(&mut adapter_files);
        }
    }

    files.sort_by(|a, b| a.path.cmp(&b.path));
    Ok(CompileResult { files })
}
```

- [x] Update `use` imports: replace `ResolvedSpecs` with `ValidatedSpecs`, add `TemplatingResources`, `TemplateContext`, `resolve_fragments`

#### 4. Update `main.rs` orchestration

**File**: `src/main.rs`

- [x] Rename `load_specs()` to `load_and_validate()`, change return type from `ResolvedSpecs` to `ValidatedSpecs`:

```rust
fn load_and_validate(config: &AgentspecConfig, dirs: &SpecDirs) -> Result<ValidatedSpecs> {
    Specs::load(dirs)?
        .normalize()
        .validate(&config.presets)
        .map_err(|errors| {
            for e in &errors {
                eprintln!("error: {e}");
            }
            anyhow::anyhow!("{} semantic validation error(s)", errors.len())
        })
}
```

- [x] Add a `load_templating()` helper that builds `TemplatingResources`:

```rust
fn load_templating(config: &AgentspecConfig) -> Result<TemplatingResources> {
    let sources = config.resolve(&config.spec.sources_dir);
    TemplatingResources::load(&sources.join("fragments"))
}
```

- [x] Update `Validate` command to validate templates separately (resolve with unprefixed context, discard result — ensures template syntax is checked):

```rust
Command::Validate(_) => {
    let validated = load_and_validate(&config, &dirs)?;
    let templating = load_templating(&config)?;
    // Check template syntax by resolving with unprefixed context
    let env = templating.build_environment()?;
    let context = TemplateContext::from_specs(validated.specs());
    resolve_fragments(validated.into_specs(), &env, &context)?;
    eprintln!("validation complete");
}
```

- [x] Update `Sync` and `Compile` commands to pass `ValidatedSpecs` + `TemplatingResources` to `run_compile()`
- [x] Update `run_compile()` signature to accept the new types and forward them to `compile::run()`
- [x] Update `use` imports: remove `ResolvedSpecs`, add `TemplatingResources`, etc.

#### 5. Update library crate exports

**File**: `src/lib.rs` (if it exists) or wherever the library crate's public API is defined

- [x] Update re-exports to reflect the removed `ResolvedSpecs` and new `TemplatingResources`
- [x] Ensure `ValidatedSpecs` is accessible from the binary crate (it should already be via `specs::ValidatedSpecs`)

#### Tests for This Phase

- [x] All existing unit tests pass unchanged (`cargo test`)
- [x] All existing integration tests pass unchanged (the pipeline produces identical output)
- [x] `cargo clippy --all-targets` clean
- [x] `cargo fmt --check` passes

### Success Criteria:

#### Automated Verification:

- [x] `cargo test` — all existing tests pass with no modifications (122/122)
- [x] `cargo clippy --all-targets` — no warnings
- [x] `cargo fmt --check` — passes
- [x] Integration test `test_compile_generates_expected_files` produces identical output (golden file diff)
- [x] Integration test `test_sync_prefix` still works correctly

---

## Phase 2: Prefix-Aware Template Context with Keyed Access

### Overview

Extend `TemplateContext` to support per-provider prefix-aware names and add underscore-normalized keyed access maps (`specs.skill.gh_safe.name`). Add per-adapter `model_facing_name()` functions.

### Changes Required:

#### 1. Add per-adapter `model_facing_name()` functions

**File**: `src/adapters/claude.rs`

- [x] Add a public function:

```rust
/// Returns the name the AI model uses to reference this spec.
///
/// For Claude, all spec types use `{prefix}-{id}` when a prefix is configured.
pub fn model_facing_name(spec: &NormalizedSpec, cfg: Option<&AdapterConfig>) -> String {
    let id = spec.id();
    match cfg.and_then(|c| c.prefix.as_deref()) {
        Some(prefix) => format!("{prefix}-{id}"),
        None => id.to_owned(),
    }
}
```

**File**: `src/adapters/cursor.rs`

- [x] Add the same function (identical logic to Claude — duplication is intentional per the "provider-specific logic belongs in adapters" principle; if Cursor adopts a different naming scheme later, having a separate function avoids coupling):

```rust
pub fn model_facing_name(spec: &NormalizedSpec, cfg: Option<&AdapterConfig>) -> String {
    let id = spec.id();
    match cfg.and_then(|c| c.prefix.as_deref()) {
        Some(prefix) => format!("{prefix}-{id}"),
        None => id.to_owned(),
    }
}
```

**File**: `src/adapters/opencode.rs`

- [x] Add a function with OpenCode-specific logic:

```rust
/// Returns the name the AI model uses to reference this spec.
///
/// - **Agents**: identity comes from the filename (`{prefix}-{id}.md`), so
///   the model-facing name is prefixed.
/// - **Skills**: the frontmatter `name` field uses the unprefixed canonical ID
///   (the prefix only appears in the directory path). User-invocable skills
///   (commands) are also derived from `NormalizedSpec::Skill` — there is no
///   separate `Command` variant — and follow the same unprefixed convention.
/// - **Rules**: have no model-facing name (auto-loaded content). Returns the
///   canonical ID as a best-effort fallback; spec authors should not typically
///   reference rules by name.
pub fn model_facing_name(spec: &NormalizedSpec, cfg: Option<&AdapterConfig>) -> String {
    let id = spec.id();
    match spec {
        NormalizedSpec::Agent(_) => match cfg.and_then(|c| c.prefix.as_deref()) {
            Some(prefix) => format!("{prefix}-{id}"),
            None => id.to_owned(),
        },
        NormalizedSpec::Skill(_) | NormalizedSpec::Rule(_) => id.to_owned(),
    }
}
```

**File**: `src/adapters.rs`

- [x] Re-export the new functions:

```rust
pub use claude::model_facing_name as claude_model_facing_name;
pub use cursor::model_facing_name as cursor_model_facing_name;
pub use opencode::model_facing_name as opencode_model_facing_name;
```

#### 2. Add keyed access maps to `SpecsContext`

**File**: `src/templating/context.rs`

- [x] Add `BTreeMap` import
- [x] Add singular-named keyed maps to `SpecsContext`:

```rust
pub struct SpecsContext {
    // List access (existing — for iteration)
    pub agents: Vec<SpecEntry>,
    pub skills: Vec<SpecEntry>,
    pub rules: Vec<SpecEntry>,
    pub all: Vec<SpecEntry>,
    // Keyed access (new — for {{ specs.skill.gh_safe.name }})
    pub agent: BTreeMap<String, SpecEntry>,
    pub skill: BTreeMap<String, SpecEntry>,
    pub rule: BTreeMap<String, SpecEntry>,
}
```

- [x] Add a helper function to normalize IDs for use as map keys:

```rust
/// Replace hyphens with underscores for MiniJinja dot-access compatibility.
fn normalize_key(id: &str) -> String {
    id.replace('-', "_")
}
```

#### 3. Add provider-aware context builder

**File**: `src/templating/context.rs`

- [x] Add a new constructor that accepts a provider and adapter config, using the adapter's `model_facing_name()` to populate the `name` field:

```rust
use crate::adapters::{
    claude_model_facing_name, cursor_model_facing_name, opencode_model_facing_name,
};
use crate::compile::AdapterConfig;
use crate::provider::Provider;

impl TemplateContext {
    /// Build a provider-specific template context with prefix-aware names
    /// and keyed access maps.
    pub fn from_specs_for_provider(
        specs: &[NormalizedSpec],
        provider: Provider,
        adapter_config: Option<&AdapterConfig>,
    ) -> Self {
        let model_facing_name_fn = match provider {
            Provider::Claude => claude_model_facing_name,
            Provider::Cursor => cursor_model_facing_name,
            Provider::OpenCode => opencode_model_facing_name,
        };

        let mut agents_list = Vec::new();
        let mut skills_list = Vec::new();
        let mut rules_list = Vec::new();
        let mut agent_map = BTreeMap::new();
        let mut skill_map = BTreeMap::new();
        let mut rule_map = BTreeMap::new();

        for spec in specs {
            let entry = SpecEntry {
                name: model_facing_name_fn(spec, adapter_config),
                description: spec.description().to_owned(),
                r#type: spec.spec_type().to_owned(),
                tags: spec.tags().to_vec(),
            };
            let key = normalize_key(spec.id());

            match spec {
                NormalizedSpec::Agent(_) => {
                    agent_map.insert(key, entry.clone());
                    agents_list.push(entry);
                }
                NormalizedSpec::Skill(_) => {
                    skill_map.insert(key, entry.clone());
                    skills_list.push(entry);
                }
                NormalizedSpec::Rule(_) => {
                    rule_map.insert(key, entry.clone());
                    rules_list.push(entry);
                }
            }
        }

        agents_list.sort_unstable_by(|a, b| a.name.cmp(&b.name));
        skills_list.sort_unstable_by(|a, b| a.name.cmp(&b.name));
        rules_list.sort_unstable_by(|a, b| a.name.cmp(&b.name));

        let mut all: Vec<SpecEntry> = agents_list
            .iter()
            .chain(skills_list.iter())
            .chain(rules_list.iter())
            .cloned()
            .collect();
        all.sort_unstable_by(|a, b| a.name.cmp(&b.name));

        Self {
            specs: SpecsContext {
                agents: agents_list,
                skills: skills_list,
                rules: rules_list,
                all,
                agent: agent_map,
                skill: skill_map,
                rule: rule_map,
            },
        }
    }
}
```

- [x] Refactor `from_specs()` and `from_specs_for_provider()` to share the common loop logic (iterating specs, grouping by type, sorting, building `all`). The only difference between the two constructors is how `SpecEntry.name` is computed: canonical ID vs model-facing name. Extract a shared helper that takes a name-resolution closure:

```rust
fn build_context(
    specs: &[NormalizedSpec],
    name_fn: impl Fn(&NormalizedSpec) -> String,
) -> SpecsContext {
    // Single loop: build both lists and keyed maps per spec
    // name_fn determines whether names are canonical or prefixed
    // Returns populated SpecsContext with all six collections
}
```

Then `from_specs()` calls `build_context(specs, |s| s.id().to_owned())` and `from_specs_for_provider()` calls `build_context(specs, |s| model_facing_name_fn(s, adapter_config))`. This avoids duplicating the iteration/sorting logic and ensures both constructors stay in sync. **Note**: this means `from_specs()` gains new output — it will now populate the keyed map fields (`agent`, `skill`, `rule`) with canonical IDs. This is necessary because the `SpecsContext` struct change adds those fields, and it enables keyed access in the `Validate` command's template syntax check.

#### 4. Wire per-provider context into compile loop

**File**: `src/compile.rs`

- [x] Update `compile_specs()` to build a provider-specific context for each provider:

```rust
for &provider in &sorted_providers {
    let adapter_config = adapter_configs.get(&provider);
    let context = TemplateContext::from_specs_for_provider(specs, provider, adapter_config);
    let resolved = resolve_fragments(specs.to_vec(), &env, &context)?;

    for spec in &resolved {
        let mut adapter_files = match provider {
            Provider::Claude => adapt_claude(spec.clone(), presets, adapter_config)?,
            Provider::Cursor => adapt_cursor(spec.clone(), presets, adapter_config)?,
            Provider::OpenCode => adapt_opencode(spec.clone(), presets, adapter_config)?,
        };
        files.append(&mut adapter_files);
    }
}
```

- [x] Add imports for `TemplateContext`, `resolve_fragments`, adapter `model_facing_name` functions

#### Tests for This Phase

- [x] Unit test: `from_specs_for_provider` with Claude provider and prefix produces `tw-{id}` names in keyed maps
- [x] Unit test: `from_specs_for_provider` with OpenCode provider and prefix produces unprefixed names for skills
- [x] Unit test: `from_specs_for_provider` with no prefix produces canonical IDs
- [x] Unit test: keyed maps use underscore-normalized keys (`gh_safe` not `gh-safe`)
- [x] Unit test: template rendering with `{{ specs.skill.basic_skill.name }}` resolves to prefixed name
- [x] Unit test: `from_specs` (unprefixed) also populates keyed maps with canonical IDs
- [x] All existing tests still pass (list-based access unchanged)

### Success Criteria:

#### Automated Verification:

- [x] `cargo test` — all existing + new tests pass
- [x] `cargo clippy --all-targets` — no warnings
- [x] `cargo fmt --check` — passes
- [x] Integration test golden files unchanged (no fixture specs use keyed access yet)

---

## Phase 3: Underscore-Normalization Collision Validation

### Overview

Add a validation check that detects specs whose IDs would collide under underscore normalization (e.g., `gh-safe` and `gh_safe`). This prevents silent overwrites in the keyed access maps.

### Changes Required:

#### 1. Add collision check to `validate_semantics()`

**File**: `src/validate.rs`

- [x] After the existing duplicate-ID check loop, add per-type underscore-normalization collision detection. The check is scoped per spec type because the keyed access maps are per-type (`specs.agent.X`, `specs.skill.X`, `specs.rule.X`) — an agent and a skill with colliding normalized keys would not actually conflict:
  > **Deviation:** Used a flat `HashMap<(&str, String), Vec<...>>` keyed by `(spec_type, normalized_id)` instead of nested `HashMap`s — clippy flagged the nested type as too complex. Also added a skip for entries where all original IDs are identical (already caught by the duplicate-ID check).

```rust
// Per-type underscore-normalization collision check.
// Keyed template access normalizes hyphens to underscores, so IDs that
// differ only in hyphen/underscore placement would collide within the same map.
let mut by_type: HashMap<&str, HashMap<String, Vec<(&str, &Path)>>> = HashMap::new();
for spec in specs {
    let normalized = spec.id().replace('-', "_");
    by_type
        .entry(spec.spec_type())
        .or_default()
        .entry(normalized)
        .or_default()
        .push((spec.id(), spec.path()));
}
for (spec_type, normalized_ids) in &by_type {
    for (normalized, entries) in normalized_ids {
        if entries.len() > 1 {
            let names: Vec<&str> = entries.iter().map(|(id, _)| *id).collect();
            for (_, path) in entries {
                errors.push(SemanticError {
                    path: path.to_path_buf(),
                    message: format!(
                        "{spec_type} IDs {} all normalize to '{normalized}' (hyphens → underscores) and would collide in template keyed access",
                        names.iter().map(|n| format!("'{n}'")).collect::<Vec<_>>().join(", ")
                    ),
                });
            }
        }
    }
}
```

- [x] Add `HashMap` import if not already present

#### Tests for This Phase

- [x] Unit test: two skills `gh-safe` and `gh_safe` produces a validation error
- [x] Unit test: an agent `gh-safe` and a skill `gh_safe` does NOT produce a collision error (per-type scope)
- [x] Unit test: `foo-bar` alone does not produce a collision error
- [x] Unit test: error message includes both colliding IDs and the spec type

### Success Criteria:

#### Automated Verification:

- [x] `cargo test` — all existing + new tests pass (130/130)
- [x] `cargo clippy --all-targets` — no warnings
- [x] `cargo fmt --check` — passes

---

## Phase 4: Integration Tests and Fixture Updates

### Overview

Add integration tests that exercise the full pipeline with prefix-aware spec references. Update test fixtures to include specs that use the new keyed access syntax.

### Changes Required:

#### 1. Add fixture spec with keyed reference

**File**: `tests/fixtures/agent-config/spec/skills/basic-skill/SKILL.md`

- [x] Add a template reference to the existing test agent, e.g., append a line to the body:

```
Agent: {{ specs.agent.test_agent.name }}
```

#### 2. Update expected golden files

- [x] Update all `generated/*/` golden files for `basic-skill` to include the resolved reference:
  - `generated/claude/skills/basic-skill/SKILL.md` — should contain `Agent: test-agent`
  - `generated/cursor/skills/basic-skill/SKILL.md` — should contain `Agent: test-agent`
  - `generated/opencode/commands/basic-skill.md` — should contain `Agent: test-agent`
  > **Deviation:** Also updated `generated/codex/skills/basic-skill/SKILL.md` — the plan didn't list it but the fixture includes codex golden files.

Note: The fixture `agentspec.toml` has no `[sync.*]` sections with prefixes, so `compile` produces no adapter configs — all names resolve to canonical IDs. OpenCode has no `skills/` directory in the fixture (only `agents/`, `commands/`, `rules/`), so no OpenCode skill golden file needs updating. The sync prefix integration tests exercise prefixed resolution.

#### 3. Add integration test for prefixed keyed reference

**File**: `tests/pipeline.rs`

- [x] Add a test that configures a prefix via `agentspec.toml`, syncs, and verifies the output contains the prefixed name in the spec body:

```rust
#[test]
fn test_sync_prefix_resolves_spec_references() {
    // Setup: write agentspec.toml with prefix = "tw" for claude
    // Sync to a temp home dir
    // Read the synced basic-skill SKILL.md
    // Assert it contains "Agent: tw-test-agent" (prefixed)
}
```

- [x] Add a test for OpenCode where skill references resolve to unprefixed names:

```rust
#[test]
fn test_sync_opencode_spec_references_unprefixed() {
    // Setup: write agentspec.toml with prefix = "tw" for opencode
    // Sync to a temp home dir
    // Read the synced basic-skill body
    // Assert agent reference is "Agent: tw-test-agent" (OpenCode agents ARE prefixed)
    // If referencing a skill, assert it would be unprefixed
}
```

#### 4. Add integration test for nonexistent spec reference error

**File**: `tests/pipeline.rs`

- [x] Add a test that creates a fixture spec referencing a nonexistent spec and verifies compilation fails:

```rust
#[test]
fn test_compile_nonexistent_spec_reference_errors() {
    // Setup: create a spec body with {{ specs.skill.nonexistent_skill.name }}
    // Run compile
    // Assert failure with error message about undefined
}
```

#### Tests for This Phase

- [x] Integration test: prefixed keyed reference resolves correctly in sync output
- [x] Integration test: OpenCode agent references are prefixed, skill references are unprefixed
- [x] Integration test: nonexistent spec reference produces a compile error
- [x] All existing integration tests pass with updated golden files

### Success Criteria:

#### Automated Verification:

- [x] `cargo test` — all tests pass including new integration tests
- [x] `cargo clippy --all-targets` — no warnings
- [x] `cargo fmt --check` — passes
- [x] `test_compile_generates_expected_files` passes with updated golden files
- [x] `test_sync_prefix_resolves_spec_references` passes
- [x] `test_compile_nonexistent_spec_reference_errors` passes

---

## Performance Considerations

Template resolution now runs once per provider instead of once globally. With ~35 specs and 2-3 providers, the cost of cloning specs and re-rendering templates is negligible (sub-millisecond). MiniJinja parsing and rendering are fast by design. If spec counts grow to thousands, we could cache the environment and only rebuild the context per-provider, but this is not needed now.

## References

- Idea document: `thoughts/ideas/2026-04-11-agentspec-prefix-aware-spec-references.md`
- Built-in template variables idea: `thoughts/ideas/2026-04-10-agentspec-built-in-template-variables.md`
- Tool name resolver idea: `thoughts/ideas/2026-04-09-agentspec-tool-name-resolver.md`
- Pipeline orchestration: `src/main.rs:98-137`
- Template resolution: `src/templating.rs:35-40`, `src/templating/fragments.rs:17-44`
- Template context: `src/templating/context.rs:28-76`
- Compile dispatch: `src/compile.rs:88-125`
- Adapter config: `src/compile.rs:17-29`
- Claude adapter (name at lines 120-123): `src/adapters/claude.rs`
- Cursor adapter (name at lines 69-72, 94-97): `src/adapters/cursor.rs`
- OpenCode adapter (unprefixed name at line 162): `src/adapters/opencode.rs`
- Validation: `src/validate.rs:91-152`
- Integration tests: `tests/pipeline.rs`
- Test fixtures: `tests/fixtures/agent-config/`
