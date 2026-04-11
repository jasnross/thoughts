---
date: 2026-03-17T10:14:54Z
topic: "Rules across AI coding agents: Claude Code, OpenCode, Codex CLI, Cursor, and others"
tags: [research, web, rules, agent-config, claude-code, opencode, codex, cursor, windsurf, copilot]
status: complete
last_updated: 2026-03-17
last_updated_by: assistant
---

# Research: Rules across AI coding agents

**Date**: 2026-03-17T10:14:54Z

## Research Question

How do "rules" work across different AI coding agents (Claude Code, OpenCode, Codex CLI, Cursor)?
- Do all providers support the concept of rules?
- How are rules "installed"? Can there be global and local rules? What about plugin rules?
- What is the expected format and frontmatter (if any) for rules?
- Anything else relevant.

Context: this research is in service of adding rules as a new generatable artifact type to the
`agent-config` build system, which already generates agents and skills for Claude Code, OpenCode,
Codex CLI, and Cursor.

---

## Summary

All four target providers support rules/instructions files. The ecosystem has broadly converged on
plain Markdown as the content format. **Cursor** is the most sophisticated: `.cursor/rules/*.mdc`
files with YAML frontmatter controlling scoping. **Claude Code** closely mirrors this with
`.claude/rules/*.md` + `paths:` frontmatter (shipped v2.0.64). **OpenCode** and **Codex CLI** both
use `AGENTS.md` — plain Markdown with no frontmatter, scoped by directory tree traversal rather
than glob patterns.

An emerging cross-tool standard around `AGENTS.md` (pushed by OpenAI, adopted by Amp, Zed, and
others) makes it increasingly feasible to write a single instruction file readable by multiple
agents, though Cursor's `.mdc` format is necessarily provider-specific.

---

## Detailed Findings

### Claude Code

**File paths:**

| Scope | Path |
|---|---|
| Global (user) | `~/.claude/rules/*.md` |
| Project-local | `./.claude/rules/*.md` |
| Org/managed (macOS) | `/Library/Application Support/ClaudeCode/CLAUDE.md` |

Rules files are discovered recursively in the `rules/` directory. The older `CLAUDE.md` files
(project root or `~/.claude/CLAUDE.md`) are a separate parallel concept — plain Markdown, no
frontmatter, always applied. The `.claude/rules/` system is an overlay for modular, scoped rules.

**Format:**

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "src/**/*.{ts,tsx}"
---

# API Development Rules

