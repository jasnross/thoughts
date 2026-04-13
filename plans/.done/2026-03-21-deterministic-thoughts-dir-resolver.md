# Deterministic `THOUGHTS_DIR` Resolver Implementation Plan

## Overview

Replace prompt-only `THOUGHTS_DIR` discovery with a deterministic bundled resolver helper
that shell-capable skills can invoke directly. Standardize the contract across Claude,
OpenCode, Codex, and Cursor so callers resolve the absolute `THOUGHTS_DIR` once, then
pass it explicitly to no-shell subagents such as `learnings-discoverer`.

## Current State Analysis

- [ ] `THOUGHTS_DIR` discovery currently lives as natural-language prompt text in
      `agent-config/spec/fragments/thoughts/discover-dir.md:1`, so correctness depends on
      model interpretation rather than executable logic.
- [ ] The fragment is shared broadly across shell-capable skills and no-shell agents,
      including `agent-config/spec/skills/learn/SKILL.md:86` and
      `agent-config/spec/agents/learnings-discoverer.md:30`.
- [ ] Skill supporting-file bundling already exists for Claude, OpenCode, and Cursor via
      `agent-config/scripts/lib/parse.ts:66`, `agent-config/scripts/lib/adapters/claude.ts:79`,
      `agent-config/scripts/lib/adapters/opencode.ts:108`, and
      `agent-config/scripts/lib/adapters/cursor.ts:31`.
- [ ] Codex is the portability gap: `agent-config/scripts/lib/adapters/codex.ts:36`
      emits only `SKILL.md`, so bundled helper files are currently unavailable there.
- [ ] `learnings-discoverer` is intentionally no-shell (`read`, `glob`, `grep` only) per
      `agent-config/spec/agents/learnings-discoverer.md:10`, so it cannot directly execute a
      resolver script.
- [ ] Existing planning history already established the desired architectural direction:
      callers should resolve `THOUGHTS_DIR` first and pass it to subagents explicitly
      (`/Users/jasonr/dotfiles/thoughts/plans/2026-03-15-consistent-thoughts-dir-discovery.md:144`).

## Desired End State

- [ ] A single deterministic resolver helper exists as a bundled supporting file in a new
      or updated skill bundle under `agent-config/spec/skills/`.
- [ ] All shell-capable skills that need `THOUGHTS_DIR` invoke the helper instead of
      relying on prose-only discovery.
- [ ] All no-shell subagents that need `THOUGHTS_DIR` require the caller to provide it as
      an absolute path in the prompt; they no longer perform independent discovery.
- [ ] All four supported targets emit the helper correctly where their packaging model
      allows it, including Codex supporting-file emission.
- [ ] Generated outputs remain provider-neutral in canonical bodies, with any provider-specific packaging handled in adapters.

### Key Discoveries

- [ ] Shared fragments are compile-time only; they cannot execute logic themselves because
      `agent-config/scripts/lib/fragments.ts:49` only performs Handlebars expansion.
- [ ] The current compiler already supports bundled helper files from skill directories via
      `agent-config/scripts/lib/parse.ts:66`, so the script pattern matches existing repo
      architecture.
- [ ] `gh-safe` is the strongest precedent for “skill + bundled executable + usage contract”
      at `agent-config/spec/skills/gh-safe/SKILL.md:19` and
      `agent-config/spec/skills/gh-safe/gh-safe.sh:1`.
- [ ] OpenCode explicitly supports bundled skill resources according to prior research in
      `/Users/jasonr/dotfiles/thoughts/research/2026-03-08-opencode-skill-bundled-scripts.md:20`.
- [ ] OpenCode splits user-facing commands and agent-facing skills structurally rather than
      with a single invocability flag, so command/skill output shape must remain compatible
      with that model (`/Users/jasonr/dotfiles/thoughts/research/2026-03-08-opencode-commands-vs-skills.md:23`).
- [ ] Cross-agent portability improves when directory/path rules are deterministic code
      rather than prose because providers differ in how they expose contextual metadata
      (`/Users/jasonr/dotfiles/thoughts/research/2026-03-17-rules-across-ai-coding-agents.md:339`).

