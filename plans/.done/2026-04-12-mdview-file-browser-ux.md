# mdview File Browser UX Improvements — Implementation Plan

## Overview

Two UX improvements to the mdview file browser sidebar: (1) persist the search
query across full-page navigations so the filtered view is maintained while
working through results, and (2) add a breadcrumb overlay bar that shows the
current directory context when the user scrolls past directory headers.

## Current State Analysis

### Search

The search input (`#fileSearch`) binds an `input` event listener
(`main.js:282`) that calls `filterTree()` on every keystroke. Filtering is
purely client-side — it sets `el.style.display` on `<li>` elements and forces
`details.open = true` on matching directories. When the query is cleared,
`initDirState()` re-reads localStorage to restore saved directory open/close
state.

File links are standard `<a href>` elements. Clicking one triggers a full page
navigation, which rebuilds the DOM and resets the search input to empty. This is
why the search "clears" — it's not intentional, just a side effect of the
navigation model.

### Directory Context

The sidebar renders a nested tree of `<details>/<summary>` elements inside
`.file-list-container` (`style.css:239`, `overflow-y: auto`). There is no
breadcrumb, path indicator, or sticky header — the only way to know which
directory you're in is to scroll up until you see its `<summary>` element.

### Key Discoveries

- No existing Escape key handler — both `keydown` listeners guard on `e.ctrlKey`
  (`main.js:499, 553`), so a plain Escape press passes through without conflict
- `initDirState()` re-attaches toggle listeners on every call (`main.js:205`),
  including when search is cleared (`main.js:285`). Pre-existing bug — not in
  scope to fix, but the search persistence change avoids making it worse
- `.sidebar` has `overflow: hidden` (`style.css:180`) which would break CSS
  `position: sticky` for descendants. Not relevant for the breadcrumb overlay
  approach (which uses a fixed element outside the scroll container), but worth
  noting
- The scroll container is `.file-list-container` (`overflow-y: auto`) — the
  breadcrumb overlay and IntersectionObserver must use this as their root
- localStorage keys in use: `sidebar-collapsed`, `toc-collapsed`,
  `sidebar-width`, `toc-width`, `scroll-y:{path}`, `dir-open:{path}`. New keys:
  `file-search-query`

## Desired End State

After implementation:

1. **Search persistence**: Typing a search query filters the file list. Clicking
   a file (or pressing Ctrl+N/P) navigates to that file, and the search query +
   filtered view persist on the new page. The search is cleared only when the
   user explicitly clears it (x button or Escape from anywhere on the page).

2. **Breadcrumb overlay**: When scrolling the file list, a breadcrumb bar
   appears at the top of `.file-list-container` showing the directory context of
   the currently visible files. For deeply nested directories (>2 ancestor
   levels scrolled past), intermediate levels are collapsed to
   `[root] > … > [immediate parent]`. Clicking a breadcrumb crumb collapses that
   directory. Clicking the `…` indicator expands to show all intermediate
   levels. The bar disappears when scrolled to the top (no directories are
   scrolled past).

### Verification

- `cargo build --release` compiles
- `cargo test` passes
- Manual: open mdview on a directory with nested subdirectories, search for a
  file, click it, verify query persists. Press Escape, verify search clears.
  Scroll the file list past directory headers, verify breadcrumb bar appears
  with correct context. Click a breadcrumb, verify the directory collapses.

## What We're NOT Doing

- Changing the navigation model (full-page nav stays)
- Adding CSS `position: sticky` on `<summary>` elements (breadcrumb overlay
  approach instead)
- Fixing the duplicate toggle listener bug in `initDirState()`
- Adding search result highlighting within file content
- Adding a breadcrumb trail in the content area

## Implementation Approach

Both features are client-side-only changes: JavaScript (main.js), CSS
(style.css), and one small template addition (main.html) for the breadcrumb
container. No Rust/server changes needed.

The breadcrumb overlay uses `IntersectionObserver` with the
`.file-list-container` as root to detect which `<summary>` elements have
scrolled above the visible area. This avoids the complexities of CSS sticky
positioning inside nested `<details>` elements with overflow containers.

## Implementation

### Overview

