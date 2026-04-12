# Unified /research Skill with Dynamic Agent Discovery — Implementation Plan

## Overview

Add an optional `tags` field to all spec types in agentspec, expose it in the
template context for filtering, then use it in the dotfiles agent-config to
create a unified `/research` skill that dynamically discovers research agents.
This spans two repos: `agentspec` (Rust binary) and `dotfiles/agent-config`
(spec library).

## Current State Analysis

### Agentspec Data Model

Frontmatter structs exist in pairs (raw + normalized) for each spec type:
- `AgentFrontmatter` / `NormalizedAgentFrontmatter` (`src/spec.rs:143-158`)
- `SkillFrontmatter` / `NormalizedSkillFrontmatter` (`src/spec.rs:160-179`)
- `RuleFrontmatter` / `NormalizedRuleFrontmatter` (`src/spec.rs:181-192`)

All raw structs use `#[serde(deny_unknown_fields)]`, so adding `tags` to YAML
without updating the struct causes a parse error. The template context exposes
specs via `SpecEntry` (`src/templating/context.rs:7-13`) with only `name`,
`description`, and `type` fields.

### Dotfiles Research Skills

Two research skills exist:
- `research-codebase` — dispatches codebase agents via `sub-agent-catalog` fragment
- `research-web` — hardcodes `web-search-researcher` inline

The `sub-agent-catalog` fragment (`fragments/research/sub-agent-catalog.md`) is
static text with a `{% if include_thoughts_agents %}` toggle. Agent descriptions
already contain "Use when..." guidance suitable for the hybrid catalog approach.

### Key Discoveries

