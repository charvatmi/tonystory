# Collapsible Month Buckets Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Each month bucket starts collapsed showing one random thumbnail; clicking the header expands it to show all photos; collapsed thumbnails are greyscale except the most recent month.

**Architecture:** Single HTML file (`index.html`) with vanilla JS and inline CSS. New state fields track which months are expanded and which thumbnail index each month uses. A delegated click handler on the months container handles all expand/collapse toggles. `renderMonthBucket` gains a third `monthIndex` parameter and branches on `state.expandedMonths` to produce either a single-thumbnail collapsed view or the existing full masonry grid.

**Tech Stack:** Vanilla JS, CSS, single `index.html` file. No build step.

---

## File Map

- Modify: `index.html` — all changes are in this single file
  - CSS block (lines ~8–308): add collapsed grid styles, greyscale filter, chevron, cursor
  - `state` object (line ~335): add `expandedMonths` and `monthThumbnails`
  - `renderMonthBucket` (~line 642): add `monthIndex` param, collapsed/expanded branching, chevron
  - New function `toggleMonth`: handles expand/collapse state + DOM re-render
  - New function `initMonthClickDelegation`: single delegated listener on `#months-container`
  - `appendNextMonths` (~line 687): pass `i` as third arg, call `initMonthClickDelegation`
  - `render` (~line 388): call `initMonthClickDelegation` after appending months
  - `signOut` (~line 484): reset `expandedMonths` and `monthThumbnails`

---

### Task 1: Create feature branch

**Files:**
- No file changes — git only

- [ ] **Step 1: Create and switch to the new branch**

```bash
git checkout -b feat/collapsible-months
```

Expected output:
```
Switched to a new branch 'feat/collapsible-months'
```

- [ ] **Step 2: Verify branch**

```bash
git branch --show-current
```

Expected: `feat/collapsible-months`

---

### Task 2: Add CSS for collapsed grid, greyscale, chevron, and clickable header

**Files:**
- Modify: `index.html` — CSS block, inside the `<style>` tag

- [ ] **Step 1: Locate the end of the `/* === Masonry grid === */` section**

Find the block ending around `.photo-item:hover img { filter: brightness(1.08); }`. The new CSS goes right after the existing `.photo-item img.loaded` rule.

- [ ] **Step 2: Add the new CSS rules**

After the line `.photo-item img.loaded { opacity: 1; }`, insert:

```css
    /* === Collapsible months === */
    .month-header {
      cursor: pointer;
      user-select: none;
    }
    .month-chevron {
      color: #8aacbf;
      font-size: 0.75em;
      margin-left: auto;
    }
    .photo-grid.collapsed {
      columns: 1;
      max-width: 200px;
      filter: grayscale(100%);
    }
    .month-bucket.current-month .photo-grid.collapsed {
      filter: none;
    }
```

- [ ] **Step 3: Verify the file still parses — open `index.html` in a browser**

No JS errors in console, page loads to login screen.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "style: collapsed month grid, greyscale filter, chevron, clickable header"
```

---

### Task 3: Add `expandedMonths` and `monthThumbnails` to state

**Files:**
- Modify: `index.html` — `state` object (~line 335)

- [ ] **Step 1: Locate the `state` object**

Find:
```js
    const state = {
      token: null,
      photos: [],
      months: [],
      renderedMonths: 0,
      loading: false,
      error: null,
      config: null  // { dob: Date, milestones: Map<string, string> } or null
    };
```

- [ ] **Step 2: Add the two new fields**

Replace with:
```js
    const state = {
      token: null,
      photos: [],
      months: [],
      renderedMonths: 0,
      loading: false,
      error: null,
      config: null,  // { dob: Date, milestones: Map<string, string> } or null
      expandedMonths: new Set(),   // month labels currently expanded
      monthThumbnails: new Map()   // month label → random thumbnail index
    };
```

- [ ] **Step 3: Verify no console errors on page load**

Open `index.html` in browser, check console — no errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add expandedMonths and monthThumbnails to state"
```

---

### Task 4: Update `renderMonthBucket` to support collapsed and expanded views

**Files:**
- Modify: `index.html` — `renderMonthBucket` function (~line 642)

- [ ] **Step 1: Locate the current `renderMonthBucket` signature**

Find:
```js
    function renderMonthBucket(month, isYearStart) {
```

- [ ] **Step 2: Replace the entire function**

