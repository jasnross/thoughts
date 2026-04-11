# Unify `spec/commands/` and `spec/skills/` Implementation Plan

## Overview

Merge the `spec/commands/` directory into `spec/skills/` so that all specs live
under `spec/skills/<id>/SKILL.md`. Invocability is declared entirely in frontmatter
via two boolean flags (`user_invocable`, `agent_invocable`). The `kind` field is
inferred from source path. The `command` frontmatter block is renamed `skill`.
The `routing` and `name` (when equal to id) fields are removed. The `"command"` value
is removed from the `execution.mode` enum; the field itself is kept for agents.

## Current State Analysis

- `spec/commands/<id>.md` — 21 flat files, each with `kind: command`, a `command`
  block, `routing.trigger`, and `execution.mode: command`
- `spec/skills/<id>/SKILL.md` — 2 subdirectory specs, each with `kind: skill` and
  `skill.user_invocable: false`
- Both kinds share: `supports_skills` feature gate, identical output path pattern
  `generated/<provider>/skills/<id>/SKILL.md`, and all four provider adapters

## Desired End State

All 23 specs live at `spec/skills/<id>/SKILL.md`. Each declares:

```yaml
user_invocable: true|false
agent_invocable: true|false
```

A spec with both false is a schema validation error. No `kind`, no `routing`,
no `execution.mode: command` in skill frontmatter (the field is removed from skills
only; agents retain it for `mode: subagent` / `mode: primary`). Invocation-specific config lives in
the `skill` block (renamed from `command`). All build tooling updated to match.

### Key Discoveries

- `routing.trigger`/`routing.aliases` (`parse.ts:93-96`, `types.ts:31-34`) are
  normalized but **never read by any adapter** — safe to drop with zero risk
- `execution.mode` (`opencode.ts:59-61`) is emitted by OpenCode but OpenCode's
  own SKILL.md schema does not recognise it (silently ignored). Dead output.
- The `supports_mode: true` feature flag (`mappings/features.yaml:14`) is
  similarly dead and should be removed
- The `skill.user_invocable` path in the Claude adapter (`claude.ts:37-39`) reads
  `spec.skill?.user_invocable`; after migration this becomes top-level
  `spec.user_invocable`
- `spec.command?.disable_model_invocation` is dropped from the canonical spec (superseded
  by `agent_invocable`); the Claude adapter continues to emit `disable-model-invocation: true`
  for specs where `agent_invocable: false`, derived from the new flag. OpenCode, Codex, and
  Cursor do not emit this field at all (unchanged behavior)
- No tests exist in the project beyond the build pipeline itself
  (`npm run build:agents`); success criteria are therefore build + generated-file
  stability

## What We're NOT Doing

- No changes to `spec/agents/` structure, format, or semantics
- No changes to provider-generated output paths
- No changes to the four mapping files beyond removing `supports_mode`
- No provider-specific behaviour changes (e.g. how Claude resolves model names)
- No new features beyond what the unification requires
- No removal of the `name` field from the schema (kept as optional for future use;
  dropped from spec frontmatter during migration since it always equals `id`)

## Implementation Approach

All phases must land together since the build is all-or-nothing (schema, types,
parse, validate, adapters, and spec files are interdependent). The migration script
(Phase 1) transforms all spec files first so subsequent phases can be developed
and tested against the new format.

---

## Phase 1: Spec File Migration Script

### Overview

Write and run a one-time TypeScript migration script that:
1. Moves all 21 `spec/commands/<id>.md` → `spec/skills/<id>/SKILL.md`
2. Updates their frontmatter to the new format
3. Updates the 2 existing `spec/skills/*/SKILL.md` frontmatters in place

### Changes Required

#### 1. Migration Script

**File**: `scripts/migrate_commands_to_skills.ts` (new, to be deleted after use)

**Before/after for a former command with `disable_model_invocation: true`** (e.g.
`spec/commands/create-idea.md` → `spec/skills/create-idea/SKILL.md`):

