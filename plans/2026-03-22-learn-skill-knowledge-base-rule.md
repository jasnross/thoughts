# learn Skill: Migrate Knowledge Base Pointer to agentspec Rule

## Overview

The `learn` skill currently embeds Knowledge Base injection logic (lines
189–221 of `spec/skills/learn/SKILL.md`): after writing learnings it checks
whether the global `AGENTS.md` references `learnings-discoverer` and, if not,
offers to inject a `## Knowledge Base` block. This is fragile ad-hoc config
surgery with no drift detection and no multi-provider coverage.

This plan replaces that logic with a first-class agentspec rule at
`agent-config/spec/rules/knowledge-base.md`, compiled to all four providers by
`agentspec compile`. The `learn` skill's "Configuration File Integration"
section is then removed entirely.

## Current State Analysis

- **Rule spec directory**: `agent-config/spec/rules/` does **not yet exist**.
  The compiler supports it (fully implemented as of 2026-03-22), but no rules
  have been authored for the dotfiles agent-config yet.
- **learn/SKILL.md**: The "Configuration File Integration" section occupies
  lines 189–221. The Knowledge Base guidance text is lines 209–218. The
  "Important Guidelines" bullet at line 235 also references config-file mutation
  and must be cleaned up.
- **Generated learn outputs**: Four provider-specific files under
  `agent-config/generated/` are regenerated every `agentspec compile` run.
  Editing `learn/SKILL.md` and recompiling will update them automatically.
- **Integration tests** (`agentspec/tests/pipeline.rs`): Exercise the fixture
  directory only; spec counts there (`loaded 5 specs`) are unaffected by changes
  to `agent-config/`.
- **Conventional Commits** required for agentspec-area commits (per
  `agentspec/CLAUDE.md`). `cargo fmt` must be run before any commit touching
  `agentspec/` source.

## Desired End State

A single canonical rule file at `agent-config/spec/rules/knowledge-base.md`
contains the Knowledge Base guidance verbatim. Running `agentspec compile`
produces:

| Target   | Output path                                                    |
|----------|----------------------------------------------------------------|
| Claude   | `generated/claude/rules/knowledge-base.md` (no frontmatter)   |
| Cursor   | `generated/cursor/rules/knowledge-base.mdc` (`alwaysApply: true`) |
| Codex    | `generated/codex/rules/knowledge-base.md` (plain body)        |
| OpenCode | `generated/opencode/rules/knowledge-base/AGENTS.md`           |

`agentspec check` passes with no drift. `learn/SKILL.md` no longer contains any
config-mutation logic or references to updating `AGENTS.md`.

### Verification

- `agentspec check` exits 0 from `agent-config/`
- `learn/SKILL.md` contains no mention of `learnings-discoverer` or config
  file mutation
- Generated rule files exist at the four paths above

## What We're NOT Doing

- Changing the Knowledge Base guidance text — content is preserved verbatim
- Migrating any other skill sections to rules
- Updating the user's installed Claude/Cursor/Codex/OpenCode configs directly
  (that is a user action; the compiled outputs are the artifacts)
- Touching any Rust source in `agentspec/` — the compiler already supports rules

## Key Discoveries

- Knowledge Base body text (from `learn/SKILL.md` lines 209–218):
  ```
  ## Knowledge Base

  Before starting tasks, try to dispatch the `learnings-discoverer` subagent with a
  description of the task, technologies, and components involved. Resolve
  `THOUGHTS_DIR` first (to an absolute path), then pass it explicitly in the
  subagent prompt/context. If `THOUGHTS_DIR` cannot be resolved, continue without
  the subagent and proceed with the task. When dispatched, it searches
  `$THOUGHTS_DIR/learnings/` for relevant insights and returns them verbatim with
  source attribution. Apply any returned insights as context for the current
  task.
  ```
- Rule frontmatter format: `id`, `description`, `version`, `compat.targets`
  (no `paths:` field for an unconditional rule). See fixture at
  `agentspec/tests/fixtures/agent-config/spec/rules/general-guidance.md`.
- `agentspec compile` must be run from `agent-config/` (the directory containing
  `agentspec.toml`).
- `agentspec check` will fail if `generated/` is stale — must be run after
  compile.

---

## Phase 1: Create the `knowledge-base` Rule Spec

### Overview

Create `agent-config/spec/rules/knowledge-base.md` and compile it. This phase
is purely additive — no existing files are changed.

### Changes Required

#### 1. Create `agent-config/spec/rules/knowledge-base.md`

- [x] Create the file with the following content:

  ```markdown
  ---
  id: knowledge-base
  description: Instruct agents to dispatch learnings-discoverer before starting tasks
  version: 1
  compat:
    targets:
      - claude
      - cursor
      - codex
      - opencode
  ---

  ## Knowledge Base

  Before starting tasks, try to dispatch the `learnings-discoverer` subagent with a
  description of the task, technologies, and components involved. Resolve
  `THOUGHTS_DIR` first (to an absolute path), then pass it explicitly in the
  subagent prompt/context. If `THOUGHTS_DIR` cannot be resolved, continue without
  the subagent and proceed with the task. When dispatched, it searches
  `$THOUGHTS_DIR/learnings/` for relevant insights and returns them verbatim with
  source attribution. Apply any returned insights as context for the current
  task.
  ```

  Note: the body is the Knowledge Base guidance text taken verbatim from
  `learn/SKILL.md` lines 209–218. No `paths:` field — this rule is unconditional.

