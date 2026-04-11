# agentspec sync command

## Problem Statement

Today, `agentspec` compiles provider-neutral specs into provider-specific
configs under `generated/<provider>/`, but the "last mile" — placing those files
into each tool's expected location — is handled by a separate shell script
(`agent-config/setup.sh`). This creates a split responsibility: the Rust tool
knows how to generate files but has no concept of where they should ultimately
live, while the shell script encodes destination knowledge, symlink management,
stale-link cleanup, and profile-conditional logic in ~375 lines of Bash.

This split makes the workflow fragile and hard to extend. Adding a new provider
or changing a sync target means editing both the Rust compiler and the shell
script. The shell script also hardcodes paths and profile-specific branching
that would be better expressed as structured configuration.

## Motivation

- **Single tool, single workflow**: `agentspec sync` should compile and
  distribute in one step, replacing the current two-tool dance.
- **Portable configuration**: Sync targets, strategies (symlink vs copy), and
  paths should be expressed in `agentspec.toml` — not scattered across shell
  conditionals.
- **Profile-driven flexibility**: The same spec set should sync to different
  destinations depending on the active profile (e.g., user-level config at home,
  plugin directory at work) without separate code paths.
- **Correctness**: Rust can handle stale-link cleanup, path validation, and
  conflict detection more reliably than shell scripting.

## Context

### Current flow

```
agentspec compile          # generates files to generated/<provider>/
agent-config/setup.sh      # distributes generated files + static files
```

### setup.sh sync targets (for reference)

| Tool      | Condition          | Target                              | Method  |
|-----------|--------------------|-------------------------------------|---------|
| Claude    | profile != work    | ~/.claude/{agents,skills}/          | symlink |
| Cursor    | always             | ~/.cursor/skills/                   | symlink |
| Codex     | always             | ~/.codex/skills/                    | symlink |
| OpenCode  | always             | ~/.config/opencode/{agents,cmds,skills}/ | symlink |
| Plugin    | profile == work    | ~/Workspace/thoughts/plugin/        | copy    |

### What stays in setup.sh

The `sync` command replaces **only the distribution of generated files**. These
concerns remain outside agentspec:

- Profile resolution prompting (interactive menu)
- Settings file merges (e.g., `default-settings.json` into `settings.json`)
- Static file symlinking (e.g., `keybindings.json`, `AGENTS_GLOBAL.md`)

setup.sh will shrink significantly but will not be fully eliminated.

## Goals / Success Criteria

- [ ] `agentspec sync` compiles and distributes generated output in a single
      invocation
- [ ] Three sync target modes are supported: user-level (tool home dirs),
      project-local (`.claude/`, `.cursor/` etc. in a project root), and
      arbitrary paths (e.g., plugin copy targets)
- [ ] All spec kinds (agents, skills, rules, commands) can be synced to any
      target mode — no artificial restrictions by target type
- [ ] Sync targets (destination path, strategy, spec kinds) are configurable
      per-provider in `agentspec.toml`
- [ ] Profiles can override sync targets (e.g., home profile syncs to user-level
      dirs, work profile syncs to a plugin directory)
- [ ] CLI flags override config/profile for ad-hoc usage
- [ ] Stale file/symlink cleanup: files in the target that came from a previous
      sync but are no longer in generated output are removed
- [ ] `--no-compile` flag to skip compilation and sync from existing generated
      output
- [ ] `--dry-run` flag shows what would be synced without making changes
- [ ] Copy strategy uses a per-target `.agentspec-manifest.json` to track
      owned files and detect stale entries
- [ ] On first sync (no manifest entry), a pre-existing file is backed up before
      being overwritten; on subsequent syncs, warn and overwrite
- [ ] `sync` does not gate on tool detection — it syncs to whatever targets
      are configured, unconditionally

## Non-Goals (Out of Scope)

- Replacing setup.sh's settings merge logic (yq-based JSON merging)
- Replacing setup.sh's static file management (keybindings, AGENTS_GLOBAL.md)
- Interactive profile prompting inside agentspec (profile comes from
  `--profile` flag or `AGENTSPEC_PROFILE` env var, same as today)

## Proposed Solution

### Target modes

A sync target has a **mode** that determines where files land relative to the
project and provider:

- `user` — tool's home config directory (e.g., `~/.claude/`). Default.
- `project` — project-local config directory (e.g., `./.claude/`). Useful when
  running `agentspec sync` from within a project that has its own
  `agentspec.toml`.
- `path` — explicit destination path. Required for plugin targets.

### Sync target configuration in agentspec.toml

Targets are configured per-provider. Profile overrides follow the same
merge-over-base pattern as model presets.

```toml
# Default sync targets (user-level config directories)
[sync.claude]
mode = "user"
strategy = "symlink"

[sync.cursor]
mode = "user"
strategy = "symlink"

[sync.codex]
mode = "user"
strategy = "symlink"

[sync.opencode]
mode = "user"
strategy = "symlink"

# Work profile: override to copy to a plugin directory
[profiles.work.sync.claude]
mode = "path"
strategy = "copy"
agents = "~/Workspace/thoughts/plugin/agents"
skills = "~/Workspace/thoughts/plugin/skills"
rules = "~/Workspace/thoughts/plugin/rules"

[profiles.work.sync.cursor]
mode = "path"
strategy = "copy"
skills = "~/Workspace/.cursor/skills"
```

For `mode = "path"`, each spec kind has an explicit destination directory. For
`mode = "user"` and `mode = "project"`, destinations are derived from
provider conventions (e.g., Claude user → `~/.claude/agents`, `~/.claude/skills`,
etc.).

### CLI interface

```
agentspec sync [OPTIONS]

Options:
  --target <providers>       Comma-separated provider filter
  --profile <name>           Profile overlay (or AGENTSPEC_PROFILE env)
  --mode <user|project|path> Override target mode for all providers
  --dest <path>              Override destination root (for mode=path)
  --strategy <symlink|copy>  Override sync strategy
  --no-compile               Skip compilation, sync from existing output
  --dry-run                  Show what would be synced without making changes
  --strict                   Treat warnings as errors
```

### Sync behavior

1. Run `compile` (unless `--no-compile`)
2. For each targeted provider:
   a. Resolve sync target config (base → profile → CLI overrides)
   b. Resolve destination paths from mode/config
   c. For each spec kind (agents, skills, rules, commands):
      - **Symlink strategy**: create symlinks from target to generated files;
        replace existing symlinks pointing elsewhere; back up regular files
        before replacing
      - **Copy strategy**: check `.agentspec-manifest.json` in target dir;
        if no manifest entry for a file, back up any pre-existing file before
        overwriting; write the copy; update manifest. If manifest entry exists
        and content differs, warn and overwrite.
      - **Stale cleanup**: remove symlinks pointing to no-longer-generated
        files; for copy strategy, remove files listed in manifest that are no
        longer in generated output
3. Write/update `.agentspec-manifest.json` in each copy-strategy target dir
4. Report what was synced, created, updated, backed up, and cleaned up

## Constraints

- Must work on macOS (primary) and Linux
- Must not break existing `compile` / `check` / `validate` commands
- Profile resolution stays external — agentspec receives the profile name, it
  does not prompt for it
- Tilde expansion (`~`) must be handled since TOML values are strings
- `agentspec sync` is unconditional — tool detection belongs in the caller
  (setup.sh or user's shell)