```yaml
# BEFORE
id: create-idea
kind: command          # ← remove (inferred from path)
name: create-idea      # ← remove (equals id; normalizer defaults name to id)
description: Flesh out a rough idea into a structured problem statement
version: 1
execution:
  model_profile: planning
  mode: command        # ← remove ("command" value dropped from schema)
capabilities:
  tools: [read, write, todowrite]
command:               # ← rename block to `skill`
  accepts_args: true
  args_schema: string
  disable_model_invocation: true   # ← drop entirely (superseded by agent_invocable)
routing:               # ← remove (entire block dropped)
  trigger: create-idea
compat:
  targets: [claude, opencode, codex, cursor]

# AFTER
id: create-idea
description: Flesh out a rough idea into a structured problem statement
version: 1
user_invocable: true   # ← new top-level flag
agent_invocable: false # ← derived: disable_model_invocation was true → false
execution:
  model_profile: planning
capabilities:
  tools: [read, write, todowrite]
skill:                 # ← renamed from `command`; disable_model_invocation not carried over
  accepts_args: true
  args_schema: string
compat:
  targets: [claude, opencode, codex, cursor]
```

**Before/after for a former command without `disable_model_invocation`** (e.g.
`spec/commands/commit.md` → `spec/skills/commit/SKILL.md`):

```yaml
# BEFORE
id: commit
kind: command          # ← remove
name: commit           # ← remove
description: Create git commits with user approval and no assistant attribution
version: 1
execution:
  model_profile: balanced
  mode: command        # ← remove
capabilities:
  tools: [bash]
command:               # ← rename block to `skill`
  accepts_args: false
  args_schema: string
  # no disable_model_invocation → agent_invocable: true
routing:
  trigger: commit
compat:
  targets: [claude, opencode, codex, cursor]

# AFTER
id: commit
description: Create git commits with user approval and no assistant attribution
version: 1
user_invocable: true
agent_invocable: true  # ← derived: disable_model_invocation absent → true
execution:
  model_profile: balanced
capabilities:
  tools: [bash]
skill:
  accepts_args: false
  args_schema: string
compat:
  targets: [claude, opencode, codex, cursor]
```

**Before/after for an existing skill** (e.g. `spec/skills/gh-safe/SKILL.md`,
updated in place):

```yaml
# BEFORE
id: gh-safe
kind: skill            # ← remove (inferred from path)
description: …
version: 1
skill:
  user_invocable: false  # ← move to top-level; remove from skill block
compat:
  targets: [claude, cursor]

# AFTER
id: gh-safe
description: …
version: 1
user_invocable: false    # ← top-level flag
agent_invocable: true    # ← top-level flag
compat:
  targets: [claude, cursor]
```

Script pseudocode:

