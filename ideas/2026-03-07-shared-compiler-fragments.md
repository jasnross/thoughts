# Shared Compiler Fragments for Skill Definitions

## Problem Statement

Canonical skill definitions in `spec/skills/` contain duplicated logic blocks that
must be kept in sync manually. When one skill is updated, others silently drift.
The review skill family (code-review, code-review-loop, review-files, review-pr)
is the most affected, with three distinct patterns duplicated across four or more
files, but the problem likely extends to other skill families.

The pain is twofold: **drift risk** (skills diverge when shared logic changes in
one but not others) and **maintenance burden** (updating shared patterns requires
editing multiple files and verifying consistency by hand).

## Motivation

- **Single source of truth**: Shared behavioral contracts (like the code-reviewer
  prompt format) and common procedures (like merge-base resolution) should be
  defined once and referenced everywhere.
- **Safer refactoring**: Updating a shared pattern should propagate automatically
  to all skills that use it, eliminating silent drift.
- **Reduced skill file size**: Skills that currently embed 40-55 lines of
  duplicated bash/instructions could reference a fragment and focus on their
  unique orchestration logic.
- **Precedent exists**: The `gh-safe` script already extracts executable logic
  into a shared artifact. Compiler fragments would do the same for behavioral
  instruction text.

## Context

### Current architecture

- Canonical skill definitions live in `spec/skills/<id>/SKILL.md` as markdown
  with YAML frontmatter.
- A TypeScript compiler generates provider-specific outputs into `generated/`.
- Skills are self-contained documents — there is no include or composition
  mechanism today.
- The compiler validates frontmatter against `spec/schema/canonical.schema.json`,
  then applies mappings and adapters per provider.

### Duplicated patterns identified (review family)

1. **Base branch & merge-base resolution** (~40-55 lines of bash each)
   - code-review steps 1-2, code-review-loop steps 1-2, review-pr step 5
   - Note: This is executable logic and may be better served as a shell script
     (like `gh-safe`) rather than a compiler fragment. See Constraints.

2. **Code-reviewer prompt contract** (structured prompt format)
   - code-review steps 3-5, code-review-loop step 4b, review-files step 5,
     review-pr step 7
   - All four construct: `Project path / Scope / Base reference / Merge-base / Focus`

3. **Interactive findings triage workflow** (behavioral instructions)
   - code-review step 7, code-review-loop step 5, review-files step 7
   - code-review-loop explicitly says "the same triage workflow as code-review"
     — a fragile prose-level reference
   - review-pr has a different variant (include/edit/exclude) and would not share

4. **GitHub host detection** (small bash block)
   - review-pr step 2, review-pr-comments step 1
   - May be better folded into `gh-safe` auto-detection than a fragment

### What belongs where

Not all duplicated logic should become compiler fragments:

| Pattern | Best form | Rationale |
|---|---|---|
| Merge-base resolution | Shell script | Executable procedure with clear I/O |
| Prompt contract | Compiler fragment | Behavioral instruction text |
| Triage workflow | Compiler fragment | Behavioral instruction text |
| GitHub host detection | `gh-safe` enhancement | Small, executable, already has a home |

This idea focuses on the **compiler fragment** mechanism. Shell script extractions
are tracked separately in `TODO.md`.

## Goals / Success Criteria

- [ ] Authors can define shared text fragments in a known location
- [ ] Skills can reference fragments using inline markdown transclusion syntax
      (e.g., `{{include:_shared/prompt-contract.md}}`)
- [ ] The compiler resolves transclusions at build time, producing self-contained
      generated output (no runtime cross-references for the LLM to follow)
- [ ] Validation catches broken references (missing fragments, circular includes)
- [ ] At least the code-reviewer prompt contract is extracted and shared across
      all four review-delegation skills
- [ ] Generated output is identical (or functionally equivalent) to the current
      hand-duplicated version

## Non-Goals (Out of Scope)

- Runtime cross-references (LLM following links to other files at inference time)
- Fragment parameterization (e.g., `{{include:triage.md mode=interactive}}`) —
  this may be valuable for the triage workflow but adds complexity; evaluate
  separately after static fragments work