Add search persistence via localStorage and a breadcrumb overlay bar using
IntersectionObserver. Both features touch the same three files: `main.js`,
`style.css`, and `main.html`.

### Changes Required

#### 1. Search Persistence (`main.js`)

**File**: `static/main.js`

- [x] Modify `initSearch()` to save query to localStorage on every `input`
  event. Use key `"file-search-query"`. When the value is cleared (empty
  string), remove the key from localStorage.

```javascript
function initSearch() {
  const input = document.getElementById("fileSearch");
  if (!input) return;

  // Restore saved search query
  const saved = localStorage.getItem("file-search-query");
  if (saved) {
    input.value = saved;
    filterTree(saved);
  }

  input.addEventListener("input", function () {
    const q = input.value.trim();
    if (q) {
      localStorage.setItem("file-search-query", q);
    } else {
      localStorage.removeItem("file-search-query");
    }
    filterTree(q);
    if (!q) initDirState(); // restore open/closed state when search cleared
  });
}
```

- [x] Add a global Escape key handler. Must fire regardless of focus (no
  INPUT/TEXTAREA guard). When pressed: clear the search input, remove
  localStorage key, call `filterTree("")` and `initDirState()` to restore the
  full tree.

```javascript
function initSearchEscape() {
  document.addEventListener("keydown", function (e) {
    if (e.key !== "Escape") return;
    var input = document.getElementById("fileSearch");
    if (!input || !input.value) return; // no-op if already empty
    e.preventDefault();
    input.value = "";
    localStorage.removeItem("file-search-query");
    filterTree("");
    initDirState();
  });
}
```

- [x] Call `initSearchEscape()` from the `DOMContentLoaded` handler, after
  `initSearch()`.

#### 2. Breadcrumb Overlay Container (`main.html`)

**File**: `templates/main.html`

- [x] Add a `<div class="tree-breadcrumb">` element between `.search-container`
  and `.file-list-container`, inside `.sidebar-content`. This container starts
  empty and hidden — JavaScript populates it dynamically.

```html
<div class="sidebar-content">
    <div class="search-container">
        <input ...>
    </div>
    <div class="tree-breadcrumb" aria-label="Current directory path"></div>
    <div class="file-list-container">
        {{ render_tree(tree) }}
    </div>
</div>
```

#### 3. Breadcrumb Overlay Logic (`main.js`)

**File**: `static/main.js`

- [x] Add `initBreadcrumb()` function. Uses `IntersectionObserver` with
  `.file-list-container` as root and `rootMargin: "-1px 0px 0px 0px"` to detect
  when `<summary class="dir-summary">` elements scroll above the visible area.
  > **Post-review fixes:** Added null guard for `entry.rootBounds`; added
  > `expanded` state reset when the ancestor chain changes (prevents stale
  > expansion when scrolling to a different subtree); changed breadcrumb
  > `overflow` from `hidden` to `overflow-x: auto` so long paths are scrollable
  > instead of silently clipped.
  >
  > **Bug fix (stale stuckSummaries):** IntersectionObserver only fires when
  > `<summary>` elements cross the threshold — it doesn't fire again when the
  > rest of a `<details>` scrolls out of view. This left stale entries in
  > `stuckSummaries`, causing the breadcrumb to show a directory path that was
  > no longer visible. Fixed by: (1) pruning summaries in `getAncestorChain()`
  > whose parent `<details>` bottom edge is above the container viewport, and
  > (2) adding a rAF-throttled scroll listener to trigger re-renders so the
  > pruning runs as the user scrolls past directory boundaries.

