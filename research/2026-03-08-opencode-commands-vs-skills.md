---
date: 2026-03-08T17:30:00-05:00
topic: "OpenCode Commands vs Skills: How Custom Extensibility Works"
tags: [research, web, opencode, commands, skills, tools, agents, claude-code, extensibility]
status: complete
last_updated: 2026-03-08
last_updated_by: assistant
last_updated_note: "Added follow-up research on custom agents"
---

# Research: OpenCode Commands vs Skills — How Custom Extensibility Works

**Date**: 2026-03-08

## Research Question

How are custom OpenCode commands defined as opposed to skills? What are the differences between skills and commands in OpenCode?

In Claude Code they are now the same thing as skills and can be made non-invocable by agents by setting `disable-model-invocation` to `true`. But how does this work in OpenCode?

## Summary

OpenCode uses a **three-tier extensibility model** — Commands, Skills, and Tools — where the invoker type is determined by which artifact you use, not by a flag. This contrasts with Claude Code's unified "skills" concept where `disable-model-invocation: true` toggles between user-only and AI-only modes. In OpenCode there is **no `disable-model-invocation` equivalent**: because commands and skills are already structurally separate primitives, the split is encoded in the artifact type itself.

## Detailed Findings

### The Three-Tier Model

| Layer | Who Invokes | Format | Location |
|---|---|---|---|
| **Commands** | User (via `/name` slash syntax or `Ctrl+P` palette) | `.md` file with YAML frontmatter | `.opencode/commands/` or `~/.config/opencode/commands/` |
| **Skills** | AI agent autonomously (via built-in `skill` tool) | `SKILL.md` with YAML frontmatter | `.opencode/skills/<name>/SKILL.md` |
| **Tools** | AI agent autonomously | TypeScript/JS files | `.opencode/tools/` or `~/.config/opencode/tools/` |

### Commands: User-Invoked Slash Prompts

Custom commands are Markdown files whose filename becomes the slash command name. They are **purely user-triggered** — the AI never autonomously loads or executes them. Typing `/` opens an autocomplete dropdown; commands also appear in the `Ctrl+P` command palette.

**File format** (`.opencode/commands/test.md`):

```markdown
---
description: Run tests with coverage
agent: build
model: anthropic/claude-3-5-sonnet-20241022
subtask: true
---

Run the full test suite with coverage report.
Focus on $ARGUMENTS failing tests and suggest fixes.
Also run: !`git log --oneline -5`
See: @src/index.ts
```

**All supported frontmatter fields for commands**:

| Field | Type | Required | Purpose |
|---|---|---|---|
| `description` | string | No | Shown in TUI autocomplete and command palette |
| `agent` | string | No | Routes command to a specific named agent |
| `model` | string | No | Overrides the default model for this command |
| `subtask` | boolean | No | Forces the agent to run as a subagent, isolating context from the primary session |

**Template variable syntax**:
- `$ARGUMENTS` — full argument string passed when invoking
- `$1`, `$2`, `$3` — positional arguments
- `` !`bash-command` `` — shell output injected at render time
- `@filename` — file content included inline

Custom commands can also **override built-in commands** by using the same name (`/help`, `/init`, `/undo`, etc.).

Commands can also be defined inline in `opencode.jsonc`:
```jsonc
{
  "command": {
    "test": {
      "template": "Run tests for $ARGUMENTS with coverage",
      "description": "Run tests",
      "agent": "build",
      "model": "anthropic/claude-haiku-4-5",
      "subtask": true
    }
  }
}
```

### Skills: AI-Initiated Instruction Sets

Skills are reusable instruction sets that agents discover and load **autonomously**. The agent sees a list of available skill names and descriptions at startup; when it decides a skill is relevant, it calls the built-in `skill` tool (e.g., `skill({ name: "git-release" })`) to load the full content.

**Progressive disclosure**: Only `name` + `description` (~100 tokens) are loaded at startup for all skills. The full `SKILL.md` body is only fetched when the agent activates the skill — keeping context lean.

**Directory locations searched** (in order):
- `.opencode/skills/<name>/SKILL.md`
- `~/.config/opencode/skills/<name>/SKILL.md`
- `.claude/skills/<name>/SKILL.md` (cross-tool compatibility)
- `.agents/skills/<name>/SKILL.md` (cross-tool compatibility)

**File format** (`.opencode/skills/git-release/SKILL.md`):

```markdown
---
name: git-release
description: Semantic versioning and release tagging workflows. Use when the user asks to cut a release, tag a version, or generate a changelog.
license: MIT
metadata:
  author: yourname
  version: "1.0"
allowed-tools: Bash(git:*) Read
---

## Release Process

1. Verify all tests pass...
```