- MiniJinja supports `"value" in list` and for-loop `if` filtering natively
- Normalization functions use explicit field-by-field struct construction — the
  compiler enforces that new fields are handled (won't compile if missed)
- All test helpers (`make_agent`, `make_skill`, `make_rule`) construct
  frontmatter directly and must be updated
- The `sub-agents` rule (`spec/rules/sub-agents.md`) has a static agent table
  that could also benefit from dynamic generation but is out of scope here

## Desired End State

1. All spec types support an optional `tags: Vec<String>` in frontmatter
2. Tags are accessible in templates via `{{ spec.tags }}` and filterable via
   `{% for agent in specs.agents if "research" in agent.tags %}`
3. All research-relevant agents have a `research` tag (plus sub-category tags)
4. The `sub-agent-catalog` fragment dynamically generates its agent list from
   tagged agents
5. A `/research` skill exists that uses all `research`-tagged agents to
   investigate topics across codebase, web, and knowledge base sources

### Verification

- `cargo test` passes in agentspec with the new field
- `agentspec validate` passes against the updated agent-config specs
- `agentspec compile` produces output that includes tags in the template context
- The `/research` skill's compiled output lists all research-tagged agents

## What We're NOT Doing

- Replacing `research-codebase` or `research-web` — they remain for scoped use
- Adding tag-based validation (e.g., enforcing known tag values)
- Updating the `sub-agents` rule to use dynamic generation
- Adding tag-based filtering to adapter output (tags are template-only, not
  emitted into provider config)

## Implementation Approach

Three phases, each independently shippable. Phase 1 is an agentspec release;
Phases 2 and 3 are dotfiles changes that depend on Phase 1 being installed.

---

## Phase 1: Agentspec — Add `tags` to the Data Model

### Overview

Add an optional `tags: Vec<String>` field to all three spec types' frontmatter,
pass it through normalization, and expose it in `SpecEntry` for template access.

### Changes Required

#### 1. Frontmatter Structs

**File**: `src/spec.rs`

- [x] Add `pub tags: Option<Vec<String>>` to `AgentFrontmatter` (after `description`, before `execution`)
- [x] Add `pub tags: Option<Vec<String>>` to `NormalizedAgentFrontmatter` (same position)
- [x] Add `pub tags: Option<Vec<String>>` to `SkillFrontmatter` (after `description`, before `user_invocable`)
- [x] Add `pub tags: Option<Vec<String>>` to `NormalizedSkillFrontmatter` (same position)
- [x] Add `pub tags: Option<Vec<String>>` to `RuleFrontmatter` (after `description`)
- [x] Add `pub tags: Option<Vec<String>>` to `NormalizedRuleFrontmatter` (same position)

The field is `Option<Vec<String>>` because it must be optional — existing specs
that don't use tags should continue to parse without it. When absent, it
deserializes as `None` via serde defaults.

#### 2. NormalizedSpec Accessor

**File**: `src/spec.rs`

- [x] Add a `tags()` accessor to `NormalizedSpec` (alongside `id()`, `description()`, `spec_type()`)

```rust
pub fn tags(&self) -> &[String] {
    match self {
        NormalizedSpec::Agent(s) => s.frontmatter.tags.as_deref().unwrap_or_default(),
        NormalizedSpec::Skill(s) => s.frontmatter.tags.as_deref().unwrap_or_default(),
        NormalizedSpec::Rule(s) => s.frontmatter.tags.as_deref().unwrap_or_default(),
    }
}
```

#### 3. Normalization

**File**: `src/validate.rs`

- [x] Add `tags: spec.frontmatter.tags` to `normalize_agent_spec` (line ~43)
- [x] Add `tags: spec.frontmatter.tags` to `normalize_skill_spec` (line ~57)
- [x] Add `tags: spec.frontmatter.tags` to `normalize_rule_spec` (line ~74)

#### 4. Template Context

**File**: `src/templating/context.rs`

- [x] Add `pub tags: Vec<String>` to `SpecEntry` (after `r#type`)
- [x] Populate it in `from_specs`: `tags: spec.tags().to_vec()`

The `SpecEntry.tags` field is `Vec<String>` (not `Option`) for ergonomic
template access — `None` tags become an empty vec, so `"research" in agent.tags`
returns false rather than erroring on a null.

#### Tests

**File**: `src/spec.rs` (inline tests if accessor is added)

- [x] Test that `tags()` returns the tag list for a spec with tags
  > **Deviation:** Covered by `test_tags_exposed_in_spec_entry` in context.rs
  > and `test_normalize_agent_tags` in validate.rs rather than a separate test
  > module in spec.rs (which had no existing test module).
- [x] Test that `tags()` returns an empty slice for a spec without tags (`None`)
  > **Deviation:** Covered by `test_no_tags_produces_empty_vec` in context.rs
  > and `test_normalize_agent_no_tags` in validate.rs.

**File**: `src/validate.rs`

- [x] Update `make_agent` helper to include `tags: None`
- [x] Update `make_skill` helper to include `tags: None`
- [x] Update `make_rule` helper to include `tags: None`
- [x] Add `test_normalize_agent_tags` — verify tags pass through normalization
- [x] Add `test_normalize_agent_no_tags` — verify `None` tags normalize correctly

**File**: `src/templating/context.rs`

- [x] Update `make_agent` helper to include `tags: None`
- [x] Update `make_skill` helper to include `tags: None`
- [x] Update `make_rule` helper to include `tags: None`
- [x] Add `test_tags_exposed_in_spec_entry` — verify tagged specs produce non-empty `tags` vec
- [x] Add `test_no_tags_produces_empty_vec` — verify untagged specs produce empty `tags` vec

**File**: `src/specs.rs` (load tests)

- [x] Update `test_load_agent_specs` to verify tags field loads from YAML
  > **Deviation:** Added as a new test `test_load_agent_specs_with_tags` rather
  > than modifying the existing test, to keep tests focused.
- [x] Add a test that loads an agent spec with `tags: [research, codebase]` and
      verifies the parsed frontmatter

**File**: `tests/fixtures/agent-config/spec/agents/test-agent.md`

- [x] Add `tags: [test]` to the fixture agent's frontmatter (exercises the field
      end-to-end in the integration test)

**File**: Adapter tests in `src/adapters/claude.rs`, `cursor.rs`, `opencode.rs`

- [x] Update all `make_agent`/`make_skill`/`make_rule` helpers in adapter test
      modules to include `tags: None`

**File**: `src/templating/fragments.rs` (not in original plan)

- [x] Update all frontmatter struct constructions in fragment tests to include `tags: None`
  > **Deviation:** The plan did not list fragments.rs, but the compiler flagged
  > 12 struct constructions in its test module that also needed `tags: None`.

### Success Criteria

#### Automated Verification

- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes (all unit + integration tests — 122 tests, 0 failures)
- [x] An agent spec with `tags: [research, web]` parses without error
  > Verified by `test_load_agent_specs_with_tags`
- [x] An agent spec without `tags` still parses without error
  > Verified by existing `test_load_agent_specs` (unchanged frontmatter, still passes)
