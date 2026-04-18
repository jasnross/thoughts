# Built-in Template Variables for agentspec

## Problem Statement

agentspec's templating system currently renders spec bodies with an empty MiniJinja context — the only capabilities are fragment includes (`{% include %}`) and inline scoped variables (`{% with %}`). There is no way for a spec to reference information about other specs at render time.

This means common patterns like "list all available agents with their descriptions in a rule" require manual maintenance — the rule body must be hand-updated whenever an agent is added or removed, defeating the purpose of a compiled spec system.

## Motivation

- **Self-documenting spec libraries**: Rules and skills should be able to dynamically reference the agents and skills available in the same spec library without manual sync.
- **Reduced maintenance burden**: Adding a new agent spec should automatically surface it in any listing templates, with no second edit required.
- **Foundation for richer templating**: A well-designed variable injection system opens the door to future built-in variables (config metadata, environment info, user-defined values) without architectural rework.

## Context

The templating stage sits between validation and compilation in the pipeline:

```
ValidatedSpecs → templating::resolve() → ResolvedSpecs → compile
```

At the point templating runs, all specs have been loaded, normalized, and validated — full frontmatter metadata (name, description, type, providers, presets, etc.) is available in typed structs. The current implementation passes `minijinja::context! {}` (empty) to every spec body render.

MiniJinja supports arbitrary structured data in its context (maps, lists of objects, nested values), so the rendering engine already supports what's needed — the gap is purely that no data is injected.

Templates render in a shared MiniJinja environment where fragments are registered as named templates. Any variables injected into the render context are naturally available in both spec bodies and included fragments.

## Goals / Success Criteria

- [ ] Spec bodies and fragments can reference a `specs` variable containing metadata about all specs in the library
- [ ] The `specs` variable exposes grouped access (`specs.agents`, `specs.skills`, `specs.rules`) and a flat list (`specs.all`)
- [ ] Each entry in the variable exposes: `name`, `description`, `type`
- [ ] The variable system is designed for extensibility — adding a new built-in variable requires minimal, localized changes (ideally adding to a single registration point)
- [ ] Existing specs with no template variable usage continue to work identically (backward compatible)
- [ ] Undefined variable access remains lenient (current behavior) so partial adoption is safe

## Non-Goals (Out of Scope)

- **User-defined variables** from config files (e.g., custom key-value pairs in `agentspec.toml`) — potential future extension but not part of this effort
- **Cross-spec body access** — exposing the full rendered body of one spec inside another (circular dependency risk, large context)
- **Per-provider variable filtering** — e.g., only showing specs that target a specific provider. Users can filter with MiniJinja's `selectattr` if needed.
- **Runtime/environment variables** — date, OS, env vars. Future extension.
- **Config metadata variables** — project name, output dirs. Future extension.

## Proposed Solution Shape

The variable system should:

1. **Collect context data** from the validated specs before rendering begins
2. **Inject that data** into the MiniJinja render context for every spec body
3. **Be structured as a registry** or builder so new variables are added by registering a new provider/builder function rather than modifying rendering logic

The `specs` variable structure:

```jinja
{# Grouped access #}
{% for agent in specs.agents %}
- **{{ agent.name }}**: {{ agent.description }}
{% endfor %}

{# Flat access #}
{% for spec in specs.all %}
- [{{ spec.type }}] {{ spec.name }}: {{ spec.description }}
{% endfor %}
```

## Constraints

- Must fit within the existing pipeline typestate progression — the variable context is built from `ValidatedSpecs` data (which is fully normalized and validated)
- `TemplatingConfig` is the library-side config struct for this stage — any new configuration (if needed) should extend or compose with it, not replace it
- MiniJinja's `Lenient` undefined mode means missing variables silently evaluate as empty — this is both a safety net and a potential source of silent bugs. No change to this behavior is proposed.
- Variables are computed once before rendering begins (not lazily per-spec) since all data is available upfront

## Resolved Questions

- **Ordering**: Lists sorted alphabetically by `name` field. Deterministic, intuitive, produces readable output.
- **Self-inclusion**: Yes — all specs see all specs, including their own type. Document this behavior.
- **Naming conflicts**: MiniJinja's natural scoping applies — inner `{% with %}` scope wins over built-in variables. Document as expected behavior; no warning or error needed.
- **Performance**: Build context eagerly once before rendering, reuse for all specs. At ~35 specs with 3 string fields, overhead is negligible. Revisit only if libraries grow to thousands of specs.

## References

- Current templating implementation: `src/templating.rs`, `src/templating/fragments.rs`
- MiniJinja context docs: https://docs.rs/minijinja/latest/minijinja/macro.context.html
- Pipeline stage types: `src/specs.rs` (typestate wrappers)
- `NormalizedSpec` struct: `src/spec.rs:31`
