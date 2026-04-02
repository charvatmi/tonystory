# Age, Counter & Milestones Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add live age counter in the header, per-month age labels on the timeline, and milestone badges — all driven by a `tonystory-config.json` file fetched from the user's private Google Drive folder. All visible UI text in Czech.

**Architecture:** Single `index.html` file. `loadConfig()` runs before `loadPhotos()` and populates `state.config`. All three features read from `state.config` and silently skip if it is `null`. `groupByMonth()` is extended to store `czechMonth`, `year`, and `month` fields on each month object so rendering functions don't need to parse the label string. New CSS classes handle milestone dot styling and badges.

**Tech Stack:** Vanilla JS, Google Drive API v3, inline CSS.

---

### Task 1: Create branch + add `state.config` + `loadConfig()`

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Create the feature branch**

```bash
git checkout main && git pull && git checkout -b feat/age-milestones
```

- [ ] **Step 2: Add `config: null` to the `state` object**

Find this block (around line 244):
```js
    const state = {
      token: null,
      photos: [],
      months: [],
      renderedMonths: 0,
      loading: false,
      error: null
    };
```

Replace with:
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

- [ ] **Step 3: Add the `loadConfig()` function**

Insert this entire block after the `handleAuthError()` function and before the `// === Timeline ===` comment:

```js
    // === Config ===

    async function loadConfig() {
      try {
        const params = new URLSearchParams({
          q: `name = 'tonystory-config.json' and '${CONFIG.FOLDER_ID}' in parents and trashed=false`,
          fields: 'files(id)',
          pageSize: '1'
        });
        const res = await fetch(`https://www.googleapis.com/drive/v3/files?${params}`, {
          headers: { Authorization: `Bearer ${state.token}` }
        });
        if (!res.ok) return;
        const data = await res.json();
        if (!data.files || data.files.length === 0) return;
        const fileId = data.files[0].id;
        const fileRes = await fetch(`https://www.googleapis.com/drive/v3/files/${fileId}?alt=media`, {
          headers: { Authorization: `Bearer ${state.token}` }
        });
        if (!fileRes.ok) return;
        const json = await fileRes.json();
        state.config = {
          dob: new Date(json.dob),
          milestones: new Map((json.milestones || []).map(m => [m.month, m.label]))
        };
      } catch {
        // config is optional — silently skip on any error
      }
    }
```

- [ ] **Step 4: Wire `loadConfig()` before `loadPhotos()` in the init block**

Find (near the end of the file, inside the `waitForGSI` interval):
```js
          if (state.token) loadPhotos();
```

Replace with:
```js
          if (state.token) loadConfig().then(() => loadPhotos());
```

Also find the auth callback in `initAuth()`:
```js
      state.token = response.access_token;
      state.error = null;
      sessionStorage.setItem('gd_token', response.access_token);
      render();
      loadPhotos();
```

Replace with:
```js
      state.token = response.access_token;
      state.error = null;
      sessionStorage.setItem('gd_token', response.access_token);
      render();
      loadConfig().then(() => loadPhotos());
```

- [ ] **Step 5: Open in browser and verify**

Sign in → open DevTools console → after photos load, run `state.config`. Expected: `{ dob: Date object for 2025-10-29, milestones: Map with entries }`. If `tonystory-config.json` is not in the folder or not yet uploaded, expected: `null`.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add loadConfig() to fetch tonystory-config.json from Drive"
```

---

### Task 2: Extend `groupByMonth()` with `czechMonth`, `year`, `month` fields

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Replace the `groupByMonth()` function**

Find:
```js
    function groupByMonth(photos) {
      const map = new Map();
      for (const photo of photos) {
        const label = photo.date.toLocaleString('default', { month: 'long', year: 'numeric' });
        if (!map.has(label)) map.set(label, []);
        map.get(label).push(photo);
      }
      return Array.from(map.entries()).map(([label, photos]) => ({ label, photos }));
    }
```

Replace with:
```js
    function groupByMonth(photos) {
      const map = new Map();
      for (const photo of photos) {
        const label = photo.date.toLocaleString('default', { month: 'long', year: 'numeric' });
        if (!map.has(label)) {
          map.set(label, {
            photos: [],
            czechMonth: photo.date.toLocaleString('cs-CZ', { month: 'long' }),
            year: photo.date.getFullYear(),
            month: photo.date.getMonth()
          });
        }
        map.get(label).photos.push(photo);
      }
      return Array.from(map.entries()).map(([label, data]) => ({
        label,
        czechMonth: data.czechMonth,
        year: data.year,
        month: data.month,
        photos: data.photos
      }));
    }
```

