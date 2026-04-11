# Learn Skill: Migrate to thoughts/learnings/ Implementation Plan

## Overview

Migrate the `/learn` skill's storage from dual `agent-docs/` directories
(`<project-root>/agent-docs/` and `~/.config/agent-docs/`) to a single flat
destination: `$THOUGHTS_DIR/learnings/`. Update `learnings-discoverer` to
search the new location. Migrate existing files and delete old directories.

## Current State Analysis

- **Learn skill** (`spec/skills/learn/SKILL.md`, 301 lines): Resolves
  per-insight scope via a walk-up algorithm (looks for `AGENTS.md`/`CLAUDE.md`
  markers, falls back to git root) and writes to one of two locations. After
  writing, checks `~/.claude/CLAUDE.md` for `agent-docs` and suggests adding
  the dispatch instruction if absent.
- **learnings-discoverer** (`spec/agents/learnings-discoverer.md`): Searches
  two fixed paths — `<project-root>/agent-docs/*.md` and
  `~/.config/agent-docs/*.md` — using Glob/Grep/Read (no bash tool). Returns
  relevant H2 sections verbatim with source attribution.
- **discover-dir fragment** (`spec/fragments/thoughts/discover-dir.md`):
  Already exists. Five-step walk-up algorithm, `{{#if writeable}}` controls
  whether Step 5 creates directories or fails hard. Step 4 performs a bash
  `export THOUGHTS_DIR=<path>`. Used by 14 skills.
- **AGENTS_GLOBAL.md**: Canonical source for user-level rules, symlinked to
  `~/.claude/CLAUDE.md`, `~/.codex/AGENTS.md`, and
  `~/.config/opencode/AGENTS.md` by `setup.sh`.
- **setup.sh line 222**: Grants OpenCode a hardcoded permission for
  `~/.config/agent-docs/**`.
- **5 files to migrate**:
  - Project-scoped (agent-config): `canonical-command-conventions.md`,
    `compiler-fragments.md`, `generated-artifacts.md`
  - Global (no project context): `git-commit-strategies.md`,
    `git-integration-workflows.md`

## Desired End State

- `/learn` writes all new insights to `$THOUGHTS_DIR/learnings/<topic>.md`
- `learnings-discoverer` searches `$THOUGHTS_DIR/learnings/*.md`
- Existing 5 files migrated into `$THOUGHTS_DIR/learnings/` as-is (prose
  already carries sufficient project context)
- Old `agent-docs/` directories deleted
- `AGENTS_GLOBAL.md` Knowledge Base section updated to reference new location
- `setup.sh` OpenCode permission updated
- `agent-config/CLAUDE.md` project KB reference removed

### Verification

- Run `/learn` in a session: proposed file path shows `thoughts/learnings/`
- Run `learnings-discoverer` against the agent-config project: returns
  insights from `thoughts/learnings/` with correct source attribution
- `~/.config/agent-docs/` and `agent-config/agent-docs/` directories do not
  exist

### Key Discoveries

- `~/.claude/CLAUDE.md` is a symlink (not a file) — edit `AGENTS_GLOBAL.md`
  only
- 14 existing skills use `{{> thoughts/discover-dir}}` — use
  `{{#unless skip_export}}` instead of `{{#if exportable}}` to avoid touching
  any of them
- `learnings-discoverer` has no bash tool — the export step must be skipped
  via the new `skip_export` flag
- `setup.sh:222` hardcodes `~/.config/agent-docs/**` as an OpenCode permission;
  must be updated to `$DOTFILES_PATH/thoughts/**`

## What We're NOT Doing

- Changing the learning file format (H1 topic, H2 insights,
  `<!-- learned: -->` comment, prose body)
- Changing how insights are identified or what counts as a good learning
- Removing the confirmation step before writing
- Splitting or reorganizing topic files
- Changing the migration script to an agent skill
- Adding a transition/fallback period (immediate cutover after migration)
- Injecting `<!-- source: -->` comments during migration — prose in each file
  already carries sufficient project context

## Implementation Approach

Sequential phases with a build at the end. Phase 0 is a prerequisite for
Phase 2 but all spec changes (Phases 0–3) can be done before the first build.
Migration runs after deploy (Phase 5), not before, so the new discoverer is
live when migration completes and old directories are deleted atomically.

---

## Phase 0: Extend discover-dir Fragment

### Overview

