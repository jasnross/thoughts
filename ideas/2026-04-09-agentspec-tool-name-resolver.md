# Agentspec: Provider-Specific Tool Name Resolution in Spec Bodies

## Problem Statement

Agentspec specs often reference tools by name in their body content — for example, instructing the model to "use the `AskUserQuestion` tool" or "use the `Read` tool to inspect files." Today, these references are hardcoded to a single provider's tool names (typically Claude Code's), which means specs compiled for other providers contain tool names the model won't recognize.

The existing `ToolFrontmatter` system already maps canonical tool names to provider-specific equivalents in YAML frontmatter, but there is no mechanism for performing this translation inside the Markdown body content where instructions live.

## Motivation

- **Precision over vagueness**: Specs currently work around the problem with vague phrasing like "use your built-in question UI" instead of naming the actual tool. Provider-specific tool names are more precise and more likely to produce correct model behavior.
- **Single authoring surface**: Spec authors should write instructions once using canonical names and have them automatically resolve per provider, rather than maintaining provider-specific forks or vague language.
- **Consistency with frontmatter tools**: The canonical → provider-specific mapping already exists for frontmatter. Extending it to body content creates a unified tool vocabulary across the entire spec.

## Context

### Current State

- `ToolFrontmatter` defines 10 canonical tools: `Read`, `Write`, `Edit`, `Grep`, `Glob`, `Bash`, `WebFetch`, `WebSearch`, `Question`, `Tasks`.
- Each adapter (`claude.rs`, `cursor.rs`, `opencode.rs`) maps these canonical names to provider-specific names in frontmatter output.
- The MiniJinja template environment (`templating/fragments.rs`) currently has no custom functions, globals, or filters — only `{% include %}` for fragments with an empty render context.
- Spec body content is rendered through MiniJinja before compilation, so template syntax is already available in bodies.

### Provider Tool Name Mappings

Based on research of all three supported providers:

| Canonical Name | Claude Code         | Cursor              | OpenCode       |
|----------------|---------------------|----------------------|----------------|
| `Read`         | `Read`              | `read_file`          | `read`         |
| `Write`        | `Write`             | `edit_file`          | `write`        |
| `Edit`         | `Edit`              | `edit_file`          | `edit`         |
| `Grep`         | `Grep`              | `grep_search`        | `grep`         |
| `Glob`         | `Glob`              | `file_search`        | `glob`         |
| `Bash`         | `Bash`              | `run_terminal_cmd`   | `bash`         |
| `WebFetch`     | `WebFetch`          | —                    | `webfetch`     |
| `WebSearch`    | `WebSearch`         | `web_search`         | `websearch`    |
| `Question`     | `AskUserQuestion`   | —                    | `question`     |
| `Tasks`        | `TodoWrite` (et al) | —                    | `todowrite`    |

**Key observations:**
- Cursor lacks equivalents for `WebFetch`, `Question`, and `Tasks`.
- Claude's `Tasks` expands to multiple tools in frontmatter (`TaskCreate`, `TaskGet`, etc.), but for body references a single representative name is needed.
- `Write` and `Edit` both map to Cursor's `edit_file` — the distinction doesn't exist in Cursor's tool model.

### Proposed Syntax

A minijinja function injected into the template environment:

```
{{ tool("Question") }}
```

At compile time, this resolves to the provider-specific name:
- Claude: `AskUserQuestion`
- OpenCode: `question`
- Cursor: **compile error** (no equivalent exists)

The function outputs the raw tool name only — spec authors control formatting (e.g., wrapping in backticks: `` `{{ tool("Question") }}` ``).

## Goals / Success Criteria

- [ ] A `tool()` minijinja function is available in spec body templates
- [ ] The function accepts a canonical tool name (matching `ToolFrontmatter` variants) and returns the provider-specific name
- [ ] Compilation fails with a clear error when a spec references a tool that has no equivalent for a target provider
- [ ] The canonical tool vocabulary is shared with (or derived from) `ToolFrontmatter` — no parallel enum
- [ ] Existing specs can adopt `{{ tool("...") }}` incrementally; no breaking changes to specs that don't use it

## Non-Goals (Out of Scope)

- Expanding the canonical tool set beyond `ToolFrontmatter` (can be done later)
- Resolving tool names in frontmatter (already handled by per-adapter code)
- Provider-specific conditional blocks (e.g., `{% if provider == "cursor" %}`) — a separate, broader feature
- Wrapping output in formatting (backticks, code blocks) — spec authors handle this

## Constraints

- The MiniJinja environment is currently built in `fragments.rs` with an empty context. Adding a `tool()` function requires either passing the target provider into the template resolution stage, or deferring tool resolution to a later stage (post-template, pre-emit).
- Template resolution currently happens once for all providers. If provider context is needed, the architecture may need to resolve templates per-provider rather than once globally. Alternatively, tool resolution could be a separate post-template pass.
- The compile-time error behavior means specs using `{{ tool("Question") }}` cannot target Cursor unless Cursor gains a `Question` equivalent or the spec avoids that call conditionally.

## Open Questions

- **When in the pipeline should resolution happen?** Template resolution currently precedes compilation (and thus provider selection). The `tool()` function needs provider context, which isn't available until compile time. Options: (a) move template resolution into the per-provider compile loop, (b) make `tool()` emit a placeholder that adapters resolve, or (c) run a second template pass during compilation.
- **How should `Tasks` resolve in body content?** In Claude frontmatter, `Tasks` expands to 6 tools. For body references, should it resolve to a single representative name (e.g., `TodoWrite`), or should body references use more specific canonical names (e.g., `{{ tool("TodoWrite") }}`)?
- **Should there be a way to conditionally reference tools?** If a spec targets both Cursor and Claude, and uses `{{ tool("Question") }}`, it will fail for Cursor. Should there be a companion `{% if has_tool("Question") %}` test, or is the compile error the desired forcing function to keep specs provider-compatible?

## References

- Provider enum: `agentspec/src/provider.rs:4-24`
- ToolFrontmatter enum: `agentspec/src/spec.rs:190-203`
- Claude tool mapping: `agentspec/src/adapters/claude.rs:219-239`
- OpenCode tool mapping: `agentspec/src/adapters/opencode.rs:230-243`
- Cursor adapter (no tool mapping): `agentspec/src/adapters/cursor.rs`
- MiniJinja environment setup: `agentspec/src/templating/fragments.rs:79-91`
- Template resolution entry point: `agentspec/src/templating.rs:32-37`
- Compile stage (adapter dispatch): `agentspec/src/compile.rs:97-125`

## Affected Systems/Services

- `agentspec` compiler — template resolution and compilation stages
- All provider adapters — must expose tool name mappings for body-level resolution
- Existing specs — optional migration to use `{{ tool("...") }}` syntax