```typescript
import path from "node:path";
import fs from "node:fs/promises";
import matter from "gray-matter";

const BASE = path.resolve(".");

async function migrateCommands() {
  const commandsDir = path.join(BASE, "spec/commands");
  const entries = await fs.readdir(commandsDir);
  for (const file of entries) {
    if (!file.endsWith(".md")) continue;
    const src = path.join(commandsDir, file);
    const raw = await fs.readFile(src, "utf8");
    const parsed = matter(raw);
    const fm = parsed.data;
    const id = fm.id as string;

    // Build new frontmatter
    // agent_invocable is derived from disable_model_invocation: if the command previously
    // set disable_model_invocation: true, only users can invoke it; otherwise the model
    // could also invoke it and agent_invocable should remain true.
    const disableModelInvocation = fm.command?.disable_model_invocation === true;
    const knownFields = new Set(["id","kind","name","description","version","execution","capabilities","command","routing","compat","provider_overrides"]);
    const unknownFields = Object.keys(fm).filter((k) => !knownFields.has(k));
    if (unknownFields.length) {
      console.warn(`WARNING: ${file} has unknown frontmatter fields that will be dropped: ${unknownFields.join(", ")}`);
    }
    const newFm: Record<string, unknown> = {
      id,
      description: fm.description,
      version: fm.version,
      user_invocable: true,
      agent_invocable: !disableModelInvocation,
    };
    if (fm.execution?.model_profile) {
      newFm.execution = { model_profile: fm.execution.model_profile };
      if (fm.execution.temperature !== undefined)
        (newFm.execution as Record<string, unknown>).temperature = fm.execution.temperature;
    }
    if (fm.capabilities?.tools?.length) newFm.capabilities = { tools: fm.capabilities.tools };
    const cmd = fm.command ?? {};
    const skillBlock: Record<string, unknown> = {};
    if (cmd.accepts_args !== undefined) skillBlock.accepts_args = cmd.accepts_args;
    if (cmd.args_schema) skillBlock.args_schema = cmd.args_schema;
    if (cmd.delegate_to) skillBlock.delegate_to = cmd.delegate_to;
    // disable_model_invocation intentionally dropped — superseded by agent_invocable
    if (Object.keys(skillBlock).length) newFm.skill = skillBlock;
    if (fm.compat) newFm.compat = fm.compat;
    if (fm.provider_overrides) newFm.provider_overrides = fm.provider_overrides;

    const destDir = path.join(BASE, "spec/skills", id);
    await fs.mkdir(destDir, { recursive: true });
    const dest = path.join(destDir, "SKILL.md");
    await fs.writeFile(dest, matter.stringify(parsed.content, newFm));
    await fs.unlink(src);
    console.log(`Migrated: ${file} → spec/skills/${id}/SKILL.md`);
  }
  await fs.rmdir(commandsDir); // will fail if non-empty, which is what we want
}

async function migrateExistingSkills() {
  const skillsDir = path.join(BASE, "spec/skills");
  for (const skillId of await fs.readdir(skillsDir)) {
    const skillFile = path.join(skillsDir, skillId, "SKILL.md");
    let raw: string;
    try { raw = await fs.readFile(skillFile, "utf8"); } catch { continue; }
    const parsed = matter(raw);
    const fm = parsed.data;
    if (fm.user_invocable !== undefined || fm.agent_invocable !== undefined) continue; // already migrated
    const wasUserInvocable = fm.skill?.user_invocable ?? false;
    delete fm.kind;
    if (fm.skill) delete fm.skill.user_invocable;
    if (fm.skill && !Object.keys(fm.skill).length) delete fm.skill;
    fm.user_invocable = wasUserInvocable;
    // Migration heuristic for the two existing skills (gh-safe, git-town): both
    // currently have user_invocable: false and are agent-only, so agent_invocable = true.
    // Using `!wasUserInvocable` produces the right answer for these two cases.
    // Note this encodes mutual exclusivity — revisit if a skill ever needs both flags true.
    fm.agent_invocable = true; // existing skills had no disable_model_invocation
    // Note: the new flags will appear at the end of frontmatter (after compat) due to
    // JavaScript object insertion order. This is functionally correct; reorder manually
    // if desired after running the script.
    await fs.writeFile(skillFile, matter.stringify(parsed.content, fm));
    console.log(`Updated: spec/skills/${skillId}/SKILL.md`);
  }
}
```

- [x] Write `scripts/migrate_commands_to_skills.ts`
- [x] Run the script: `npx tsx scripts/migrate_commands_to_skills.ts`
- [x] Verify `spec/commands/` is empty and removed
- [x] Verify all 21 former commands appear as `spec/skills/<id>/SKILL.md`
- [x] Verify the 2 existing skills have updated frontmatter
- [x] Manually inspect 3-4 migrated files to confirm frontmatter correctness:
  - One with `disable_model_invocation: true` → `agent_invocable: false` (e.g. `create-idea`)
  - One with `delegate_to` (e.g. `code-review`)
  - One without `disable_model_invocation` → `agent_invocable: true` (e.g. `commit` or `learn`)
  - One existing skill (e.g. `gh-safe`)

### Success Criteria

#### Automated Verification:
- [x] `spec/commands/` directory no longer exists
- [x] `ls spec/skills/ | wc -l` outputs `23` (21 former commands + 2 existing skills)
- [x] Every `spec/skills/*/SKILL.md` has `user_invocable:` and `agent_invocable:` at top-level
- [x] No `spec/skills/*/SKILL.md` has a `kind:` field
- [x] No `spec/skills/*/SKILL.md` has a `routing:` block
- [x] No `spec/skills/*/SKILL.md` has `execution.mode:`

#### Manual Verification:
- [ ] Spot-check 4 files as listed above confirm correct frontmatter structure

---

## Phase 2: Schema Update

### Overview

Update `spec/schema/canonical.schema.json` to match the new frontmatter format.

### Changes Required

**File**: `spec/schema/canonical.schema.json`

