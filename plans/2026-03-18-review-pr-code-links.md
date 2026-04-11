# Add PR Code Links to review-pr Finding Walkthrough

## Overview

Update `spec/skills/review-pr/SKILL.md` so that each finding presented during
the step 9 triage walkthrough includes a clickable GitHub blob permalink to the
relevant code. This lets the reviewer quickly jump to the file in the browser
while deciding how to disposition each finding.

## Current State Analysis

- The code-reviewer agent outputs findings with `**[File:line]**` references
  (relative paths with absolute line numbers in the new file).
- Step 9 presents findings as plain text: severity, file, line, description.
- `$PR_URL` is captured in step 3 but only used at the end (step 12) for the
  posted review URL.
- The HEAD commit SHA is not captured anywhere in the current flow.
- No mechanism exists to construct GitHub URLs from file:line references.

### Key Discoveries:

- `**[File:line]**` references from the code-reviewer are relative paths with
  absolute line numbers — exactly what blob URLs need (`code-reviewer.md:126`).
- `$PR_URL` follows the pattern `https://{host}/{owner}/{repo}/pull/{number}`,
  so the repo base URL is derivable by stripping `/pull/{number}`
  (`SKILL.md:56`).
- The PR branch checkout in step 4 ensures HEAD matches the PR tip, so
  `git rev-parse HEAD` gives the correct SHA to pin permalinks to
  (`SKILL.md:58-66`).

## Desired End State

When walking through findings in step 9, each finding includes a blob permalink
like:

```
**[Blocker]** `src/auth.ts:42` ([view](https://github.com/owner/repo/blob/abc123/src/auth.ts#L42))
Missing null check on session token before database lookup.
```

The link is pinned to the HEAD SHA at review time, making it stable regardless
of subsequent commits to the PR.

### Verification:

- [x] `npm --prefix agent-config run build:agents` passes after changes
- [x] Generated output for review-pr skill includes the new permalink
      construction instructions in step 5 and the updated presentation format
      in step 9

## What We're NOT Doing

- Not linking to the PR files/diff view (those URLs use MD5 file-path hashes
  and are fragile to construct).
- Not changing the code-reviewer agent's output format — it already provides
  exactly the data we need.
- Not adding a `gh-safe` subcommand — blob URLs are constructed from data
  already available in the flow.
- Not changing how step 10 maps findings to inline comments — that logic is
  unrelated.

## Implementation Approach

Two small, self-contained edits to `spec/skills/review-pr/SKILL.md`:

1. Extend step 5 to also capture `HEAD_SHA` and `REPO_URL`.
2. Update step 9's finding presentation format to include a blob permalink.

No new files, fragments, or commands needed.

## Phase 1: Capture permalink components in step 5

### Overview

After the merge-base is resolved, capture the two additional values needed to
construct blob permalinks.

### Changes Required:

#### 1. `spec/skills/review-pr/SKILL.md` — Step 5

**File**: `spec/skills/review-pr/SKILL.md`

- [x] Add `HEAD_SHA` and `REPO_URL` capture after the existing merge-base block

The updated step 5 should read:

~~~markdown
### 5. Resolve merge-base and permalink components

Use remote-tracking refs for an accurate merge-base, matching the pattern from `code-review`:

```bash
git fetch origin "$PR_BASE_BRANCH" --quiet 2>/dev/null
MERGE_BASE=$(git merge-base HEAD "origin/$PR_BASE_BRANCH")
```

Capture values for constructing code permalinks during triage:

```bash
HEAD_SHA=$(git rev-parse HEAD)
REPO_URL=$(echo "$PR_URL" | sed 's|/pull/[0-9]*$||')
```

Store `$HEAD_SHA` and `$REPO_URL` for use in step 9.
~~~

### Success Criteria:

#### Automated Verification:

- [x] `npm --prefix agent-config run build:agents` passes

## Phase 2: Add permalink to finding presentation in step 9

### Overview

Update the step 9 finding walkthrough to include a blob permalink with each
finding, so the reviewer can click through to the code.

### Changes Required:

#### 1. `spec/skills/review-pr/SKILL.md` — Step 9

**File**: `spec/skills/review-pr/SKILL.md`

- [x] Update the finding presentation instructions (lines 119-126) to include
      a constructed blob permalink

The updated step 9 opening should read:

~~~markdown
### 9. Walk through findings

Work through each remaining finding collaboratively, one at a time, in severity order: Blockers → Major → Minor. For each finding:

1. Present the finding with a permalink to the code:
   - Severity, file, line, and the issue description
   - Construct a blob permalink: `$REPO_URL/blob/$HEAD_SHA/<file>#L<line>`
   - Format: `**[Severity]** \`file:line\` ([view](<permalink>)) — description`
2. Ask the user what to do:
   - **Include** — add to the PR review as-is
   - **Edit** — adjust the comment text, severity, or remove the file/line reference before including
   - **Exclude** — drop it from the review entirely
3. If editing, present the updated comment for confirmation before moving on.
~~~

### Success Criteria:

#### Automated Verification:

- [x] `npm --prefix agent-config run build:agents` passes

#### Manual Verification:

- [ ] Run `/review-pr` on a test PR and confirm each finding in step 9 includes
      a clickable `(view)` link that opens the correct file and line on GitHub

## Testing Strategy

### Automated:

- Build validation via `npm --prefix agent-config run build:agents`

### Manual:

1. Run `/review-pr` against a PR with known findings
2. Verify each finding in the triage walkthrough includes a `(view)` link
3. Click the link — confirm it opens the correct file at the correct line
4. Confirm the link uses the HEAD SHA (check the URL contains a commit hash,
   not a branch name)

## References

- Code-reviewer output format: `spec/agents/code-reviewer.md:118-145`
- Current step 9: `spec/skills/review-pr/SKILL.md:117-140`
- Current step 5: `spec/skills/review-pr/SKILL.md:68-75`
- PR metadata capture: `spec/skills/review-pr/SKILL.md:56`
