# TonyStory — Collapsible Month Buckets Design Spec

**Date:** 2026-04-03
**Status:** Approved

---

## Overview

Each month bucket on the timeline starts collapsed, showing only a single random thumbnail. Clicking the month header expands it to reveal all thumbnails (the full masonry grid, exactly as today). Images for a month are loaded only after that month is expanded. Collapsed thumbnails appear greyscale except for the most recent month, which stays in full color.

---

## State

Two new entries added to `state`:

```js
state.expandedMonths = new Set()         // month labels (e.g. "April 2026") that are expanded
state.monthThumbnails = new Map()        // month label → index of the random thumbnail chosen for that month
```

`expandedMonths` starts empty (all months collapsed). `monthThumbnails` is populated lazily the first time each month bucket is rendered — one `Math.floor(Math.random() * month.photos.length)` call per month, never re-randomised on re-render.

---

## Rendering

### Collapsed state

The month bucket renders:
- The existing `month-header` (click target, full width)
- A single `.photo-grid.collapsed` containing one `.photo-item` — the thumbnail at the stored random index
- The thumbnail image loads immediately (it's already a 200px Drive thumbnail, negligible cost)
- CSS: `filter: grayscale(100%)` on `.photo-grid.collapsed`, **except** for month index `0` (the most recent month), which gets no filter

### Expanded state

The month bucket re-renders with:
- The existing `month-header` (click target)
- Full `.photo-grid` with all photos — identical to the current implementation
- Images carry `data-src` and are observed by the existing `IntersectionObserver` (lazy loading unchanged)
- No greyscale filter

### Chevron indicator

The month label gets a small inline chevron: `▸` when collapsed, `▾` when expanded. Styled in muted colour (`#8aacbf`), `font-size: 0.75em`, `margin-left: 6px`.

---

## Interaction

Clicking anywhere on `.month-header` toggles the month:

```
if expandedMonths.has(label):
    expandedMonths.delete(label)
    re-render that bucket (collapsed)
else:
    expandedMonths.add(label)
    re-render that bucket (expanded)
    call attachImageObserver() to register new lazy images
    call attachPhotoClickHandlers() to wire up lightbox clicks
```

Re-render is done by replacing the existing `.month-bucket[data-month="<label>"]` element's `outerHTML` — same pattern used in `appendNextMonths`.

---

## CSS Changes

```css
/* Collapsed grid — single thumbnail, constrained width */
.photo-grid.collapsed {
  columns: 1;
  max-width: 200px;
}

/* Greyscale overlay for collapsed thumbnails */
.photo-grid.collapsed {
  filter: grayscale(100%);
  transition: filter 0.2s;
}

/* Most-recent month: no greyscale (applied via .month-bucket.current-month) */
.month-bucket.current-month .photo-grid.collapsed {
  filter: none;
}

/* Clickable header */
.month-header {
  cursor: pointer;
  user-select: none;
}

/* Expand/collapse animation */
.photo-grid:not(.collapsed) {
  animation: none; /* grid is just inserted; browser paints normally */
}
```

---

## Lazy Loading Behaviour

- **Collapsed:** The single thumbnail image is rendered with a real `src` (not `data-src`) since it loads immediately and is small. No observer needed.
- **Expanded:** All photos in the expanded grid are rendered with `data-src` (lazy), same as today. `attachImageObserver()` is called after the expanded HTML is inserted.

This means images are **never fetched** for a month until it is expanded — satisfying the requirement exactly.

---

## Branch

Work is done on a new branch `feat/collapsible-months` branched from `feat/age-milestones` (current branch). User tests locally before any push.

---

## Out of Scope

- Remembering expand/collapse state across page reloads (always starts fully collapsed)
- "Expand all" / "Collapse all" controls
- Keyboard accessibility for the toggle (future enhancement)
- Any change to the lightbox behaviour
