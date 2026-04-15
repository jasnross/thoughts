# Agentspec: Prefix-Aware Spec References in Body Content

## Problem Statement

Agentspec specs frequently reference other specs by name in their body content — for example, instructing the model to "Load the 'gh-safe' skill" or listing available agents with `subagent_type: "code-reviewer"`. These references are hardcoded to the canonical (unprefixed) spec ID, but when `agentspec sync` is configured with a prefix (e.g., `tw`), the output artifacts use prefixed names like `tw-gh-safe` and `tw-code-reviewer`.

This creates a mismatch: the AI model sees an instruction to load `gh-safe`, but the actual skill is named `tw-gh-safe`. The model either fails to find the skill or the spec author must manually hardcode the prefix — defeating the purpose of a configurable prefix system.

## Motivation

- **Correctness**: Cross-spec references in output must match the actual names produced by sync. A reference that doesn't resolve is a silent bug that degrades model behavior.
- **Prefix independence**: Spec authors should write references using canonical IDs and have the correct prefixed name appear in output automatically. Changing the prefix in `agentspec.toml` should not require editing every spec body.
- **No new syntax**: The existing MiniJinja template system already renders spec bodies. Extending the template context to include prefix-aware names leverages infrastructure that's already in place (or planned — see built-in template variables idea).
- **Free validation**: MiniJinja's undefined behavior means referencing a nonexistent spec via template syntax (e.g., `{{ specs.skills.nonexistent.name }}`) will error at render time. No custom validation pass is needed — the template engine enforces referential integrity.

## Context

### Current State

- Specs reference other specs as plain text in body content: `Load the 'gh-safe' skill`
- The prefix is configured per-provider in `agentspec.toml` (e.g., `prefix = "tw"`)
- Adapters apply the prefix to output file paths and frontmatter `name` fields, but **not** to body content
- The MiniJinja template environment renders spec bodies before compilation, but currently with no spec metadata in the context
- The [built-in template variables idea](2026-04-10-agentspec-built-in-template-variables.md) proposes a `{{ specs }}` context exposing spec metadata — but with unprefixed canonical IDs as the `name` field
- The [tool name resolver idea](2026-04-09-agentspec-tool-name-resolver.md) addresses a similar problem for provider tool names (e.g., `{{ tool("Question") }}` → `AskUserQuestion`)

### Scope of Cross-References in Current Specs

Extensive cross-references exist across the dotfiles spec library:

- **Skill → skill**: `gh-safe`, `git-town`, `thoughts-dir-resolver`, `resolve-conflicts`, `discover-repo` (referenced from 10+ locations)
- **Rule → agent**: `learnings-discoverer`, `code-reviewer` (referenced from sub-agents rule, knowledge-base rule)
- **Skill → agent**: `code-reviewer` (referenced from review-pr, code-review-loop, review-files skills)
- **Fragment → skill**: `thoughts-dir-resolver` (referenced from discover-dir, discover-root fragments)
- **Rule → skill**: `gh-safe` (referenced from git-conventions rule — currently hardcoded as `tw:gh-safe`)
- **Dynamic listing**: `sub-agents.md` rule already uses `{{ agent.name }}` via MiniJinja to list agents, but the template context exposes unprefixed IDs

### Pipeline Architecture

Template resolution currently happens **once, before compilation** — before a target provider (and its prefix) is known:

```
ValidatedSpecs → templating::resolve() → ResolvedSpecs → compile (per-provider)
```

For prefix-aware references, template resolution needs access to the prefix. This means either:
1. Moving template resolution into the per-provider compile loop
2. Running template resolution once per provider
3. Using a placeholder that adapters resolve during compilation

This is the same architectural constraint identified in the tool name resolver idea — both features need provider context at template render time.

## Goals / Success Criteria