- [x] The integration test in `tests/pipeline.rs` passes (16 tests)

---

## Phase 2: Dotfiles — Tag Agents and Update Sub-Agent Catalog

### Overview

Add `tags` to all agent frontmatter in the dotfiles spec, then rewrite the
`sub-agent-catalog` fragment to dynamically generate its agent list from
`research`-tagged agents.

### Changes Required

#### 1. Tag Existing Agents

**File**: `spec/agents/codebase-locator.md`
- [x] Add `tags: [research, codebase]` to frontmatter

**File**: `spec/agents/codebase-analyzer.md`
- [x] Add `tags: [research, codebase]` to frontmatter

**File**: `spec/agents/codebase-pattern-finder.md`
- [x] Add `tags: [research, codebase]` to frontmatter

**File**: `spec/agents/web-search-researcher.md`
- [x] Add `tags: [research, web]` to frontmatter

**File**: `spec/agents/thoughts-locator.md`
- [x] Add `tags: [research, knowledge-base]` to frontmatter

**File**: `spec/agents/thoughts-analyzer.md`
- [x] Add `tags: [research, knowledge-base]` to frontmatter

**File**: `spec/agents/learnings-discoverer.md`
- [x] Add `tags: [research, knowledge-base]` to frontmatter

**File**: `spec/agents/code-reviewer.md`
- [x] No changes — no tags (or add non-research tags if appropriate later)

#### 2. Rewrite Sub-Agent Catalog Fragment

**File**: `spec/fragments/research/sub-agent-catalog.md`

- [x] Replace the static agent list with a dynamic tag-based listing
- [x] Remove the `include_thoughts_agents` conditional (no longer needed —
      knowledge-base agents are included if they have the `research` tag)

**Behavioral change**: The current fragment restricts web research to "only if
user explicitly asks" and hides thoughts agents behind an `include_thoughts_agents`
toggle that no caller activates. The dynamic version shows all `research`-tagged
agents unconditionally. This is intentional — the calling skill (not the catalog)
should decide which agents to use. If `research-codebase` needs to suppress web
agents, it can use its own `{% for %}` loop with a `"codebase" in agent.tags`
filter instead of including the catalog.

Proposed content:

```markdown
{% for agent in specs.agents if "research" in agent.tags %}
- **{{ agent.name }}**: {{ agent.description }}
{% endfor %}

The key is to use these agents intelligently:

- Start with locator agents to find what exists
- Then use analyzer agents on the most promising findings to document how they work
- Run multiple agents in parallel when they're searching for different things
- Each agent knows its job — just tell it what you're looking for
- Don't write detailed prompts about HOW to search — the agents already know
- Remind agents they are documenting, not evaluating or improving
```

#### 3. Update Skills That Include the Fragment

The `sub-agent-catalog` fragment no longer uses the `include_thoughts_agents`
variable, but callers that pass it via `{% with %}` won't break (MiniJinja
ignores unused variables). However, we should clean up the call sites.

**File**: `spec/skills/research-codebase/SKILL.md`

- [x] Verify the `{% include "research/sub-agent-catalog.md" %}` invocation
      still works (the indent filter is unchanged; the fragment just generates
      different content)

**File**: `spec/skills/iterate-research/SKILL.md`

- [x] Verify the `{% include "research/sub-agent-catalog.md" %}` invocation
      still works

#### Tests

- [x] Run `agentspec validate` against the updated spec — all specs should pass
- [x] Run `agentspec compile` and inspect the compiled `research-codebase` skill
      output to verify the dynamic agent list renders correctly
- [x] Verify the compiled output includes all 7 research-tagged agents in the
      catalog section
- [x] Verify `code-reviewer` does NOT appear in the catalog

### Success Criteria

#### Automated Verification

- [x] `agentspec validate` passes with no errors
- [x] `agentspec compile` completes successfully (154 files for 3 providers)

#### Manual Verification

- [x] Inspect compiled `research-codebase` skill output: the sub-agent-catalog
      section lists all 7 research agents dynamically (manual-only: requires
      reading generated output to confirm content correctness)

---

## Phase 3: Dotfiles — Create the `/research` Skill

### Overview

Create a new `/research` skill that combines codebase and web research
capabilities, using the dynamic agent catalog to dispatch all available research
agents based on the topic.

### Changes Required

