# Template Variables Documentation Update Plan

## Overview

Add documentation for the new built-in `specs` template variable to the README and update the CLAUDE.md pipeline description to reflect that the templating stage now injects context variables.

## Current State Analysis

The README Templating section (lines 158-227) documents:
- Fragment includes (`{% include %}`)
- Scoped variables (`{% with %}`)
- Indented includes (`{% filter indent() %}`)

There is no mention of built-in template variables. The CLAUDE.md pipeline stage 4 description says: "renders MiniJinja `{% include %}` and `{% with %}` tags in spec bodies" — no mention of injected context.

### Key Discoveries:

- README Templating section ends at line 227 with the "Fragments can include other fragments" note
- The `#### Variables` subsection (lines 184-204) documents `{% with %}` scoped variables only
- CLAUDE.md pipeline stage 4 (line in "Pipeline Stages" section) describes templating as fragment/include resolution only

## Desired End State

- Users reading the README understand that `specs` (and future built-in variables) are available in templates
- The documentation shows practical usage examples with correct syntax
- CLAUDE.md accurately reflects that the templating stage injects a context with built-in variables

### Verification:

- Documentation is accurate (matches the implementation)
- Examples use correct MiniJinja syntax
- No broken Markdown formatting

## What We're NOT Doing

- Adding a dedicated docs/ directory or separate templating guide
- Documenting internal implementation details (TemplateContext, SpecEntry structs)
- Adding examples to test fixtures (that's a separate concern)

## Implementation

### Overview

Add a `#### Built-in variables` subsection to the README Templating section and update CLAUDE.md pipeline stage 4.

### Changes Required:

#### 1. README — Add built-in variables documentation

**File**: `README.md`

- [x] Add `#### Built-in variables` subsection after the `#### Indented include` subsection (after line 226), before the closing "Fragments can include other fragments" note
- [x] Document the `specs` variable structure (`specs.agents`, `specs.skills`, `specs.rules`, `specs.all`)
- [x] Document each entry's fields (`name`, `description`, `type`)
- [x] Include a practical example: listing all agents with name and description
- [x] Include a brief example using `specs.all` with the `type` field
- [x] Note that lists are sorted alphabetically by name
- [x] Mention extensibility: "Additional built-in variables may be added in future versions"

Content to add (insert before "Fragments can include other fragments" line):

````markdown
#### Built-in variables

In addition to user-defined `{% with %}` variables, agentspec provides built-in
variables that expose metadata about all specs in the library.

##### `specs`

The `specs` variable contains all specs grouped by type and as a flat list.
Each entry exposes `name`, `description`, and `type` fields. Lists are sorted
alphabetically by name.

| Field         | Type            | Description                                    |
| ------------- | --------------- | ---------------------------------------------- |
| `specs.agents` | list of entries | All agent specs                                |
| `specs.skills` | list of entries | All skill specs                                |
| `specs.rules`  | list of entries | All rule specs                                 |
| `specs.all`    | list of entries | All specs regardless of type                   |

Each entry has:

| Field         | Description                                  |
| ------------- | -------------------------------------------- |
| `name`        | The spec's `id` field                        |
| `description` | The spec's description (empty string if not set) |
| `type`        | One of `agent`, `skill`, or `rule`           |

Example — listing all available agents in a rule:

```
{% for agent in specs.agents %}
- **{{ agent.name }}**: {{ agent.description }}
{% endfor %}
```

Example — listing all specs with their type:

```
{% for spec in specs.all %}
- [{{ spec.type }}] {{ spec.name }}
{% endfor %}
```

Built-in variables are available in both spec bodies and included fragments.
Additional built-in variables may be added in future versions.
````

#### 2. CLAUDE.md — Update pipeline stage 4 description

**File**: `CLAUDE.md`

- [x] Update stage 4 description to mention built-in template variables alongside fragment resolution

Change from:
```
4. **Template resolution** — `templating.rs` renders MiniJinja `{% include %}`
   and `{% with %}` tags in spec bodies → `ResolvedSpecs`
```

To:
```
4. **Template resolution** — `templating.rs` renders MiniJinja templates in
   spec bodies with a context containing built-in variables (e.g., `specs`)
   and resolves `{% include %}` fragment references → `ResolvedSpecs`
```

#### Tests

- [x] Verify Markdown renders correctly by checking `README.md` with no broken table formatting or code blocks (visual inspection or a Markdown linter if available)

### Success Criteria:

#### Automated Verification:

- [x] No broken Markdown links or formatting: run a basic check that the README parses cleanly
- [x] `cargo test` still passes (documentation changes shouldn't affect anything, but confirms no accidental edits to code)

#### Manual Verification:

- [ ] README examples match the actual template variable behavior (manual-only: requires reading the implementation to confirm accuracy)

## References

- Implementation: `src/templating/context.rs` (TemplateContext, SpecsContext, SpecEntry)
- Current README Templating section: `README.md:158-227`
- CLAUDE.md pipeline stages: `CLAUDE.md` (Pipeline Stages section)
- Idea document: `thoughts/ideas/2026-04-10-agentspec-built-in-template-variables.md`
