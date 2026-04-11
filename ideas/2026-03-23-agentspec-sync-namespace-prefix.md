# Agentspec Sync Namespace Prefixing

## Problem Statement

When agentspec syncs compiled skills, agents, and commands to user-level config
directories, the generated names can collide with skills/agents the user already
has. A user who has a personal `commit` skill and syncs an agentspec-managed
`commit` skill will get a silent collision — the existing file is backed up with
a timestamp suffix, but the user may not notice.

Claude Code solves this for plugins with automatic namespace prefixing (e.g.,
`tw:commit`), but agentspec's user-level sync has no equivalent mechanism. As
agentspec spec libraries grow and are shared between users, the collision risk
increases.

## Motivation

- **Name collisions are silent today.** Symlink sync backs up regular files but
  doesn't warn prominently. Copy sync with manifest tracking can overwrite
  without the user realizing a collision occurred.
- **Shared spec libraries compound the problem.** If multiple people publish
  agentspec specs with common names (`commit`, `review-pr`, `describe-pr`),
  users consuming more than one library will hit collisions between libraries,
  not just between agentspec and their own skills.
- **Claude Code plugins already have this solved.** The `plugin.json` `name`
  field creates a colon-separated namespace. Agentspec should offer something
  comparable for non-plugin sync targets.
- **OpenCode uses subdirectory-based namespacing.** Files placed in subdirectories
  under `commands/` are invoked as `/subdir/command-name`. Note: the `user:` /
  `project:` prefix convention in community docs is not implemented in the source
  — the actual separator is `/` derived from the filesystem path.

## Context

### Current Sync Behavior

Agentspec sync operates in three modes:

- **User-level** (default): syncs to `~/.claude/skills/`, `~/.cursor/rules/`, etc.
- **Project-level**: syncs to `./<tool>/skills/`, etc.
- **Explicit path**: per-kind custom destination paths

The `strip_name` option already exists for plugin contexts — it removes the
`name:` frontmatter line so the downstream plugin can apply its own prefix. This
is the inverse of the proposed feature: `strip_name` defers naming to the
plugin, while a prefix config would have agentspec apply naming itself.

### Provider Namespace Conventions

| Provider    | Separator      | Prefix Source                     | Configurable?         |
| ----------- | -------------- | --------------------------------- | --------------------- |
| Claude Code | `:` (colon)    | `plugin.json` `name` field        | Yes                   |
| OpenCode    | `/` (slash)    | Subdirectory under `commands/`    | Via directory layout  |
| Cursor      | None           | N/A (no namespacing)              | N/A                   |
| Codex       | None           | N/A (no custom commands)          | N/A                   |

Note: Community docs often describe OpenCode using `user:` / `project:` prefixes with
colon separators, but this is not implemented in the source. The actual behavior is
that subdirectory names map directly to `/`-separated command path segments.

### Collision Detection Today

- **Symlink strategy**: if target is a regular file, backs it up as
  `<name>.bak.<timestamp>` and creates symlink (`BackedUp` action)
- **Symlink strategy**: if target is a symlink pointing elsewhere, replaces it
  (`Updated` action)