## What We're NOT Doing

- [ ] Re-introducing independent discovery logic into no-shell subagents after caller
      handoff is adopted.
- [ ] Supporting multiple `thoughts/` directories for a single invocation.
- [ ] Changing the layout under `thoughts/` (`learnings/`, `plans/`, `research/`, etc.).
- [ ] Adding provider-specific canonical prose for path discovery when adapter/tooling
      changes can handle portability.
- [ ] Replacing the entire fragment system or broader compiler architecture.
- [ ] Building a live end-to-end agent-behavior harness that depends on nondeterministic LLM
      interpretation.

## Implementation Approach

- [ ] Introduce a small deterministic resolver helper that prints the resolved absolute
      `THOUGHTS_DIR` and exits nonzero on failure.
- [ ] Package it using the same supporting-file mechanism already used by bundled-skill
      patterns like `gh-safe`.
- [ ] Refactor the shared `thoughts/discover-dir` fragment so shell-capable skills call the
      helper instead of restating the traversal algorithm.
- [ ] Refactor no-shell subagents to accept caller-provided `THOUGHTS_DIR` only, removing
      ownership ambiguity and duplicate discovery.
- [ ] Add tests around deterministic compiler behavior and resolver semantics, not wording.

---

## Phase 1: Define the Resolver Contract

### Overview

- [x] Specify the canonical behavior of the deterministic resolver before any code changes
      so every caller, adapter, and subagent uses the same contract.

### Changes Required:

#### 1. Canonical resolver behavior

**File**: `agent-config/spec/fragments/thoughts/discover-dir.md`

- [x] Rewrite the fragment so it no longer describes the full traversal algorithm inline.
  > **Deviation:** Added a temporary inline compatibility fallback (legacy traversal)
  > because the helper skill from Phase 2 does not exist yet. This avoids breaking
  > current flows before the helper ships.
- [x] Document the helper contract instead: use env override first, otherwise invoke the
      bundled resolver helper from the current skill directory.
- [x] Preserve the distinction between writeable and non-writeable call sites only for
      fallback handling, not for path-search semantics.
- [x] State that callers must treat resolver output as an absolute path and store it in
      `$THOUGHTS_DIR` for subsequent steps.

#### 2. No-shell subagent contract

**Files**:

- `agent-config/spec/agents/learnings-discoverer.md`
- `agent-config/spec/fragments/research/sub-agent-catalog.md`
- `agent-config/AGENTS_GLOBAL.md`
- `agent-config/spec/skills/learn/SKILL.md`

- [x] Remove independent discovery expectations from no-shell subagents.
  > **Deviation:** Initially landed with a temporary compatibility fallback, then
  > removed it after review to enforce fail-fast caller ownership in
  > `agent-config/spec/agents/learnings-discoverer.md`.
- [x] Add explicit instruction that the parent caller must pass an absolute `THOUGHTS_DIR`
      in the subagent prompt/context.
- [x] Keep missing-learnings-directory behavior non-blocking (report no learnings and stop)
      to avoid first-run deadlocks.
- [x] Update shared subagent-catalog guidance so planning/research skills consistently pass
      `THOUGHTS_DIR` into thoughts-related subagents.
  > **Deviation:** Also updated `agent-config/AGENTS_GLOBAL.md` in Phase 1 so the global
  > knowledge-base dispatch contract resolves and passes `THOUGHTS_DIR` explicitly,
  > preventing an interim caller/subagent mismatch.
  > **Deviation:** Also updated the pointer template in `agent-config/spec/skills/learn/SKILL.md`
  > so newly suggested Knowledge Base sections match the new caller contract.

### Success Criteria:

#### Automated Verification:

- [x] `grep -n "thoughts/discover-dir" agent-config/spec/agents/learnings-discoverer.md`
      shows the old include removed or replaced as intended.
  > **Deviation:** Verified include removal with
  > `grep -n "{{> thoughts/discover-dir" agent-config/spec/agents/learnings-discoverer.md`
  > because the phrase `thoughts/discover-dir` now appears in explanatory prose.
