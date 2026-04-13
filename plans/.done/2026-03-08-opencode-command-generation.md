# OpenCode Command Generation Implementation Plan

## Overview

The OpenCode adapter currently emits all skills as `SKILL.md` files, ignoring the
`user_invocable` and `agent_invocable` flags entirely. OpenCode has a three-tier
extensibility model — Commands (user-triggered), Skills (AI-triggered), and Tools — and
user-invocable skills need to be emitted as command files (`.opencode/commands/<id>.md`),
not as skills. This routing is an OpenCode-specific implementation detail and lives
entirely in the adapter, with no changes to shared schema, types, or feature mappings.

## Current State Analysis

The OpenCode adapter (`scripts/lib/adapters/opencode.ts`) handles skills at lines 72–86:
it always emits `generated/opencode/skills/<id>/SKILL.md` regardless of `user_invocable`
or `agent_invocable`. It never reads either flag.

**Invocability breakdown across the 26 canonical skills:**

- `user_invocable: true` only: ~14 skills (e.g., `implement-plan`, `describe-pr`,
  `review-pr`, `split-branch`)
- `agent_invocable: true` only: ~2 skills (e.g., `git-town`, `gh-safe`)
- **Both flags true**: ~10 skills (e.g., `commit`, `code-review`, `learn`,
  `research-web`, `create-plan`, `resolve-conflicts`, `review-files`, `validate-plan`,
  `code-review-loop`, `research-codebase`)

**The problem with dual-invocable skills**: OpenCode has no unified artifact that serves
both roles — a command can only be user-triggered; a skill can only be AI-triggered. The
resolution is to generate **both artifacts** when both flags are set.

## Desired End State

After this plan is complete:

- Skills with `user_invocable: true` produce `generated/opencode/commands/<id>.md`
- Skills with `agent_invocable: true` produce `generated/opencode/skills/<id>/SKILL.md`
  (current behavior, now gated on the flag)
- Dual-invocable skills produce **both** files
- Agent specs are unchanged
- No changes to `features.yaml`, `features-mapping.schema.json`, or `types.ts` — the
  command/skill routing is an internal adapter concern

### Verification

```bash
npm --prefix agent-config run build:agents
```

Must pass without errors. Then verify:

- `generated/opencode/commands/` exists and contains `.md` files for all user_invocable skills
- `generated/opencode/skills/` contains only agent_invocable skills (subset of current output)
- Dual-invocable skills (e.g., `commit`) appear in BOTH directories

### Key Discoveries

- `NormalizedSpec.user_invocable: boolean` and `NormalizedSpec.agent_invocable: boolean`
  are already populated — the adapter just needs to read them (`scripts/lib/types.ts:86-87`)
- `NormalizedSpec.skill?.delegate_to?: string` maps to the OpenCode `agent:` command
  frontmatter field (`scripts/lib/types.ts:91`, `spec/skills/code-review/SKILL.md:15`)
- `GeneratedFile[]` return type already supports multiple files per adapter call
  (`scripts/lib/types.ts:131-136`) — no pipeline changes needed
- Semantic validation already ensures every skill has at least one invocability flag —
  no edge case where both are false
- OpenCode command frontmatter fields per research: `description`, `agent` (optional),
  `model` (optional), `subtask` (optional boolean) — **no** `tools` boolean map,
  **no** `name`, **no** `temperature`, **no** `variant`

## What We're NOT Doing

- Not changing the Claude, Codex, or Cursor adapters
- Not changing the compile/dispatch logic in `scripts/lib/compile.ts`
- Not changing the canonical schema or adding new canonical frontmatter fields
- Not changing `NormalizedSpec` shape
- Not changing `features.yaml`, `features-mapping.schema.json`, or `types.ts` — the
  command routing is an OpenCode-specific adapter detail, not a cross-cutting feature flag
