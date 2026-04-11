# `/learn` Command Implementation Plan

## Overview

Create a new `/learn` command spec that captures reusable insights from the
current conversation and writes them to topic-based `agent-docs/` files. The
agent automatically categorizes each insight as either **project-specific**
(written to `agent-docs/` in the project root) or **general** (written to
`~/.config/agent-docs/` for cross-project knowledge). The user confirms the
categorization and content before anything is written. This follows the
progressive disclosure pattern — keeping CLAUDE.md lean while building growing
knowledge bases that future sessions can reference on demand.

## Current State Analysis

The `agent-config` system has 12 existing command specs in `spec/commands/` that
compile through a validate → normalize → compile pipeline into provider-specific
output for Claude, OpenCode, Codex, and Cursor. The closest analogue to `/learn`
is `create-idea`, which also:

- Writes structured markdown files to user project directories
- Accepts optional arguments
- Uses the `planning` model profile
- Needs `read`, `write`, and `todowrite` tools

### Key Discoveries:

- Command specs use YAML frontmatter validated against
  `spec/schema/canonical.schema.json` — required fields: `id`, `kind`,
  `description`, `version`, plus `command` block for `kind: command`
- Tool names are canonical (`read`, `write`, `glob`, `grep`) and mapped to
  provider-specific names via `mappings/tools.yaml`
- Model profiles (`planning`, `balanced`, `deep_review`) are defined in
  `mappings/models.yaml` with per-provider resolution
- The build system (`npm run build:agents`) handles all generated output — no
  hand-editing of `generated/` files
- Instruction body text must be provider-neutral; provider-specific behavior
  belongs in mappings/adapters

## Desired End State

A new file `spec/commands/learn.md` exists and:

1. Passes schema validation (`npm run validate:agents`)
2. Compiles successfully to all four providers (`npm run compile:agents`)
3. Generated output matches compiled state (`npm run check:agents`)
4. The full build pipeline succeeds: `npm run build:agents`

The command body provides clear, complete instructions for an agent to:
- Analyze the current conversation for reusable insights
- Categorize each insight as project-specific or general
- Write project insights to `agent-docs/` in the project root
- Write general insights to `~/.config/agent-docs/` for cross-project use
- Append to existing topics or create new files (with user confirmation)
- Check CLAUDE.md for `agent-docs/` pointers and offer to add them

### How to Verify:

```bash
npm --prefix agent-config run build:agents
```

This single command runs validate → compile → check. If it exits 0, the spec is
valid and all generated output is correct.

## What We're NOT Doing

- **`/validate-learnings` command** — deferred to post-v1 per idea doc
- **Auto-retrieval system** — how future sessions discover `agent-docs/` is out
  of scope (handled by CLAUDE.md pointers and agent behavior)
- **Sensitivity filtering** — user confirmation step serves as the safety gate
- **Schema changes** — no modifications to `canonical.schema.json` needed; the
  existing schema supports everything `/learn` requires
- **Mapping changes** — no new tools, model profiles, or features needed

## Implementation Approach

This is a single-file addition. The command spec follows the exact same pattern
as `create-idea.md`: YAML frontmatter defining metadata + a markdown body
containing the full agent instruction set. The build system handles all
downstream generation.

---

## Phase 1: Create the Command Spec

### Overview

Write the canonical spec file `spec/commands/learn.md` with frontmatter and
instruction body.

### Changes Required:

#### 1. New file: `spec/commands/learn.md`

**Frontmatter:**

```yaml
---
id: learn
kind: command
name: learn
description: Capture reusable insights into project or global agent-docs topic files
version: 1
execution:
  model_profile: planning
  mode: command
capabilities:
  tools:
    - glob
    - grep
    - read
    - write
command:
  accepts_args: true
  args_schema: string
routing:
  trigger: learn
compat:
  targets:
    - claude
    - opencode
    - codex
    - cursor
---
```

**Design rationale for frontmatter choices:**

- `model_profile: planning` — conversation synthesis requires the strongest
  model (maps to `opus` on Claude)
- `tools: [glob, grep, read, write]` — `glob` to discover existing
  `agent-docs/` files (both project and global), `grep` to search CLAUDE.md for
  existing pointers, `read` to read existing topic files and CLAUDE.md, `write`
  to create/update files in both locations
