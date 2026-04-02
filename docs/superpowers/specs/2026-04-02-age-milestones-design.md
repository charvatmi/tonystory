# TonyStory — Age, Counter & Milestones Design Spec

**Date:** 2026-04-02
**Status:** Approved

---

## Overview

Three features added to TonyStory:

1. **Live age counter** in the sticky header
2. **Age label** on each month bucket
3. **Milestone badges** on the timeline rail

All visible text on the page is in Czech. Config, code, and comments stay in English. Feature data (DOB, milestones) is loaded from a JSON file in the user's private Google Drive folder — nothing personal is stored in the repo.

---

## Config File

A file named `tonystory-config.json` lives in the same Drive folder as the photos. The app searches for it by name after authenticating. If not found, all three features silently skip — the rest of the app works normally.

```json
{
  "dob": "2025-10-29",
  "milestones": [
    { "month": "October 2025", "label": "Narozen" },
    { "month": "November 2025", "label": "První úsměv" }
  ]
}
```

`month` values must match the month labels produced by the Drive API date grouping (English month name + year, e.g. `"October 2025"`). Labels are freeform — written by the user in whatever language they prefer.

---

## Data Flow

After OAuth succeeds and before `loadPhotos()` runs, a new `loadConfig()` function:

1. Calls Drive API `files.list` with `q: name = 'tonystory-config.json' and '<folderId>' in parents and trashed=false`
2. If a file is found, fetches its content with `?alt=media` and the auth token
3. Parses JSON, stores in `state.config`:

```js
state.config = {
  dob: new Date('2025-10-29'),
  milestones: new Map([
    ['October 2025', 'Narozen'],
    ['November 2025', 'První úsměv']
  ])
}
```

4. If anything fails (file not found, parse error, network error): `state.config = null`, silently continue

`loadConfig()` and `loadPhotos()` run sequentially: config first, then photos. Config is fast (one small file) so this adds negligible delay.

---

## Feature 6 — Live Age Counter (Header)

Replaces the center header meta (`N photos · date range`) with:

> Tonymu je 1 rok a 5 měsíců · 142 fotek

Calculated once on page load from `state.config.dob` to today's date.

### Age formatting (Czech)

| Condition | Format |
|---|---|
| config not loaded | show original date range meta |
| same day as birthday | `"Tonymu je dnes X rok/roky/let!"` (same pluralization) |
| under 1 month | `"Tonymu je méně než měsíc"` |
| under 12 months | `"Tonymu je X měsíc/měsíce/měsíců"` |
| exactly 12+ months, 0 extra months | `"Tonymu je X rok/roky/let"` |
| 12+ months with remainder | `"Tonymu je X rok/roky/let a Y měsíc/měsíce/měsíců"` |

### Czech pluralization rules

**Roky (years):**
- 1 → rok
- 2–4 → roky
- 5+ → let

**Měsíce (months):**
- 1 → měsíc
- 2–4 → měsíce
- 5+ → měsíců

Photo count: `1 fotka`, `2–4 fotky`, `5+ fotek`

---

## Feature 1 — Age on Month Bucket

Displayed as a small muted line directly below the month label in each month header.

| Condition | Text shown |
|---|---|
| config not loaded | nothing |
| month is before birth month | nothing |
| month is birth month | `novorozenec` |
| 1–11 months after birth | `X měsíc/měsíce/měsíců` (same pluralization) |
| exactly N years | `X rok/roky/let` |
| N years + M months | `X rok/roky/let a Y měsíc/měsíce/měsíců` |

Age is calculated as of the **first day of that month** relative to DOB.

Month labels are in Czech via `cs-CZ` locale: `toLocaleString('cs-CZ', { month: 'long', year: 'numeric' })`.

---

## Feature 5 — Milestone Badges

When `state.config.milestones` has an entry for a given month:

- The timeline dot (`.month-bucket::before`) changes from an empty ring to a **filled blue circle** (`background: #3a7abd`)
- A small inline badge appears after the month label: the label text from config, styled as a pill (`background: #dbeeff`, `color: #1a4a7a`, small font, rounded)

Milestone matching: compare `month.label` (the internal English grouping key, e.g. `"October 2025"`) against the milestone map keys. Labels are displayed as-is from the config.

---

## Czech UI Strings

All static visible strings translated to Czech:

| Current (English) | Czech |
|---|---|
| "Sign in with Google" | "Přihlásit se přes Google" |
| "A little boy growing up, one month at a time" | "Jak Tony roste, měsíc po měsíci" |
| "Fetching photos…" | "Načítám fotky…" |
| "Fetching photos… (N found)" | "Načítám fotky… (nalezeno N)" |
| "Sign out" | "Odhlásit se" |
| "Sign out and retry" | "Odhlásit se a zkusit znovu" |
| "No photos found in this folder" | "V této složce nejsou žádné fotky" |
| "Make sure the folder ID…" | "Zkontroluj FOLDER_ID v konfiguraci." |
| photo count suffix "photo/photos" | "fotka / fotky / fotek" |

Month names rendered via `cs-CZ` locale automatically.

---

## Error Handling

- Config file not found: `state.config = null`, all three features skip gracefully
- Config JSON malformed: same fallback
- Config fetch 401: same fallback (don't re-trigger full auth error for a secondary file)
- All other photo/auth errors unchanged

---

## Out of Scope

- Editing milestones or DOB from within the UI
- Milestone icons or colours beyond the filled dot
- Any server-side logic
