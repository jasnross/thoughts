# Triage TODOs Skill Implementation Plan

## Overview

Create a new `triage-todos` skill in `agent-config/spec/skills/` that scans a
project directory for TODO items (both TODO.md files and inline `TODO` code
comments), groups related items semantically, ranks groups by ambiguity, presents
them for user selection, and routes the chosen group to either
`/tw-create-plan` (unambiguous) or `/tw-create-idea` (ambiguous).

## Current State Analysis

- No skill exists for TODO discovery or triage
- The `create-idea` and `create-plan` skills both accept free-form input
  describing a problem or task — the triage skill will format its TODO context as
  that input
- Skills delegate to other skills via prose instructions:
  "Load the `/{{ specs.skill.<id>.name }}` skill" (pattern from `gt-sync`,
  `describe-pr`, `create-pr`)
- The `thoughts/discover-dir.md` fragment with `writeable = true` handles
  `$THOUGHTS_DIR` resolution

### Key Discoveries:

- Existing skills omit the `version` field despite README listing it as required
  — the compiler defaults it. Follow the existing convention and omit it.
- The `skill` frontmatter block (`accepts_args`, `args_schema`) is not used by
  any existing skill despite being documented. The triage-todos skill accepts an
  optional directory argument, but since no existing skill uses this block, we'll
  document the argument in the skill body prose instead.
- The `planning` execution preset maps to opus on Claude, which is appropriate
  for the analysis and judgment calls this skill requires (ambiguity scoring,
  semantic grouping).
- Skills reference other skills via `{{ specs.skill.<id_underscored>.name }}`
  where hyphens in the `id` become underscores in the template variable.

## Desired End State

A single file at `spec/skills/triage-todos/SKILL.md` that:

1. Compiles cleanly via `agentspec compile`
2. Appears in the slash-command menu as `/tw-triage-todos`
3. Scans a target directory for TODO.md files and inline `TODO` comments
4. Groups related TODOs semantically, ranks by ambiguity, and presents to user
5. Routes the selected group to `/tw-create-plan` or `/tw-create-idea`

Verification: `agentspec compile` exits zero with no errors or warnings related
to the new skill.

## What We're NOT Doing

- Adding new fragments — the skill body is self-contained except for the
  existing `thoughts/discover-dir.md` include
- Modifying existing skills — `create-idea` and `create-plan` already accept
  free-form input
- Adding the `version` or `skill.accepts_args` fields — no existing skill uses
  them
- Supporting FIXME/HACK/XXX markers — TODO only for v1
- Persistent state tracking — each invocation is a fresh scan

## Implementation Approach

Create a single SKILL.md that instructs the agent through a linear pipeline:
scan → parse → group → rank → present → route. The skill is an orchestrator
that does its own analysis work (no sub-agent dispatch needed for scanning) and
then loads a downstream skill for document creation.

## Implementation

### Overview

Create `spec/skills/triage-todos/SKILL.md` with YAML frontmatter and a
structured skill body covering the full triage pipeline.

### Changes Required:

#### 1. Skill Spec File

**File**: `spec/skills/triage-todos/SKILL.md`

- [x] Create directory `spec/skills/triage-todos/`
- [x] Create `SKILL.md` with the following frontmatter:

```yaml
---
id: triage-todos
description: >-
  Scan a project for TODO items, group related ones, rank by ambiguity, and
  route to create-idea or create-plan.
user_invocable: true
agent_invocable: true
execution:
  preset: planning
capabilities:
  tools:
    - bash
    - glob
    - grep
    - read
---
```

Key tool choices:
- `bash` — needed for `pwd` to determine CWD, and for any file-system
  operations during scanning
- `glob` — find TODO.md files across the directory tree
- `grep` — scan for inline `TODO` comments in code
- `read` — read TODO.md contents and surrounding code context
- No `write` or `edit` — this skill doesn't create files; it delegates that to
  the downstream skill
- No `tasks` — scanning and grouping happen in the main agent context; the
  downstream skill handles its own sub-agent dispatch

- [x] Write skill body with the following sections:

**Section: Your Role** — Establish that this is a triage/routing skill, not an
implementation skill. The agent's job is to discover, organize, and hand off.

**Section: Input Parsing** — Accept an optional directory argument. If provided,
use it as the scan target. If not, use the current working directory (`pwd`).

**Section: Step 1 — Scan** — Two parallel scan operations:
1. Glob for `**/TODO.md` files in the target directory, then read each one
2. Grep for `TODO` and `TODO:` patterns in code files (excluding common
   non-source directories like `node_modules/`, `.git/`, `vendor/`, `target/`,
   `dist/`, `build/`)

