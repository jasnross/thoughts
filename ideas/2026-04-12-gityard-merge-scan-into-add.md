# Merge `scan` Into `add` Command

## Problem Statement

The `gityard scan` and `gityard add` commands do fundamentally the same thing —
register git repositories into the registry via `Registry::add()`. The only
difference is where the list of paths comes from: `add` takes explicit paths
from the user, while `scan` discovers them by walking a directory tree. Having
two separate commands for what is conceptually the same operation ("add repos to
gityard") creates unnecessary cognitive overhead and fragments the CLI surface.

## Motivation

- **Simpler mental model**: "add" is the one command for getting repos into
  gityard, regardless of whether you're pointing at a single repo or a directory
  full of them.
- **Reduced CLI surface**: Fewer top-level commands makes gityard easier to
  learn and document.
- **Natural behavior**: When a user runs `gityard add ~/projects`, the intuitive
  expectation is that gityard figures out what to do with the path — not that it
  errors because the path isn't a git repo.

## Context

Currently in gityard:

- `gityard add <path>...` accepts one or more paths to git repositories.
  Validates that `.git` exists at each path, canonicalizes, deduplicates, and
  saves to the registry.
- `gityard scan <dir> [--depth N]` accepts a single directory, walks it with
  `walkdir` up to a configurable depth, discovers repos by checking for `.git`,
  then registers each one via the same `Registry::add()` method.
- Both commands share the same underlying registration logic in `Registry::add()`
  (`registry.rs`).
- The scanner module (`scanner.rs`) is only used by the `scan` command.

## Goals / Success Criteria

- [ ] `gityard add <path>` works for both single repos and directories
  containing repos, auto-detecting which case applies
- [ ] `--depth` flag available on `add` for controlling scan depth (only
  meaningful when scanning a directory)
- [ ] `scan` subcommand is removed entirely
- [ ] `scanner.rs` module is used by `add`'s handler directly
- [ ] Output is clear about what happened — how many repos were added vs.
  already registered, whether scanning was triggered

## Non-Goals (Out of Scope)

- Glob pattern support (e.g., `gityard add ~/projects/*/`) — may be interesting
  later but not part of this change
- Recursive auto-scan on startup or background scanning
- Changes to the registry format or `Registry::add()` internals

## Proposed Approach

**Auto-detect based on path contents:**

When `gityard add` receives a path:

1. If the path contains a `.git` directory/file → add it directly (current `add`
   behavior)
2. If the path does NOT contain `.git` → treat it as a directory to scan for
   repos underneath (current `scan` behavior)

The `--depth` flag carries over from `scan` to `add`, defaulting to the
configured `config.scan.depth` value. It is only meaningful when a path triggers
scanning; it is ignored (or warned about) when all paths are direct repos.

**Breaking change**: The `scan` subcommand is removed entirely. This is
acceptable because gityard is pre-1.0 with a single user.

## Resolved Questions

- **Mixed paths in one invocation**: Handle seamlessly. Each path is classified
  independently — repos are added directly, non-repo directories are scanned.
  No restrictions on mixing.
- **`--no-scan` escape hatch**: Not needed. Bare repos and unusual layouts are
  rare enough that this doesn't warrant a flag. Easy to add later if the need
  arises.
- **Output format**: Per-path feedback always ("added: ...", "already
  registered: ...", "error: ..."). When scanning was triggered, also print an
  aggregate summary at the end. Single direct-repo adds skip the summary since
  it would be redundant.

## Affected Systems

- `src/cli.rs` — Remove `Scan` variant, add `--depth` to `Add`
- `src/main.rs` — Merge `cmd_scan` logic into `cmd_add`, remove `cmd_scan`
- `src/scanner.rs` — No changes needed, but now called from `cmd_add`
- `src/config.rs` — No changes needed (`scan.depth` config still used)
