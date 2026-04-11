# Compiler Fragments

<!-- migrated-from: /Users/jasonr/dotfiles/agent-config/agent-docs/compiler-fragments.md -->

<!-- learned: 2026-03-22 -->

In the `agent-config/` compiler (`agentspec`), shared instruction text in canonical spec bodies uses
MiniJinja templates resolved at compile time.

## Key Details

- Fragments live in `spec/fragments/` as `.md` files (subdirectories allowed).
- Fragment names: path relative to `spec/fragments/`, **including** the `.md` extension
  (e.g., `review/prompt-contract.md`).
- Referenced in spec bodies with `{% include %}`:
  ```
  {% include "review/prompt-contract.md" %}
  ```
- Variables passed via `{% with %}` blocks:
  ```
  {% with key = "value" %}{% include "review/prompt-contract.md" %}{% endwith %}
  ```
- Indented includes use the `indent` filter:
  ```
  {% filter indent(2, first=false) %}{% include "review/prompt-contract.md" %}{% endfilter %}
  ```
- Conditionals: `{% if flag %}...{% endif %}`, negated with `{% if not flag %}`.
- Whitespace control: `{%-` strips preceding whitespace/newlines at conditional boundaries.
- Variables output with `{{ variable_name }}`.
- Optional boolean flags default to falsy when not passed — no need for `| default(false)`.
- Use single quotes for string values containing double quotes.
- Fragment files must end with a trailing newline.
- Resolution happens after spec loading and before schema validation in the pipeline
  (`agentspec/src/fragments.rs`).
- Fragments can include other fragments (nested MiniJinja includes).
- The compiler is a standalone Rust binary (`agentspec/`), not TypeScript.
