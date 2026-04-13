# Built-in Template Variables Implementation Plan

## Overview

Add a built-in `specs` template variable to agentspec's templating stage, allowing spec bodies and fragments to dynamically reference metadata (name, description, type) about all specs in the library. The variable system uses a typed struct (`TemplateContext`) on `TemplatingConfig` that is straightforward to extend with new variables in the future.

## Current State Analysis

The templating stage (`src/templating.rs:32`) renders spec bodies through MiniJinja with an empty context (`minijinja::context! {}`). The only template capabilities are fragment includes and inline `{% with %}` blocks.

### Key Discoveries:

- `resolve_fragments()` at `src/templating/fragments.rs:28` passes empty context to every render call
- `TemplatingConfig` (`src/templating.rs:15-17`) has only `fragments_dir: PathBuf`
- `NormalizedSpec` (`src/spec.rs:37-61`) has `id()`, `body()`, `path()` methods but no `description()` or `spec_type()` accessors
- Agent `description` is `String` (required); Skill/Rule `description` is `Option<String>`
- `main.rs:98-117` has access to both `validated.specs()` and `config` when constructing `TemplatingConfig`
- MiniJinja `Lenient` undefined mode (`fragments.rs:83`) means templates referencing the new variable in older spec libraries will silently produce empty output

## Desired End State

Spec bodies and fragments can use the `specs` variable to generate dynamic content:

```jinja
{% for agent in specs.agents %}
- **{{ agent.name }}**: {{ agent.description }}
{% endfor %}
```

The variable is built once from validated specs, injected into `TemplatingConfig`, and passed to every render call. Adding a new built-in variable requires: (1) a new struct, (2) a new field on `TemplateContext`, (3) population at construction in `main.rs`.

### Verification:

- `cargo test` passes with new unit tests covering variable rendering
- `cargo clippy --all-targets` clean
- `cargo fmt --check` clean
- Existing integration tests in `tests/pipeline.rs` continue to pass (backward compat)

## What We're NOT Doing

- User-defined variables from config files
- Exposing spec bodies (only name/description/type)
- Per-provider filtering of the specs variable
- Runtime/environment variables (date, OS, env vars)
- Config metadata variables (project name, output dirs)
- Trait-based plugin registry (struct fields are sufficient for built-in variables)

## Implementation Approach

Single atomic change — no meaningful intermediate checkpoint exists. The work progresses bottom-up: add accessor methods on `NormalizedSpec`, create the context types, wire them through `TemplatingConfig` to the render call, then add tests.

## Implementation

### Overview

Add a `TemplateContext` struct (with a nested `SpecsContext`) to the templating module, populate it from validated specs in `main.rs`, carry it on `TemplatingConfig`, and inject it into MiniJinja's render context.

### Changes Required:

#### 1. Add accessor methods to `NormalizedSpec`

**File**: `src/spec.rs`

- [x] Add `pub fn description(&self) -> &str` method that returns description (or `""` for `None` variants)
- [x] Add `pub fn spec_type(&self) -> &'static str` method returning `"agent"`, `"skill"`, or `"rule"`

```rust
pub fn description(&self) -> &str {
    match self {
        NormalizedSpec::Agent(s) => &s.frontmatter.description,
        NormalizedSpec::Skill(s) => s.frontmatter.description.as_deref().unwrap_or_default(),
        NormalizedSpec::Rule(s) => s.frontmatter.description.as_deref().unwrap_or_default(),
    }
}

pub fn spec_type(&self) -> &'static str {
    match self {
        NormalizedSpec::Agent(_) => "agent",
        NormalizedSpec::Skill(_) => "skill",
        NormalizedSpec::Rule(_) => "rule",
    }
}
```

#### 2. Create context types module

**File**: `src/templating/context.rs` (new file)

- [x] Define `SpecEntry` struct with `name`, `description`, `r#type` fields, deriving `Serialize`
- [x] Define `SpecsContext` struct with `agents`, `skills`, `rules`, `all` fields, deriving `Serialize`
- [x] Define `TemplateContext` struct with `specs` field, deriving `Serialize`
- [x] Implement `TemplateContext::from_specs(specs: &[NormalizedSpec]) -> Self` constructor that builds the context, sorting entries alphabetically by name

```rust
use serde::Serialize;

use crate::spec::NormalizedSpec;

/// A single spec entry exposed to templates.
#[derive(Clone, Serialize)]
pub struct SpecEntry {
    pub name: String,
    pub description: String,
    #[serde(rename = "type")]
    pub r#type: String,
}

/// The `specs` variable available in templates.
#[derive(Clone, Serialize)]
pub struct SpecsContext {
    pub agents: Vec<SpecEntry>,
    pub skills: Vec<SpecEntry>,
    pub rules: Vec<SpecEntry>,
    pub all: Vec<SpecEntry>,
}

/// Top-level template context. Extend by adding fields here.
#[derive(Clone, Serialize)]
pub struct TemplateContext {
    pub specs: SpecsContext,
}

impl TemplateContext {
    /// Build the template context from validated specs.
    ///
    /// Entries are sorted alphabetically by name within each group and in `all`.
    pub fn from_specs(specs: &[NormalizedSpec]) -> Self {
        let mut agents = Vec::new();
        let mut skills = Vec::new();
        let mut rules = Vec::new();

        for spec in specs {
            let entry = SpecEntry {
                name: spec.id().to_owned(),
                description: spec.description().to_owned(),
                r#type: spec.spec_type().to_owned(),
            };
            match spec {
                NormalizedSpec::Agent(_) => agents.push(entry),
                NormalizedSpec::Skill(_) => skills.push(entry),
                NormalizedSpec::Rule(_) => rules.push(entry),
            }
        }

        agents.sort_by(|a, b| a.name.cmp(&b.name));
        skills.sort_by(|a, b| a.name.cmp(&b.name));
        rules.sort_by(|a, b| a.name.cmp(&b.name));

        let mut all: Vec<SpecEntry> = agents.iter()
            .chain(skills.iter())
            .chain(rules.iter())
            .cloned()
            .collect();
        all.sort_by(|a, b| a.name.cmp(&b.name));

        Self {
            specs: SpecsContext { agents, skills, rules, all },
        }
    }
}
```