- **Copy strategy**: manifest tracks ownership; unowned files are not touched
  (but name collision means the agentspec file simply isn't synced)

## Goals / Success Criteria

- [ ] Users can configure an optional namespace prefix in `agentspec.toml`
- [ ] Prefix is applied during sync to skill/agent/command names
- [ ] Prefix behavior is configurable per sync mode (user / project / path)
- [ ] Prefix separator follows provider conventions (colon for Claude Code and
      OpenCode, dash-in-filename for Cursor/Codex)
- [ ] Sync **errors** (aborts) when it detects a name collision without prefix configured
- [ ] Existing `strip_name` behavior is preserved and composable with prefix

## Non-Goals (Out of Scope)

- **Alias generation**: making both `/commit` and `/tw:commit` work is up to
  the downstream tool, not agentspec
- **Cross-library deduplication**: resolving collisions between two different
  agentspec spec libraries (this is a bigger problem that prefix helps but
  doesn't fully solve)
- **Runtime collision detection**: agentspec only checks at sync time, not at
  invocation time

## Proposed Solutions (Optional)

### Option A: Config-level prefix with per-provider separator

Add a `prefix` field to sync config, with the separator determined by provider
convention:

```toml
[sync.claude]
mode = "user"
prefix = "tw"
# Results in name: tw:commit, tw:describe-pr (colon separator, native to Claude Code)

[sync.opencode]
mode = "user"
prefix = "tw"
# Results in commands synced to ~/.config/opencode/commands/tw/commit.md → /tw/commit

[sync.cursor]
mode = "user"
prefix = "tw"
# Results in: tw-commit.mdc, tw-describe-pr.mdc (dash separator, filename-only)

[profiles.work.sync.claude]
mode = "path"
prefix = ""  # No prefix for work profile plugin path
strip_name = true
```

**Pros**: Simple, follows existing config patterns, separator is automatic per
provider.
**Cons**: Users can't override the separator character per provider.

### Option B: Configurable prefix AND separator

```toml
[sync.claude]
prefix = "tw"
separator = ":"  # default for claude

[sync.cursor]
prefix = "tw"
separator = "-"  # default for cursor (no native namespacing)
```

**Pros**: Full control.
**Cons**: More config surface, separator choice is almost always dictated by
the provider anyway.

## Constraints

- Must not break existing sync behavior when no prefix is configured
- Must compose correctly with `strip_name` (prefix + strip_name should probably
  be mutually exclusive or have clear precedence)
- Prefix must be applied to both the filename and the `name:` frontmatter field
  (where applicable)
- For OpenCode, prefix is applied as a subdirectory: syncing to
  `~/.config/opencode/commands/<prefix>/<name>.md` yields `/<prefix>/<name>`
- `prefix` and `strip_name` are mutually exclusive — `strip_name` defers naming
  to the plugin; `prefix` asserts naming. Setting both is a config error.

## Resolved Decisions

- **OpenCode interaction**: OpenCode uses subdirectory-based namespacing (not
  colon-separated `user:`/`project:` prefixes as community docs suggest). A
  prefix of `tw` means syncing into `~/.config/opencode/commands/tw/`, yielding
  `/tw/commit`. No special handling beyond directory placement is required.
- **Per-kind prefixing**: Prefix applies to **skills, agents, and commands**
  only. Rules are excluded — their names are not user-invoked and filename
  collisions in the rules directory have low impact.
- **Multiple spec libraries**: Each library has its own independent
  `agentspec.toml` and syncs independently. Prefix is a per-library setting;
  no multi-source aggregation is needed.
- **Default behavior**: No automatic default. Prefix is always opt-in via
  explicit config. Sync **errors and aborts** when a collision is detected
  and no prefix is configured, forcing the user to resolve it.

## References

- Claude Code plugin namespacing: `plugin.json` `name` field with colon separator
- OpenCode command namespacing: subdirectory path becomes `/`-separated command
  name segments (source: `packages/opencode/src/config/config.ts` `loadCommand`)
- Existing `strip_name` implementation: `agentspec/src/sync/strategy.rs` `apply_strip_name()`
- Sync config resolution: `agentspec/src/config.rs` `resolve_sync_target()`
- Related idea: `thoughts/ideas/2026-03-22-agentspec-sync-command.md`

## Affected Systems/Services

- `agentspec/src/sync.rs` — orchestrator, prefix application point
- `agentspec/src/sync/strategy.rs` — filename and frontmatter manipulation
- `agentspec/src/sync/provider.rs` — provider-specific separator logic
- `agentspec/src/config.rs` — `SyncTargetConfig` gains `prefix` field
- `agentspec.toml` schema — new `prefix` (and possibly `separator`) fields