Add `{{#unless skip_export}}` around the bash export line in Step 4 of
`spec/fragments/thoughts/discover-dir.md`. This allows the learnings-discoverer
(which has no bash tool) to use the fragment with `skip_export=true`. All 14
existing callers are unaffected — `skip_export` is undefined (falsy), so the
export step still runs for them.

### Changes Required

#### 1. `spec/fragments/thoughts/discover-dir.md`

**File**: `spec/fragments/thoughts/discover-dir.md`

- [x] Wrap the export bullet in Step 4 with `{{#unless skip_export}}` /
      `{{/unless}}`

The current Step 4 export line (line 27–29):
```
- Export it to the shell environment: run `export THOUGHTS_DIR=<absolute-path>` via bash.
  This ensures subsequent skill invocations within the same session find `$THOUGHTS_DIR`
  already set in Step 1 and skip re-discovery entirely.
```

Replace Step 4 with:
```markdown
**Step 4 — Select best match and export:**
- Use the first `thoughts/` found — that is, the one with the fewest `../` hops from the
  workspace root (the one deepest in the filesystem path, closest to the workspace).
- Resolve the selected path to an absolute path. Store as `$THOUGHTS_DIR`.
{{#unless skip_export}}
- Export it to the shell environment: run `export THOUGHTS_DIR=<absolute-path>` via bash.
  This ensures subsequent skill invocations within the same session find `$THOUGHTS_DIR`
  already set in Step 1 and skip re-discovery entirely.
{{/unless}}
```

### Success Criteria

#### Automated Verification

- [x] Fragment file contains `{{#unless skip_export}}` and `{{/unless}}`
- [x] Build succeeds: `npm --prefix /Users/jasonr/dotfiles/agent-config run build:agents`
- [x] No generated output changes for existing skills (since `skip_export` is
      not passed by any current caller, the rendered output should be identical
      to before): verified `create-idea` still renders the export line while
      `learnings-discoverer` does not

---

## Phase 1: Update Learn Skill

### Overview

Replace the per-insight scope resolution algorithm with `$THOUGHTS_DIR`
discovery. All insights go to `$THOUGHTS_DIR/learnings/<topic>.md`. Remove the
project/global distinction. Update the post-write CLAUDE.md config check.

### Changes Required

#### 1. `spec/skills/learn/SKILL.md`

**File**: `spec/skills/learn/SKILL.md`

- [x] **Remove the scope resolution algorithm** (lines 93–135): the entire
      block describing how to determine `project:<root>` vs `global` scope —
      the walk-up for `AGENTS.md`/`CLAUDE.md` markers, git root fallback,
      workspace guardrail, and tie-breaker logic
- [x] **Add `$THOUGHTS_DIR` discovery** in place of scope resolution:

  ```
  ## Discover $THOUGHTS_DIR

  {{> thoughts/discover-dir writeable=true}}
  ```

- [x] **Update storage location** (lines 215–222): replace the two-path model
      with a single destination:

  ```
  **Storage location**: `$THOUGHTS_DIR/learnings/<topic-name>.md`
  ```

  Remove all references to `<project-root>/agent-docs/` and
  `~/.config/agent-docs/`.

- [x] **Update the Propose Learnings section** (lines 149–184): remove scope
      labels (`project:` / `global:`) from the proposal format. The proposal
      format becomes:

  ```
  - `$THOUGHTS_DIR/learnings/<topic>.md` → [new file / append]
    ## <Insight Title>
    <!-- learned: YYYY-MM-DD -->
    <preview>
  ```

