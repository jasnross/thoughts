# gityard config subcommand

## Problem Statement

gityard has a capable config system (`config.toml` with four sections, serde
defaults, and `deny_unknown_fields` validation) but no CLI surface for users to
interact with it. Users must know the config file exists, find it on disk, and
edit it manually with no guidance about available fields or their defaults.

This makes the config system effectively invisible to new users and inconvenient
for experienced ones.

## Motivation

- **Discoverability**: Users have no way to learn what config knobs exist
  without reading source code or docs. A `config show` that displays all
  effective values (with default markers) makes the config self-documenting.
- **Convenience**: Common operations like changing a single value require
  finding the file, opening an editor, remembering TOML syntax, and hoping you
  don't introduce a typo in a field name (which `deny_unknown_fields` will
  reject with an opaque parse error).
- **Parity with similar tools**: `git config`, `gh config`, and most modern CLIs
  provide config subcommands. Users expect this surface.

## Context

### Current config system

- **File**: `config.toml` in the platform config directory
  (`~/Library/Application Support/gityard/` on macOS, overridable via
  `GITYARD_CONFIG_DIR`)
- **Format**: TOML with four sections: `status`, `scan`, `defaults`, `fetch`
- **Defaults**: All fields have serde defaults; the file is entirely optional
- **Validation**: `deny_unknown_fields` on every section catches typos but
  produces raw serde error messages
- **No CLI exposure**: Config is loaded in `main()` and threaded through to
  commands, but there's no subcommand for viewing or modifying it

### Config fields (current)

| Section    | Key                | Type           | Default                                    |
| ---------- | ------------------ | -------------- | ------------------------------------------ |
| status     | columns            | Vec\<String\>  | ["repo", "branch", "status", "main", "remote"] |
| scan       | depth              | usize          | 3                                          |
| defaults   | main_branches      | Vec\<String\>  | ["main", "master"]                         |
| fetch      | all                | bool           | false                                      |
| fetch      | tags               | Option\<bool\> | None (git auto-follow)                     |
| fetch      | force              | bool           | false                                      |
| fetch      | prune              | bool           | true                                       |
| fetch      | prune_tags         | bool           | false                                      |
| fetch      | recurse_submodules | bool           | false                                      |
| fetch      | jobs               | Option\<usize\>| None                                       |
| fetch      | depth              | Option\<usize\>| None                                       |

## Goals / Success Criteria

- [ ] Users can inspect the full effective config (defaults + overrides) from the CLI
- [ ] Users can discover the config file path without platform-specific knowledge
- [ ] Users can open the config in their `$EDITOR` with one command
- [ ] `config show` defaults to an annotated human-readable format, with a
      `--toml` flag for valid TOML output

## Non-Goals (Out of Scope)

- **CRUD operations (get/set/unset)** -- programmatic reading and writing of
  individual config values. The goal for this subcommand isn't clear yet, and
  `config edit` covers the modification case. Can be revisited as a follow-up
  once usage patterns emerge.
- **Config init / scaffolding** -- generating a starter config.toml with
  commented fields. Could be a follow-up but not essential given `config show`
  makes defaults visible.
- **Config validation subcommand** -- a dedicated `config validate` command.
  Validation errors should surface clearly in normal operations instead.
- **Schema / list-keys subcommand** -- listing all available keys with types.
  `config show` with default annotations serves this purpose.
- **JSON output format** -- TOML output via `--toml` is sufficient for now;
  JSON can be added later if needed.

## Proposed Subcommands

### `gityard config show`

Display the full effective config with all defaults filled in. Output uses
TOML-like formatting with `# (default)` inline comments on values the user
hasn't explicitly set:

```
[status]
columns = ["repo", "branch", "status", "main", "remote"]  # (default)

[scan]
depth = 3  # (default)

[defaults]
main_branches = ["main", "master"]  # (default)

[fetch]
all = false  # (default)
prune = true
prune_tags = false  # (default)
recurse_submodules = false  # (default)
```

A `--toml` flag outputs valid TOML (no annotations) suitable for piping to a
file.

### `gityard config path`

Print the path to the config file (whether or not it exists). Useful for
scripting and for telling users where to look.

### `gityard config edit`

Open the config file in `$EDITOR` (or `$VISUAL`). If the config file doesn't
exist, create it with a commented defaults template before opening:

```toml
# gityard configuration
# Uncomment and modify values to override defaults.

# [status]
# columns = ["repo", "branch", "status", "main", "remote"]

# [scan]
# depth = 3

# [defaults]
# main_branches = ["main", "master"]

# [fetch]
# all = false
# tags =           # unset = git auto-follow
# force = false
# prune = true
# prune_tags = false
# recurse_submodules = false
# jobs =           # unset = no limit
# depth =          # unset = full clone
```

## Constraints

- The config file may not exist (gityard works with pure defaults), so all
  read operations must handle this gracefully

## Resolved Questions

- **`config edit` bootstrapping**: Create a commented defaults template with all
  fields present but commented out. Users uncomment what they want to change.
- **Output format for `config show`**: TOML-like output with `# (default)`
  inline comments on values that haven't been explicitly set by the user.
- **List value syntax for `set`**: Deferred — CRUD operations (get/set/unset)
  are out of scope for this iteration.

## References

- Current config implementation: `src/config.rs`
- CLI structure: `src/cli.rs`
- Command dispatch: `src/main.rs`
- Similar prior art: `git config`, `gh config`, `cargo config` (proposed)