- Not writing commands for agent specs (only skill specs)
- Not adding `variant` to command frontmatter (not documented in OpenCode command spec)
- Not adding `temperature` to command frontmatter (not documented in OpenCode command spec)
- Not changing how supporting files (e.g., the `gh-safe` shell script) are handled —
  they are only relevant to skill output, not commands

## Implementation Approach

**One file changes**: `scripts/lib/adapters/opencode.ts`. The skill block is replaced
with invocability-flag-driven routing that builds command and/or skill files as needed.

The new command frontmatter shape for OpenCode:

```yaml
description: <spec.description>
agent: <skill.delegate_to> # if delegate_to is set
model: <resolved model> # if model_profile resolves to a model
subtask: true
```

The skill frontmatter shape is unchanged:

```yaml
name: <spec.name>
description: <spec.description>
tools:
  bash: true
  read: false
  # ... boolean map of all known OpenCode tools
model: <resolved model> # if model_profile resolves to a model
variant: <variant> # if model options include variant
temperature: <temperature> # if execution.temperature is set
```

---

## Phase 1: OpenCode Adapter Refactor

### Overview

Refactor `scripts/lib/adapters/opencode.ts` to route skills based on their invocability
flags. Replace the single unconditional skill branch with three cases:

1. `user_invocable && !agent_invocable` → command file only
2. `!user_invocable && agent_invocable` → skill file only (existing behavior, now gated)
3. `user_invocable && agent_invocable` → both files

### Changes Required

#### 1. OpenCode Adapter

**File**: `scripts/lib/adapters/opencode.ts`

The current skill block (lines 72–86) is:

```typescript
// Skills emit as skill directories
if (spec.kind === "skill") {
  frontmatter.name = spec.name;

  const skillDir = path.join("generated", "opencode", "skills", spec.id);
  return {
    files: [
      {
        provider: "opencode",
        path: path.join(skillDir, "SKILL.md"),
        content: renderMarkdownWithFrontmatter(frontmatter, spec.body),
      },
    ],
    warnings,
  };
}
```

- [x] Replace the skill block with routing logic that builds command and skill file
      objects separately, then returns the appropriate combination based on flags:

  ```typescript
  if (spec.kind === "skill") {
    const files: GeneratedFile[] = [];

    if (spec.user_invocable) {
      const commandFrontmatter: Record<string, unknown> = {
        description: spec.description,
        subtask: true,
      };
      if (spec.skill?.delegate_to) {
        commandFrontmatter.agent = spec.skill.delegate_to;
      }
      if (resolvedModel.model) {
        commandFrontmatter.model = resolvedModel.model;
      }
      files.push({
        provider: "opencode",
        path: path.join("generated", "opencode", "commands", `${spec.id}.md`),
        content: renderMarkdownWithFrontmatter(commandFrontmatter, spec.body),
      });
    }

    if (spec.agent_invocable) {
      frontmatter.name = spec.name;
      const skillDir = path.join("generated", "opencode", "skills", spec.id);
      files.push({
        provider: "opencode",
        path: path.join(skillDir, "SKILL.md"),
        content: renderMarkdownWithFrontmatter(frontmatter, spec.body),
      });
    }

    return { files, warnings };
  }
  ```

- [x] Ensure the shared `frontmatter` object (built at lines 51–70 with `description`,
      `tools`, `model`, `variant`, `temperature`) is only used for the skill file — the
      command file builds its own `commandFrontmatter` object independently.

- [x] Remove the `frontmatter.name = spec.name` assignment from the old block (it now
      lives inside the `agent_invocable` branch only).

- [x] Confirm `GeneratedFile` is in the import from `"../types.js"` (already present
      at line 7 — no change needed).

### Success Criteria

#### Automated Verification

- [x] Full build passes: `npm --prefix agent-config run build:agents`
- [x] TypeScript type-check passes (included in build)
- [x] Check step passes (validates generated output matches expected):
      `npm --prefix agent-config run check:agents`

#### Manual Verification