- [x] `npm --prefix agent-config run validate:agents` passes.

#### Manual Verification:

- [x] Read the updated contract text and confirm there is exactly one owner of discovery:
      the shell-capable caller.
- [x] Confirm no unresolved ambiguity remains about who discovers `THOUGHTS_DIR` for
      `learnings-discoverer`.

**Implementation Note**: After completing this phase and all automated verification passes,
pause here for manual confirmation that the contract is clear before proceeding to the next
phase.

---

## Phase 2: Add the Bundled Resolver Helper

### Overview

- [x] Implement the deterministic resolver as a bundled supporting file following the
      existing `gh-safe` pattern.

### Changes Required:

#### 1. Create the helper skill bundle

**Files**:

- `agent-config/spec/skills/thoughts-dir-resolver/SKILL.md`
- `agent-config/spec/skills/thoughts-dir-resolver/resolve-thoughts-dir.sh`

- [x] Add a dedicated skill bundle for the resolver helper, or add the helper to an existing
      shared skill if that keeps ownership clear.
  > **Deviation:** Scoped initial compatibility targets to Claude/OpenCode/Cursor.
  > Codex remains deferred to Phase 3 because Codex supporting-file emission is not yet
  > implemented. Phase 3 completed Codex supporting-file emission and re-enabled
  > Codex compatibility for this skill.
- [x] Ensure the skill body explains exactly how callers invoke the helper by full path,
      similar to the `gh-safe` guidance in `agent-config/spec/skills/gh-safe/SKILL.md:34`.
- [x] Make the helper executable and packaged as a supporting file.

#### 2. Implement resolver semantics in the script

**File**: `agent-config/spec/skills/thoughts-dir-resolver/resolve-thoughts-dir.sh`

- [x] Implement env-var override: if `THOUGHTS_DIR` is already set and non-empty, print it
      unchanged and exit success.
  > **Deviation:** Tightened during review: pre-set `THOUGHTS_DIR` must be absolute.
  > Relative overrides now fail fast with a clear error to preserve the absolute-path
  > contract used by caller-to-subagent handoff.
- [x] Accept an explicit starting path argument so callers can pass the session-launch
      directory rather than relying on mutable shell CWD.
- [x] Check `<starting-path>/thoughts` first, then walk parent-by-parent until filesystem
      root.
- [x] Print the resolved absolute path on stdout only.
- [x] Exit nonzero with a clear error message on stderr if no `thoughts/` directory is
      found.
- [x] Keep selection criteria strictly proximity-based; do not reintroduce git-backing as a
      discriminator.

```bash
#!/usr/bin/env bash
set -euo pipefail

# Contract sketch:
# 1. honor THOUGHTS_DIR if already set
# 2. require a starting path argument
# 3. check current level first, then parents
# 4. print absolute path on success
# 5. print actionable error and exit 1 on failure
```

### Success Criteria:

#### Automated Verification:

- [x] Helper file is executable after generation: emitted mode includes `0o755`.
- [x] `npm --prefix agent-config run build:agents` passes.
- [x] A script-focused test fixture covers env override, current-level hit, one-level-up
      hit, and not-found failure behavior.
  > **Deviation:** Implemented as
  > `agent-config/scripts/__tests__/thoughts-dir-resolver.test.ts` with hermetic
  > environment handling so tests do not inherit ambient `THOUGHTS_DIR`.

#### Manual Verification:

- [x] Run the helper from a generated skill directory with a starting path of
      `/Users/jasonr/dotfiles/agent-config` and confirm it prints
      `/Users/jasonr/dotfiles/thoughts`.
- [x] Run it again with `THOUGHTS_DIR=/tmp/custom-thoughts` and confirm it prints the
      override unchanged.

**Implementation Note**: After completing this phase and all automated verification passes,
pause here for manual confirmation that the helper behaves deterministically before
proceeding to the next phase.

---

## Phase 3: Extend Compiler Support for Cross-Provider Packaging