```javascript
function initBreadcrumb() {
  var container = document.querySelector(".file-list-container");
  var bar = document.querySelector(".tree-breadcrumb");
  if (!container || !bar) return;

  // Track which summaries have scrolled above the viewport
  var stuckSummaries = new Set();
  var expanded = false; // whether the "…" has been expanded

  var observer = new IntersectionObserver(
    function (entries) {
      entries.forEach(function (entry) {
        var summary = entry.target;
        // Only track summaries whose parent <details> is open
        var details = summary.parentElement;
        if (!details || !details.open) {
          stuckSummaries.delete(summary);
          return;
        }
        if (!entry.isIntersecting && entry.boundingClientRect.top < entry.rootBounds.top) {
          stuckSummaries.add(summary);
        } else {
          stuckSummaries.delete(summary);
        }
      });
      renderBreadcrumb();
    },
    { root: container, threshold: 0, rootMargin: "-1px 0px 0px 0px" }
  );

  // Observe all directory summaries
  container.querySelectorAll(".dir-summary").forEach(function (s) {
    observer.observe(s);
  });

  // Re-evaluate on toggle (directory open/close changes what's stuck)
  container.querySelectorAll(".dir-details").forEach(function (d) {
    d.addEventListener("toggle", function () {
      // When a directory is collapsed, remove its summary and all descendant
      // summaries from stuckSummaries
      if (!d.open) {
        stuckSummaries.delete(d.querySelector(":scope > .dir-summary"));
        d.querySelectorAll(".dir-summary").forEach(function (s) {
          stuckSummaries.delete(s);
        });
      }
      renderBreadcrumb();
    });
  });

  function getAncestorChain() {
    // Sort stuck summaries by DOM order (shallowest first)
    var sorted = Array.from(stuckSummaries).sort(function (a, b) {
      return a.compareDocumentPosition(b) & Node.DOCUMENT_POSITION_FOLLOWING
        ? -1
        : 1;
    });

    // Filter to only include ancestors of the current scroll position:
    // A stuck summary is relevant only if it's an ancestor of content
    // currently visible in the container. We keep only summaries where each
    // subsequent one is a descendant of the previous.
    var chain = [];
    for (var i = 0; i < sorted.length; i++) {
      if (chain.length === 0) {
        chain.push(sorted[i]);
      } else {
        var prev = chain[chain.length - 1];
        if (prev.parentElement.contains(sorted[i])) {
          chain.push(sorted[i]);
        }
      }
    }
    return chain;
  }

  function renderBreadcrumb() {
    var chain = getAncestorChain();
    bar.innerHTML = "";

    if (chain.length === 0) {
      bar.style.display = "none";
      return;
    }

    bar.style.display = "flex";

    // Intelligent collapse: if > 2 levels and not expanded, show
    // [root] > … > [immediate]
    var crumbs;
    if (chain.length > 2 && !expanded) {
      crumbs = [
        { summary: chain[0], type: "crumb" },
        { summary: null, type: "ellipsis", hidden: chain.slice(1, -1) },
        { summary: chain[chain.length - 1], type: "crumb" },
      ];
    } else {
      crumbs = chain.map(function (s) {
        return { summary: s, type: "crumb" };
      });
    }

    crumbs.forEach(function (item, i) {
      if (i > 0) {
        var sep = document.createElement("span");
        sep.className = "breadcrumb-sep";
        sep.textContent = "›";
        bar.appendChild(sep);
      }

      if (item.type === "ellipsis") {
        var btn = document.createElement("button");
        btn.className = "breadcrumb-ellipsis";
        btn.textContent = "…";
        btn.title = item.hidden
          .map(function (s) {
            return s.textContent.trim();
          })
          .join(" › ");
        btn.addEventListener("click", function () {
          expanded = true;
          renderBreadcrumb();
        });
        bar.appendChild(btn);
      } else {
        var crumb = document.createElement("button");
        crumb.className = "breadcrumb-item";
        crumb.textContent = item.summary.textContent.trim();
        crumb.title = "Click to collapse " + item.summary.textContent.trim();
        crumb.addEventListener("click", function () {
          // Collapse this directory
          var details = item.summary.parentElement;
          if (details) details.open = false;
          expanded = false;
        });
        bar.appendChild(crumb);
      }
    });
  }

  // Reset expanded state when user scrolls to top
  container.addEventListener(
    "scroll",
    function () {
      if (container.scrollTop === 0) expanded = false;
    },
    { passive: true }
  );
}
```

- [x] Call `initBreadcrumb()` from the `DOMContentLoaded` handler, after
  `initDirState()`.

#### 4. Breadcrumb Overlay Styles (`style.css`)

**File**: `static/style.css`

- [x] Add styles for `.tree-breadcrumb` and its children. Place after the
  `.search-container` styles (after line 395).