- [x] `generated/opencode/commands/` directory exists and is non-empty
- [x] A pure user_invocable skill (e.g., `describe-pr`) appears in `commands/` only —
      NOT in `skills/`
- [x] A pure agent_invocable skill (e.g., `gh-safe`) appears in `skills/` only —
      NOT in `commands/`
- [x] A dual-invocable skill (e.g., `commit`) appears in BOTH `commands/` AND `skills/`
- [x] `generated/opencode/commands/code-review.md` contains `agent: code-reviewer` in
      its frontmatter (verifying `delegate_to` mapping)
- [x] `generated/opencode/commands/commit.md` contains `subtask: true` in its frontmatter
- [x] `generated/opencode/commands/commit.md` does NOT contain a `tools:` boolean map
- [x] Spot-check: `generated/opencode/skills/gh-safe/SKILL.md` still contains the
      `tools:` boolean map (skill output unchanged)
  > **Deviation:** `gh-safe` targets only `claude` and `cursor` via `compat.targets`, so
  > it is excluded from OpenCode entirely — no `SKILL.md` is generated for it. The
  > skill output (tools map) was verified on `git-town/SKILL.md` instead.

**Implementation Note**: After Phase 1 passes automated verification, pause for manual
spot-check of a few generated files before proceeding.

---

## Phase 2: Regenerate and Verify Setup Sync

### Overview

Confirm the full build produces the correct output and investigate whether `setup.sh`
needs updating to sync the new `commands/` directory.

### Changes Required

#### 1. Build and Regenerate

- [x] Run `npm --prefix agent-config run build:agents` to regenerate `generated/opencode/`

#### 2. Verify Setup Sync (Investigation Only)

**File**: `setup.sh` (at repo root)

- [x] Read `setup.sh` to confirm it syncs `generated/opencode/` to the correct
      destination. If `commands/` is not currently synced, note it as a follow-up but
      do NOT change `setup.sh` in this plan.
  > **Finding:** `setup.sh` does not sync any OpenCode generated files — neither
  > `skills/` nor `commands/`. Syncing the new `commands/` directory to `.opencode/commands/`
  > is a follow-up task.

### Success Criteria

#### Automated Verification

- [x] `npm --prefix agent-config run build:agents` exits 0
- [x] `git diff --stat generated/opencode/` shows:
  - New `generated/opencode/commands/` directory and files added (untracked — 24 new files)
  - `generated/opencode/skills/` reduced (pure user_invocable skills removed)
  - Dual-invocable skills still present in `generated/opencode/skills/`

#### Manual Verification

- [x] The manifest (`generated/manifest.json`) reflects the new file paths (24 `opencode/commands/` entries confirmed)
- [x] No unexpected files remain in `generated/opencode/skills/` (only agent_invocable
      skills should be present)

---

## Testing Strategy

### Automated

- `npm --prefix agent-config run build:agents` — full pipeline (typecheck + validate +
  compile + check)

### Manual Spot Checks

1. `generated/opencode/commands/commit.md` — has `subtask: true`, no `tools:` map
2. `generated/opencode/commands/code-review.md` — has `agent: code-reviewer`
3. `generated/opencode/skills/commit/SKILL.md` — still exists (dual-invocable), has `tools:` map
4. `generated/opencode/skills/describe-pr/` — should NOT exist (pure user_invocable)
5. `generated/opencode/skills/gh-safe/SKILL.md` — still exists (pure agent_invocable)
6. Total file counts: `ls generated/opencode/commands/ | wc -l` should equal 24 (all
   user_invocable skills); `ls generated/opencode/skills/ | wc -l` should equal 12 (all
   agent_invocable skills)

## References

- Research: `./thoughts/research/2026-03-08-opencode-commands-vs-skills.md`
- OpenCode adapter (current): `scripts/lib/adapters/opencode.ts`
- Type definitions: `scripts/lib/types.ts`
- Example dual-invocable spec: `spec/skills/code-review/SKILL.md`
- OpenCode commands docs: https://opencode.ai/docs/commands/
