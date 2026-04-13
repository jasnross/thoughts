# agentspec: Rules Support

## Problem Statement

agentspec compiles provider-neutral specs into configurations for Claude Code, Cursor, Codex, and
OpenCode. Today it supports two spec types — **agents** and **skills** — but has no concept of
**rules**: path-scoped instruction files that inject persistent context into the AI's window based
on which files are being worked on.

All four target providers have some form of project-rules or instruction mechanism, but the
semantics and file formats differ significantly. Without first-class rules support in agentspec,
teams must maintain provider-specific rule files manually and outside the compile pipeline, losing
the normalization and multi-target benefits that motivated agentspec in the first place.

## Motivation

- **DRY authoring**: Write a rule once (e.g., "API design guidelines for src/api/**") and compile
  it to all providers rather than maintaining `.claude/rules/api.md`, `.cursor/rules/api.mdc`, and
  Codex-specific `src/api/AGENTS.md` separately.
- **Consistent validation**: Rule IDs, provider targets, and path patterns go through the same
  schema and semantic checks as agents and skills.
- **Version-controlled, reviewable output**: `agentspec check` can detect drift between spec source
  and generated rule files, the same as it does for agents and skills today.
- **One compile command**: `agentspec compile` produces the complete configured workspace for all
  providers in one pass.

## Context

### Provider Rules Matrix

| Provider | Format | Location | Path-scoped? | Notes |
|---|---|---|---|---|
| **Claude** | `.md` + optional YAML frontmatter | `.claude/rules/<id>.md` | Yes — `paths:` list of globs; lazy-loaded when matching file opened | `paths:` absent → unconditional (loads at session start) |
| **Cursor** | `.mdc` + YAML frontmatter | `.cursor/rules/<id>.mdc` | Yes — `globs:` field; 4 activation types | Fields: `description`, `globs`, `alwaysApply` |
| **Codex** | Plain markdown | `<directory>/AGENTS.md` | Directory-level only | Walks git root → cwd; one AGENTS.md per directory |
| **OpenCode** | Plain markdown | `<id>/AGENTS.md` + `opencode.json` entry | No (all load at session start) | `opencode.json` `instructions:` glob loads multiple files |

### Canonical → Provider Mapping Summary

- `paths:` absent → Claude unconditional rule; Cursor `alwaysApply: true`; Codex/OpenCode: body always included
- `paths:` present (directory-prefix globs like `src/api/**`) → Claude path-scoped; Cursor `globs:`; Codex: emits `src/api/AGENTS.md`
- `paths:` present (file-extension globs like `**/*.ts`) → Claude/Cursor: native; Codex: warning + falls back to root AGENTS.md; OpenCode: path metadata dropped

### Existing Pipeline

agentspec's compile pipeline runs: Load → Fragment resolution → Schema validation →
Normalization → Preset resolution → Semantic validation → Compile → Emit.

Rules will enter at the **Load** stage alongside agents and skills, and be dispatched to
provider adapters at the **Compile** stage. Because rules have no `tools`, `execution`, or
`user_invocable` fields, a simpler schema section suffices.

## Goals / Success Criteria

- [ ] New `spec/rules/` directory is supported alongside `spec/agents/` and `spec/skills/`
- [ ] Canonical rule frontmatter schema validated: `id`, `description` (optional), `paths`
  (optional list of glob strings), `compat.targets`
- [ ] `agentspec compile` emits Claude rules to `.claude/rules/<id>.md` with correct `paths:` frontmatter
- [ ] `agentspec compile` emits Cursor rules to `.cursor/rules/<id>.mdc` with correct `globs:` /
  `alwaysApply:` / `description:` fields
- [ ] `agentspec compile` emits directory-scoped Codex `AGENTS.md` files; file-extension-only
  globs produce a `MissingDirectoryMapping` warning and fall back to root `AGENTS.md`
- [ ] `agentspec compile` emits OpenCode rules as `<id>/AGENTS.md` files and registers them in
  `opencode.json` under the `instructions` key
- [ ] `agentspec check` detects drift in generated rule files
- [ ] `agentspec validate` catches unknown provider targets and malformed `paths:` patterns
- [ ] Integration tests cover a fixture with at least one unconditional and one path-scoped rule

## Non-Goals (Out of Scope)

- **Codex Starlark execution rules** (command allow/prompt/forbid policies) — those are a
  completely different concept and a separate future idea
- **OpenCode `paths:`-equivalent lazy loading** — OpenCode has no per-file activation trigger;
  path scoping is intentionally dropped
- **Rule priority / ordering** — providers handle load order; agentspec does not attempt to
  control it
- **Merging rules into existing AGENTS.md** — if an `AGENTS.md` already exists in a directory,
  that is an open question (see below)

## Proposed Solution Sketch

A canonical rule file lives at `spec/rules/<id>.md` and looks like:

```markdown
---
id: api-design
description: API design conventions for REST endpoints
paths:
  - "src/api/**"
compat:
  targets:
    - claude
    - cursor
    - codex
    - opencode
---

# API Design Rules

- All endpoints must validate input at the boundary
- Use the standard error response shape from `src/api/errors.ts`
- Include OpenAPI doc comments on every handler
```

Provider output:

- **Claude**: `.claude/rules/api-design.md` with `paths: ["src/api/**"]` frontmatter
- **Cursor**: `.cursor/rules/api-design.mdc` with `globs: ["src/api/**"]` and `description:`
- **Codex**: `src/api/AGENTS.md` containing the rule body
- **OpenCode**: `generated/opencode/rules/api-design/AGENTS.md` + entry in `opencode.json`

## Constraints

- Must not break existing agent and skill compilation
- Schema changes must keep both `schemas/canonical.schema.json` and
  `agent-config/spec/schema/canonical.schema.json` in sync
- Codex directory-scoped output must not silently overwrite any user-authored `AGENTS.md` files
  (the emit strategy needs to handle collisions — likely by emitting to `generated/` and letting
  users symlink or `include` them)

## Open Questions

- **Codex collision handling**: If two rules target overlapping directories (e.g., `src/**` and
  `src/api/**`), should they be merged into one `AGENTS.md` or emitted as separate files? And
  what happens when a user already has a handwritten `AGENTS.md` in that directory?
- **OpenCode `opencode.json` ownership**: Should agentspec own the full `opencode.json` or only
  manage the `instructions` key? Overwriting a user's config would be destructive.
- **Cursor rule type for `description:`-only rules**: Without `paths:` and without
  `alwaysApply: true`, Cursor's "apply intelligently" type requires `description:` to be
  meaningful. Should agentspec require `description:` when `paths:` is absent?
- **`agentspec check` behavior**: Should the check command compare rule output the same way it
  compares agents/skills, or does the directory-distributed Codex output require a different
  diffing strategy?

## References

- [Claude Code rules documentation](https://code.claude.com/docs/en/memory)
- [Cursor rules documentation](https://cursor.com/docs/rules)
- [Codex AGENTS.md documentation](https://developers.openai.com/codex/guides/agents-md)
- [OpenCode rules documentation](https://opencode.ai/docs/rules/)
