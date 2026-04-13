# Add Frontmatter Name Prefixing for Cursor Agents and Skills

## Problem Statement

The Cursor adapter applies file-path prefixing (e.g., `agents/tw-my-agent.md`)
but does not prefix the frontmatter `name` field. Claude's adapter already
handles both — file path and frontmatter name — making Cursor the only adapter
with incomplete prefix support.

When multiple spec libraries sync into the same Cursor workspace, agents and
skills from different libraries can collide on `name`. File-path prefixing
prevents filename collisions, but the user-facing name in Cursor's UI remains
ambiguous without a namespace prefix.

## Motivation

- **Namespace consistency**: Users who configure a prefix expect it to apply
  everywhere — file paths and display names. Claude already does this; Cursor
  should too.
- **Disambiguation in Cursor's UI**: Without name prefixing, two libraries
  syncing an agent called `code-review` would show identical names in Cursor's
  agent picker despite having different file paths.
- **Parity across adapters**: Reducing behavioral differences between adapters
  makes the prefix feature easier to reason about and document.

## Context

The agentspec compile pipeline dispatches each `(spec, provider)` pair to a
provider adapter. Adapters receive an `Option<&AdapterConfig>` containing
`prefix: Option<String>` and `strip_name: bool`.

Current state by adapter:

| Transform         | Claude Agent | Claude Skill | Cursor Agent | Cursor Skill |
|-------------------|--------------|--------------|--------------|--------------|
| File path prefix  | Yes          | Yes          | Yes          | Yes          |
| Name prefix       | Yes (`:`)    | Yes (`:`)    | **No**       | **No**       |
| Name strip        | N/A          | Yes          | N/A          | No-op        |

Claude uses `:` as the name prefix delimiter (`tw:my-agent`). The Cursor adapter
should use `-` (`tw-my-agent`), matching the file-path convention.

Relevant code locations:
- `src/adapters/cursor.rs` lines 51 (agent name) and 87 (skill name) — both
  assign `name` directly from `id` with no prefix logic
- `src/adapters/claude.rs` lines 122-125 (agent) and 179-186 (skill) — full
  prefix implementation to model from

## Goals / Success Criteria

- [ ] Cursor agent adapter prefixes frontmatter `name` with `-` delimiter when
      `prefix` is configured (e.g., `tw-my-agent`)
- [ ] Cursor skill adapter prefixes frontmatter `name` with `-` delimiter when
      `prefix` is configured (e.g., `tw-my-skill`)
- [ ] `strip_name` remains a no-op for Cursor (Cursor requires a name field)
- [ ] Prefix delimiter is hardcoded per adapter (`:` for Claude, `-` for Cursor),
      not user-configurable
- [ ] Unit tests cover prefixed name output for both Cursor agents and skills
- [ ] Existing Claude prefixing behavior is unchanged

## Non-Goals (Out of Scope)

- Making the prefix delimiter configurable in `agentspec.toml`
- Changing `strip_name` behavior for Cursor
- Adding name prefixing for the OpenCode adapter (separate concern)
- Modifying Claude's existing `:` delimiter

## Constraints

- The `-` delimiter means the prefixed name (`tw-my-agent`) is identical to the
  prefixed filename (`tw-my-agent.md`). This is intentional and accepted.
- Cursor requires a `name` field in frontmatter — it cannot be omitted.

## Open Questions

- None — this is a straightforward parity feature with clear design decisions
  already made.

## References

- TODO.md item #8
- `src/adapters/claude.rs` — reference implementation for prefix transforms
- `src/adapters/cursor.rs` — target for changes
- Prior idea: `2026-03-23-agentspec-sync-namespace-prefix.md` — original prefix
  feature design
