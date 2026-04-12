# Migrate Hardcoded Spec References in Dotfiles to Template Syntax

## Problem Statement

The dotfiles spec library (`agent-config/spec/`) contains ~30+ hardcoded references to other spec names in body content — skill names like `'gh-safe'`, agent names like `"code-reviewer"`, and one instance of a manually prefixed name (`tw:gh-safe`). When `agentspec sync` applies a prefix, these hardcoded names don't match the actual output names, causing the AI model to reference specs that don't exist under that name.

The new prefix-aware keyed access syntax (`{{ specs.skill.gh_safe.name }}`) solves this by resolving to the correct model-facing name at compile time. The existing references need to be migrated to use it.

## Motivation

- **Correctness**: Hardcoded references silently break when a prefix is configured. The model sees `'gh-safe'` but the actual skill is `tw-gh-safe`.
- **Maintainability**: Renaming or removing a spec should produce a compile error, not silently leave stale references. Template references provide this safety net.
- **Consistency**: The `sub-agents.md` rule already uses `{{ agent.name }}` for dynamic agent listing. Migrating the remaining hardcoded references makes the entire library consistent.
- **The `tw:gh-safe` hack**: `git-conventions.md` manually hardcodes the prefix as `tw:gh-safe` — a brittle workaround that would break if the prefix changes. Template syntax eliminates this entirely.

## Context

### Current References by Category

**Skill-to-skill references** (~10 locations):
- `gh-safe` — referenced from `create-pr`, `describe-pr`, `review-pr`, `review-pr-comments`, `split-branch`
- `git-town` — referenced from `gt-sync`, `split-branch`
- `resolve-conflicts` — referenced from `gt-sync`
- `thoughts-dir-resolver` — referenced from fragments `discover-dir.md`, `discover-root.md`

**Rule-to-agent references**:
- `learnings-discoverer` — referenced from `knowledge-base.md`, `sub-agents.md`

**Rule-to-skill references**:
- `tw:gh-safe` — hardcoded with prefix in `git-conventions.md`

**Skill/fragment-to-agent references** (~15+ locations):
- `code-reviewer` — referenced from `review-files`, `review-pr`, `code-review-loop`, `code-review`, fragment `delegation-rules.md`, fragment `implementation-workflow.md`
- `codebase-locator`, `codebase-analyzer`, `codebase-pattern-finder`, `web-search-researcher`, `thoughts-locator`, `thoughts-analyzer` — referenced from `create-plan`, `iterate-plan`, `create-learning-plan`, `research-web`, `vet-repo`, fragment `sub-agent-catalog.md`
- `learnings-discoverer` — referenced from fragment `sub-agent-catalog.md`

### Reference Patterns

Two distinct patterns exist:

1. **Prose references**: `Load the 'gh-safe' skill` — appears in instructional text
2. **Tool parameter references**: `subagent_type: "code-reviewer"` — appears in code/YAML examples that the model copies into tool calls

Both should be migrated. The template resolves before the model sees the content, so the model receives the correct prefixed name regardless of where it appears.

### Migration Syntax

| Before | After |
|--------|-------|
| `Load the 'gh-safe' skill` | `Load the '{{ specs.skill.gh_safe.name }}' skill` |
| `subagent_type: "code-reviewer"` | `subagent_type: "{{ specs.agent.code_reviewer.name }}"` |
| `load the 'tw:gh-safe' skill` | `Load the '{{ specs.skill.gh_safe.name }}' skill` |

## Goals / Success Criteria

- [ ] All hardcoded spec name references in `agent-config/spec/` are replaced with `{{ specs.*.name }}` template syntax
- [ ] The `tw:gh-safe` hardcoded prefix in `git-conventions.md` is eliminated
- [ ] `agentspec validate` passes (template syntax is valid)
- [ ] `agentspec compile` produces output with correct resolved names
- [ ] `agentspec sync --dry-run` shows expected prefixed names in sync output
- [ ] No remaining hardcoded spec names in spec bodies (verified by grep)

## Non-Goals (Out of Scope)

- **Fragment path references** (`{% include "review/prompt-contract.md" %}`) — these use filesystem paths, not spec names; they're not affected by prefixing
- **agentspec code changes** — the template feature is already implemented and shipped
- **Adding new specs or changing spec behavior** — this is a pure reference migration

## Constraints

- **Template syntax is more verbose**: `{{ specs.skill.gh_safe.name }}` is longer than `gh-safe`. This is an acceptable tradeoff for correctness and safety.
- **Fragment references**: Fragments that reference specs also need migration. Since fragments share the same MiniJinja context, the same `{{ specs.*.name }}` syntax works.
- **The `sub-agents.md` rule**: Already uses `{{ agent.name }}` in a `{% for %}` loop for listing agents dynamically. The hardcoded references elsewhere in that file (and in `knowledge-base.md`) should use the keyed access syntax instead.

## Resolved Questions

- **Batch vs incremental**: One batch — the change is mechanical, low-risk, and a single `agentspec compile` run verifies everything.
- **Prose references in `sub-agents.md`**: Migrate these too. Even prose-only references are confusing to agents when the name doesn't match the actual spec name after prefixing.

## References

- Prefix-aware spec references implementation: `thoughts/plans/2026-04-11-agentspec-prefix-aware-spec-references.md`
- Idea document: `thoughts/ideas/2026-04-11-agentspec-prefix-aware-spec-references.md`
- README documentation: `agentspec/README.md` (Built-in variables section)

## Affected Systems/Services

- `dotfiles/agent-config/spec/skills/` — ~8 skill specs with hardcoded references
- `dotfiles/agent-config/spec/rules/` — 3 rule specs with hardcoded references
- `dotfiles/agent-config/spec/fragments/` — ~4 fragments with hardcoded references
- `dotfiles/agent-config/generated/` — all generated output will have resolved names (cosmetic change in compile output, functional change in sync output with prefix)