### Overview

- [x] Ensure bundled helper files are emitted consistently for every supported target that
      can consume them.

### Changes Required:

#### 1. Codex supporting-file emission

**File**: `agent-config/scripts/lib/adapters/codex.ts`

- [x] Update the Codex adapter to emit `supportingFiles` alongside `SKILL.md`, matching the
      behavior already present in `agent-config/scripts/lib/adapters/claude.ts:79` and
      `agent-config/scripts/lib/adapters/cursor.ts:31`.
- [x] Preserve executable mode when emitting script helpers.
- [x] Keep file placement consistent with existing provider directory layout:
      `generated/codex/skills/<id>/...`.

#### 2. Packaging and manifest checks

**Files**:

- `agent-config/scripts/lib/emit.ts`
- `agent-config/scripts/__tests__/parse.test.ts`
- `agent-config/scripts/__tests__/emit.test.ts`
- `agent-config/scripts/__tests__/compile.test.ts`

- [x] Add test coverage that skills with supporting files compile and emit correctly for
      Codex as well as existing providers.
- [x] Verify manifest/check behavior continues to treat helper files as first-class
      generated artifacts.
  > **Deviation:** Additional hardening later updated
  > `agent-config/scripts/lib/emit.ts` and `agent-config/scripts/__tests__/emit.test.ts`
  > to make `check:agents` detect file-mode drift (including unexpected executable bits)
  > as part of end-to-end generated artifact validation.

### Success Criteria:

#### Automated Verification:

- [x] `npm --prefix agent-config test` passes.
- [x] `npm --prefix agent-config run build:agents` passes.
- [x] `generated/codex/skills/thoughts-dir-resolver/resolve-thoughts-dir.sh` exists after build.

#### Manual Verification:

- [x] Spot-check generated outputs for Claude, OpenCode, Codex, and Cursor to confirm the
      resolver script is emitted where expected.
- [x] Confirm executable mode is preserved for generated helper scripts on disk.

---

## Phase 4: Refactor Shell-Capable Skills to Use the Helper

### Overview

- [x] Replace prose-only discovery in shell-capable skills with a helper-invocation
      workflow while keeping canonical prompt text provider-neutral.

### Changes Required:

#### 1. Shared fragment rewrite

**File**: `agent-config/spec/fragments/thoughts/discover-dir.md`

- [x] Replace the inline traversal steps with instructions to invoke the bundled resolver
      helper by full path.
- [x] Keep the fragment focused on the call contract, export/store behavior, and writeable
      vs. read-only fallback handling.
- [x] Ensure the fragment does not depend on provider-specific metadata labels beyond the
      already-established “session-launch directory” concept.

#### 2. Shell-capable skill call sites

**Representative files**:

- `agent-config/spec/skills/learn/SKILL.md`
- `agent-config/spec/skills/create-plan/SKILL.md`
- `agent-config/spec/skills/research-codebase/SKILL.md`
- `agent-config/spec/skills/research-web/SKILL.md`
- `agent-config/spec/skills/implement-plan/SKILL.md`

- [x] Update each shell-capable skill that currently includes `thoughts/discover-dir` so
      the rendered instructions consistently call the helper.
- [x] Where a skill dispatches thought-related subagents, add explicit instructions to pass
      the resolved `THOUGHTS_DIR` value forward.
  > **Deviation:** Explicit caller-handoff wording was added to
  > `agent-config/spec/skills/create-plan/SKILL.md`; skills that already rely on
  > `research/sub-agent-catalog` inherited the absolute-path handoff wording there.
- [x] Preserve existing absolute-path usage for downstream file references like
      `$THOUGHTS_DIR/learnings/`, `$THOUGHTS_DIR/plans/`, and `$THOUGHTS_DIR/research/`.

### Success Criteria:

#### Automated Verification:

- [x] `npm --prefix agent-config run build:agents` passes.
- [x] Generated outputs for helper-using skills mention invoking the resolver helper rather
      than restating the full walk-up algorithm.

#### Manual Verification:

