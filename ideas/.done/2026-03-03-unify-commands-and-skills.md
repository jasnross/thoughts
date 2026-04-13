# Unify `spec/commands/` and `spec/skills/` into a Single `spec/skills/` Directory

## Problem Statement

The `spec/` directory currently contains two separate subdirectories — `commands/` and
`skills/` — that represent a conceptual split between user-invocable specs and
agent-invocable specs. However, this split is already leaking: `spec/skills/git-town/SKILL.md`
carries `user_invocable: false` and `spec/commands/create-idea.md` carries
`disable_model_invocation: true`, both of which are frontmatter flags that express
invocability semantics.

The result is a system where the directory name implies one thing ("this is a command" /
"this is a skill") but the frontmatter can override it. This dual-encoding creates
confusion about where the authoritative definition of invocability lives, and makes it
harder to switch a spec from one mode to another without moving files.

## Motivation

- **Single source of truth**: Invocability should be declared in frontmatter, not encoded
  in directory structure.
- **Flexibility**: Switching a spec between user-invocable and agent-invocable (or enabling
  both) should require only a frontmatter change, not a file move.
- **Reduced conceptual overhead**: Contributors shouldn't need to decide "is this a command
  or a skill?" before writing a spec. That decision is just a flag.
- **Already mostly unified**: The system already treats both kinds identically in many ways
  — same feature gate (`supports_skills`), same output path pattern, same provider adapters.
  The split is artificial.

## Context

The current structure:

- `spec/commands/<id>.md` — flat files, user-invocable, always have `execution`,
  `capabilities.tools`, `routing.trigger`, and the `command` block
  (`accepts_args`, `disable_model_invocation`, etc.)
- `spec/skills/<id>/SKILL.md` — subdirectory per skill, agent-invocable, pure knowledge
  documents with no execution config

Both kinds share: the `supports_skills` feature gate, the
`generated/<provider>/skills/<id>/SKILL.md` output path, and all four provider adapters.

`spec/agents/` is out of scope — it stays as-is.

## Goals / Success Criteria

- [ ] All specs live under `spec/skills/<id>/SKILL.md` (subdirectory per spec)
- [ ] Invocability is declared via two boolean flags in frontmatter:
  `user_invocable` and `agent_invocable`
- [ ] Both flags default to `false`; a spec with neither set to `true` is a validation error
- [ ] The slash-command trigger name is derived from `id` automatically; `routing.trigger`
  and `routing.aliases` are removed from the schema
- [ ] `execution` and `capabilities` are optional on any spec regardless of invocability
  mode (present when needed, inert when not applicable)
- [ ] `execution.mode` is removed from the schema (was silently ignored by OpenCode's skill
  system; the other three providers never used it)
- [ ] `kind` is inferred from file path (files in `spec/skills/` are skills); no `kind`
  field in skill frontmatter
- [ ] All 21 existing commands and 2 existing skills are migrated to the new structure
  with no behavioral change in generated output
- [ ] Build, validation, and schema tooling updated to enforce new conventions

## Non-Goals (Out of Scope)

- Changes to `spec/agents/` structure or semantics
- Changes to the generated output format beyond what the migration requires
- Provider-specific behavior changes
- Renaming or restructuring the `generated/` directory

## Proposed Solution

Merge `spec/commands/` into `spec/skills/` using frontmatter to express invocability:

**Before (command):**
```
spec/commands/commit.md
```
```yaml
kind: command
command:
  accepts_args: true
  disable_model_invocation: true
routing:
  trigger: commit
```

**After (unified skill):**
```
spec/skills/commit/SKILL.md
```
```yaml
user_invocable: true
agent_invocable: false
skill:
  accepts_args: true
  disable_model_invocation: true
```

**Before (skill):**
```
spec/skills/git-town/SKILL.md
```
```yaml
kind: skill
skill:
  user_invocable: false
```

**After (unified skill):**
```
spec/skills/git-town/SKILL.md
```
```yaml
user_invocable: false
agent_invocable: true
```

## Constraints

- The `command` block is renamed to `skill`. Its fields `accepts_args`, `args_schema`,
  and `delegate_to` are preserved. `disable_model_invocation` is **dropped** — the new
  top-level `user_invocable`/`agent_invocable` flags give providers sufficient signal and
  make an additional boolean redundant
- Schema conditionals currently keyed on `kind: command` need to be re-keyed on
  `user_invocable: true`
- Parsing logic (`scripts/lib/parse.ts`) currently uses separate loading paths for commands
  vs. skills; both should use the subdirectory loader after migration
- A one-time migration script will handle the mechanical work: creating subdirectories,
  renaming files to SKILL.md, and updating frontmatter fields across all 23 specs

## Open Questions

None — all open questions resolved during idea development.

## Affected Systems/Services

- `spec/commands/` — removed (all files migrated to `spec/skills/`)
- `spec/skills/` — gains all former commands as subdirectories
- `spec/schema/canonical.schema.json` — updated schema
- `scripts/lib/parse.ts` — unified loading logic
- `scripts/lib/validate.ts` — updated validation rules
- `scripts/lib/compile.ts` — `kindFeatureKey` map updated
- All provider adapters in `scripts/lib/adapters/` — updated frontmatter field handling
