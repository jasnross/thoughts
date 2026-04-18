# gityard README

## Problem Statement

gityard has no README. Users who discover the repository have no way to
understand what the tool does, how to install it, or how to use it without
reading source code. A README is the front door to any open-source project.

## Motivation

- First impressions matter: a clear README lets potential users evaluate the tool
  in seconds.
- Installation instructions reduce friction for new users.
- Command documentation with examples demonstrates the tool's value faster than
  a dry feature list.
- A quick-start walkthrough shows the intended workflow, not just individual
  commands.

## Goals / Success Criteria

- [ ] Users can understand what gityard does from a brief summary
- [ ] Users can install gityard with a single command
- [ ] Users can see all available commands with descriptions and representative
      example output
- [ ] Users can follow a quick-start walkthrough to go from zero to productive
- [ ] License is clearly stated

## Proposed Sections

### 1. Summary

One-paragraph description: multi-repository git management CLI providing rich
summarized status across many repos and safe bulk operations with pre-flight
validation.

### 2. Installation

```
cargo install --git https://github.com/jasnross/gityard
```

Single method for now. Can expand later (Homebrew, binaries, etc.).

### 3. Quick Start

Short walkthrough covering the core workflow:

1. Add repos (`gityard add ~/projects`)
2. Check status (`gityard status`)
3. Fetch all (`gityard fetch`)

End with a brief mention of `gityard config show` for customizing defaults.
Shows the tool in action rather than just listing features.

### 4. Commands

Each command with a brief description and example output where it helps
illustrate the tool's value. Key commands to show examples for:

- `status` -- the flagship command; show a sample status table
- `branches` -- cross-repo branch overview
- `fetch` / `pull` -- bulk operations with pre-flight validation
- `add` / `remove` / `list` -- registry management
- `worktrees` -- cross-repo worktree listing
- `trim` -- stale branch cleanup
- `group` subcommands -- repo grouping
- `config` subcommands -- configuration management
- `completions` -- shell completion generation

### 5. License

One-liner noting dual-license: MIT and Apache 2.0.

## Non-Goals (Out of Scope)

- Detailed configuration reference (can be a follow-up or separate doc)
- Contributing guide
- CI badges or project status indicators
- Comparison with similar tools
- Repo groups deep-dive (mention in commands, but no dedicated section yet)
- Filtering flags reference (mention in command descriptions, but no dedicated
  section)

## Resolved Questions

- **Repo URL**: `https://github.com/jasnross/gityard` (confirmed from git remote)
- **Example style**: Stylized/hand-crafted output — shows the shape without
  coupling to exact formatting, reducing maintenance burden
- **Quick-start scope**: Core add-status-fetch loop, with a brief closing
  mention of `gityard config show` for customization

## Constraints

- Example output must be kept in sync when commands change — consider this
  maintenance cost when deciding how much output to include.
- The README should work well both on GitHub (rendered markdown) and in terminal
  viewers.

## References

- Existing CLAUDE.md has project description and command reference (internal dev
  docs, not user-facing)
- CLI is defined in `src/cli.rs` with clap derive macros — command help text
  there is the source of truth for descriptions