Replace the full `renderMonthBucket` function (from `function renderMonthBucket` through its closing `}`) with:

```js
    function renderMonthBucket(month, isYearStart, monthIndex) {
      const yearMarker = isYearStart ? `<div class="year-marker">${month.year}</div>` : '';

      // Age label below month name
      let ageLabel = '';
      if (state.config && state.config.dob) {
        const age = formatMonthAge(month.year, month.month, state.config.dob);
        if (age) ageLabel = `<span class="month-age">${age}</span>`;
      }

      // Milestone badge
      const milestoneText = state.config && state.config.milestones.has(month.label)
        ? state.config.milestones.get(month.label)
        : null;
      const milestoneBadge = milestoneText
        ? `<span class="milestone-badge">${escapeHtml(milestoneText)}</span>`
        : '';

      // Pick random thumbnail index once per month
      if (!state.monthThumbnails.has(month.label)) {
        state.monthThumbnails.set(month.label, Math.floor(Math.random() * month.photos.length));
      }
      const thumbIdx = state.monthThumbnails.get(month.label);

      const isExpanded = state.expandedMonths.has(month.label);
      const isMostRecent = monthIndex === 0;
      const chevron = isExpanded ? '▾' : '▸';

      let gridHtml;
      if (isExpanded) {
        const items = month.photos.map((photo, idx) => `
          <div class="photo-item" data-photo-id="${photo.id}" data-month-label="${month.label}" data-month-idx="${idx}">
            <img
              class="lazy"
              data-src="${photo.thumbnailUrl}"
              alt="${escapeHtml(photo.name)}"
              loading="lazy"
            />
          </div>`).join('');
        gridHtml = `<div class="photo-grid">${items}</div>`;
      } else {
        const photo = month.photos[thumbIdx];
        gridHtml = `
          <div class="photo-grid collapsed">
            <div class="photo-item" data-photo-id="${photo.id}" data-month-label="${month.label}" data-month-idx="${thumbIdx}">
              <img
                src="${photo.thumbnailUrl || ''}"
                alt="${escapeHtml(photo.name)}"
                class="loaded"
              />
            </div>
          </div>`;
      }

      return `
        <div class="month-bucket${milestoneText ? ' has-milestone' : ''}${isMostRecent ? ' current-month' : ''}" data-month="${month.label}">
          ${yearMarker}
          <div class="month-header">
            <span class="month-label">${escapeHtml(month.czechMonth)}</span>
            ${milestoneBadge}
            <span class="month-count">${pluralCzFotek(month.photos.length)}</span>
            <span class="month-chevron">${chevron}</span>
          </div>
          ${ageLabel}
          ${gridHtml}
        </div>`;
    }
```

- [ ] **Step 3: Update the call site in `appendNextMonths`**

Find this line inside `appendNextMonths`:
```js
        container.insertAdjacentHTML('beforeend', renderMonthBucket(month, isYearStart));
```

Replace with:
```js
        container.insertAdjacentHTML('beforeend', renderMonthBucket(month, isYearStart, i));
```

- [ ] **Step 4: Verify in browser**

After logging in, all months should appear collapsed with a single thumbnail. The most recent month thumbnail should be in colour. All others should be greyscale. Each month header should show `▸`. Console: no errors.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: render month buckets collapsed by default with single thumbnail"
```

---

### Task 5: Add `toggleMonth` and `initMonthClickDelegation`

**Files:**
- Modify: `index.html` — add two new functions after `appendNextMonths`

- [ ] **Step 1: Locate the end of `appendNextMonths`**

Find the closing `}` of `appendNextMonths`. The new functions go immediately after it.

- [ ] **Step 2: Insert the two new functions**

After `appendNextMonths`'s closing `}`, insert:

```js
    function toggleMonth(label) {
      if (state.expandedMonths.has(label)) {
        state.expandedMonths.delete(label);
      } else {
        state.expandedMonths.add(label);
      }
      const bucket = document.querySelector(`.month-bucket[data-month="${label}"]`);
      if (!bucket) return;
      const monthIndex = state.months.findIndex(m => m.label === label);
      const prevYear = monthIndex > 0 ? state.months[monthIndex - 1].year : null;
      const isYearStart = monthIndex === 0 || state.months[monthIndex].year !== prevYear;
      bucket.outerHTML = renderMonthBucket(state.months[monthIndex], isYearStart, monthIndex);
      if (state.expandedMonths.has(label)) {
        attachImageObserver();
        attachPhotoClickHandlers();
      }
    }

    function initMonthClickDelegation() {
      const container = document.getElementById('months-container');
      if (!container || container.dataset.monthClickAttached) return;
      container.dataset.monthClickAttached = 'true';
      container.addEventListener('click', (e) => {
        const header = e.target.closest('.month-header');
        if (!header) return;
        const bucket = header.closest('.month-bucket');
        if (!bucket) return;
        const label = bucket.dataset.month;
        if (label) toggleMonth(label);
      });
    }