- [x] Remove `"kind"` from the `"required"` array (top-level)
- [x] Remove the `"kind"` property from `"properties"` (or make it optional — agents still have it, but skills won't)
- [x] Remove the `"routing"` property from `"properties"`
  > **Deviation:** `routing` was kept in the schema because agents still use it
  > and the schema has `additionalProperties: false`. Removing it caused agent
  > validation failures. Skills simply don't include this block.
- [x] In `execution.properties`, remove `"command"` and `"skill"` from the `mode`
  enum — neither value is used by any spec (agents use `"subagent"` / `"primary"`;
  `"skill"` and `"command"` are both dead). After removal the enum is `["subagent", "primary"]`
- [x] Remove the `"command"` property from `"properties"`
- [x] Update the `"skill"` property to include three fields from the former `command`
  block (`disable_model_invocation` is dropped — superseded by `agent_invocable`):
  ```json
  "skill": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "accepts_args": { "type": "boolean" },
      "args_schema":  { "type": "string" },
      "delegate_to":  { "type": "string", "pattern": "^[a-z0-9]+(?:-[a-z0-9]+)*$" }
    }
  }
  ```
- [x] Add `"user_invocable"` to top-level `"properties"`: `{ "type": "boolean" }`
- [x] Add `"agent_invocable"` to top-level `"properties"`: `{ "type": "boolean" }`
- [x] Remove the `"allOf"` array entirely — it has exactly one entry (the
  `kind: command → require command` conditional) and nothing else

### Success Criteria

#### Automated Verification:
- [x] Schema file is valid JSON: `node -e "JSON.parse(require('fs').readFileSync('spec/schema/canonical.schema.json','utf8'))"`
- [x] `npm run validate:agents` passes (schema validation step)

---

## Phase 3: TypeScript Types Update

### Overview

Update `scripts/lib/types.ts` to remove command-specific types and add the new
invocability flags.

### Changes Required

**File**: `scripts/lib/types.ts`

- [x] Update `SpecKind` (line 2): remove `"command"` → `"agent" | "skill"`
- [x] Remove `CanonicalRouting` interface (lines 31-34)
- [x] In `CanonicalExecution` (line 22): remove `"command"` and `"skill"` from the
  `mode` union (both are dead values; agents use only `"subagent"` or `"primary"`).
  Result: `mode?: "subagent" | "primary"`
- [x] Remove `CanonicalCommand` interface (lines 36-41)
- [x] Update `CanonicalSkill` interface (lines 43-45) to include the former
  command-block fields (`disable_model_invocation` excluded — dropped from schema):
  ```typescript
  export interface CanonicalSkill {
    accepts_args?: boolean;
    args_schema?: string;
    delegate_to?: string;
  }
  ```
- [x] Update `CanonicalFrontmatter` (lines 60-73):
  - Keep `kind?: SpecKind` but make it **optional** (skills won't have it in frontmatter;
    agents still do; Phase 4 injects it during loading so downstream code always sees it)
  - Add `user_invocable?: boolean`
  - Add `agent_invocable?: boolean`
  - Remove `routing?: CanonicalRouting`
  - Remove `command?: CanonicalCommand`
  - `skill?: CanonicalSkill` stays (with expanded type)
- [x] In `MappingFeatures` (lines 108-122): remove `supports_mode?: boolean` (line 119)
  — this feature flag is being removed from `mappings/features.yaml` in Phase 7
- [x] Update `NormalizedSpec` (lines 82-98):
  - Add `user_invocable: boolean`
  - Add `agent_invocable: boolean`
  - Remove `routing: CanonicalRouting`
  - Remove `command?: CanonicalCommand`
  - `skill?: CanonicalSkill` stays

### Success Criteria

#### Automated Verification:
- [x] TypeScript compiles without errors: `npx tsc --noEmit -p tsconfig.json` (or equivalent)

---

## Phase 4: Parse Logic Update

### Overview

Remove the `spec/commands` load path and ensure `loadSkillSpecs` infers
`kind = "skill"` for all loaded specs.

### Changes Required

**File**: `scripts/lib/parse.ts`

- [x] In `loadCanonicalSpecs` (line 83): remove `spec/commands` from the spec roots
  array so only `spec/agents` is loaded via `walkMarkdownFiles`
- [x] In `loadSkillSpecs` (lines 26-80): mutate `parsed.data` to inject `kind`
  **before** pushing to the specs array (the object is constructed inline, so there
  is no `spec` variable to mutate after the fact):
  ```typescript
  // Before the specs.push({...}) call (around line 71), add:
  (parsed.data as CanonicalFrontmatter).kind = "skill"; // inferred from spec/skills/ location
  ```

### Success Criteria

#### Automated Verification:
- [x] `npm run validate:agents` passes
- [x] `npm run compile:agents` produces files (no crash)

---

## Phase 5: Validate Logic Update

### Overview

Update `normalizeSpecs` to handle the new top-level invocability flags and remove
routing. Update `validateSemantics` to enforce the new rules.

### Changes Required

**File**: `scripts/lib/validate.ts`

#### `normalizeSpecs` (lines 50-107):

- [x] Update the valid-kinds check (line 68-69) from
  `["agent", "command", "skill"]` to `["agent", "skill"]`
- [x] **Do not remove** `"kind"` from `requiredStringFields` (line 52) — Phase 4
  injects `spec.fm.kind = "skill"` during loading, so `kind` is always present by
  the time `normalizeSpecs` runs. The existing required-field check still passes.
- [x] Remove routing normalization block (lines 93-96) entirely
- [x] Remove the `command` passthrough (line 97)
- [x] Add `user_invocable` and `agent_invocable` normalization:
  ```typescript
  user_invocable: spec.fm.user_invocable ?? false,
  agent_invocable: spec.fm.agent_invocable ?? false,
  ```
- [x] Update the `skill` passthrough (line 98) — already passes `spec.fm.skill`
  through, no change needed since `CanonicalSkill` now holds the right fields

#### `validateSemantics` (lines 109-173):

- [x] Replace the kind-specific `command` block checks (lines 125-131) with a new
  invocability check (guarded with `spec.kind === "skill"` since agents don't use these flags):
  ```typescript
  if (!spec.user_invocable && !spec.agent_invocable) {
    errors.push(`${spec.sourcePath}: at least one of user_invocable or agent_invocable must be true`);
  }
  ```
- [x] Remove the routing collision check (lines 133-143) entirely — routing is gone
  and ID uniqueness is already enforced by the duplicate-ID check above it
- [x] Update the `delegate_to` reference check (lines 145-152) to read from
  `spec.skill?.delegate_to`:
  ```typescript
  if (spec.skill?.delegate_to) {
    const target = spec.skill.delegate_to;
    if (!idSet.has(target) && !specs.some((s) => s.id === target)) {
      errors.push(
        `${spec.sourcePath}: skill.delegate_to '${target}' does not match any canonical id`,
      );
    }
  }
  ```

### Success Criteria

#### Automated Verification:
- [x] `npm run validate:agents` passes
- [x] Temporarily add a spec with both flags false and confirm it fails validation
  with a clear error message (then revert)

---

## Phase 6: Compile + Adapter Updates

### Overview

Update `compile.ts` to remove the `"command"` kind entry, and update all four
adapters to remove kind-based branching and `disable_model_invocation` handling
entirely.

### Changes Required

#### 1. Compile Orchestrator

**File**: `scripts/lib/compile.ts`

- [x] Remove `"command": "supports_skills"` from `kindFeatureKey` (lines 24-28):
  ```typescript
  const kindFeatureKey: Record<SpecKind, keyof MappingFeatures["providers"][Provider]> = {
    agent: "supports_agents",
    skill: "supports_skills",
  };
  ```

#### 2. Claude Adapter

**File**: `scripts/lib/adapters/claude.ts`

The current 3-way kind branch (skill / command / agent) becomes 2-way (skill / agent).
The unified skill case merges the former branches. **Note**: the two existing skills
(`gh-safe`, `git-town`) previously skipped tool mapping and model resolution; they will
now go through these code paths. Since both have no `capabilities.tools` and no
`execution.model_profile`, the generated output is unchanged — `mapTools` returns an
empty list and `resolveProviderModelConfig` returns nothing. No output diff expected.

- [x] Replace the `kind === "skill"` and `kind === "command"` branches with a
  single unified skill branch. The canonical `agent_invocable` flag is mapped to
  Claude Code's `disable-model-invocation` field; the canonical `user_invocable`
  flag maps to Claude Code's `user-invocable` field:
  ```typescript
  // skill branch (formerly kind === "skill" or kind === "command")
  // mapTools signature: (spec, mappings, warnings) — third arg is the CompileWarning array
  const mappedTools = mapTools(spec, mappings, warnings);
  const fm: Record<string, unknown> = {
    name: spec.name,
    description: spec.description,
  };
  if (mappedTools.length > 0) fm["allowed-tools"] = mappedTools.join(", ");
  // resolveProviderModelConfig takes a profile mapping object, not the profile name string.
  // Match the existing command branch pattern at claude.ts:74-80:
  const profile = spec.execution.model_profile;
  if (profile) {
    const resolved = resolveProviderModelConfig(mappings.models.profiles[profile], "claude");
    if (resolved.model) fm.model = resolved.model; // resolved always returns an object
  }
  // Only emit user-invocable when false — Claude Code defaults to true,
  // so we emit the field only when hiding the skill from the slash-command menu.
  if (!spec.user_invocable) fm["user-invocable"] = false;
  // Map agent_invocable: false → disable-model-invocation: true.
  // Claude Code's disable-model-invocation removes the description from Claude's
  // context entirely, preventing autonomous invocation. Only emit when restricting
  // model access; omit when the model CAN invoke (the default behavior).
  if (!spec.agent_invocable) fm["disable-model-invocation"] = true;
  const outDir = `generated/claude/skills/${spec.id}/`;
  // Use renderMarkdownWithFrontmatter (imported from "../format.js"), not matter.stringify,
  // to match the project's existing frontmatter formatting convention:
  const files: GeneratedFile[] = [{ provider: "claude", path: `${outDir}SKILL.md`, content: renderMarkdownWithFrontmatter(fm, spec.body) }];
  for (const sf of spec.supportingFiles) {
    files.push({ provider: "claude", path: `${outDir}${sf.relativeName}`, content: sf.content, mode: sf.mode });
  }
  return files;
  ```

#### 3. OpenCode Adapter

**File**: `scripts/lib/adapters/opencode.ts`

- [x] In the shared preamble (lines 59-61): restrict `execution.mode` emission to
  agents only (agents use `mode: subagent` / `mode: primary`; skills never used
  this field meaningfully):
  ```typescript
  // REPLACE:
  if (spec.execution.mode) {
    frontmatter.mode = spec.execution.mode;
  }
  // WITH:
  if (spec.execution.mode && spec.kind === "agent") {
    frontmatter.mode = spec.execution.mode;
  }
  ```
- [x] Update the branch condition at line 68 from
  `spec.kind === "command" || spec.kind === "skill"` to just `spec.kind === "skill"`.
  After Phase 3 removes `"command"` from `SpecKind`, the string literal `"command"`
  is not assignable to the type and causes a TypeScript compile error if left in place.
- [x] In the same branch (lines 68-84): remove the entire `disable_model_invocation`
  block — no `disable-model-invocation` emitted at all:
  ```typescript
  // DELETE these lines:
  if (spec.kind === "command" && spec.command?.disable_model_invocation !== undefined)
    frontmatter["disable-model-invocation"] = spec.command.disable_model_invocation;
  ```

#### 4. Codex Adapter

**File**: `scripts/lib/adapters/codex.ts`

- [x] Remove the `spec.kind === "command"` block (lines 28-30) entirely —
  no `disable-model-invocation` emitted

#### 5. Cursor Adapter

**File**: `scripts/lib/adapters/cursor.ts`

- [x] Remove the `spec.kind === "command"` block (lines 14-16) entirely —
  no `disable-model-invocation` emitted

### Success Criteria

#### Automated Verification:
- [x] `npm run compile:agents` produces files without error or warning
- [x] `npm run check:agents` passes (generated files match compiled output)
- [x] For OpenCode: confirm no `mode:` key appears in any
  `generated/opencode/skills/*/SKILL.md`

---

## Phase 7: Mappings Update

### Overview

Remove the dead `supports_mode` feature flag from the OpenCode provider mapping and
its definition from the features schema.

### Changes Required

**File**: `mappings/features.yaml`

- [x] Remove `supports_mode: true` from the `opencode:` provider block (line 14)

**File**: `spec/schema/features-mapping.schema.json`

- [x] Remove `"supports_mode"` from the `providerFeatures` definition (line 39)
- [x] Remove `"supports_commands"` from the `providerFeatures` definition (line 33) —
  this property is unused in all YAML files and in the TypeScript types; removing it
  now prevents future confusion

### Success Criteria

#### Automated Verification:
- [x] `npm run build:agents` passes end-to-end
- [x] `npm run build:agents:home` passes
- [x] `npm run build:agents:work` passes

---

## Phase 8: End-to-End Verification

### Overview

Verify the full build produces correct output and that the generated files
have no unintended regressions.

### Changes Required

- [x] Delete the migration script: `rm scripts/migrate_commands_to_skills.ts`

### Success Criteria

#### Automated Verification:
- [x] `npm run build:agents` passes (validate + compile + check)
- [x] `npm run build:agents:home` passes
- [x] `npm run build:agents:work` passes
- [x] `generated/claude/skills/` contains 23 subdirectories (21 commands + 2 skills;
  all target claude)
- [x] `generated/opencode/skills/` contains 22 subdirectories (21 commands + `git-town`;
  `gh-safe` targets claude+cursor only, so it is excluded from opencode)
- [x] `generated/codex/skills/` contains 22 subdirectories (21 commands + `git-town`;
  `gh-safe` targets only claude+cursor and is therefore excluded from codex as well)
- [x] `generated/cursor/skills/` contains 23 subdirectories (21 commands + 2 skills;
  all target cursor)
- [x] No `generated/claude/skills/*/SKILL.md` has `allowed-tools:` missing for
  specs that had tools (e.g. `commit` had `bash`)
- [x] No `generated/opencode/skills/*/SKILL.md` has a `mode:` field

#### Manual Verification:
- [ ] Diff the generated output before and after the migration for 3-4 key specs:
  - `commit` (had `bash` tool, `disable_model_invocation` absent)
  - `create-idea` (had `disable_model_invocation: true`)
  - `code-review` (had `delegate_to`)
  - `git-town` (existing skill, agent-only, no tools)
- [ ] Confirm the only diffs are expected:
  - OpenCode: removal of `mode: command` from all 21 former command specs
  - Claude: no field changes for the 8 commands without `disable_model_invocation`
    (field was absent before, still absent after)
  - Claude: no change for the 13 commands with `disable_model_invocation: true`
    (`disable-model-invocation: true` was emitted before, still emitted after via new mapping)
  - Claude: no `user-invocable` field added to former commands — field is only emitted
    when `false`; former commands have `user_invocable: true` so it is omitted
  - Claude: `gh-safe` and `git-town` output unchanged (already had `user-invocable: false`)

---

## Testing Strategy

No automated test suite exists. Verification relies on the build pipeline.

### Build Commands:
- `npm run validate:agents` — schema + semantic validation only
- `npm run compile:agents` — full compile (skips check)
- `npm run check:agents` — verify generated files are up-to-date
- `npm run build:agents` — all three in sequence

### Regression Approach:
Before starting **Phase 1** (before any changes), capture a snapshot of the generated output:
```bash
cp -r generated/ generated_before_migration/
```
After Phase 8, diff to confirm only expected changes:
```bash
diff -r generated_before_migration/ generated/
```

## Migration Notes

**Generated output is largely unchanged after migration.** The main visible diff is:

- **OpenCode only**: `mode: command` is removed from all 21 former command specs. This
  was always dead output (OpenCode's skill schema does not recognise the field).

**Claude adapter** — field-by-field:

- `disable-model-invocation: true` continues to be emitted for exactly the same 13 specs
  that had `disable_model_invocation: true` in their canonical command block. The mapping
  changes internally (`agent_invocable: false` → `disable-model-invocation: true`) but
  the generated output is identical.
- `user-invocable` is only emitted when `false`. Former commands all have
  `user_invocable: true`, so no `user-invocable` field appears in their output (same as
  before). The two existing skills (`gh-safe`, `git-town`) already had `user-invocable: false`
  and continue to emit it.

**Canonical spec** — the canonical `disable_model_invocation` field in the `command` block
is dropped (superseded by the top-level `agent_invocable` flag). The equivalent generated
output field `disable-model-invocation` is retained in the Claude adapter's output,
derived from `agent_invocable: false`.

## References

- Idea document: `./thoughts/ideas/2026-03-03-unify-commands-and-skills.md`
- Schema: `spec/schema/canonical.schema.json`
- Types: `scripts/lib/types.ts`
- Parse: `scripts/lib/parse.ts` (loadSkillSpecs lines 26-80, loadCanonicalSpecs lines 82-115)
- Validate: `scripts/lib/validate.ts` (normalizeSpecs lines 50-107, validateSemantics lines 109-173)
- Compile: `scripts/lib/compile.ts` (kindFeatureKey lines 24-28)
- Adapters: `scripts/lib/adapters/{claude,opencode,codex,cursor}.ts`
- Mappings: `mappings/features.yaml`
