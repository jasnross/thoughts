# mdview File Browser UX Improvements

## Problem Statement

The mdview file browser sidebar has two UX friction points that make it harder to navigate large collections of markdown files:

1. **Search results don't persist across navigation.** When searching for files, clicking a result triggers a full page reload that clears the search input. Users who want to work through several search results must re-type their query after every click.

2. **Directory context is lost when scrolling.** In directories with many files, scrolling the sidebar causes the parent directory header to scroll off-screen. There's no visual indicator of which directory the currently visible files belong to, making it easy to lose orientation.

## Motivation

- mdview is designed for browsing large collections of markdown files. The file browser is the primary navigation mechanism, and these friction points compound with scale — the more files you have, the more painful both issues become.
- Search is the fastest way to find files, but its value is diminished if it can't be used as a persistent filter while working through results.
- Directory context is a solved problem in many file explorers (sticky headers, breadcrumbs), and the absence of it feels like a gap.

## Context

The file browser is rendered server-side via MiniJinja templates. Each file click triggers a full page navigation — there is no client-side routing. The sidebar tree uses `<details>` elements for directory expand/collapse, with state persisted in localStorage. Search filtering is entirely client-side via fuzzy matching that hides/shows `<li>` elements.

Key files:
- `src/app.rs` — server-side file tree construction and rendering
- `templates/main.html` — sidebar template with recursive `render_tree` macro
- `static/main.js` — client-side search, directory state, keyboard navigation
- `static/style.css` — sidebar layout and styling

## Goals / Success Criteria

- [ ] Search query persists when navigating between files; the file list stays filtered to matching results
- [ ] Search can be cleared via the native `x` button or by pressing Escape
- [ ] When Escape clears the search, the full unfiltered tree is restored
- [ ] Directory headers stick to the top of the sidebar scroll area when their content is visible
- [ ] Sticky directory headers are interactive — clicking one collapses that directory
- [ ] Nested directories stack their sticky headers naturally (child under parent)
- [ ] Both features work correctly with keyboard navigation (Ctrl+N / Ctrl+P)

## Non-Goals (Out of Scope)

- Changing the navigation model (e.g., moving to client-side routing or SPA)
- Adding a breadcrumb trail or path bar (sticky headers address the same need)
- Changing the search algorithm (fuzzy matching is working well)
- Adding search result highlighting within file content

## Proposed Solutions (High-Level)

### Search Persistence

Persist the search query in localStorage (or a URL query parameter) so it survives full page reloads. On page load, if a saved search query exists, populate the input and re-run the filter. Clear the saved query when the user explicitly clears the input (x button or Escape).

**localStorage vs. URL parameter:** localStorage is simpler and doesn't pollute URLs. A URL parameter would make search state shareable/bookmarkable but adds complexity. localStorage is likely sufficient since search is a transient navigation aid.

### Sticky Directory Headers

Use CSS `position: sticky` on directory `<summary>` elements to pin them to the top of the scroll container as the user scrolls. Nested directories should stack with increasing `top` offsets. Clicking a sticky header should collapse that directory (this already works since `<summary>` elements toggle their parent `<details>`).

**Challenge:** CSS sticky positioning with nested elements and dynamic nesting depths requires careful handling of `top` values and stacking context. May need JavaScript to compute `top` offsets dynamically based on the actual nesting level and the heights of already-stickied ancestors.

## Constraints

- Must work within the existing full-page-navigation architecture
- Should not add external JavaScript dependencies (the project currently has zero)
- Must not break existing keyboard navigation (Ctrl+N/P, Ctrl+Shift+C/E)
- Sticky headers must handle directories with varying nesting depths

## Resolved Questions

- **Escape scope:** Global — pressing Escape anywhere on the page clears the search and restores the full tree. This avoids awkward focus-juggling between the search input (where Ctrl+N/P is disabled) and the rest of the page.
- **Sticky header stacking depth:** No hard cap. All ancestor levels are sticky, but intermediate levels collapse intelligently — only the most immediate ancestor and the root are shown, with intermediate levels collapsed into a compact indicator. This maximizes context while keeping vertical space under control.
- **Keyboard nav + search persistence:** Ctrl+N/P preserves the search query, same as clicking a link. The user is working within their search scope regardless of navigation method. (Note: keyboard nav at `main.js:523-546` already filters out hidden elements, so it naturally respects the preserved search filter.)

## Affected Systems

- `static/main.js` — search persistence logic, sticky header JS (if needed)
- `static/style.css` — sticky positioning styles
- `templates/main.html` — possible template changes for sticky header markup