#### 3. Wire context into templating module

**File**: `src/templating.rs`

- [x] Add `mod context;` declaration and re-export `TemplateContext`
- [x] Add `pub context: TemplateContext` field to `TemplatingConfig`
- [x] Pass `&config.context` to `resolve_fragments()`

```rust
mod context;
mod fragments;

pub use context::TemplateContext;

pub struct TemplatingConfig {
    pub fragments_dir: PathBuf,
    pub context: TemplateContext,
}

pub fn resolve(validated: ValidatedSpecs, config: &TemplatingConfig) -> Result<ResolvedSpecs> {
    let fragment_map = load_fragments(&config.fragments_dir)?;
    let env = build_environment(&fragment_map)?;
    let specs = resolve_fragments(validated.into_specs(), &env, &config.context)?;
    Ok(ResolvedSpecs { specs })
}
```

#### 4. Inject context into MiniJinja render calls

**File**: `src/templating/fragments.rs`

- [x] Add `context: &TemplateContext` parameter to `resolve_fragments()`
- [x] Convert `TemplateContext` to a `minijinja::Value` once before the loop
- [x] Replace `minijinja::context! {}` with the serialized context value

```rust
use minijinja::Value;

use super::context::TemplateContext;

pub fn resolve_fragments(
    specs: Vec<NormalizedSpec>,
    env: &Environment<'_>,
    context: &TemplateContext,
) -> Result<Vec<NormalizedSpec>> {
    let ctx = Value::from_serialize(context);
    let mut resolved = Vec::with_capacity(specs.len());

    for mut spec in specs {
        let template = env
            .template_from_str(spec.body())
            .with_context(|| format!("failed to parse template in {}", spec.path().display()))?;

        let body = template
            .render(&ctx)
            .with_context(|| format!("failed to resolve fragments in {}", spec.path().display()))?;

        match &mut spec {
            NormalizedSpec::Agent(s) => s.body = body,
            NormalizedSpec::Skill(s) => s.body = body,
            NormalizedSpec::Rule(s) => s.body = body,
        }

        resolved.push(spec);
    }

    Ok(resolved)
}
```

#### 5. Construct context in main.rs

**File**: `src/main.rs`

- [x] Import `TemplateContext` from the library crate
- [x] Build `TemplateContext::from_specs(validated.specs())` before constructing `TemplatingConfig`
- [x] Pass it as the `context` field

```rust
let context = TemplateContext::from_specs(validated.specs());

let sources = config.resolve(&config.spec.sources_dir);
let templating_config = TemplatingConfig {
    fragments_dir: sources.join("fragments"),
    context,
};
```

#### Tests

**File**: `src/templating/context.rs` (unit tests at bottom)

- [x] Test `TemplateContext::from_specs` with mixed spec types, verifying grouped lists and `all` are sorted alphabetically
- [x] Test that specs with `None` description produce empty string in entry
- [x] Test that `all` contains entries from all types

**File**: `src/templating/fragments.rs` (extend existing test module)

- [x] Test rendering `{{ specs.agents | length }}` produces correct count
- [x] Test rendering `{% for agent in specs.agents %}{{ agent.name }}\n{% endfor %}` produces alphabetically sorted names
- [x] Test rendering `{{ specs.all[0].type }}` produces the correct type string
- [x] Test that fragments can also access the `specs` variable (include a fragment that uses `{{ specs.skills | length }}`)
- [x] Test that existing specs with no variable usage continue to render identically (backward compat — already partly covered by existing tests, but verify they still pass with non-empty context)

### Success Criteria:

#### Automated Verification:

- [x] `cargo test` passes (all existing + new tests)
- [x] `cargo clippy --all-targets` clean (no new warnings)
- [x] `cargo fmt --check` clean
- [x] Integration tests in `tests/pipeline.rs` pass unchanged (backward compatibility)

## Performance Considerations

Context is built once via `Value::from_serialize()` before the render loop and reused immutably for all specs. At ~35 specs with 3 string fields each, the serialized `Value` is trivially small. No per-spec allocation or cloning occurs during rendering.

## References

- Idea document: `thoughts/ideas/2026-04-10-agentspec-built-in-template-variables.md`
- Current templating: `src/templating.rs`, `src/templating/fragments.rs`
- Spec types: `src/spec.rs:30-61`
- Pipeline orchestration: `src/main.rs:98-117`
- MiniJinja `Value::from_serialize`: https://docs.rs/minijinja/latest/minijinja/value/struct.Value.html#method.from_serialize
