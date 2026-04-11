# Convert learn/SKILL.md Knowledge Base Integration to an Agentspec Rule

## Problem Statement

The `learn` skill's "Configuration File Integration" section (lines 189–221 of
`agent-config/spec/skills/learn/SKILL.md`) contains logic that, after writing
learning files, checks whether the user's global agent config references
`learnings-discoverer`. If no reference is found, it offers to inject a
`## Knowledge Base` section directly into the global `AGENTS.md`.

This approach has several problems:

- It is ad-hoc config surgery done at `learn`-time — the agent modifies a
  global configuration file as a side effect of capturing learnings.
- It has no drift detection: if the injected content drifts from the canonical
  text in the skill, `agentspec check` cannot catch it.
- It only targets the Claude global AGENTS.md. Other providers (Cursor, Codex,
  OpenCode) receive no equivalent instruction.
- The canonical text for the Knowledge Base pointer lives in the skill itself
  (lines 207–219) — a prose document — rather than in a versioned, compiled
  spec artifact.

Now that agentspec supports first-class rules (fully implemented as of
2026-03-22), this guidance can be expressed as a generated rule instead.

## Motivation

- **Single source of truth**: The Knowledge Base pointer text lives in one
  canonical place (`spec/rules/knowledge-base.md`) and is compiled to all
  providers by `agentspec compile`.
- **Drift detection**: `agentspec check` catches stale generated files,
  eliminating silent divergence.
- **Multi-provider coverage**: The rule reaches Cursor, Codex, and OpenCode
  in addition to Claude — all with correct formatting.
- **Simpler `learn` skill**: The "Configuration File Integration" section is
  removed entirely — no conditional check, no config-mutation logic, no
  user-facing prompt.
- **Reviewable changes**: When the Knowledge Base guidance needs to evolve,
  a PR to `spec/rules/knowledge-base.md` shows an auditable diff instead of
  an agent silently patching a config file.

## Context

### Current `learn` Skill Behavior

After writing learning files, the skill:

1. Looks up the global agent config path from session context.
2. Greps for `learnings-discoverer` in that file.
3. If absent, offers to append the `## Knowledge Base` block.
4. Requires explicit user approval before writing.

This works but is fragile — it depends on the agent having the right config
path in context, and the text it injects is embedded in the skill prose with
no version control beyond the skill file itself.

### Agentspec Rules Format

A canonical rule file lives at `spec/rules/<id>.md`:

```markdown
---
id: knowledge-base
description: Instruct agents to dispatch learnings-discoverer before starting tasks
compat:
  targets:
    - claude
    - opencode
    - codex
    - cursor
---

## Knowledge Base

Before starting tasks, try to dispatch the `learnings-discoverer` subagent with
a description of the task, technologies, and components involved. ...
```

No `paths:` field → unconditional rule, loaded at every session start.

Compiled output:
- **Claude**: `generated/claude/rules/knowledge-base.md` (no frontmatter — unconditional)
- **Cursor**: `generated/cursor/rules/knowledge-base.mdc` (`alwaysApply: true`)
- **Codex**: `generated/codex/rules/knowledge-base.md`
- **OpenCode**: `generated/opencode/rules/knowledge-base/AGENTS.md`

### What Changes in `learn/SKILL.md`

The "Configuration File Integration" section (lines 189–221) is removed
entirely. The `knowledge-base` rule is deployed as part of the same
skills/agents/rules set — it is always present, so the skill has nothing to
check or offer. The skill no longer reads, greps, or modifies any global
configuration file.

## Goals / Success Criteria

- [ ] New rule at `agent-config/spec/rules/knowledge-base.md` with the
  `learnings-discoverer` Knowledge Base guidance as its body
- [ ] Rule compiles cleanly with `agentspec compile` for all four targets
- [ ] `agentspec check` passes after compilation (no drift)
- [ ] `learn/SKILL.md` "Configuration File Integration" section removed entirely
- [ ] Generated rule files committed to `agent-config/generated/`

## Non-Goals (Out of Scope)

- Changing the guidance text itself — the content of the Knowledge Base block
  stays the same; only its home changes
- Migrating any other skill sections to rules
- Automatically detecting whether the user has run `agentspec compile`

## Proposed Solution

1. Create `agent-config/spec/rules/knowledge-base.md` with the existing
   Knowledge Base text as the rule body (extracted verbatim from learn/SKILL.md
   lines 207–219).
2. Run `agentspec compile` to generate the four provider output files.
3. Edit `learn/SKILL.md`: remove the "Configuration File Integration" section
   entirely.
4. Commit both the new rule spec and the updated skill.

## Constraints

- The Knowledge Base rule must be unconditional (no `paths:`) so it loads at
  every session start.
- The existing text in the rule body should be preserved verbatim to avoid
  changing behavior for existing users who already have the AGENTS.md entry.

## References

- `agent-config/spec/skills/learn/SKILL.md` — current skill, lines 189–221
- `thoughts/ideas/2026-03-22-agentspec-rules-support.md` — rules concept doc
- `thoughts/plans/2026-03-22-agentspec-rules-support.md` — rules implementation
  plan (all items checked — fully implemented)
