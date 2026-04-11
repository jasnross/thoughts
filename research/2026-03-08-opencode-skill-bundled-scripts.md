---
date: 2026-03-08T23:29:11Z
topic: "Can OpenCode skills include bundled scripts like Claude Code (e.g., gh-safe)?"
tags: [research, web, opencode, skills, scripts, claude-compat]
status: complete
last_updated: 2026-03-08
last_updated_by: assistant
---

# Research: Can OpenCode skills include bundled scripts like Claude Code (e.g., gh-safe)?

**Date**: 2026-03-08T23:29:11Z

## Research Question

Can OpenCode include and use scripts bundled with a skill directory, similar to Claude Code skills such as `gh-safe`?

## Summary

Yes. OpenCode supports bundled skill resources (including scripts) and exposes a base directory when a skill is loaded, so relative paths like `scripts/...` are intended to work. The strongest evidence is in OpenCode's source code for the `skill` tool, which explicitly states support for bundled resources and returns "Base directory for this skill" plus guidance that relative paths are skill-directory-relative.

Two caveats came up:

1. OpenCode's public skills docs currently focus on `SKILL.md` discovery/frontmatter and do not document script execution mechanics in detail.
2. There was a real bug (reported for OpenCode 1.1.1) where relative script paths were not resolved correctly; that issue was closed by PR #11737, indicating behavior was fixed after the report.

## Detailed Findings

### OpenCode officially supports skill loading and Claude-compatible skill locations

- Official OpenCode docs show skills are discovered from `.opencode/skills/...`, `~/.config/opencode/skills/...`, and Claude-compatible locations (`.claude/skills/...`, `~/.claude/skills/...`).
- Docs state only a minimal frontmatter set is recognized (`name`, `description`, optional `license`, `compatibility`, `metadata`), and unknown frontmatter fields are ignored.

Implication: You can place Claude-style skill folders in OpenCode-discovered paths, but Claude-specific frontmatter features are not guaranteed unless explicitly implemented by OpenCode.

### OpenCode source code explicitly describes bundled resources (scripts/references/templates)

- In `packages/opencode/src/tool/skill.ts`, the skill tool description says loading a skill gives access to "bundled resources (scripts, references, templates)."
- The tool output includes:
  - a reported base directory for the skill, and
  - explicit text: "Relative paths in this skill (e.g., scripts/, reference/) are relative to this base directory."
- It also samples non-`SKILL.md` files from the skill directory and includes them in a `<skill_files>` section.

Implication: OpenCode intends skills to be multi-file bundles, not just a single markdown file.

### OpenCode does not appear to perform Claude-style content substitution in SKILL.md

- In the same `skill.ts`, OpenCode injects `skill.content.trim()` directly into the tool output, then separately appends base-directory guidance.
- This suggests OpenCode does not preprocess markdown placeholders like `${CLAUDE_SKILL_DIR}` or custom `{baseDir}` tokens inside `SKILL.md` itself.

Practical takeaway: For portability, prefer explicit relative script paths in instructions (for example, `scripts/gh-safe`), and rely on OpenCode's base-directory guidance for path resolution.

### Evidence of real-world script-path behavior and maturity

- Issue #6900 (Jan 2026) reports a bug where Claude-compatible relative script paths (e.g., `uv run scripts/tasks.py`) failed in OpenCode 1.1.1.
- The issue is closed and linked to PR #11737, indicating a fix landed.

Interpretation: Script execution with skill-relative paths is a supported direction, but version-specific behavior mattered and older OpenCode versions had edge cases.

### Comparison with Claude Code and Agent Skills standard

- Claude Code docs explicitly document bundled scripts and supporting files in skills.
- Agent Skills documentation describes scripts as a first-class part of the open skill format and recommends referencing them via relative paths from the skill root.

Interpretation: OpenCode's implementation direction is aligned with the broader Agent Skills ecosystem, but OpenCode-specific docs are less explicit than Claude's docs about script execution details.

## Sources

- [OpenCode docs: Agent Skills](https://opencode.ai/docs/skills) - Official OpenCode skill discovery/frontmatter/permissions behavior.
- [OpenCode source: `packages/opencode/src/tool/skill.ts` (dev)](https://raw.githubusercontent.com/anomalyco/opencode/dev/packages/opencode/src/tool/skill.ts) - Explicit wording about bundled resources and relative-path base directory.
- [OpenCode source: `packages/opencode/src/skill/skill.ts` (dev)](https://raw.githubusercontent.com/anomalyco/opencode/dev/packages/opencode/src/skill/skill.ts) - Skill discovery/loading behavior across `.opencode`, `.claude`, `.agents`, and configured paths.
- [OpenCode issue #6900: skill can't find scripts path](https://github.com/anomalyco/opencode/issues/6900) - Bug report on relative script resolution in v1.1.1, later closed via PR link.
- [OpenCode issue #6007: ability for skills to read references and execute scripts](https://github.com/anomalyco/opencode/issues/6007) - Community request/discussion showing historical gap and expectations.
- [Claude Code docs: Extend Claude with skills](https://docs.anthropic.com/en/docs/claude-code/skills) - Reference behavior for bundled scripts/supporting files in Claude.
- [Agent Skills: Using scripts in skills](https://agentskills.io/skill-creation/using-scripts) - Open standard guidance for script bundling and relative path conventions.

## Key Takeaways

- You can do the Claude-style "skill + script" pattern in OpenCode.
- For OpenCode compatibility, structure skills as a directory with `SKILL.md` plus `scripts/` and call scripts via skill-relative paths.
- Do not rely on Claude-specific placeholder substitution inside `SKILL.md` unless you verify OpenCode support for that exact feature.
- If behavior seems off, check your OpenCode version against known script-path fixes (issue #6900 / PR #11737).

## Open Questions

- OpenCode docs do not clearly document whether markdown variable substitutions (for example `${CLAUDE_SKILL_DIR}`-style tokens) are supported, partially supported, or intentionally unsupported.
- It is unclear from docs alone which OpenCode release first included the #6900 fix; changelog confirmation would remove version ambiguity.