#### 2. Compile and verify

- [x] Run `agentspec compile` from `agent-config/`
- [x] Verify the four generated rule files exist:
  - `generated/claude/rules/knowledge-base.md`
  - `generated/cursor/rules/knowledge-base.mdc`
  - `generated/codex/rules/knowledge-base.md`
  - `generated/opencode/rules/knowledge-base/AGENTS.md`
- [x] Run `agentspec check` from `agent-config/` and confirm it exits 0

### Success Criteria

#### Automated Verification

- [x] `agentspec compile` exits 0 from `agent-config/`
- [x] `agentspec check` exits 0 from `agent-config/` (no drift)
- [x] `generated/claude/rules/knowledge-base.md` exists and has **no** YAML
  frontmatter block (unconditional rule → no `---` delimiters)
- [x] `generated/cursor/rules/knowledge-base.mdc` exists and contains
  `alwaysApply: true` in its frontmatter
- [x] `generated/codex/rules/knowledge-base.md` exists and contains no
  frontmatter block
- [x] `generated/opencode/rules/knowledge-base/AGENTS.md` exists and contains
  the Knowledge Base body text

#### Manual Verification

- [x] Inspect `generated/claude/rules/knowledge-base.md` — contents match the
  existing `## Knowledge Base` block in `~/.claude/CLAUDE.md` exactly (same
  guidance text, same section heading)

---

## Phase 2: Remove the "Configuration File Integration" Section from `learn/SKILL.md`

### Overview

With the rule in place, the `learn` skill no longer needs to check or modify
any global configuration file. Remove the "Configuration File Integration"
section and the stale reference in "Important Guidelines", then recompile so
all four generated learn skill files are updated.

### Changes Required

#### 1. `agent-config/spec/skills/learn/SKILL.md` — Remove config-mutation logic

**File**: `agent-config/spec/skills/learn/SKILL.md`

- [x] Remove the entire "Configuration File Integration" section — from the
  `## Configuration File Integration` heading (line 189) through the closing
  `**Never modify any configuration file without explicit user approval.**` line
  (line 221), inclusive. This is a complete section deletion with no replacement.

- [x] In the "Important Guidelines" section, update the bullet that currently
  reads:
  ```
  **Never modify source code** — this command only writes to
  `$THOUGHTS_DIR/learnings/` and optionally updates configuration files
  ```
  Change it to:
  ```
  **Never modify source code** — this command only writes to
  `$THOUGHTS_DIR/learnings/`
  ```
  (Remove "and optionally updates configuration files" — the skill no longer
  touches any config file.)

#### 2. Recompile and verify

- [x] Run `agentspec compile` from `agent-config/` to regenerate the four
  provider-specific `learn` skill files
- [x] Run `agentspec check` from `agent-config/` to confirm no drift

### Success Criteria

#### Automated Verification

- [x] `agentspec compile` exits 0 from `agent-config/`
- [x] `agentspec check` exits 0 from `agent-config/`
- [x] `grep -r "learnings-discoverer" agent-config/spec/skills/learn/` returns
  no matches (the section is gone)
- [x] `grep -r "Configuration File Integration" agent-config/spec/skills/learn/`
  returns no matches

#### Manual Verification

- [x] Inspect `agent-config/generated/claude/skills/learn/SKILL.md` — confirm
  the "Configuration File Integration" section is absent from the generated
  output
- [x] Inspect `agent-config/generated/cursor/skills/learn/SKILL.md` — same
  check

> **Post-review fix:** The "Your Role" section item 4 ("Maintain the connection
> between the learnings directory and configuration files") was also removed —
> it was a stale role description that referenced the deleted functionality.

---

## Testing Strategy

### Automated

- `agentspec check` is the authoritative test: it compiles and diffs against
  disk. A clean exit guarantees the spec and generated files are in sync.
- After each phase, `agentspec check` must pass before proceeding.

### Manual

- Compare `generated/claude/rules/knowledge-base.md` against the
  `## Knowledge Base` block currently in `~/.claude/CLAUDE.md` to confirm the
  guidance text is identical.

## References

- Idea file: `thoughts/ideas/2026-03-22-learn-skill-knowledge-base-rule.md`
- learn skill: `agent-config/spec/skills/learn/SKILL.md` (lines 189–221, 235)
- Rule fixture example: `agentspec/tests/fixtures/agent-config/spec/rules/general-guidance.md`
- Rules implementation plan (completed): `thoughts/plans/2026-03-22-agentspec-rules-support.md`
- agentspec workflow: `agent-config/CLAUDE.md`