- [ ] **Step 2: Update `renderMonthBucket()` to use the new fields**

Find inside `renderMonthBucket()`:
```js
      const year = month.label.split(' ').pop();
      const monthName = month.label.split(' ').slice(0, -1).join(' ');
      const yearMarker = isYearStart ? `<div class="year-marker">${year}</div>` : '';
```

Replace with:
```js
      const yearMarker = isYearStart ? `<div class="year-marker">${month.year}</div>` : '';
```

Then find:
```js
            <span class="month-label">${monthName}</span>
```

Replace with:
```js
            <span class="month-label">${month.czechMonth}</span>
```

- [ ] **Step 3: Update `appendNextMonths()` to use `month.year`**

Find:
```js
        const prevYear = i > 0 ? state.months[i - 1].label.split(' ').pop() : null;
        const isYearStart = i === 0 || month.label.split(' ').pop() !== prevYear;
```

Replace with:
```js
        const prevYear = i > 0 ? state.months[i - 1].year : null;
        const isYearStart = i === 0 || month.year !== prevYear;
```

- [ ] **Step 4: Open in browser and verify**

After sign-in: month labels appear in Czech (e.g. "říjen" instead of "October"). Year markers still appear correctly. Lightbox still opens and navigates (it uses `month.label` internally which is unchanged English key).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: extend groupByMonth with czechMonth, year, month fields"
```

---

### Task 3: Czech UI strings

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Replace all visible English strings in JS**

**Login screen subtitle** — find:
```js
            <p>A little boy growing up, one month at a time</p>
```
Replace with:
```js
            <p>Jak Tony roste, měsíc po měsíci</p>
```

**Sign in button text** — find:
```js
              Sign in with Google
```
Replace with:
```js
              Přihlásit se přes Google
```

**Loading messages** — find:
```js
          <p id="loading-msg">Načítám fotky…</p>
```
(Already Czech if previously set — if still English `Fetching photos…` replace with `Načítám fotky…`)

Find inside `loadPhotos()`:
```js
          if (msg) msg.textContent = `Fetching photos… (${fetched} found)`;
```
Replace with:
```js
          if (msg) msg.textContent = `Načítám fotky… (nalezeno ${fetched})`;
```

**Sign out button** — find:
```js
          <button id="btn-signout">Sign out</button>
```
Replace with:
```js
          <button id="btn-signout">Odhlásit se</button>
```

**Error banner retry link** — find:
```js
          <a id="btn-retry-signout">Sign out and retry</a>
```
Replace with:
```js
          <a id="btn-retry-signout">Odhlásit se a zkusit znovu</a>
```

**Empty state** — find:
```js
        return `<div id="empty-state">
          <h2>No photos found in this folder</h2>
          <p>Make sure the folder ID in CONFIG is correct and contains images.</p>
        </div>`;
```
Replace with:
```js
        return `<div id="empty-state">
          <h2>V této složce nejsou žádné fotky</h2>
          <p>Zkontroluj FOLDER_ID v konfiguraci.</p>
        </div>`;
```

**Auth error message** — find:
```js
      state.error = 'Session expired. Please sign in again.';
```
Replace with:
```js
      state.error = 'Relace vypršela. Přihlas se znovu.';
