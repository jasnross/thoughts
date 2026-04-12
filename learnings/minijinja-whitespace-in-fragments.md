<!-- learned: 2026-04-11 -->

## MiniJinja whitespace control with `{% filter indent %}` and `{% for %}` loops

When a fragment uses a `{% for %}` loop to generate a list, the loop produces a leading
`\n` before the first item (from the newline between the `%}` closing tag and the list
item text). This leading newline causes problems when the fragment is included inside a
`{% filter indent(N, first=false) %}` wrapper:

- The empty first line (from the leading `\n`) passes through as the "first line"
- `first=false` leaves it unindented, creating a visible blank line before the list
- All subsequent lines get indented correctly

### Fix approaches

**Caller-side (preferred):** Use `{%- filter indent(N) -%}` with full whitespace
stripping on the filter tag. The `{%-` strips the newline between the preceding text and
the filter block, so the fragment's leading `\n` becomes the natural line break between
the intro text and the list content. This avoids modifying the fragment.

Example:
```
   - We have specialized agents:
{%- filter indent(5) -%}
{%- include "research/sub-agent-catalog.md" -%}
{%- endfilter -%}
```

**Fragment-side (alternative):** Strip the newline after the `for` tag with `-%}` and
use `{{ "\n" if not loop.first }}` to insert separators between items but not before the
first:

```
{%- for item in list -%}
{{ "\n" if not loop.first }}- **{{ item.name }}**: {{ item.description }}
{%- endfor %}
```

### Trailing newline stripping

The agentspec Claude adapter strips trailing newlines from agent and skill body content
during serialization. This means `{%- endwith %}` vs `{% endwith %}` on the last line of
a skill spec has no effect on the generated output — the trailing newline is lost
regardless. This is an agentspec adapter behavior, not a MiniJinja issue.