- [x] Spot-check at least one generated skill per provider to confirm the helper-invocation
      instructions render cleanly and remain understandable.
- [x] Confirm no shell-capable skill still claims the model itself should perform the
      parent-directory walk-up manually.

---

## Phase 5: Refactor No-Shell Subagents and Caller Handoff

### Overview

- [x] Complete the architectural split: callers discover, no-shell subagents consume.
  > **Deviation:** Completed earlier than originally phased (in Phase 1) to prevent
  > caller/subagent contract mismatch during rollout.

### Changes Required:

#### 1. `learnings-discoverer`

**File**: `agent-config/spec/agents/learnings-discoverer.md`

- [x] Remove the `thoughts/discover-dir` include.
- [x] Add a clear precondition stating the caller must provide an absolute `THOUGHTS_DIR`
      value in the prompt/context.
- [x] Update search instructions to operate only on the provided
      `$THOUGHTS_DIR/learnings/` location.
- [x] Fail clearly if the caller omitted the path or provided an invalid/non-absolute
      directory value.
  > **Deviation:** Missing `$THOUGHTS_DIR/learnings/` is intentionally non-blocking in
  > `learnings-discoverer`; it reports no learnings and stops.

#### 2. Caller prompt templates and shared guidance

**Files**:

- `agent-config/AGENTS_GLOBAL.md`
- `agent-config/spec/fragments/research/sub-agent-catalog.md`
- `agent-config/spec/agents/thoughts-locator.md`
- `agent-config/spec/agents/thoughts-analyzer.md`
- any thoughts-dispatching skills that launch `learnings-discoverer`, `thoughts-locator`,
  or `thoughts-analyzer`

- [x] Update guidance so parent agents resolve `THOUGHTS_DIR` before dispatching thought-related subagents.
- [x] Ensure subagent prompts explicitly include the absolute path they should use.
- [x] Keep this rule consistent across planning, research, and learning workflows.
  > **Deviation:** Also tightened no-shell consumers (`thoughts-locator` and
  > `thoughts-analyzer`) to fail when `THOUGHTS_DIR` is missing, non-absolute,
  > or unreadable, so caller errors surface clearly.

### Success Criteria:

#### Automated Verification:

- [x] `grep -n "thoughts/discover-dir" agent-config/spec/agents/learnings-discoverer.md`
      returns no matches.
- [x] `npm --prefix agent-config run build:agents` passes.

#### Manual Verification:

- [x] Trigger `learnings-discoverer` from a caller that passes `THOUGHTS_DIR` explicitly and
      confirm it succeeds without attempting self-discovery.
- [x] Review generated `learnings-discoverer` output and confirm ownership boundaries are
      unambiguous.

---

## Phase 6: Testing Strategy and Verification

### Overview

- [x] Verify actual deterministic behavior in tooling and generation, without freezing
      incidental wording.

### Changes Required:

#### 1. Resolver behavior tests

**Files**:

- `agent-config/scripts/__tests__/` (new or expanded tests)
- test fixtures under `agent-config/scripts/__tests__/fixtures/`

- [x] Add tests for the resolver helper itself using temporary directory fixtures.
- [x] Cover env override, exact-level match, parent-level match, and not-found failure.
- [x] Avoid brittle wording assertions beyond stable interface guarantees like stdout path
      and nonzero exit codes.

#### 2. Compiler/output tests

**Files**:

- `agent-config/scripts/__tests__/parse.test.ts`
- `agent-config/scripts/__tests__/emit.test.ts`
- `agent-config/scripts/__tests__/compile.test.ts`

- [x] Add compile/emission coverage for Codex supporting files.
- [x] Add at least one test ensuring bundled helper files survive the full load → compile →
      emit pipeline.
- [x] Add a lightweight assertion that generated skills contain the resolver helper path
      reference rather than exact multi-line prompt text.

### Success Criteria:

#### Automated Verification:

- [x] `npm --prefix agent-config test` passes.
- [x] `npm --prefix agent-config run build:agents` passes.
- [x] `npm --prefix agent-config run check:agents` passes.
  > **Deviation:** `check:agents` was verified after a fresh `compile:agents`
  > invocation because `build:agents` formats generated files after its internal
  > check step.

