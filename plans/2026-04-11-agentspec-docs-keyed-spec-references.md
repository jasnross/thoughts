# Document Keyed Spec References in agentspec README

## Overview

Update the agentspec README to document the new keyed access syntax (`specs.skill.gh_safe.name`) and the prefix-aware name resolution behavior. This enables spec authors to discover and adopt the feature.

## Current State Analysis

The README at `agentspec/README.md:228-271` documents the `specs` built-in variable with list-based access (`specs.agents`, `specs.skills`, `specs.rules`, `specs.all`). Each entry is documented as having `name`, `description`, and `type` fields.

Missing from the documentation:

1. **Keyed access maps** — `specs.agent`, `specs.skill`, `specs.rule` (singular) for direct lookup by underscore-normalized ID
2. **Prefix-aware name resolution** — `name` resolves to the model-facing name (prefixed when a sync prefix is configured), not the canonical ID
3. **The `tags` field** — added in a prior commit but not yet documented in the README
4. **Best practice guidance** — spec authors should use `{{ specs.skill.gh_safe.name }}` instead of hardcoding `gh-safe` or `tw-gh-safe`
5. **Underscore normalization** — hyphens in IDs become underscores in keyed access (e.g., `gh-safe` → `specs.skill.gh_safe`)

### Key Discoveries:

- README template variable section: `agentspec/README.md:228-271`
- The `name` field description at line 250 says "The spec's `id` field" — this is now inaccurate since `name` can be the prefixed model-facing name during compilation
- The `tags` field is already implemented in code (`context.rs:13`) but not listed in the README's entry field table at line 246-253

## Desired End State

After this plan is complete, the agentspec README:

1. Documents keyed access syntax with examples
2. Explains prefix-aware name resolution behavior
3. Lists all entry fields including `tags`
4. Includes a best-practice note recommending keyed references over hardcoded names
5. Documents the underscore normalization convention for keyed access

### Verification:

- The README accurately describes the implemented behavior
- Examples use correct syntax that matches the implementation
- No stale descriptions remain (e.g., `name` = "The spec's `id` field")

## What We're NOT Doing

- Migrating existing dotfiles specs to use the new syntax (separate plan)
- Adding new agentspec features or code changes
- Updating CHANGELOG.md (that happens at release time)

## Implementation

### Overview

Update the Built-in variables section of `agentspec/README.md` to document keyed access, prefix-aware names, tags, and best practices.

### Changes Required:

#### 1. Update the `specs` variable documentation

**File**: `agentspec/README.md`

- [x] Add keyed access maps to the field table (after the existing list-access table):

```markdown
Keyed access maps let you reference a specific spec by its ID (with
hyphens replaced by underscores):

| Field        | Type                  | Description                              |
| ------------ | --------------------- | ---------------------------------------- |
| `specs.agent`| map of name → entry   | Agents keyed by underscore-normalized ID |
| `specs.skill`| map of name → entry   | Skills keyed by underscore-normalized ID |
| `specs.rule` | map of name → entry   | Rules keyed by underscore-normalized ID  |
```

- [x] Update the entry field table to include `tags` and clarify `name`:

```markdown
| Field         | Description                                                    |
| ------------- | -------------------------------------------------------------- |
| `name`        | The spec's name as the model sees it (may include sync prefix) |
| `description` | The spec's description (empty string if not set)               |
| `type`        | One of `agent`, `skill`, or `rule`                             |
| `tags`        | List of tags from frontmatter (empty list if not set)          |
```

- [x] Add a keyed access example after the existing list examples:

```markdown
Example — referencing a specific skill by name:

\`\`\`
Load the '{{ specs.skill.gh_safe.name }}' skill before proceeding.
\`\`\`

When compiled with `prefix = "tw"` for Claude, this resolves to:

\`\`\`
Load the 'tw-gh-safe' skill before proceeding.
\`\`\`

Without a prefix, it resolves to the canonical ID (`gh-safe`).
```

- [x] Add a note about underscore normalization:

```markdown
**Key normalization**: hyphens in spec IDs are replaced with underscores
for keyed access. A spec with `id: gh-safe` is accessed as
`specs.skill.gh_safe`.
```

- [x] Add a best-practice note recommending keyed references:

```markdown
> **Best practice**: Use keyed references (`{{ specs.skill.gh_safe.name }}`)
> instead of hardcoding spec names in body text. This ensures references
> stay correct when the sync prefix changes, and produces a compile error
> if the referenced spec is renamed or removed.
```

- [x] Add a note about `subagent_type` references in code examples:

```markdown
This also applies to `subagent_type` values in tool-call examples:

\`\`\`
subagent_type: "{{ specs.agent.code_reviewer.name }}"
\`\`\`
```

#### Tests

No code tests — this is a documentation-only change. Verification is manual.

### Success Criteria:

#### Automated Verification:

- [x] No agentspec code changes — `cargo test` still passes as-is

#### Manual Verification:

- [ ] README keyed access table is accurate (field names match `SpecsContext` struct)
- [ ] README entry field table matches `SpecEntry` struct (name, description, type, tags)
- [ ] Examples use valid MiniJinja syntax that would compile successfully
- [ ] Best-practice note is clear and actionable for spec authors

## References

- Current README template section: `agentspec/README.md:228-271`
- `SpecsContext` struct: `agentspec/src/templating/context.rs:28-44`
- `SpecEntry` struct: `agentspec/src/templating/context.rs:7-14`
- `normalize_key` function: `agentspec/src/templating/context.rs:56-58`
- Implementation plan: `thoughts/plans/2026-04-11-agentspec-prefix-aware-spec-references.md`
- Idea document: `thoughts/ideas/2026-04-11-agentspec-prefix-aware-spec-references.md`
