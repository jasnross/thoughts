# Move `/learn` Skill Storage to `thoughts/` Directory

## Problem Statement

The `/learn` skill currently writes captured insights to two locations:
`<project-root>/agent-docs/` for project-scoped learnings and
`~/.config/agent-docs/` for global learnings. Neither location reliably follows
the user across machines. Project-level `agent-docs/` directories are only
present when checked into the specific repo, and `~/.config/agent-docs/` lives
outside any version-controlled dotfiles sync.

Additionally, the per-project directory model is brittle: project roots can
move, two projects can share the same directory name, and projects without a
GitHub org have no natural namespace. This makes the scoping model harder to
reason about and maintain over time.

## Motivation

- Learnings captured today should be available in a fresh shell tomorrow, on a
  different machine, without manual syncing.
- The `thoughts/` directory is already in a git-tracked dotfiles repo and
  follows the user everywhere.
- A flat, topic-based structure is simpler to browse and reason about than a
  nested project hierarchy.
- Removing per-project directory scoping eliminates a whole class of ambiguity
  around project identity (same name, no org, moved roots).

## Context

The current `/learn` skill resolves scope per-insight using a walk-up algorithm
that finds `AGENTS.md`/`CLAUDE.md` markers or git repo roots. This produces
either a `project:<root>` or `global` scope, driving the write destination.

The `learnings-discoverer` subagent currently searches
`<project-root>/agent-docs/` and `~/.config/agent-docs/`. It is referenced in
`~/.claude/CLAUDE.md` and dispatched at the start of tasks.

Existing files that would need migration:

- `/Users/jasonr/dotfiles/agent-config/agent-docs/generated-artifacts.md`
- `/Users/jasonr/dotfiles/agent-config/agent-docs/compiler-fragments.md`
- `/Users/jasonr/dotfiles/agent-config/agent-docs/canonical-command-conventions.md`
- `/Users/jasonr/.config/agent-docs/git-commit-strategies.md`
- `/Users/jasonr/.config/agent-docs/git-integration-workflows.md`

## Goals / Success Criteria

- [ ] `/learn` writes all insights to `$THOUGHTS_DIR/learnings/` as flat
      topic files (e.g., `auth-patterns.md`, `database-gotchas.md`)
- [ ] Project context is captured as inline prose within each insight (e.g.,
      "In `myorg/myapp`, the auth middleware...") rather than as directory
      structure
- [ ] The skill uses the walk-up `$THOUGHTS_DIR` discovery algorithm (same as
      `create-idea`) to locate the learnings directory — no reliance on env var
      being pre-set
- [ ] `learnings-discoverer` subagent is updated to look in
      `$THOUGHTS_DIR/learnings/` instead of project/global `agent-docs/` paths
- [ ] `~/.claude/CLAUDE.md` knowledge base section is updated to reference the
      new location
- [ ] Existing `agent-docs/` files are migrated into `$THOUGHTS_DIR/learnings/`
      with project context added inline where appropriate
- [ ] The per-project scope resolution algorithm is removed from the learn skill
      (no more `project:<root>` vs `global` distinction)

## Non-Goals (Out of Scope)

- Changing the file format (H1 topic title, H2 insights, `<!-- learned: -->` date comment)
- Changing how insights are identified or what counts as a good learning
- Removing the confirmation step before writing
- Splitting or reorganizing topic files (that's a separate concern)

## Proposed Solution

Replace the dual-destination model with a single flat destination:
`$THOUGHTS_DIR/learnings/`. The scope resolution algorithm is replaced with
the `$THOUGHTS_DIR` discovery algorithm (walk up from workspace root looking for
a `thoughts/` directory). All insights go to the same place; project context is
expressed in the insight prose itself.

The `learnings-discoverer` subagent needs a corresponding update to its search
path. Since `$THOUGHTS_DIR` may not be set in the subagent's environment, it
should run the same walk-up discovery to find `thoughts/learnings/`.

The discovery algorithm could be extracted into a shared fragment
(`spec/fragments/thoughts/discover-dir.md`) to avoid duplicating it across
multiple skills.

### Migration Script

Migration should be a shell script committed to dotfiles rather than a Claude
Code skill. The source project is derivable directly from the file path, so no
LLM judgment is needed — the operation is mechanical:

1. Walk known `agent-docs/` source directories
2. For each `.md` file, derive the project name from the path
3. Prepend a `<!-- source: <project-name> -->` comment to each insight section
4. Copy into `$THOUGHTS_DIR/learnings/`, concatenating into an existing topic
   file if one already exists at the destination
5. Skip files already migrated (idempotent — safe to run on multiple machines)

A script is preferable to a skill here because it runs without Claude Code,
is self-documenting in dotfiles, and can be invoked in automation or on a
fresh machine before any agent session.

## Constraints

- The `$THOUGHTS_DIR` discovery algorithm must work in fresh sessions where the
  env var is not pre-exported
- The change to `learnings-discoverer` must be coordinated with the learn skill
  change so there's no window where new learnings are written somewhere the
  discoverer can't find them
- Migration of existing `agent-docs/` files should happen atomically (write new,
  then remove old) to avoid duplicate discovery

## Open Questions

- Should the old `agent-docs/` directories be deleted after migration, or left
  in place with a deprecation notice?
- Should `learnings-discoverer` fall back to old locations during a transition
  period, or cut over immediately?
- Is there a shared fragment for `$THOUGHTS_DIR` discovery already, or does one
  need to be created?

## References

- Current learn skill: `spec/skills/learn/SKILL.md`
- Current create-idea skill (has `$THOUGHTS_DIR` discovery algorithm):
  `~/.claude/skills/create-idea/`
- `learnings-discoverer` subagent: `spec/agents/learnings-discoverer/`
- Existing agent-docs: `agent-config/agent-docs/`, `~/.config/agent-docs/`

## Affected Systems/Services

- `spec/skills/learn/SKILL.md` — primary change
- `spec/agents/learnings-discoverer/` — update search paths
- `~/.claude/CLAUDE.md` — update knowledge base section
- `spec/fragments/` — possibly add a shared `thoughts/discover-dir` fragment
- Existing `agent-docs/` files — migration