For inline TODOs, capture surrounding context (file path, line number, 2-3
lines above and below) to aid grouping and ambiguity assessment.

**Section: Step 2 — Parse & Normalize** — Extract individual TODO items into a
uniform structure regardless of source:
- Source (file path + line number, or TODO.md section)
- Description (the TODO text itself)
- Context (surrounding code or document section)

**Section: Step 3 — Group** — Cluster related TODOs using semantic-first
grouping:
- Primary signal: shared topic, referenced components, described feature
  (keywords, function names, module names)
- Secondary signal: file/directory proximity (same module or package)
- TODOs that don't relate to anything else become singleton groups
- Each group gets a short descriptive label

**Section: Step 4 — Rank by Ambiguity** — Score each group as low, medium, or
high ambiguity using these heuristics:

*Low ambiguity (high priority):*
- Concrete action verb (add, remove, rename, extract, fix, replace)
- Specific file, function, or component referenced
- Small, well-scoped change
- Surrounding code makes the intent clear

*Medium ambiguity:*
- Clear problem but multiple possible approaches
- References a system but not a specific location
- May need minor research or a design choice

*High ambiguity (lower priority):*
- Vague language (clean up, improve, refactor, revisit)
- No surrounding context or stale context
- Touches multiple systems or unclear boundaries
- Requires significant research or design decisions

**Section: Step 5 — Present** — Show the ranked groups to the user using
`AskUserQuestion` with one option per group. Each option includes:
- Label: group name + ambiguity level
- Description: list of TODOs in the group with file locations
- Groups ordered by ambiguity (low → medium → high)

Include a "None — skip for now" option.

**Section: Step 6 — Route** — Based on the selected group's ambiguity:
- **Low ambiguity** → Explain the routing decision, then instruct: "Load the
  `/{{ specs.skill.create_plan.name }}` skill" and pass the TODO group context
  as input (descriptions, file locations, surrounding code)
- **Medium or High ambiguity** → Explain the routing decision, then instruct:
  "Load the `/{{ specs.skill.create_idea.name }}` skill" and pass the TODO
  group context as input

Before loading the downstream skill, present a brief explanation of why this
group is being routed to idea vs. plan, citing the specific ambiguity factors.

**Section: THOUGHTS_DIR Resolution** — ~~Include the standard fragment for
directory discovery since the downstream skills will need it.~~
  > **Deviation:** Removed the `discover-dir.md` include. Code review identified
  > that both `create-idea` and `create-plan` already include this fragment
  > themselves, so pre-resolving `$THOUGHTS_DIR` in the triage skill is redundant
  > and adds an unnecessary user-facing prompt step.

#### Tests

- [x] Run `agentspec validate` — exits zero, no errors for `triage-todos`
- [x] Run `agentspec compile` — exits zero, generates output for the new skill
- [x] Run `agentspec check` — shows the new skill in the diff (expected)
  > **Deviation:** `agentspec check` subcommand does not exist. Validated via
  > `agentspec compile` + manual inspection of `generated/` output instead.
- [x] Verify generated output includes the skill with correct name prefix
      (`tw-triage-todos`) by inspecting `generated/claude/` output

### Success Criteria:

#### Automated Verification:

- [x] `agentspec validate` exits zero with no errors (run from `agent-config/`)
- [x] `agentspec compile` exits zero with no errors (run from `agent-config/`)
- [x] The skill file exists at `spec/skills/triage-todos/SKILL.md`
- [x] The generated output references `tw-triage-todos` (grep `generated/` for `triage-todos`)
- [x] Cross-skill references resolve — no unresolved `{{ specs.` template variables in the generated output for this skill

#### Manual Verification:

- [ ] Invoke `/tw-triage-todos` in a project with TODO items and confirm it scans, groups, ranks, and presents options correctly (manual-only: requires interactive skill invocation and judgment of grouping quality)

## References

- Idea document: `thoughts/ideas/2026-04-13-triage-todos-skill.md`
- Pattern reference — skill-loads-skill: `spec/skills/gt-sync/SKILL.md:52,188`
- Pattern reference — AskUserQuestion for selection: `spec/skills/code-review-loop/SKILL.md`
- Fragment: `spec/fragments/thoughts/discover-dir.md`
- Cross-skill reference syntax: `{{ specs.skill.create_idea.name }}`, `{{ specs.skill.create_plan.name }}`
