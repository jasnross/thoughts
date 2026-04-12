# Unified /research Skill with Dynamic Agent Discovery

## Problem Statement

The current research workflow requires the user to choose between `/research-codebase`
and `/research-web` up front, even when a topic spans both domains. Many research
questions naturally cross boundaries — understanding how a library works requires
reading the codebase *and* checking external docs, or investigating a bug means
searching code *and* looking up known issues online. Forcing users to pick a lane (or
run both skills separately and mentally merge the results) adds friction and produces
fragmented research output.

Additionally, on different machines the user has different research agents available
(e.g., additional specialized agents beyond the core set). The existing skills
hard-code their agent lists, so adding a new research agent on one machine doesn't
automatically make it available to the research skills.

## Motivation

- **Reduce friction:** A single `/research` entry point that automatically decides
  which research methods to use based on the topic eliminates the "which skill do I
  use?" decision.
- **Better research quality:** Cross-domain topics get a unified investigation rather
  than two separate partial ones. The skill can correlate findings from code and web
  sources in a single synthesis.
- **Portability across machines:** Different environments have different agents
  available. A dynamic agent discovery mechanism means the skill automatically
  incorporates whatever research agents are present — no manual updates needed.
- **Extensibility:** Future research agents (e.g., a database-schema-researcher or an
  API-spec-analyzer) automatically become available to the `/research` skill when
  tagged appropriately.

## Context

### Current Architecture

- **Skills:** `research-codebase`, `research-web`, and `iterate-research` are
  user-facing entry points that dispatch sub-agents for focused research tasks.
- **Agents:** Seven agents exist, of which six are research-relevant (codebase-locator,
  codebase-analyzer, codebase-pattern-finder, web-search-researcher,
  thoughts-locator, thoughts-analyzer, learnings-discoverer).
- **Fragment:** A shared `sub-agent-catalog` fragment lists available agents, but it's
  static text — not dynamically generated from the agent definitions.
- **Sub-agents rule:** `spec/rules/sub-agents.md` provides a behavioral guideline table
  listing all agents, but it's a generic rule visible to all skills, not scoped to
  research.

### Agentspec Capabilities

- **Templates:** Spec bodies use MiniJinja templates with `{% include %}` for fragments.
- **Spec context (recently added):** Templates can access `{{ specs.agents }}` to
  iterate over all agents with their `name`, `description`, and `type` fields.
- **No categorization:** Agent frontmatter currently has no tags, categories, or other
  grouping mechanism. The `specs.agents` list is flat and unfiltered.

## Goals / Success Criteria

- [ ] A `/research` skill exists that automatically dispatches appropriate research
      agents (codebase, web, knowledge base) based on the research topic
- [ ] The skill dynamically discovers available research agents via a `tags` field in
      agent frontmatter, rather than hard-coding agent names
- [ ] Agentspec's data model supports a `tags` field on agent specs, exposed in the
      template context for filtering (e.g.,
      `{% for agent in specs.agents if "research" in agent.tags %}`)
- [ ] The skill produces output in the same format and location
      (`$THOUGHTS_DIR/research/`) as existing research skills, compatible with
      `iterate-research`
- [ ] Adding a new research agent on any machine (with the appropriate tag) makes it
      automatically available to the `/research` skill without modifying the skill
      template

## Non-Goals (Out of Scope)

- Replacing the existing `research-codebase` and `research-web` skills — they remain
  useful for scoped research when the user knows what they want
- Adding tags to skills or rules — agent tags are the initial scope; other spec types
  can be considered later if needed
- User-directed agent selection (e.g., "web only") — the skill decides automatically;
  users who want scoped research can use the existing specialized skills
- Changing the research output format or storage location

## Proposed Solutions

### Agent Tagging in Agentspec

Add an optional `tags` field to agent frontmatter:

```yaml
---
id: web-search-researcher
description: Researches topics using web search and URL fetching.
tags:
  - research
  - web
execution:
  preset: balanced
capabilities:
  tools:
    - tasks
    - webfetch
    - websearch
---
```

Expose tags in the template context:

```rust
pub struct SpecEntry {
    pub name: String,
    pub description: String,
    pub r#type: String,
    pub tags: Vec<String>,  // NEW
}
```

This enables template filtering:

```markdown
{% for agent in specs.agents if "research" in agent.tags %}
- **{{ agent.name }}**: {{ agent.description }}
{% endfor %}
```

### Skill Template

The `/research` skill template would use the tag-filtered agent list to dynamically
build its sub-agent dispatch instructions. The skill body would include logic like:

```markdown
## Available Research Agents

{% for agent in specs.agents if "research" in agent.tags %}
- **{{ agent.name }}**: {{ agent.description }}
{% endfor %}

Analyze the research question and dispatch the appropriate agents above...
```

This replaces the current static `sub-agent-catalog` fragment with a dynamic,
tag-driven listing.

## Constraints

- The `tags` field must be optional to avoid breaking existing specs that don't use it
- The skill template must degrade gracefully if no agents have the `research` tag
  (e.g., on a minimal setup)

## Resolved Questions

### MiniJinja `in` operator
**Confirmed working.** MiniJinja supports `"value" in list` and for-loop `if` filtering
natively. `{% for agent in specs.agents if "research" in agent.tags %}` works out of the
box with no custom filters needed.

### Tag assignments for existing agents
Agreed upon:
- `codebase-locator`: `[research, codebase]`
- `codebase-analyzer`: `[research, codebase]`
- `codebase-pattern-finder`: `[research, codebase]`
- `web-search-researcher`: `[research, web]`
- `thoughts-locator`: `[research, knowledge-base]`
- `thoughts-analyzer`: `[research, knowledge-base]`
- `learnings-discoverer`: `[research, knowledge-base]`
- `code-reviewer`: *(no tags)*

### Sub-agent-catalog: hybrid approach
The `sub-agent-catalog` fragment will switch to **dynamic tag-based generation** for the
agent list, but usage guidance (e.g., "find WHERE code lives") will be co-located in
each agent's `description` field rather than hardcoded in the fragment. This keeps the
catalog always in sync with available agents while preserving actionable guidance.

### Tags scope: all spec types
Tags will be added to **all spec types** (agents, skills, and rules) for a consistent
data model, even though only agent tags have an immediate use case.

## Open Questions

*All questions resolved — ready for planning.*

## References

- Existing skills: `spec/skills/research-codebase/`, `spec/skills/research-web/`
- Sub-agent catalog fragment: `spec/fragments/research/sub-agent-catalog.md`
- Agentspec template context: `agentspec/src/templating/context.rs`
- Agentspec spec model: `agentspec/src/spec.rs`
- MiniJinja docs: https://docs.rs/minijinja/latest/minijinja/

## Affected Systems/Services

- **agentspec** (Rust binary): Add `tags` to frontmatter schema, normalization,
  validation, and template context
- **dotfiles/agent-config/spec/**: Add tags to agent frontmatter, create new
  `/research` skill template, potentially update `sub-agent-catalog` fragment