- [x] **Update existing-file discovery** (lines 228–230): replace the two Glob
      patterns (`agent-docs/*.md` in project root and
      `~/.config/agent-docs/*.md`) with a single:

  ```
  Glob `$THOUGHTS_DIR/learnings/*.md` to discover existing topic files.
  ```

- [x] **Update post-write config check** (lines 255–284): replace the check
      for `agent-docs` string in CLAUDE.md with a check for
      `learnings-discoverer`. If the string `learnings-discoverer` is absent
      from `~/.claude/CLAUDE.md`, suggest adding the Knowledge Base section.
      Update the suggested text to reference `$THOUGHTS_DIR/learnings/` instead
      of the old agent-docs paths.

### Success Criteria

#### Automated Verification

- [x] No remaining references to `agent-docs` in `spec/skills/learn/SKILL.md`:
      only remaining matches are the `learnings-discoverer` subagent name
      (expected — that's the agent's ID, not a path reference)
- [x] Fragment include present: `grep -c 'thoughts/discover-dir' spec/skills/learn/SKILL.md`
      returns `1`
- [x] Build succeeds: `npm --prefix /Users/jasonr/dotfiles/agent-config run build:agents`

#### Manual Verification

- [ ] Invoke `/learn` in a session, propose a learning: the proposed file path
      shows `$THOUGHTS_DIR/learnings/<topic>.md`
- [ ] Confirm and write: file appears at `$THOUGHTS_DIR/learnings/<topic>.md`
- [ ] No scope labels (`project:` / `global:`) appear in the proposal

**Implementation Note**: After completing this phase and all automated
verification passes, pause here for manual confirmation that `/learn` proposes
and writes to the correct location before proceeding to Phase 2.

---

## Phase 2: Update learnings-discoverer

### Overview

Replace the two fixed search paths with `$THOUGHTS_DIR/learnings/` discovered
via the `discover-dir` fragment (using `skip_export=true` since the discoverer
has no bash tool). Update search algorithm, constraints, and output format.

### Changes Required

#### 1. `spec/agents/learnings-discoverer.md`

**File**: `spec/agents/learnings-discoverer.md`

- [x] **Add `$THOUGHTS_DIR` discovery** before the search algorithm section.
      Use:

  ```
  ## Discover $THOUGHTS_DIR

  {{> thoughts/discover-dir skip_export=true}}
  ```

  Since `writeable` is not passed (falsy), the fragment will fail hard if no
  `thoughts/` directory is found — correct behavior for a read-only subagent.

- [x] **Update Step 1** (discover available topics): replace the two Glob
      patterns with a single:

  ```
  Glob `$THOUGHTS_DIR/learnings/*.md`
  If no files exist, report that no learnings are available and stop.
  ```

- [x] **Update Step 3** (keyword grep): replace the two directory targets with
      `$THOUGHTS_DIR/learnings/`

- [x] **Update the file structure section** (lines 39–60): replace the
      `agent-docs/` directory description with `$THOUGHTS_DIR/learnings/`

- [x] **Update the "Must not" constraint** (line 147): change "Don't read files
      outside of `agent-docs/` directories" to "Don't read files outside of
      `$THOUGHTS_DIR/learnings/`"

- [x] **Update the output format** (lines 94–122): remove the
      `### Project: /path` / `### Global (~/.config/agent-docs/)` grouping.
      Replace with a single flat listing:

  ```
  ### Learnings ($THOUGHTS_DIR/learnings/)

  **<filename.md> — <Insight Title>**
  <!-- learned: YYYY-MM-DD -->
  <verbatim insight content>
  ```

  Source context (which project the insight came from) is embedded in the
  insight prose itself — the existing file content already makes this clear.

### Success Criteria

#### Automated Verification

- [x] No remaining `agent-docs` references in
      `spec/agents/learnings-discoverer.md`:
      only remaining matches are the agent's own ID `learnings-discoverer`
      (expected)
- [x] Fragment include present:
      `grep -c 'thoughts/discover-dir' spec/agents/learnings-discoverer.md`
      returns `1`
- [x] `skip_export=true` present in the fragment include
- [x] Build succeeds: `npm --prefix /Users/jasonr/dotfiles/agent-config run build:agents`

#### Manual Verification

- [ ] Dispatch `learnings-discoverer` from a fresh session in the agent-config
      project: it returns insights from `$THOUGHTS_DIR/learnings/` (not from
      `agent-docs/`)

**Implementation Note**: Pause after manual verification before proceeding.

---

## Phase 3: Update Configuration Files

### Overview

Update `AGENTS_GLOBAL.md` (which is symlinked as `~/.claude/CLAUDE.md`,
`~/.codex/AGENTS.md`, `~/.config/opencode/AGENTS.md`), remove the redundant
project KB section from `agent-config/CLAUDE.md`, and update the OpenCode
external-directory permission in `setup.sh`.

### Changes Required

#### 1. `AGENTS_GLOBAL.md`

**File**: `AGENTS_GLOBAL.md`

- [x] Update the `## Knowledge Base` section (lines 3–10). Replace:

  ```markdown
  Before starting tasks, dispatch the `learnings-discoverer` subagent with the
  project root path and a description of the task, technologies, and components
  involved. It searches both project-level (`<project-root>/agent-docs/`) and
  global (`~/.config/agent-docs/`) knowledge bases, returning relevant insights
  verbatim with source attribution. Apply any returned insights as context for
  the current task.
  ```

  With:

  ```markdown
  Before starting tasks, dispatch the `learnings-discoverer` subagent with a
  description of the task, technologies, and components involved. It searches
  `$THOUGHTS_DIR/learnings/` for relevant insights and returns them verbatim
  with source attribution. Apply any returned insights as context for the
  current task.
  ```

#### 2. `agent-config/CLAUDE.md`

**File**: `CLAUDE.md` (project root)

- [x] Remove the `## Project Knowledge Base` section (lines 26–29):

  ```markdown
  ## Project Knowledge Base

  Before starting tasks, check `agent-docs/` for project-specific patterns,
  gotchas, and conventions captured from previous sessions.
  ```

  The global `AGENTS_GLOBAL.md` rule already handles knowledge base dispatch
  and makes this section redundant.

#### 3. `setup.sh`

**File**: `setup.sh`

- [x] Update line 222 in `setup_opencode_default_settings()`:

  Change:
  ```bash
  yq -i -o=json '.permission.external_directory["~/.config/agent-docs/**"] = "allow"' "$opencode_settings"
  ```
  To:
  ```bash
  yq -i -o=json '.permission.external_directory["'"$DOTFILES_PATH"'/thoughts/**"] = "allow"' "$opencode_settings"
  ```

### Success Criteria

#### Automated Verification

- [x] No `agent-docs` references remain in `AGENTS_GLOBAL.md`:
      only remaining match is `learnings-discoverer` subagent name (expected)
- [x] No `agent-docs` references remain in `CLAUDE.md`:
      `grep -c 'agent-docs' CLAUDE.md` returns `0`
- [x] `setup.sh` references `$DOTFILES_PATH/thoughts/**`:
      `grep 'DOTFILES_PATH.*thoughts' setup.sh` returns a match
- [x] Build succeeds: `npm --prefix /Users/jasonr/dotfiles/agent-config run build:agents`

---

## Phase 4: Write Migration Script

### Overview

Write `scripts/migrate-agent-docs.sh` — a shell script that copies existing
`agent-docs/` files into `$THOUGHTS_DIR/learnings/`, prepending a
per-source-file marker before each contribution. The marker serves as both
idempotency guard and source attribution. Safe to run multiple times and across
machines; handles same-named files from different sources correctly.

### Changes Required

#### 1. `scripts/migrate-agent-docs.sh`

**File**: `scripts/migrate-agent-docs.sh` (new file)

- [x] Create the script with `#!/usr/bin/env bash` and `set -euo pipefail`
- [x] **Resolve `$THOUGHTS_DIR`**: walk up from `$DOTFILES_PATH` looking for a
      `thoughts/` subdirectory. Fail with a clear error if not found.
- [x] **Create `$THOUGHTS_DIR/learnings/`** if it does not exist
- [x] **`migrate_file <source_file>`** function:
  - Derive `dest_file` as `$THOUGHTS_DIR/learnings/$(basename "$source_file")`
  - Compute the per-source idempotency marker:
    `<!-- migrated-from: <absolute-source-path> -->`
  - If `dest_file` already contains this exact marker: skip (already migrated
    from this source)
  - If `dest_file` already exists (content from a different source): append a
    blank line, then the marker, then the source file content
  - If `dest_file` does not exist: write the marker followed by the source
    file content
- [x] **Migrate project files** from
      `$DOTFILES_PATH/agent-config/agent-docs/*.md`
- [x] **Migrate global files** from `$HOME/.config/agent-docs/*.md`
- [x] **Print post-migration instructions** telling the user to review the
      migrated files, then manually delete the source directories:
      ```
      rm -rf $DOTFILES_PATH/agent-config/agent-docs
      rm -rf $HOME/.config/agent-docs
      ```
      Do NOT auto-delete — manual review step required
- [x] Make the script executable: `chmod +x scripts/migrate-agent-docs.sh`

Example output for a same-named file present in both sources:
```markdown
# Git Commit Strategies

<!-- migrated-from: /Users/jasonr/dotfiles/agent-config/agent-docs/git-commit-strategies.md -->

## Project-specific insight
<!-- learned: 2026-02-12 -->
...

<!-- migrated-from: /Users/jasonr/.config/agent-docs/git-commit-strategies.md -->

## Global insight
<!-- learned: 2026-02-15 -->
...
```

Note: for new files, the H1 is placed first, then the marker, then the body.
For appended content, the H1 is stripped (the file already has one) and the
marker precedes the body.

### Success Criteria

#### Automated Verification

- [x] Script is executable: `test -x scripts/migrate-agent-docs.sh`
- [x] Script passes shellcheck: `shellcheck scripts/migrate-agent-docs.sh`
      (SC1091 info about not following `_common.sh` source — same pattern as
      `setup.sh`, not a real issue)

#### Manual Verification

- [ ] Run the script: `scripts/migrate-agent-docs.sh`
- [ ] Verify `$THOUGHTS_DIR/learnings/` contains the 5 expected files
- [ ] Verify each file contains `<!-- migrated-from: <absolute-path> -->`
- [ ] Run the script a second time: no files are re-migrated (idempotency check)

**Implementation Note**: After verifying migration output, manually delete the
source directories and confirm they are gone before proceeding.

---

## Phase 5: Build, Deploy, and Verify

### Overview

Run the full build, deploy via `setup.sh`, and run end-to-end verification.

### Changes Required

- [x] **Build**: `npm --prefix /Users/jasonr/dotfiles/agent-config run build:agents`
- [x] **Run migration script**: `scripts/migrate-agent-docs.sh`
- [x] **Review migrated files** in `$THOUGHTS_DIR/learnings/`
- [x] **Delete old directories**:
  - `rm -rf /Users/jasonr/dotfiles/agent-config/agent-docs`
  - `rm -rf /Users/jasonr/.config/agent-docs`
- [x] **Run setup.sh**: `setup.sh` (re-links AGENTS_GLOBAL.md and updates
      OpenCode permissions)
- [x] **Commit all spec and generated changes** to dotfiles

### Success Criteria

#### Automated Verification

- [x] Build produces no errors
- [x] `$THOUGHTS_DIR/learnings/` contains 5 migrated files
- [x] `agent-config/agent-docs/` directory does not exist
- [x] `~/.config/agent-docs/` directory does not exist
- [x] Generated discoverer output references `$THOUGHTS_DIR/learnings/`:
      confirmed via grep on generated file
- [x] `~/.claude/CLAUDE.md` (symlink target) does not mention `agent-docs/`:
      `grep -c 'agent-docs/' AGENTS_GLOBAL.md` returns `0`
- [x] OpenCode settings contain the thoughts permission:
      `/Users/jasonr/dotfiles/thoughts/**` key confirmed in opencode.json
      > **Deviation:** The old `~/.config/agent-docs/**` permission key was
      > not removed from opencode.json — `yq` adds the new key but doesn't
      > remove the old one. The key is harmless since the directory no longer
      > exists. Cleaning it up would require a separate `yq del` call in
      > `setup.sh`, which is out of scope.

#### Manual Verification

- [x] Start a fresh Claude Code session in `agent-config/`
- [x] Invoke `/learn` and propose a new learning: path shows
      `$THOUGHTS_DIR/learnings/`
- [x] Dispatch `learnings-discoverer` for an agent-config task: insights from
      migrated files are returned correctly
- [x] Confirm no references to old `agent-docs/` paths appear in any agent
      output

---

## Testing Strategy

### Automated

- Build passes after each phase
- Grep assertions confirm no stale `agent-docs` references in edited specs

### Manual

- End-to-end `/learn` flow in a fresh session
- End-to-end `learnings-discoverer` dispatch in a fresh session
- Idempotency check on the migration script

## Migration Notes

- The migration script must run **after** `setup.sh` deploys the new
  `learnings-discoverer` (so the discoverer can find the migrated content the
  moment migration completes)
- Old directories are deleted **after** verifying migration output — never
  auto-deleted by the script
- The per-source `<!-- migrated-from: <path> -->` markers make the script safe
  to run on a second machine that already has the new system deployed but
  hasn't run migration yet

## References

- Idea file: `agent-config/thoughts/ideas/2026-03-17-learn-skill-thoughts-dir.md`
- Learn skill: `spec/skills/learn/SKILL.md`
- Discoverer: `spec/agents/learnings-discoverer.md`
- Fragment: `spec/fragments/thoughts/discover-dir.md`
- Setup script: `setup.sh`
- Existing agent-docs: `agent-config/agent-docs/`, `~/.config/agent-docs/`
