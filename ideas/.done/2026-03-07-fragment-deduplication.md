# Deduplicate Spec Files Using Handlebars Fragments

## Problem Statement

The agent-config spec directory contains significant duplication across 8 agents and 28 skills. Identical or near-identical instruction blocks are copied between files, leading to wording drift, inconsistent behavior between related skills, and maintenance burden when updating shared logic. A bug fix or wording improvement in one file can easily miss its duplicates elsewhere.

The Handlebars fragment system is already in place (one fragment exists: `review/prompt-contract`), and the compiler supports fragment resolution including nested fragments. The infrastructure is ready — we just need to author the fragments and migrate the specs.

## Motivation

- **Consistency**: Related skills (e.g., `implement-plan` and `implement-plan-worktree`) should behave identically for shared concerns. Currently they drift in punctuation, wording, and occasionally logic.
- **Correctness**: Some duplicated blocks contain critical logic (git merge-base resolution, PR hunk line-number parsing) where a bug fix in one copy must propagate to all copies. Fragments make this automatic.
- **Maintainability**: Updating a shared behavior (e.g., review delegation rules) currently requires finding and editing 3-6 files. A fragment reduces this to one edit.
- **Established pattern**: The fragment system and compiler already exist and work. This effort extends an existing, proven pattern rather than introducing new infrastructure.

## Context

A research analysis ([2026-03-07-spec-duplication-fragment-candidates.md](../research/2026-03-07-spec-duplication-fragment-candidates.md)) identified **13 high-value fragment candidates** organized into 5 thematic clusters:

| Cluster | Fragments | Files Affected |
|---------|-----------|----------------|
| Agent Behavioral Contracts | documentarian-mandate, file-line-references, no-critique-bullets | 5 agents |
| Review Workflow | delegation-rules, triage-workflow, merge-base-resolution, feedback-processing | 6 skills |
| Research & Planning | document-conventions, sub-agent-catalog, implementation-workflow | 6 skills |
| GitHub & PR Operations | host-detection, pr-inline-review, pr-description-rules | 6 skills |
| Cross-cutting (deferred) | Small 1-3 line patterns | Many files |

The compiler supports:
- Fragment resolution via `{{> category/name}}` syntax
- Variable passing via `{{> category/name variable="value"}}`
- Nested fragments (fragments can include other fragments)
- Handlebars conditionals (`{{#if}}`, `{{#unless}}`)

## Goals / Success Criteria

- [ ] All 13 identified fragment candidates are evaluated — each is either implemented as a fragment or explicitly decided against with rationale
- [ ] Specs that consumed duplicated text now use `{{> fragment}}` invocations instead
- [ ] No behavioral change in generated output — fragments produce identical (or intentionally improved) text compared to the originals
- [ ] Wording drift between related specs is eliminated for shared concerns
- [ ] The build passes: `npm --prefix agent-config run build:agents` succeeds after all changes
- [ ] Tests pass: `npm --prefix agent-config test` succeeds after all changes

## Non-Goals (Out of Scope)

- **Small cross-cutting patterns**: Single-line patterns like "no attribution", "never git add -A", "read files fully" are deferred. These may be better served by template variables rather than fragments.
- **New instructions or behavioral changes**: This is a deduplication effort. Fragment content should match existing spec text (with drift reconciled to the best version), not introduce new behaviors.
- **Compiler feature work beyond what's needed**: If a compiler enhancement is needed to support a specific fragment design, it's in scope. Speculative compiler improvements are not.

## Design Guidance

### Fragment conditionals
Use Handlebars conditionals (`{{#if}}`) when the concern is well-scoped and the variants are minor (e.g., swapping a path label, toggling an optional section). Avoid creating "god fragments" that handle multiple distinct concerns through extensive conditional branching. When variants diverge significantly, prefer separate fragments.

### Fragment granularity
For large shared blocks (e.g., the ~100-line implementation workflow), decide during implementation whether to keep as one fragment or split into sub-fragments. Read the source files side-by-side and let the actual content guide the decision. Nested fragment support means sub-fragments can compose cleanly.

### Drift reconciliation
When duplicated text has drifted between files, choose the best version as the canonical fragment content. Don't blindly pick one file's version — compare them and take the clearest, most correct wording.

### Fragment file organization
Follow the established convention: `spec/fragments/<category>/<name>.md`. Proposed categories align with the research clusters: `agents/`, `review/`, `research/`, `plans/`, `github/`.

## Open Questions

1. **Parameterization design**: Some fragments need variables (e.g., `item_type` for feedback-processing, `project_path_label` for implementation-workflow). The exact variable names and defaults should be decided when authoring each fragment — the research document has initial suggestions.

2. **Fragment evaluation**: The research was exploratory. Some candidates may not justify extraction once examined closely (e.g., `agents/file-line-references` is a single line). Each should be evaluated on its merits during implementation.

3. **Ordering**: The research provides a priority ranking by duplication volume and drift risk. Implementation can follow this order but should adapt if dependencies between fragments suggest a different sequence.

## References

- Research document: `thoughts/research/2026-03-07-spec-duplication-fragment-candidates.md`
- Existing fragment example: `spec/fragments/review/prompt-contract.md`
- Compiler fragment docs: `agent-docs/compiler-fragments.md`
- Build command: `npm --prefix agent-config run build:agents`