- `accepts_args: true` — user can optionally hint at what to capture (e.g.,
  `/learn auth flow gotcha`)
- No `delegate_to` — this command runs inline, not via a sub-agent
- All four providers targeted

**Instruction body** (markdown after frontmatter):

The body covers the complete `/learn` workflow in these sections:

1. **Role and purpose** — what the command does and why
2. **Getting started** — how to handle args vs. no-args invocation
3. **Conversation analysis** — how to identify learnable moments
4. **Scope categorization** — project-specific vs. general insights
5. **Content structure** — the topic file format
6. **File operations** — discovery, append vs. create, naming conventions,
   dual storage locations
7. **User confirmation** — present proposed content with scope labels
8. **CLAUDE.md integration** — check for pointers at both levels
9. **Guidelines** — what makes a good learning, what to avoid

Full body content for the spec:

````markdown
# Learn

Capture reusable insights from the current conversation and write them to
topic-based `agent-docs/` files. Each insight is categorized as either
**project-specific** (stored in the project's `agent-docs/`) or **general**
(stored in `~/.config/agent-docs/` for cross-project use). These knowledge bases
help future agent sessions be more effective without bloating configuration
files.

## Your Role

You are a knowledge curator. Your job is to:

1. **Identify** high-signal, reusable insights from the current conversation
2. **Categorize** each insight as project-specific or general
3. **Organize** them into topic-based files in the appropriate `agent-docs/`
4. **Present** proposed content to the user for confirmation before writing
5. **Maintain** the connection between `agent-docs/` and configuration files

You are capturing operational knowledge — gotchas, patterns, architectural
decisions, debugging strategies, API conventions, tooling quirks — anything that
would save a future agent session from rediscovering the same insights.

## Getting Started

The user invokes `/learn` in one of two ways:

**With arguments** (e.g., `/learn auth flow gotcha`):
- Focus your analysis on the topic area hinted at by the arguments
- Search the conversation for insights related to that topic

**Without arguments**:
- Scan the full conversation for learnable moments
- Look for patterns like: surprises, corrections, workarounds, discoveries,
  decisions with non-obvious rationale, gotchas that caused debugging time

## Identifying Insights

Good learnings are:

- **Reusable** — would help a future session working in the same area
- **Non-obvious** — not something an agent would figure out from reading the
  code alone
- **Actionable** — tells the reader what to do or avoid, not just what exists
- **Specific** — references concrete files, patterns, or behaviors

Examples of good learnings:
- "The `auth` middleware silently swallows 403 errors and returns 200 — always
  check the response body for an `error` field"
- "Database migrations must be run with `--lock-timeout=5s` or they'll deadlock
  against the connection pool"
- "The `UserService` class has a hidden dependency on `ConfigStore` being
  initialized first — this isn't enforced by the type system"

Examples of poor learnings (avoid these):
- Generic programming advice ("always write tests")
- Things obvious from reading the code ("the User model has an email field")
- Temporary state ("there's currently a bug in the login page")

## Categorizing Scope

For each insight, determine whether it is **project-specific** or **general**:

**Project-specific** insights are tied to this particular codebase:
- References specific files, modules, or architecture in this project
- Describes conventions or patterns unique to this project
- Documents gotchas caused by this project's specific setup
- Examples: "The auth middleware swallows 403s", "Migrations need --lock-timeout
  because of our connection pool config"

**General** insights apply across any project:
- Language features, tool behaviors, or API quirks not specific to this codebase
- Broadly applicable techniques, patterns, or debugging strategies
- Examples: "Markdown fenced code blocks can be nested using more backticks on
  the outer fence", "PostgreSQL NOTIFY payloads are limited to 8000 bytes"

When in doubt, lean toward **project-specific** — it's the safer default. The
user will review your categorization before anything is written and can correct
it.

## Proposing Learnings

After analyzing the conversation, propose one or more learnings. For each:

1. **Identify the scope** — project-specific or general.

2. **Identify the topic** — a broad category this belongs to (maps to a
   filename). Use descriptive kebab-case names like `auth-patterns`,
   `database-gotchas`, `api-conventions`, `build-system`, `testing-strategies`.

3. **Draft the insight content** — concise, actionable markdown.

4. **Check for existing topic files** — use Glob to search for matching files:
   - Project: `agent-docs/*.md` in the project root
   - General: `~/.config/agent-docs/*.md`
   If a matching topic file exists, read it and propose appending. If not,
   propose creating a new file.

Present the proposed learnings to the user with scope labels:

```
I found [N] insight(s) from our conversation:

**[Project] auth-patterns** (new file → agent-docs/auth-patterns.md)
> The auth middleware silently swallows 403 errors...

**[General] markdown-formatting** (new file → ~/.config/agent-docs/markdown-formatting.md)
> Fenced code blocks can be nested using more backticks...

Shall I write these?
```

Wait for explicit user confirmation before writing any files. The user may
change the scope, topic, or content of any proposed learning.

## File Format

Each topic file in `agent-docs/` (both project and global) follows this format:

```markdown
# [Topic Title]

## [Insight Title]
<!-- learned: YYYY-MM-DD -->

[Concise, actionable description of the insight]

## [Another Insight]
<!-- learned: YYYY-MM-DD -->

[...]
```

Keep it simple:
- No YAML frontmatter — plain markdown for easy reading
- HTML comment for the date — lightweight metadata that doesn't clutter
- Each insight is an H2 section within the topic file
- The H1 is the human-readable topic title (e.g., "Authentication Patterns")

## File Operations

### Storage locations

There are two `agent-docs/` directories:

- **Project**: `agent-docs/` at the project root — for project-specific insights
- **Global**: `~/.config/agent-docs/` — for general cross-project insights

Both use the same file format and naming conventions.

### Discovering existing files

Use Glob to check for existing topic files in both locations:
- `agent-docs/*.md` in the project root
- `~/.config/agent-docs/*.md` for global learnings

This tells you which topics already have files and where.

### Creating new topic files

- Filename: `{topic-name}.md` in kebab-case (e.g., `auth-patterns.md`)
- Create the target `agent-docs/` directory if it doesn't exist
- Start with the H1 topic title, then the first insight as an H2

### Appending to existing files

- Read the existing file first
- Add the new insight as a new H2 section at the end of the file
- Avoid duplicating insights that are already captured

### File size guidance

If a topic file is growing large (many insights), suggest to the user that it
might be worth splitting into subtopics. But don't enforce a hard limit — let
the user decide.

## Configuration File Integration

After writing the learning files, check whether the relevant configuration files
contain pointers to `agent-docs/`:

### Project-level pointer

Check if the project's CLAUDE.md or AGENTS.md references `agent-docs/`. Use
Grep to search for the string `agent-docs` in these files at the project root.

If no reference exists, suggest adding a pointer:

```
I notice your project's CLAUDE.md doesn't reference `agent-docs/` yet. Future
agent sessions won't know to check it unless there's a pointer. Would you like
me to add this section?

## Project Knowledge Base

Before starting tasks, check `agent-docs/` for project-specific patterns,
gotchas, and conventions captured from previous sessions.
```

If neither CLAUDE.md nor AGENTS.md exists, mention that the user may want to
create one with the pointer, but don't create it automatically.

### Global pointer

If general insights were written, check if the user's global CLAUDE.md
(`~/.claude/CLAUDE.md`) references `~/.config/agent-docs/`. Use Grep to search
for the string `agent-docs` in that file.

If no reference exists, suggest adding a pointer:

```
Your global CLAUDE.md (~/.claude/CLAUDE.md) doesn't reference the general
knowledge base yet. Would you like me to add this section?

## General Knowledge Base

Before starting tasks, check `~/.config/agent-docs/` for general patterns,
gotchas, and techniques captured from previous sessions across all projects.
```

**Never modify any configuration file without explicit user approval.**

## Important Guidelines

- **Always confirm before writing** — present proposed content and wait for
  approval
- **Keep insights concise** — a few sentences per insight is ideal; if it takes
  a paragraph, it might be two insights
- **Use the conversation as source of truth** — don't invent insights or
  speculate about things not discussed
- **Prefer specific over general** — reference actual file paths, function
  names, and behaviors when possible
- **One topic per file** — don't mix unrelated insights into the same file
- **Never modify source code** — this command only writes to `agent-docs/`
  directories and optionally updates configuration files
- **Default to project scope when uncertain** — it's easier for the user to
  promote a project insight to general than to untangle a misplaced general one
````

### Success Criteria:

#### Automated Verification:

- [x] Schema validation passes: `npm --prefix agent-config run validate:agents`
- [x] Compilation succeeds: `npm --prefix agent-config run compile:agents`
- [x] Generated output matches: `npm --prefix agent-config run check:agents`
- [x] Full build pipeline passes: `npm --prefix agent-config run build:agents`

#### Manual Verification:

- [ ] Review `spec/commands/learn.md` — frontmatter follows conventions, body
      covers all workflow steps from the idea doc
- [ ] Review generated Claude output in
      `generated/claude/commands/learn.md` — correct tool list, model, body
- [ ] Review generated OpenCode output in
      `generated/opencode/commands/learn.md` — correct boolean tool map, model,
      variant
- [ ] Review generated Codex output in
      `generated/codex/skills/learn/SKILL.md` — correct model, reasoning effort

**Implementation Note**: After completing this phase and all automated
verification passes, pause here for manual confirmation from the human that the
manual testing was successful before proceeding.

---

## Phase 2: Verify Cross-Provider Output

### Overview

Spot-check the generated output across all providers to confirm the build system
correctly handled the new spec.

### Expected Generated Files:

1. `generated/claude/commands/learn.md` — frontmatter with `tools: Glob, Grep, Read, Write` and `model: opus`
2. `generated/opencode/commands/learn.md` — boolean tool map with `glob: true, grep: true, read: true, write: true` (others `false`), `model: anthropic/claude-opus-4-6`, `variant: max`
3. `generated/codex/skills/learn/SKILL.md` — `model: gpt-5.3-codex`, `model_reasoning_effort: xhigh`, no tools field
4. `generated/cursor/commands/learn.md` — body preserved, minimal frontmatter
5. `generated/manifest.json` — updated with new file entries and new source hash

### Success Criteria:

#### Automated Verification:

- [ ] `npm --prefix agent-config run check:agents` exits 0 (already covered in
      Phase 1, but re-run to confirm)

#### Manual Verification:

- [ ] Each generated file listed above exists and contains expected frontmatter
- [ ] The markdown instruction body is preserved identically across all providers
- [ ] `manifest.json` includes entries for all four new generated files
- [ ] No unexpected warnings in `manifest.json` related to the learn command
      (some standard warnings like `DROPPED_FIELD` for Codex tools are expected)

---

## Testing Strategy

### Automated Tests:

The build pipeline (`npm run build:agents`) serves as the automated test suite:
- JSON Schema validation catches frontmatter errors
- Semantic validation catches routing collisions, missing profile references,
  and broken `delegate_to` references
- Check step catches any mismatch between compiled and on-disk output

### Manual Testing Steps:

1. Run `/learn` in a Claude Code session within a test project
2. Verify it analyzes the conversation and proposes insights
3. Confirm each insight is labeled with a scope ([Project] or [General])
4. Confirm it discovers existing `agent-docs/` files in both locations (if any)
5. Confirm it presents content for user approval before writing
6. Verify the written file follows the specified format
7. Verify project-level CLAUDE.md pointer check works when no pointer exists
8. Verify global `~/.claude/CLAUDE.md` pointer check works for general insights
9. Run `/learn some-topic` with args and verify it focuses on that topic
10. Verify a general insight is written to `~/.config/agent-docs/`, not the
    project directory

## Performance Considerations

None significant. The `/learn` command is invoked manually and performs a small
number of file I/O operations. The `planning` model profile uses the strongest
available model, which is appropriate for the synthesis task but means slightly
higher latency per invocation.

## Migration Notes

No migration needed. This is a net-new command with no existing state to
migrate. Both `agent-docs/` directories (project-level and
`~/.config/agent-docs/`) are created on first use.

## References

- Idea document: `thoughts/ideas/learn-command-persistent-learnings.md`
- Closest existing command: `spec/commands/create-idea.md`
- Schema: `spec/schema/canonical.schema.json`
- Build entry point: `scripts/compile_agents.ts`
- [Progressive Disclosure for AI Coding Tools](https://alexop.dev/posts/stop-bloating-your-claude-md-progressive-disclosure-ai-coding-tools/)