**All supported SKILL.md frontmatter fields** (per the [Agent Skills specification](https://agentskills.io/specification) that OpenCode implements):

| Field | Required | Constraints |
|---|---|---|
| `name` | Yes | 1–64 chars, lowercase alphanumeric + hyphens, must match directory name |
| `description` | Yes | 1–1024 chars, describes what the skill does and when to use it |
| `license` | No | License name or reference |
| `compatibility` | No | 1–500 chars, environment requirements |
| `metadata` | No | Arbitrary string key-value map |
| `allowed-tools` | No | Space-delimited pre-approved tool list (experimental — parsed but not enforced) |

**User-facing skill browsing**: Skills are deliberately filtered out of the `/` slash autocomplete. Users can browse available skills via the `/skills` command. This reinforces the design intent: skills are for AI consumption, commands are for users.

### Tools: Autonomous AI-Callable Functions

Custom tools are TypeScript/JS files placed in `.opencode/tools/`. The LLM calls them like any built-in tool (e.g., `read`, `write`, `bash`). Custom tools can even override built-ins by using the same name.

### No `disable-model-invocation` Equivalent

OpenCode has no direct equivalent to Claude Code's `disable-model-invocation: true` frontmatter field. The architectural reason: since commands and skills are already separate primitives, there's no need for a flag to toggle a single artifact between user-only and AI-only modes.

The closest approximations in OpenCode:

- **Prevent AI from loading any skills**: set `"tools": { "skill": false }` in `opencode.jsonc` (disables all skills globally)
- **Make something only user-triggered**: use a **command** (`.md` in `commands/`)
- **Make something only AI-triggered**: use a **skill** (`SKILL.md` in `skills/<name>/`) or **tool**
- **Permission control on tools**: use `"permission": { "edit": "ask", "bash": "deny" }` to require approval or block tool calls

**Important**: Claude Code-specific frontmatter fields like `disable-model-invocation` and `user-invocable` are **silently ignored** by OpenCode when present in `SKILL.md` files — unknown fields are not validated.

### Comparison: Claude Code vs OpenCode

| Concept | Claude Code | OpenCode |
|---|---|---|
| User-triggered prompts | Skill with `user-invocable: true` (default) | Command (separate artifact type) |
| AI-triggered instruction sets | Skill with `disable-model-invocation: false` | Skill (separate artifact type) |
| AI-callable functions | Tool use | Tool (TypeScript files in `tools/`) |
| Toggle between modes | `disable-model-invocation` flag | Not needed — use different artifact types |
| Isolation from main context | N/A | `subtask: true` on commands |
| Cross-tool compatibility | `.claude/skills/` | Reads `.claude/skills/` and `.agents/skills/` |

## Sources

- [Commands | OpenCode](https://opencode.ai/docs/commands/) — Primary command format reference
- [Agent Skills | OpenCode](https://opencode.ai/docs/skills/) — Official skills documentation
- [Custom Tools | OpenCode](https://opencode.ai/docs/custom-tools/) — Tool authoring guide
- [Config | OpenCode](https://opencode.ai/docs/config/) — Full `opencode.jsonc` schema
- [Agent Skills Specification](https://agentskills.io/specification) — Cross-tool SKILL.md standard OpenCode implements
- [TUI Commands Reference | DeepWiki](https://deepwiki.com/anomalyco/opencode/9.3-tui-commands-reference) — Internal command registry breakdown
- [Skills System | sst/opencode | DeepWiki](https://deepwiki.com/sst/opencode/5.7-skills-system) — Internal skills architecture
- [GitHub - opencode-ai/opencode](https://github.com/opencode-ai/opencode)
- [Feature Request: Custom commands #132](https://github.com/opencode-ai/opencode/issues/132) — Original request (explicitly modeled on Claude Code)
- [How Coding Agents Actually Work: Inside OpenCode](https://cefboud.com/posts/coding-agents-internals-opencode-deepdive/) — Deep dive into agent autonomy
- [playbooks.com writing-commands skill](https://playbooks.com/skills/ian-pascoe/dotfiles/writing-commands) — Real-world examples

## Key Takeaways

1. **OpenCode uses artifact type to encode invoker, not a flag.** Commands = user-invoked. Skills = AI-invoked. Tools = AI-callable functions. No `disable-model-invocation` needed.

2. **Skills are on an AI-discoverable registry with progressive disclosure** — only names/descriptions are loaded until activation. This keeps token usage low for large skill libraries.

3. **`subtask: true` on commands** is the closest thing to isolated context execution — it forces the command to run as a subagent, keeping the main session clean.

4. **Cross-tool compatibility**: OpenCode reads `.claude/skills/` (in addition to its own `.opencode/skills/`), so Claude Code skills are automatically available in OpenCode. Unknown frontmatter fields (like `disable-model-invocation`) are silently ignored.

5. **If you want both user and AI access to the same content**, you must duplicate it as both a command and a skill. OpenCode has no equivalent to Claude Code's unified skill that can serve both roles.

## Open Questions

- Whether `allowed-tools` in SKILL.md will become enforced in OpenCode (currently parsed but ignored).
- The DeepWiki source mentions TypeScript skill files (`.opencode/skill/*.ts`) as a potential advanced feature not yet in official docs — unclear if this is experimental or undocumented.
- Whether OpenCode will eventually add a `disable-skill-invocation` or `user-only` flag to skill files to avoid the content-duplication problem.

---

## Follow-up Research: Custom Agents (2026-03-08)

### Does OpenCode Have Custom Agents?

Yes — custom agents are a first-class feature, more capable than Claude Code's in several respects. Agents can be defined two ways:

- **Markdown files** with YAML frontmatter: `.opencode/agents/<name>.md` (project) or `~/.config/opencode/agents/<name>.md` (global)
- **JSON block** in `opencode.json` under the `"agent"` key

CLI scaffolding: `opencode agent create`

### Agent File Format

```markdown
---
description: Reviews code for best practices and security
mode: subagent
model: anthropic/claude-sonnet-4-5
temperature: 0.1
tools:
  write: false
  edit: false
permission:
  bash:
    "*": "deny"
color: "#ff6b6b"
---

You are a code reviewer. Focus on security, performance, and maintainability.
Never make changes to files. Only produce analysis and recommendations.
```

### All Configurable Agent Properties

| Property | Type | Description |
|---|---|---|
| `description` | string | Required. Explains purpose; orchestrators use this to decide delegation |
| `mode` | `"primary"`, `"subagent"`, `"all"` | Who can invoke it: user-facing, AI-delegated, or both |
| `model` | string | Provider/model override (e.g. `anthropic/claude-sonnet-4-5`) |
| `prompt` | string | System prompt — inline text or `{file:./path}` reference (JSON only) |
| `temperature` | 0.0–1.0 | Sampling temperature |
| `top_p` | 0.0–1.0 | Nucleus sampling |
| `steps` | number | Max agentic iterations |
| `tools` | object | Per-tool enable/disable (`true`/`false`) |
| `permission` | object | Per-tool permission rules with glob patterns: `"allow"`, `"ask"`, `"deny"` |
| `permission.task` | object | Controls which subagents this agent may invoke |
| `color` | string | Hex color for TUI display |
| `hidden` | boolean | Hides from `@` autocomplete (still invocable programmatically) |
| `disable` | boolean | Deactivates entirely |

### How Agents Are Invoked

**Primary agents**: User cycles with `Tab`, or sets `opencode --agent <name>`, or configures `"default_agent"` in `opencode.json`.

**Subagents** — two paths:
1. Primary agent calls the `task` tool internally → new child session, returns result
2. User types `@agentname` in the TUI to directly invoke a subagent

### Built-in Agents

| Agent | Mode | Purpose |
|---|---|---|
| `build` | primary | Default; full tool access for development |
| `plan` | primary | Read-only planning; write exception for `.opencode/plans/*.md` |
| `general` | subagent | General-purpose task execution |
| `explore` | subagent | Fast read-only codebase exploration |
| `compaction` | system | Context window management (hidden) |
| `title` | system | Session title generation (hidden) |

### OpenCode vs Claude Code: Custom Agents Comparison

| Dimension | OpenCode | Claude Code |
|---|---|---|
| Definition format | Markdown + YAML frontmatter OR JSON | Markdown + YAML frontmatter |
| Config location | `.opencode/agents/` or `~/.config/opencode/agents/` | `.claude/agents/` or `~/.claude/agents/` |
| Agent modes | `primary`, `subagent`, `all` — explicit hierarchy | Always subagents; no primary mode concept |
| User can directly invoke subagent | Yes, via `@agentname` | No, only via orchestrator `Task` tool |
| Tool control | Per-tool `allow`/`ask`/`deny` with glob patterns | Tool list in frontmatter (less granular) |
| Restrict which subagents may be delegated to | `permission.task` glob rules | No equivalent |
| Temperature / top_p per agent | Yes | No |
| Max steps per agent | Yes (`steps`) | No |
| System-prompt file reference | `{file:./path}` syntax (JSON only) | No direct equivalent |
| **Key distinction** | Custom agents can be `primary` (user-selectable top-level) | Custom agents are always subagents |

### Gaps and Known Limitations

- `{file:}` prompt reference syntax does not work in Markdown agent files — only in JSON config (open issue)
- Agents with `mode: subagent` don't appear in the TUI's primary agent list — no separate subagent browser either (open issue)
- No documented sub-subagent nesting examples, though architecturally possible

### Additional Sources for Agents

- [Agents | OpenCode](https://opencode.ai/docs/agents/)
- [Permissions | OpenCode](https://opencode.ai/docs/permissions/)
- [Config | OpenCode](https://opencode.ai/docs/config/)
- [DeepWiki: opencode agent system](https://deepwiki.com/baumaxt/opencode/8.2-agents)
- [How Coding Agents Actually Work: Inside OpenCode](https://cefboud.com/posts/coding-agents-internals-opencode-deepdive/)