- [ ] Spec authors can reference other specs by canonical ID using MiniJinja syntax, and the output contains the correct prefixed name for each target provider
- [ ] Referencing a nonexistent spec ID produces a clear compile-time error (leveraging MiniJinja's undefined behavior)
- [ ] Changing the prefix in config automatically updates all spec references in output — no spec body edits required
- [ ] Works correctly when prefix is absent (references resolve to unprefixed canonical IDs)
- [ ] Works correctly across spec types (skill referencing an agent, rule referencing a skill, etc.)
- [ ] Existing specs using plain-text references continue to compile (no breaking change) — migration to template syntax is incremental
- [ ] The template context for spec references aligns with / extends the built-in template variables design

## Non-Goals (Out of Scope)

- **Custom MiniJinja macros or functions** (e.g., `{{ skill_name("gh-safe") }}`) — may be added later for ergonomics, but the first version uses the `{{ specs }}` context directly
- **Validating plain-text references** — only template-based references (`{{ specs.skills.gh_safe.name }}`) are validated; plain-text like `'gh-safe'` is an authoring error agentspec can't catch
- **Provider-specific tool name resolution** — covered by the separate [tool name resolver idea](2026-04-09-agentspec-tool-name-resolver.md)
- **Cross-library references** — referencing specs from a different spec library / agentspec project

## Constraints

- **Template resolution timing**: Currently runs once before compilation. Prefix-aware resolution requires provider context, necessitating a pipeline change. This constraint is shared with the tool name resolver idea — solving it for one likely solves it for both.
- **Per-provider prefixes**: Prefixes can differ between providers (e.g., `tw` for Claude, something else for Cursor). The template context must reflect the correct prefix for the provider being compiled.
- **MiniJinja `Lenient` mode**: Currently set for optional boolean flags. Chained attribute access on undefined values (`specs.skills.nonexistent.name`) already errors under `Lenient`, so validation works without switching to `Strict`. However, a simple `{{ specs.skills.nonexistent }}` (no `.name` access) would silently resolve to empty under `Lenient` — worth documenting this footgun.
- **Canonical ID as context key**: Template syntax uses dot-access, so canonical IDs with hyphens (e.g., `gh-safe`) need a key format compatible with MiniJinja attribute access. MiniJinja supports bracket notation (`specs.skills["gh-safe"].name`) or the IDs could be normalized to underscores (`specs.skills.gh_safe.name`).

## Resolved Questions

- **Key format for hyphenated IDs**: Use underscore-normalized keys — `specs.skills.gh_safe.name` rather than `specs.skills["gh-safe"].name`. The hyphen-to-underscore mapping is a common convention and more ergonomic. To prevent subtle collisions, validation must error if two specs within the same type normalize to the same key (e.g., both `gh-safe` and `gh_safe` exist).
- **Pipeline restructuring scope**: Bundle the pipeline refactor and feature in one plan, with the refactor as phase 1. The feature phases validate that the refactor supports the use case. The refactor itself is bounded: move template resolution to run per-provider (so it has access to `AdapterConfig`/prefix), update typestate types, verify tests pass. This also unblocks the tool name resolver idea.
- **Fragment references**: Resolved — fragments share the same MiniJinja render context (same `Environment`, same `ctx` value passed to `template.render()`). They automatically get `{{ specs }}` access. No special handling needed.
- **Adapter-specific name formats**: `.name` returns the **model-facing name** — whatever the AI model needs to use to invoke the spec for the provider being compiled. This varies by provider and spec type:
  - Claude agents/skills: `tw-gh-safe` (prefixed)
  - OpenCode commands: `gh-safe` (unprefixed, prefix is a subdirectory)
  - OpenCode skills: `gh-safe` (unprefixed in frontmatter)
  - Cursor agents/skills: `tw-gh-safe` (prefixed)
  
  Each adapter already knows its name format — they expose this as a function the template context can call during per-provider rendering.

## References

- Built-in template variables idea: `thoughts/ideas/2026-04-10-agentspec-built-in-template-variables.md`
- Tool name resolver idea: `thoughts/ideas/2026-04-09-agentspec-tool-name-resolver.md`
- Prefix configuration: `agentspec/src/config.rs:247-262`
- Template environment setup: `agentspec/src/templating/fragments.rs:82-94`
- Template context (current, unprefixed): `agentspec/src/templating/context.rs:36-74`
- Claude adapter prefix application: `agentspec/src/adapters/claude.rs:117-123, 173-174, 208-209`
- Cursor adapter prefix application: `agentspec/src/adapters/cursor.rs:65-72, 94-97, 145-146`
- OpenCode adapter prefix application: `agentspec/src/adapters/opencode.rs:99-103, 142-147, 194-196`
- MiniJinja UndefinedBehavior: `https://docs.rs/minijinja/latest/minijinja/enum.UndefinedBehavior.html`

## Affected Systems/Services

- `agentspec` compiler — template resolution stage (needs provider/prefix context), template context construction
- All provider adapters — must expose how they format prefixed names for each spec type
- Existing specs — optional, incremental migration from plain-text references to `{{ specs.*.name }}` syntax
