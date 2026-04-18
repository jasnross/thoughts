# Triage TODOs Skill

## Problem Statement

There is no systematic way to discover, prioritize, and act on TODO items
scattered across a project. TODOs accumulate in two places — dedicated TODO.md
files and inline `TODO` comments in code — but nothing connects them to the
existing idea/plan workflow. Without a triage step, TODOs either rot in place or
get addressed ad hoc without considering related items that could be tackled
together.

## Motivation

- **TODOs decay**: The longer a TODO sits untouched, the more context is lost
  about why it was added and what it means. A triage skill turns stale TODOs into
  actionable artifacts before that context disappears.
- **Related TODOs should be grouped**: Addressing TODOs one at a time misses
  opportunities to batch related changes. Grouping them into a single idea or
  plan reduces churn and produces more coherent changes.
- **Ambiguity varies**: Some TODOs are clear instructions ("TODO: add index on
  `users.email`") while others are vague aspirations ("TODO: clean this up").
  Prioritizing unambiguous TODOs first maximizes throughput — they can go straight
  to a plan without a research/ideation phase.
- **Reuses existing workflow**: Rather than inventing its own document format,
  the skill delegates to `/tw-create-idea` or `/tw-create-plan`, keeping the
  pipeline consistent.

## Context

The `agent-config/spec/skills/` directory contains existing skills for creating
ideas (`create-idea`) and plans (`create-plan`). Both are `user_invocable` and
`agent_invocable`, and both produce documents in `$THOUGHTS_DIR`. The new
`triage-todos` skill sits upstream of these — it discovers and organizes the raw
material, then hands off to the appropriate downstream skill.

Skills reference each other via `{{ specs.skill.<id>.name }}` in MiniJinja
templates and can load shared fragments from `spec/fragments/`.

## Goals / Success Criteria

- [ ] Skill scans both TODO.md files and inline `TODO` comments in code, merging
      results into a unified list
- [ ] Related TODOs are grouped (e.g., by file proximity, shared topic, or
      explicit cross-references)
- [ ] Groups are ranked by ambiguity — unambiguous groups with clear context and
      solutions are presented first
- [ ] The user selects which group to tackle from the ranked list
- [ ] Unambiguous groups route directly to `/tw-create-plan`; ambiguous groups
      route to `/tw-create-idea`
- [ ] The skill explains its routing decision before proceeding
- [ ] The skill accepts an optional directory argument (defaults to CWD)
- [ ] The skill delegates all document creation to the downstream skill — it does
      not invent its own idea/plan format

## Non-Goals (Out of Scope)

- **Resolving TODOs**: The skill triages and routes; it does not fix code or
  remove TODO comments
- **Persistent TODO tracking**: No database, no state file — each invocation is
  a fresh scan
- **Cross-project scanning**: The skill operates within a single project
  directory tree
- **Custom TODO syntax**: The skill handles standard `TODO` and `TODO:` markers
  plus TODO.md files; it does not support arbitrary tag formats (FIXME, HACK,
  XXX) unless we decide to expand scope later

## Proposed Solutions

### Workflow

1. **Scan**: Grep for `TODO` comments in code and read any TODO.md files in the
   target directory
2. **Parse**: Extract each TODO item with its surrounding context (file, line,
   nearby code, any description)
3. **Group**: Cluster related TODOs by proximity (same file/module), shared
   topic, or explicit references to each other
4. **Rank**: Score each group by ambiguity:
   - **Low ambiguity** (high priority): Clear action, specific location, obvious
     solution
   - **Medium ambiguity**: Clear problem but multiple possible approaches
   - **High ambiguity** (lower priority): Vague, lacking context, or requiring
     research
5. **Present**: Show the ranked groups to the user via the selection UI
6. **Route**: Based on the selected group's ambiguity:
   - Low ambiguity → load `/tw-create-plan` with the TODO context
   - Medium/High ambiguity → load `/tw-create-idea` with the TODO context
7. **Explain**: Before loading the downstream skill, explain why this group is
   being routed to idea vs. plan

### Ambiguity Heuristics

Factors that reduce ambiguity (make a TODO more actionable):

- Specific file/function referenced
- Concrete action verb ("add", "remove", "rename", "extract")
- Related code is nearby and readable
- Small, well-scoped change

Factors that increase ambiguity:

- Vague language ("clean up", "improve", "refactor")
- No surrounding context
- Touches multiple systems or has unclear boundaries
- Requires design decisions or research

## Constraints

- Must follow the `agent-config/spec/` skill spec format with YAML frontmatter
- Must use `{{ specs.skill.create_idea.name }}` and
  `{{ specs.skill.create_plan.name }}` for cross-skill references
- Should use the `thoughts/discover-dir.md` fragment for `$THOUGHTS_DIR`
  resolution
- The skill needs code search tools (`grep`, `glob`, `read`) to scan for TODOs,
  plus the ability to invoke downstream skills

## Resolved Questions

- **Marker scope**: TODO only for v1. FIXME, HACK, and XXX have different
  semantics (bugs, shortcuts, unknowns) and would add noise. Can expand later
  if needed.
- **Grouping strategy**: Semantic-first. Group primarily by topic/intent (shared
  keywords, referenced components, described feature). Use file proximity as a
  secondary signal/tiebreaker. This ensures cross-file TODOs about the same
  feature are grouped together.
- **State tracking**: No tracking. Each invocation is a fresh scan. The skill is
  stateless — once a TODO is resolved and removed from code, it naturally
  disappears from future scans. The user can skip already-triaged items manually.

## References

- Existing skill specs: `agent-config/spec/skills/create-idea/SKILL.md`,
  `agent-config/spec/skills/create-plan/SKILL.md`
- Fragment for THOUGHTS_DIR discovery: `spec/fragments/thoughts/discover-dir.md`
- MiniJinja cross-skill reference pattern: `{{ specs.skill.<id>.name }}`