#### Manual Verification:

- [x] Start fresh sessions in at least Claude/OpenCode/Codex/Cursor-generated environments
      and confirm the helper is present in the generated skill directories.
  > **Deviation:** Verified helper presence directly in generated outputs on disk for all
  > providers via `glob` and executable invocation, instead of launching four separate fresh
  > interactive sessions.
- [x] Validate one end-to-end learnings-discoverer invocation where the parent resolves and
      passes `THOUGHTS_DIR`, and the subagent searches the expected location.
  > **Deviation:** Verified with an explicit subagent dispatch from the implementation
  > session using `THOUGHTS_DIR=/Users/jasonr/dotfiles/thoughts` rather than a separate
  > external caller harness.
  > **Manual Confirmation:** User confirmed this verification approach is accepted for
  > Phase 6 completion.

---

## Testing Strategy

### Unit Tests:

- [x] Resolver helper returns the env override unchanged when `THOUGHTS_DIR` is set.
- [x] Resolver helper rejects relative `THOUGHTS_DIR` overrides with a nonzero exit and
      clear stderr.
- [x] Resolver helper finds `thoughts/` in the starting directory before checking parents.
- [x] Resolver helper finds the nearest parent `thoughts/` when the current level misses.
- [x] Resolver helper exits nonzero with actionable stderr when no `thoughts/` exists.
- [x] Codex adapter emits supporting files with executable mode preserved.

### Integration Tests:

- [x] Supporting-file skills compile and emit correctly for Claude, OpenCode, Codex, and
      Cursor.
- [x] A generated shell-capable skill references the resolver helper and a generated no-shell
      subagent no longer contains independent discovery instructions.

### Manual Testing Steps:

- [x] Build generated outputs: `npm --prefix agent-config run build:agents`
- [x] Run the generated resolver helper from a skill directory with starting path
      `/Users/jasonr/dotfiles/agent-config` and confirm it prints
      `/Users/jasonr/dotfiles/thoughts`
- [x] Run it with `THOUGHTS_DIR=/tmp/custom-thoughts` and confirm the override wins.
- [x] Dispatch `learnings-discoverer` from a caller prompt that includes
      `THOUGHTS_DIR=/Users/jasonr/dotfiles/thoughts` and confirm it searches only that
      location.

## Performance Considerations

- [ ] Resolver execution should remain trivial: a short bounded walk to filesystem root with
      no network or repository introspection.
- [ ] Avoid repeated resolver calls inside a single workflow by resolving once and passing
      the absolute path forward.
- [ ] Keep helper output parseable and minimal so callers do not need complex shell glue.

## Migration Notes

- [ ] Existing fragment-based discovery text can be replaced incrementally as long as the
      new helper contract and no-shell caller handoff land together.
- [ ] Codex supporting-file emission should land before any canonical spec depends on a
      bundled helper for Codex compatibility.
- [ ] Because current generated outputs are version-controlled, review diffs carefully for
      any provider-specific wording regressions introduced by the fragment rewrite.

## References

- `agent-config/spec/fragments/thoughts/discover-dir.md:1`
- `agent-config/spec/agents/learnings-discoverer.md:25`
- `agent-config/spec/skills/gh-safe/SKILL.md:19`
- `agent-config/spec/skills/gh-safe/gh-safe.sh:1`
- `agent-config/scripts/lib/parse.ts:66`
- `agent-config/scripts/lib/adapters/claude.ts:79`
- `agent-config/scripts/lib/adapters/opencode.ts:108`
- `agent-config/scripts/lib/adapters/codex.ts:36`
- `agent-config/scripts/lib/adapters/cursor.ts:31`
- `/Users/jasonr/dotfiles/thoughts/plans/2026-03-15-consistent-thoughts-dir-discovery.md:144`
- `/Users/jasonr/dotfiles/thoughts/research/2026-03-08-opencode-skill-bundled-scripts.md:20`