```

- [ ] **Step 3: Call `initMonthClickDelegation` from `render`**

Find inside the `render` function this block:
```js
        if (state.months.length > 0) {
          appendNextMonths();
          initSentinelObserver();
        }
```

Replace with:
```js
        if (state.months.length > 0) {
          appendNextMonths();
          initSentinelObserver();
          initMonthClickDelegation();
        }
```

- [ ] **Step 4: Verify expand/collapse in browser**

Click a month header → it should expand, showing all thumbnails and `▾`. Click again → collapses back to single thumbnail and `▸`. Check console: no errors. Lightbox still opens when clicking a photo inside an expanded month.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: toggle month expand/collapse on header click"
```

---

### Task 6: Clean up state on sign-out

**Files:**
- Modify: `index.html` — `signOut` function (~line 484)

- [ ] **Step 1: Locate `signOut`**

Find:
```js
    function signOut() {
      google.accounts.oauth2.revoke(state.token, () => {});
      state.token = null;
      state.photos = [];
      state.months = [];
      state.renderedMonths = 0;
      sessionStorage.removeItem('gd_token');
      render();
    }
```

- [ ] **Step 2: Add the two new state resets**

Replace with:
```js
    function signOut() {
      google.accounts.oauth2.revoke(state.token, () => {});
      state.token = null;
      state.photos = [];
      state.months = [];
      state.renderedMonths = 0;
      state.expandedMonths = new Set();
      state.monthThumbnails = new Map();
      sessionStorage.removeItem('gd_token');
      render();
    }
```

- [ ] **Step 3: Verify**

Sign out and sign back in — all months should start collapsed again with fresh random thumbnails.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "fix: reset expandedMonths and monthThumbnails on sign-out"
```

---

### Task 7: Full manual test pass

- [ ] **Step 1: Load the page and verify initial state**

All months collapsed. Most recent month thumbnail in full colour. All other collapsed thumbnails are greyscale. Each month header shows `▸`.

- [ ] **Step 2: Expand a non-recent month**

Click the month header → `▸` becomes `▾`, full masonry grid appears in full colour (no greyscale). Images load as you scroll (lazy). Clicking a photo opens the lightbox.

- [ ] **Step 3: Collapse the same month**

Click the header again → grid disappears, single greyscale thumbnail returns, `▾` becomes `▸`.

- [ ] **Step 4: Expand the most recent month**

Click the most recent month header → expands normally, full colour.

- [ ] **Step 5: Scroll to trigger sentinel (if > 3 months of photos)**

New months append as collapsed. No JS errors in console.

- [ ] **Step 6: Click the collapsed thumbnail directly**

Clicking the single thumbnail photo in a collapsed month should open the lightbox for that photo (not toggle the month). Verify this works.

- [ ] **Step 7: Sign out and sign back in**

All months start collapsed again. Random thumbnails may differ from the previous session (re-randomised on sign-in).

- [ ] **Step 8: Final commit if any tweaks were made during testing**

```bash
git add index.html
git commit -m "fix: <describe any tweaks>"
```

---

### Task 8: Notify user the branch is ready for local testing

- [ ] **Step 1: Confirm the branch is `feat/collapsible-months` and all commits are in place**

```bash
git log --oneline feat/collapsible-months
```

Expected output (newest first):
```
<hash> fix: reset expandedMonths and monthThumbnails on sign-out
<hash> feat: toggle month expand/collapse on header click
<hash> feat: render month buckets collapsed by default with single thumbnail
<hash> feat: add expandedMonths and monthThumbnails to state
<hash> style: collapsed month grid, greyscale filter, chevron, clickable header
<hash> docs: collapsible month buckets design spec
...
```

- [ ] **Step 2: Tell the user the branch is ready**

All changes are on `feat/collapsible-months`. Open `index.html` locally to test. No push until the user approves.