- All API endpoints must include input validation
- Use the standard error response format
```

**Supported frontmatter fields:**

| Field | Type | Notes |
|---|---|---|
| `paths` | array of glob strings | Triggers rule only when files matching globs are read. Must be quoted. |

Rules without `paths` are loaded unconditionally. Rules with `paths` trigger only when Claude reads
a matching file (not on Write — an active bug).

**Rule types (derived from fields):**
- No `paths`: always-on (equivalent to CLAUDE.md)
- With `paths`: auto-attached when matching files are read

**Activation history:** Shipped in Claude Code v2.0.64.

**Known bugs (early 2026):**
- `paths:` in user-level rules (`~/.claude/rules/`) is ignored ([Issue #21858](https://github.com/anthropics/claude-code/issues/21858))
- Path-scoped rules only trigger on Read, not Write ([Issue #23478](https://github.com/anthropics/claude-code/issues/23478))
- Git worktree path resolution ignores `paths:` filtering ([Issue #23569](https://github.com/anthropics/claude-code/issues/23569))

**Sharing/plugin rules:** No registry. Sharing via symlinks or `@~/.claude/my-file.md` import
syntax within CLAUDE.md.

**Sources:** [Official Docs](https://code.claude.com/docs/en/memory), [paddo.dev post](https://paddo.dev/blog/claude-rules-path-specific-native/)

---

### OpenCode

**File paths:**

| Scope | Path |
|---|---|
| Global (user) | `~/.config/opencode/AGENTS.md` |
| Project-local | `AGENTS.md` (project root, or anywhere up the tree) |
| Multi-file (via config) | `opencode.json` `instructions` array |

OpenCode traverses upward from the current directory collecting all `AGENTS.md` files it finds,
then also loads the global file. All matching files are combined (not first-match-wins).

**Compatibility fallbacks:** Reads `CLAUDE.md` if no `AGENTS.md` is present (can be disabled with
`OPENCODE_DISABLE_CLAUDE_CODE=1`).

**Format:** Plain Markdown. **No frontmatter fields defined or consumed** by the native system.

**Multi-file loading via `opencode.json`:**

```json
{
  "$schema": "https://opencode.ai/config.json",
  "instructions": [
    "CONTRIBUTING.md",
    "docs/guidelines.md",
    ".cursor/rules/*.md",
    "packages/*/AGENTS.md",
    "https://raw.githubusercontent.com/org/repo/main/standards.md"
  ]
}
```

The `instructions` array supports file globs, individual paths, and remote URLs (5s timeout). This
is the primary mechanism for monorepo and multi-file setups.

**Third-party plugin (`opencode-rules` by frap129):** Adds Cursor-style frontmatter scoping via
OpenCode's hook system. Rule files go in `.opencode/rules/` (project) or
`~/.config/opencode/rules/` (global). Supported frontmatter fields:

| Field | Type | Notes |
|---|---|---|
| `globs` | array of glob strings | Matches when context files match patterns |
| `keywords` | array of strings | Matches when user prompt contains term (case-insensitive) |
| `tools` | array | Matches when named MCP/built-in tool is available |
| `model` | array | Matches current model |
| `agent` | array | Matches current agent |
| `command` | array | Matches active slash command |
| `project` | array | Matches project type (node, python, etc.) |
| `branch` | array | Matches git branch pattern |
| `os` | array | Matches operating system |
| `ci` | boolean | Matches whether running in CI |
| `match` | `"any"` or `"all"` | OR (default) or AND logic for conditions |

**Sources:** [OpenCode Rules Docs](https://opencode.ai/docs/rules/), [Config Docs](https://opencode.ai/docs/config/), [opencode-rules plugin](https://github.com/frap129/opencode-rules)

---

### Codex CLI

**Important:** Codex CLI uses the term "rules" for two distinct and unrelated concepts. Do not
conflate them.

**Concept 1 — AGENTS.md (instruction files):**

| Scope | Path |
|---|---|
| Global (user) | `~/.codex/AGENTS.md` (or `~/.codex/AGENTS.override.md`) |
| Project-local | `AGENTS.md` at any directory level in the repo tree |
| Override | `AGENTS.override.md` in any subdirectory (takes precedence over `AGENTS.md` there) |
| Fallback filenames | Configurable via `project_doc_fallback_filenames` in `~/.codex/config.toml` |

Resolution: Codex walks from the Git root down to the current directory, collecting one file per
directory level (override wins over non-override). All collected files are concatenated. Most deeply
nested takes precedence on conflicts.

Format: Plain Markdown. **No frontmatter.** Scoping is directory-based, not glob-based.

**Concept 2 — Execution policy `.rules` files (Starlark):**

These are NOT instruction files. They govern which shell commands the agent may run without user
prompting. They use the **Starlark** language (Python-like configuration DSL), not Markdown.

| Scope | Path |
|---|---|
| Global (user) | `~/.codex/rules/default.rules` |
| Project/team | `.codex/rules/` (committed to repo) |

```python
prefix_rule(
    pattern       = ["gh", "pr", ["view", "list"]],
    decision      = "prompt",                # "allow" | "prompt" | "forbidden"
    justification = "PR read ops require approval",
    match         = [["gh", "pr", "view", "123"]],      # inline unit tests
    not_match     = [["gh", "pr", "merge", "123"]],
)
```

Decision precedence: `forbidden > prompt > allow`.

**For `agent-config` purposes:** The AGENTS.md concept is the analog to other agents' rules/CLAUDE.md
systems. The `.rules` Starlark files are a Codex-specific execution safety mechanism with no
analog in other agents.

**Sources:** [AGENTS.md guide](https://developers.openai.com/codex/guides/agents-md), [Rules docs](https://developers.openai.com/codex/rules/), [execpolicy README](https://github.com/openai/codex/blob/main/codex-rs/execpolicy/README.md)

---

### Cursor

**File paths:**

| Scope | Path |
|---|---|
| Project-local (modern) | `.cursor/rules/*.mdc` |
| Project-local (legacy, deprecated) | `.cursorrules` (project root) |
| Global (user) | Cursor app Settings UI → "Rules for AI" (not a file on disk) |
| Team/org (Cursor 1.7+) | Cursor organization dashboard (cloud-managed) |

**Format:** `.mdc` files (MDC = "Markdown Cursor") with YAML frontmatter.

```markdown
---
description: TypeScript and React component conventions
globs: ["**/*.tsx", "**/*.ts"]
alwaysApply: false
---

# TypeScript/React Rules

- Use functional components with explicit return types
- Prefer `const` arrow functions
```

**Supported frontmatter fields (complete set — only 3 fields):**

| Field | Type | Notes |
|---|---|---|
| `description` | string | Used by agent to decide relevance in "Agent Requested" mode; also shown in UI |
| `globs` | string or array of strings | Gitignore-style glob patterns for auto-attachment |
| `alwaysApply` | boolean | If `true`, injected into every conversation |

**Rule types (derived from field combinations):**

| Type | `alwaysApply` | `globs` | `description` | Behavior |
|---|---|---|---|---|
| Always | `true` | (ignored) | optional | Injected into every conversation |
| Auto Attached | `false` | set | optional | Auto-attached when matching file is open/referenced |
| Agent Requested | `false` | unset | set | AI agent pulls it in when it deems it relevant |
| Manual | `false` | unset | unset | Only via `@ruleName` in chat |

**Important limitation:** Subdirectory support within `.cursor/rules/` is unreliable — rules in
`.cursor/rules/subfolder/rule.mdc` do not work reliably (known bug, multiple reports, no confirmed
fix as of early 2026). Files must be flat under `.cursor/rules/`.

The `@` reference syntax inside rule bodies: writing `@src/components/Button.tsx` in the rule body
includes that file's contents when the rule fires.

**Sources:** [Cursor Rules Docs](https://docs.cursor.com/context/rules), [Community deep-dive forum](https://forum.cursor.com/t/a-deep-dive-into-cursor-rules-0-45/60721), [Cursor 1.7 changelog](https://cursor.com/changelog/1-7)

---

### Other Agents (Windsurf, Cline, Copilot, Zed, Amp)

**Windsurf:**
- Project: `.windsurf/rules/*.md` — YAML frontmatter with `trigger` (`always_on`, `manual`, `model_decision`, `glob`), `description`, `globs`
- Legacy: `.windsurfrules` (project root, no frontmatter)
- Global: `global_rules.md` (managed in Windsurf app settings, not a filesystem path)
- Limits: 6,000 chars per rule file; 12,000 chars combined

**Cline:**
- Project: `.clinerules` (single file) or `.clinerules/*.md` (directory, v3.7.0+)
- Global: `~/Documents/Cline/Rules/`
- Format: Plain Markdown. No frontmatter.

**GitHub Copilot:**
- Repo-wide: `.github/copilot-instructions.md` (plain Markdown, no frontmatter, always applied)
- Path-scoped: `.github/instructions/*.instructions.md` (YAML frontmatter: `applyTo` glob string, `excludeAgent`)
- Global: VS Code `settings.json` `github.copilot.chat.codeGeneration.instructions`

**Zed:**
- Reads a single project-level file, first match from priority list: `.rules`, `.cursorrules`, `.windsurfrules`, `.clinerules`, `.github/copilot-instructions.md`, `AGENT.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`
- No path-scoped rules; no global file (uses in-app Rules Library UI for named personal rules)

**Amp:**
- Project: `AGENTS.md` at root and subdirectories; `AGENT.md` as fallback
- Global: `~/.config/AGENTS.md`
- Subtree scoping: `AGENTS.md` in subdirectories included when agent reads files there
- Referenced files support frontmatter: `globs` (array), `alwaysApply` (boolean), `description` (string)

---

## Comparison Table — Target Providers

| Feature | Claude Code | OpenCode | Codex CLI | Cursor |
|---|---|---|---|---|
| Primary file | `.claude/rules/*.md` | `AGENTS.md` | `AGENTS.md` | `.cursor/rules/*.mdc` |
| Secondary/legacy | `CLAUDE.md` | `CLAUDE.md` (fallback) | — | `.cursorrules` (deprecated) |
| Global scope | `~/.claude/rules/` | `~/.config/opencode/AGENTS.md` | `~/.codex/AGENTS.md` | App settings UI |
| Project scope | `./.claude/rules/` | `AGENTS.md` (upward walk) | `AGENTS.md` (downward walk) | `.cursor/rules/` |
| Frontmatter | `paths:` (globs array) | None (native); rich via plugin | None | `description`, `globs`, `alwaysApply` |
| Scoping mechanism | Glob patterns on file read | Directory tree position | Directory tree position | Glob + always + agent-request |
| Subdirectory support | Yes (recursive scan) | Via directory traversal | Via directory traversal | **No** (known bug) |
| Remote/URL rules | No | Yes (via `opencode.json`) | No | No |
| Plugin rules | No | Yes (`opencode-rules`) | No | No |
| Execution policy rules | No | No | Yes (Starlark `.rules`) | No |

---

## Sources

- [Claude Code — How Claude remembers your project](https://code.claude.com/docs/en/memory)
- [Claude Code — Settings](https://code.claude.com/docs/en/settings)
- [paddo.dev — Claude Code gets path-specific rules](https://paddo.dev/blog/claude-rules-path-specific-native/)
- [Claude Code Issue #21858 — paths in user-level rules ignored](https://github.com/anthropics/claude-code/issues/21858)
- [OpenCode — Rules](https://opencode.ai/docs/rules/)
- [OpenCode — Config](https://opencode.ai/docs/config/)
- [opencode-rules plugin](https://github.com/frap129/opencode-rules)
- [Codex CLI — Custom instructions with AGENTS.md](https://developers.openai.com/codex/guides/agents-md)
- [Codex CLI — Rules (execution policy)](https://developers.openai.com/codex/rules/)
- [Codex CLI — execpolicy README](https://github.com/openai/codex/blob/main/codex-rs/execpolicy/README.md)
- [Cursor — Rules docs](https://docs.cursor.com/context/rules)
- [Cursor — Deep dive forum post](https://forum.cursor.com/t/a-deep-dive-into-cursor-rules-0-45/60721)
- [Cursor 1.7 changelog](https://cursor.com/changelog/1-7)
- [Windsurf — Cascade Memories Docs](https://docs.windsurf.com/windsurf/cascade/memories)
- [Cline — Rules docs](https://docs.cline.bot/features/cline-rules)
- [GitHub Copilot — Custom instructions](https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot)
- [Zed — AI rules docs](https://zed.dev/docs/ai/rules)
- [Amp — AGENT.md announcement](https://ampcode.com/news/AGENT.md)
- [Amp — Globs in AGENTS.md](https://ampcode.com/news/globs-in-AGENTS.md)
- [lbb00/ai-rules-sync — cross-agent rules sync tool](https://github.com/lbb00/ai-rules-sync)
- [roderik/ai-rules — example shared rules for multiple agents](https://github.com/roderik/ai-rules)
- [AGENTS.md interoperability issue #1624](https://github.com/openai/codex/issues/1624)

---

## Key Takeaways

1. **All four target providers support rules.** Claude Code and Cursor use provider-specific
   directories (`.claude/rules/`, `.cursor/rules/`). OpenCode and Codex CLI both use `AGENTS.md`
   plain Markdown with no frontmatter.

2. **Cursor's `.mdc` frontmatter is the richest scoping system** among the target providers:
   `description` + `globs` + `alwaysApply` → four rule activation modes. It's also the only
   target provider with a dedicated file extension (`.mdc`).

3. **Claude Code's `paths:` frontmatter is a simplified version of Cursor's `globs:`** — same
   concept, but only that one field (and with several active bugs at the user-level scope).

4. **OpenCode and Codex use plain-markdown AGENTS.md** — no frontmatter, scoping via directory
   placement only. The `opencode-rules` third-party plugin adds richer frontmatter for OpenCode if
   needed, but it's not official.

5. **`AGENTS.md` is becoming the de-facto cross-agent standard** for instruction content. OpenCode
   reads it natively. Codex CLI is its origin. Amp, Zed, and others also consume it. This is
   relevant if we want agent-config rules to also be usable across tools.

6. **For `agent-config` purposes:** A rule spec likely needs to support:
   - A `paths`/`globs` field for Claude Code and Cursor scoping
   - An `alwaysApply` boolean for Cursor
   - A `description` string for Cursor "Agent Requested" mode
   - A way to distinguish between Claude Code `.claude/rules/` output, Cursor `.cursor/rules/`
     output, and OpenCode/Codex `AGENTS.md` aggregation

7. **Codex's Starlark `.rules` execution policy is a completely different concept** — not an
   instruction file. It has no analog in other agents and should likely be out of scope for
   agent-config rules generation (or treated as a completely separate artifact type).

---

## Open Questions

1. **How should scoping fields map across providers?** Cursor's `globs` and Claude Code's `paths`
   are semantically similar but syntactically distinct field names. Should the canonical spec use
   a single `globs` field that maps to `paths:` for Claude Code and `globs:` for Cursor?

2. **Should OpenCode/Codex rules aggregate into AGENTS.md, or stay separate?** Since OpenCode and
   Codex consume plain AGENTS.md with no frontmatter, individual rule files would need to be
   stripped of frontmatter and concatenated, or emitted as separate files into the `instructions`
   array in `opencode.json`.

3. **How should `alwaysApply` map to Claude Code?** Claude Code has no `alwaysApply` equivalent —
   a rule without `paths:` is always applied. So `alwaysApply: true` in the canonical spec could
   simply mean "omit the paths: field in Claude Code output."

4. **What is the right canonical frontmatter schema for a rule spec?** Candidate fields:
   `id`, `description`, `version`, `globs` (array), `alwaysApply` (boolean), with
   `compat.targets` controlling which providers the rule is generated for.

5. **Should Codex execution policy (Starlark `.rules`) be in scope?** It's a categorically
   different artifact from instruction files. Worth a separate discussion before including.

6. **How should Windsurf's `trigger` field map?** Windsurf's `trigger: model_decision` is
   analogous to Cursor's "Agent Requested" mode. If Windsurf is added as a future target, this
   would need a mapping in `features-mapping.yaml` or a new field.