- Extracting executable logic into fragments (use shell scripts instead)
- Refactoring non-review skill families (solve the review family first, then
  evaluate broader applicability)
- Changing the generated output format or provider adapters

## Proposed Solutions

### Option A: Existing templating engine (e.g., Handlebars, Nunjucks, Liquid)

- Adopt a mature templating engine with built-in support for includes/partials,
  variables, and conditionals
- Fragments live in a shared directory and are referenced via the engine's
  native partial/include syntax
- Variables could be populated from mappings or frontmatter context, enabling
  provider-specific text within shared fragments (e.g., tool names, invocation
  syntax, or behavioral notes that differ between claude/cursor/codex/opencode)
- The compiler invokes the template engine as a pre-processing step before
  provider mapping/adaptation

Pros: Battle-tested syntax and semantics, parameterization comes for free,
conditionals available if needed, community documentation. Cons: Adds a runtime
dependency, template syntax may be unfamiliar to skill authors, risk of
over-engineering if only simple includes are needed initially.

Candidate engines:
- **Handlebars**: `{{> partial}}`, `{{variable}}`, `{{#if}}` — widely known,
  JS-native, logic-less by default but extensible
- **Nunjucks**: Jinja2-style, `{% include %}`, `{{ variable }}` — more powerful
  but heavier
- **Liquid**: `{% include 'fragment' %}`, `{{ variable }}` — popular in static
  site generators, sandboxed by design

### Option B: Custom compiler-resolved transclusion

- Fragments live in `spec/skills/_shared/` (or similar conventional path)
- Skills reference them inline: `{{include:_shared/prompt-contract.md}}`
- Compiler reads the fragment file and splices its content in place of the
  transclusion marker before applying provider mappings
- Generated output remains a single self-contained document
- Validation step checks for missing files and circular references

Pros: No external dependencies, minimal syntax to learn, tight integration with
existing compiler. Cons: Must build and maintain include resolution, any future
need for variables or conditionals requires custom implementation.

### Option C: Frontmatter-declared fragments

- Fragments declared in YAML frontmatter: `fragments: [prompt-contract, triage]`
- Compiler appends or inserts fragment content at designated anchor points
- Less flexible positioning than inline transclusion

Pros: Structured, schema-validatable. Cons: Less intuitive placement, harder to
control where in the document the fragment appears, no path to variables.

## Constraints

- Must work within the existing TypeScript compiler pipeline
- Generated output must remain self-contained (the LLM should not need to fetch
  external documents at runtime)
- Fragment resolution must happen before provider-specific mapping/adaptation
- Schema validation must be extended to cover fragment references
- Should not break existing skills that don't use fragments

## Open Questions

- Should fragments support variables and parameterization? Two known use cases:
  (1) The triage workflow has variants (interactive vs. auto-fix vs. PR
  disposition) that could be parameterized. (2) Provider-specific behavioral
  differences — values that differ between claude/cursor/codex/opencode targets
  (e.g., tool invocation syntax, capability references) could be injected as
  template variables from the existing mapping context. An existing templating
  engine would handle both for free; a custom solution would need to grow into
  this over time.
- What is the right location for shared fragments? `spec/skills/_shared/` is
  intuitive but might conflict with the per-skill directory convention. Alternatives:
  `spec/fragments/`, `spec/shared/`.
- Are there non-review skill families that would benefit immediately? An audit of
  the plan skills (create-plan, implement-plan, validate-plan, iterate-plan) and
  git skills (git-town, gt-sync, split-branch) would answer this.
- Should fragments be allowed to include other fragments (nested transclusion)?
  Adds power but also complexity and circular reference risk.

## References

- `TODO.md` — tracks all review family extraction items (scripts + fragments)
- `spec/skills/code-review/SKILL.md` — primary source of duplicated patterns
- `spec/skills/code-review-loop/SKILL.md` — explicitly references code-review's
  triage workflow by prose
- `scripts/` — compiler implementation that would need fragment support
- `spec/schema/canonical.schema.json` — schema that may need updates