#### 1. Create Skill Directory and File

**File**: `spec/skills/research/SKILL.md`

- [x] Create the skill with frontmatter:

```yaml
---
id: research
description: Research a topic using all available methods — codebase, web, and knowledge base
user_invocable: true
agent_invocable: true
execution:
  preset: deep_review
capabilities:
  tools:
    - bash
    - edit
    - glob
    - grep
    - read
    - tasks
    - webfetch
    - websearch
    - write
---
```

- [x] Write the skill body combining the best of both `research-codebase` and
      `research-web`:

The skill body should follow this structure:

1. **Role statement**: "You are tasked with conducting comprehensive research
   using all available methods — codebase analysis, web sources, and knowledge
   base — to answer user questions."

2. **Critical stance**: Document and explain, don't evaluate. Same as
   `research-codebase` but extended: "For codebase findings, document as-is
   without suggesting improvements. For web findings, attribute to sources and
   note recency."

3. **Initial setup**: Same prompt pattern as existing skills — ready message,
   wait for query, or proceed if argument provided.

4. **$THOUGHTS_DIR discovery**: `{% with writeable = true %}{% include "thoughts/discover-dir.md" %}{% endwith %}`

5. **Research steps**:
   - Read project guidelines (AGENTS.md, CLAUDE.md) for each area explored
   - Read any directly mentioned files fully before spawning agents
   - Analyze and decompose the research question — decide which research
     methods are appropriate (codebase, web, knowledge base, or all)
   - Spawn parallel sub-agents using the dynamic catalog:
     `{% include "research/sub-agent-catalog.md" %}`
   - Additional guidance for the unified skill:
     - For codebase questions: prefer codebase agents first
     - For external/ecosystem questions: prefer web agents first
     - For cross-domain questions: dispatch both in parallel
     - Always check the knowledge base (thoughts/learnings) for prior research
   - Wait for all agents and synthesize

6. **Output format**: A merged format that includes git metadata when codebase
   research was performed, and source URLs when web research was performed:

   ```markdown
   ---
   date: [ISO datetime]
   git_commit: [if codebase research was performed]
   branch: [if codebase research was performed]
   repository: [if codebase research was performed]
   topic: "[Research question]"
   tags: [research, relevant-tags]
   research_methods: [codebase, web, knowledge-base]  # which methods were used
   status: complete
   last_updated: [YYYY-MM-DD]
   last_updated_by: assistant
   ---
   ```

   Sections: Research Question, Summary, Detailed Findings (with file:line
   refs and/or source URLs as appropriate), Sources (if web research), Code
   References (if codebase research), Open Questions.

7. **GitHub permalinks**: Same as `research-codebase` (if applicable).

8. **Follow-up handling**: Same append pattern as existing skills.

9. **Document conventions**: `{% with include_git_commit_example = true %}{% include "research/document-conventions.md" %}{% endwith %}`

#### Tests

- [x] Run `agentspec validate` — the new skill should pass validation
- [x] Run `agentspec compile` — verify the skill compiles for all providers (158 files)
- [x] Inspect compiled output: the skill body should contain the dynamically
      generated agent list from the sub-agent-catalog fragment

### Success Criteria

#### Automated Verification

- [x] `agentspec validate` passes with no errors
- [x] `agentspec compile` completes successfully (158 files for 3 providers)
- [x] The compiled skill output file exists at the expected path for each
      provider

#### Manual Verification

- [ ] Invoke `/research` in a Claude Code session and verify it responds with
      the ready message (manual-only: requires interactive Claude Code session)
- [ ] Run a test research query that spans codebase and web (e.g., "How does
      agentspec's MiniJinja integration compare to other template engines?") and
      verify it dispatches both codebase and web agents (manual-only: requires
      live agent execution)

---

## References

- Idea document: `$THOUGHTS_DIR/ideas/2026-04-11-unified-research-skill.md`
- Agentspec template context: `agentspec/src/templating/context.rs`
- Agentspec spec model: `agentspec/src/spec.rs`
- Agentspec normalization: `agentspec/src/validate.rs`
- Existing skills: `dotfiles/agent-config/spec/skills/research-codebase/`, `spec/skills/research-web/`
- Sub-agent catalog fragment: `dotfiles/agent-config/spec/fragments/research/sub-agent-catalog.md`
- MiniJinja docs: https://docs.rs/minijinja/latest/minijinja/