```css
/* Directory breadcrumb overlay */
.tree-breadcrumb {
    display: none; /* hidden until JS populates it */
    align-items: center;
    gap: 2px;
    padding: 4px 8px;
    background: var(--panel-bg);
    border-bottom: 1px solid var(--border-color);
    font-size: 11px;
    font-family: var(--font-family-code);
    min-height: 0;
    flex-shrink: 0;
    overflow: hidden;
    white-space: nowrap;
}

.breadcrumb-item,
.breadcrumb-ellipsis {
    background: none;
    border: none;
    color: var(--text-color);
    cursor: pointer;
    padding: 2px 4px;
    border-radius: 4px;
    font-size: 11px;
    font-family: inherit;
    flex-shrink: 0;
}

.breadcrumb-item:hover,
.breadcrumb-ellipsis:hover {
    background: var(--interactive-hover-bg);
}

.breadcrumb-sep {
    color: var(--blockquote-color);
    padding: 0 1px;
    flex-shrink: 0;
}
```

#### 5. Interaction Between Features

No special integration code is needed — the two features operate on the same
DOM independently:

- When search is active and the breadcrumb is visible, both coexist. The
  breadcrumb shows context for the filtered tree — directories containing
  matches are expanded, and the breadcrumb reflects whichever of those
  directories has scrolled past.
- `collapseAll()` and `expandAll()` trigger breadcrumb updates automatically
  because setting `.open` programmatically fires the `toggle` event, which
  `initBreadcrumb()`'s toggle handlers already listen for.

#### Tests

mdview has no frontend test infrastructure (no browser testing framework). The
project uses `cargo test` for Rust-level tests. These changes are client-side
JS/CSS only, so verification is manual.

- [ ] Manual: search for a term, click a result, verify query and filtered view
  persist on the new page
- [ ] Manual: press Escape from anywhere on the page, verify search clears and
  full tree restores
- [ ] Manual: press Escape when no search is active, verify no side effects
- [ ] Manual: use Ctrl+N/P to navigate through filtered results, verify search
  persists across navigations
- [ ] Manual: clear search via the native x button, verify localStorage is
  cleared and full tree restores
- [ ] Manual: scroll the file list past directory headers in a nested structure,
  verify breadcrumb bar appears with correct ancestor chain
- [ ] Manual: scroll past >2 levels of directories, verify breadcrumb shows
  `[root] › … › [immediate]` with ellipsis
- [ ] Manual: click the `…` in the breadcrumb, verify it expands to show all
  intermediate levels
- [ ] Manual: click a breadcrumb crumb, verify the corresponding directory
  collapses and the breadcrumb updates
- [ ] Manual: scroll back to top, verify breadcrumb bar disappears
- [ ] Manual: with both search and breadcrumb active, verify they coexist
  without visual or behavioral conflict
- [ ] Manual: use Ctrl+Shift+C (collapse all), verify breadcrumb clears
- [ ] Manual: verify dark mode and light mode styling of the breadcrumb bar

### Success Criteria

#### Automated Verification

- [x] `cargo build --release` compiles successfully
- [x] `cargo test` passes (no server-side changes, but confirms nothing is
  broken)

#### Manual Verification

- [ ] Search query persists across file navigation via click and Ctrl+N/P
- [ ] Search clears via x button and global Escape, restoring the full tree
- [ ] Breadcrumb bar appears when scrolling past directory headers, showing
  correct ancestor chain
- [ ] Breadcrumb intelligently collapses at >2 depth levels with expandable `…`
- [ ] Clicking a breadcrumb crumb collapses that directory
- [ ] Breadcrumb disappears when scrolled to top
- [ ] Both features work together without conflict
- [ ] Styling is correct in both dark and light themes

## References

- Idea document: `thoughts/ideas/2026-04-12-mdview-file-browser-ux.md`
- VS Code sticky scroll for tree views: https://github.com/microsoft/vscode/issues/167782
- IntersectionObserver sticky detection: https://developer.chrome.com/docs/css-ui/sticky-headers
- CSS `overflow: clip` vs `hidden`: https://www.terluinwebdesign.nl/en/css/position-sticky-not-working-try-overflow-clip-not-overflow-hidden/