```

**Lightbox date locale** — find:
```js
        photo.date.toLocaleString('default', { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' });
```
Replace with:
```js
        photo.date.toLocaleString('cs-CZ', { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' });
```

- [ ] **Step 2: Open in browser and verify**

All visible text is in Czech. Lightbox date shows Czech weekday/month names.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: Czech UI strings and cs-CZ locale"
```

---

### Task 4: Helper functions + Feature 6 — live age counter in header

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add `pluralCz()`, `pluralCzFotek()`, and `formatAge()` helper functions**

Insert this block immediately before the `// === Render ===` comment:

```js
    // === Czech helpers ===

    function pluralCz(n, one, few, many) {
      if (n === 1) return `${n} ${one}`;
      if (n >= 2 && n <= 4) return `${n} ${few}`;
      return `${n} ${many}`;
    }

    function pluralCzFotek(n) {
      return pluralCz(n, 'fotka', 'fotky', 'fotek');
    }

    function formatAge(dob, today) {
      const totalDays = Math.floor((today - dob) / 86400000);
      if (totalDays < 0) return null;

      let years = today.getFullYear() - dob.getFullYear();
      let months = today.getMonth() - dob.getMonth();
      if (today.getDate() < dob.getDate()) months--;
      if (months < 0) { years--; months += 12; }

      if (today.getDate() === dob.getDate() && today.getMonth() === dob.getMonth() && years > 0) {
        return `Tonymu je dnes ${pluralCz(years, 'rok', 'roky', 'let')}!`;
      }
      if (totalDays < 30) return 'Tonymu je méně než měsíc';
      if (years === 0) return `Tonymu je ${pluralCz(months, 'měsíc', 'měsíce', 'měsíců')}`;
      if (months === 0) return `Tonymu je ${pluralCz(years, 'rok', 'roky', 'let')}`;
      return `Tonymu je ${pluralCz(years, 'rok', 'roky', 'let')} a ${pluralCz(months, 'měsíc', 'měsíce', 'měsíců')}`;
    }
```

- [ ] **Step 2: Update `buildHeader()` to use age counter and Czech photo count**

Find the `buildHeader()` function:
```js
    function buildHeader() {
      const total = state.photos.length;
      let meta = '';
      if (total > 0) {
        const oldest = state.photos[state.photos.length - 1].date;
        const newest = state.photos[0].date;
        const fmt = d => d.toLocaleString('default', { month: 'short', year: 'numeric' });
        meta = `${total} photos · ${fmt(oldest)} – ${fmt(newest)}`;
      }
```

Replace with:
```js
    function buildHeader() {
      const total = state.photos.length;
      let meta = '';
      if (state.config && state.config.dob) {
        const ageStr = formatAge(state.config.dob, new Date());
        if (ageStr) {
          meta = total > 0 ? `${ageStr} · ${pluralCzFotek(total)}` : ageStr;
        }
      } else if (total > 0) {
        const oldest = state.photos[state.photos.length - 1].date;
        const newest = state.photos[0].date;
        const fmt = d => d.toLocaleString('cs-CZ', { month: 'short', year: 'numeric' });
        meta = `${pluralCzFotek(total)} · ${fmt(oldest)} – ${fmt(newest)}`;
      }
```

- [ ] **Step 3: Update photo count in `renderMonthBucket()`**

Find:
```js
            <span class="month-count">${month.photos.length} photo${month.photos.length !== 1 ? 's' : ''}</span>
```
Replace with:
```js
            <span class="month-count">${pluralCzFotek(month.photos.length)}</span>
```

- [ ] **Step 4: Open in browser and verify**

Header center shows e.g. "Tonymu je 5 měsíců · 42 fotek". If config not loaded, falls back to Czech date range. Photo count per month shows "1 fotka" / "3 fotky" / "12 fotek" correctly.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: live age counter in header with Czech pluralization"
```

---

### Task 5: Feature 1 (month age) + Feature 5 (milestone badges)

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add `formatMonthAge()` helper**

Add this function immediately after `formatAge()`:

```js
    function formatMonthAge(bucketYear, bucketMonth, dob) {
      const ageMonths = (bucketYear - dob.getFullYear()) * 12 + (bucketMonth - dob.getMonth());
      if (ageMonths < 0) return null;
      if (ageMonths === 0) return 'novorozenec';
      const years = Math.floor(ageMonths / 12);
      const months = ageMonths % 12;
      if (years === 0) return pluralCz(months, 'měsíc', 'měsíce', 'měsíců');
      if (months === 0) return pluralCz(years, 'rok', 'roky', 'let');
      return `${pluralCz(years, 'rok', 'roky', 'let')} a ${pluralCz(months, 'měsíc', 'měsíce', 'měsíců')}`;
    }
```

- [ ] **Step 2: Add CSS for age label, milestone badge, and filled milestone dot**

Add this block inside the `<style>` tag, after the `.month-count` rule:

```css
    .month-age {
      display: block;
      font-size: 0.75rem;
      color: #8aacbf;
      margin-top: -6px;
      margin-bottom: 10px;
    }
    .milestone-badge {
      font-size: 0.72rem;
      background: #dbeeff;
      color: #1a4a7a;
      padding: 2px 8px;
      border-radius: 10px;
      font-weight: 500;
    }
    .month-bucket.has-milestone::before {
      background: #3a7abd;
      border-color: #3a7abd;
    }
```

- [ ] **Step 3: Replace the full `renderMonthBucket()` function**

Replace the entire `renderMonthBucket()` function with:

```js
    function renderMonthBucket(month, isYearStart) {
      const yearMarker = isYearStart ? `<div class="year-marker">${month.year}</div>` : '';

      // Feature 1: age below month label
      let ageLabel = '';
      if (state.config && state.config.dob) {
        const age = formatMonthAge(month.year, month.month, state.config.dob);
        if (age) ageLabel = `<span class="month-age">${age}</span>`;
      }

      // Feature 5: milestone badge
      const milestoneText = state.config && state.config.milestones.has(month.label)
        ? state.config.milestones.get(month.label)
        : null;
      const milestoneBadge = milestoneText
        ? `<span class="milestone-badge">${escapeHtml(milestoneText)}</span>`
        : '';

      const items = month.photos.map((photo, idx) => `
        <div class="photo-item" data-photo-id="${photo.id}" data-month-label="${month.label}" data-month-idx="${idx}">
          <img
            class="lazy"
            data-src="${photo.thumbnailUrl}"
            alt="${escapeHtml(photo.name)}"
            loading="lazy"
          />
        </div>`).join('');

      return `
        <div class="month-bucket${milestoneText ? ' has-milestone' : ''}" data-month="${month.label}">
          ${yearMarker}
          <div class="month-header">
            <span class="month-label">${month.czechMonth}</span>
            ${milestoneBadge}
            <span class="month-count">${pluralCzFotek(month.photos.length)}</span>
          </div>
          ${ageLabel}
          <div class="photo-grid">${items}</div>
        </div>`;
    }
```

- [ ] **Step 4: Open in browser and verify**

- Each month shows age below the label: "novorozenec", "3 měsíce", "1 rok a 2 měsíce", etc. Months before Tony's birth show no age label.
- Months with milestones show a blue pill badge ("Narozen", "První úsměv", etc.) next to the month name.
- Timeline dot for milestone months is solid blue instead of an empty ring.
- Months without config loaded show no age or badges (test by temporarily removing the config file from Drive).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: per-month age label and milestone badges on timeline"
```

---

### Task 6: noindex meta tag + robots.txt

**Files:**
- Modify: `index.html`
- Create: `robots.txt`

- [ ] **Step 1: Add noindex meta tag to `index.html`**

Find:
```html
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
```

Add this line immediately after it:
```html
  <meta name="robots" content="noindex, nofollow" />
```

- [ ] **Step 2: Create `robots.txt`**

Create the file `/home/misakch/tonystory/robots.txt` with this exact content:

```
User-agent: *
Disallow: /
```

- [ ] **Step 3: Commit**

```bash
git add index.html robots.txt
git commit -m "chore: prevent search engine indexing"
```

---

### Task 7: Push branch and open PR

- [ ] **Step 1: Push the branch**

```bash
git push -u origin feat/age-milestones
```

- [ ] **Step 2: Create the PR**

```bash
gh pr create --title "feat: age counter, month age labels, milestone badges, noindex" --body "$(cat <<'EOF'
## Summary
- Fetches `tonystory-config.json` from the Drive folder (DOB + milestones) — no personal data in the repo
- Header shows live Czech age: "Tonymu je 5 měsíců · 42 fotek"
- Each month bucket shows Tony's age at that month below the label
- Milestone months get a filled blue dot on the rail and a badge pill next to the month name
- All visible UI text translated to Czech
- `robots.txt` and noindex meta tag prevent search engine indexing

## Test Plan
- [ ] Sign in → header shows age in Czech
- [ ] Scroll timeline → each month shows correct age label
- [ ] Months with milestones in config show blue dot + badge
- [ ] Months before October 2025 show no age label
- [ ] Remove `tonystory-config.json` from Drive → all three features silently absent, app works normally
- [ ] `robots.txt` accessible at `/robots.txt` on the deployed site

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

---

## Completion Checklist

- [ ] `loadConfig()` fetches `tonystory-config.json` from Drive, silently skips on failure
- [ ] `state.config` is `null` when config unavailable, features all gracefully absent
- [ ] `groupByMonth()` returns `{ label, czechMonth, year, month, photos }` per bucket
- [ ] Czech pluralization: rok/roky/let, měsíc/měsíce/měsíců, fotka/fotky/fotek
- [ ] Header: age counter with photo count
- [ ] Month buckets: age label below month name
- [ ] Milestone months: filled blue dot + badge
- [ ] All visible UI text in Czech
- [ ] `robots.txt` created with `Disallow: /`
- [ ] `<meta name="robots" content="noindex, nofollow">` in `<head>`
- [ ] PR open against `main`
