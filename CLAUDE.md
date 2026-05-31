# CLAUDE.md — Daily Dilemmas Label Verification Task

## Project Overview

A single-file web annotation app (`dilemmas_annotation.html`) built for Prolific. Participants verify the quality of moral foundation labels assigned to everyday ethical dilemmas. The app collects ratings and submits them to a Google Sheet backend.

The theoretical framework is **Moral Foundations Theory (MFQ-2)** — Atari et al. (2023), *Journal of Personality and Social Psychology*. The six foundations are: Care, Equality, Proportionality, Loyalty, Authority, Purity.

---

## File Structure

Everything lives in a **single HTML file** — no build pipeline, no framework, no dependencies. It can be deployed by dropping it on Vercel, Netlify, or GitHub Pages.

```
dilemmas_annotation.html
├── <style>          — All CSS (CSS variables, layout, components)
├── Screens 1–5      — HTML for each screen (hidden/shown via JS)
└── <script>
    ├── Constants        — PROLIFIC_CODE, PROLIFIC_COMPLETION_URL, SHEET_URL
    ├── FOUNDATION_META  — Labels, definitions and examples for the 6 foundations
    ├── RAW              — The 22 dilemma objects (hardcoded data)
    ├── State            — currentIdx, responses {}
    └── Functions        — navigation, rendering, saving, submitting
```

---

## Screen Flow

```
Screen 1: Welcome
    ↓ (Next button)
Screen 2: Consent
    ↓ (checkbox must be ticked before Next unlocks)
Screen 3: Instructions
    ↓ (Next button)
Screen 4: Annotation (×22 dilemmas, shuffled)
    ↓ (Submit button — only enabled when all ratings are filled and open pill panels have a selection)
Screen 5: Completion (shows Prolific code, redirects to Prolific)
```

Navigation is handled by `goTo(screenId)` which shows/hides `.screen` divs.

---

## Annotation Screen Logic

Each dilemma has:
- A situation text shown at the top
- Two columns: **Action A** (internally `do`) and **Action B** (internally `notdo`)
- Each column lists the moral foundations assigned to that action in the source data
- Foundations are ordered: shared foundations first, differentiating foundation last (marked with `*` prefix and a dashed separator)
- A **"+ Add foundation"** button at the bottom of each column (always visible)
- A **"?" button** in the top-right corner of the card that opens the cheatsheet modal

### Per-foundation row:
1. Foundation name with a hover tooltip showing definition + example
2. A 3-point dot slider: **Disagree · Neutral · Agree** (colour-coded red/amber/green)

### "+ Add foundation" pills:
- One button per column (Action A / Action B), always visible
- Clicking opens a panel of 6 foundation pills that toggle on/off
- Multiple foundations can be selected
- Each pill has a hover tooltip with the foundation definition and example
- Button label changes to "− Add foundation" when open
- If panel is open, at least one pill must be selected before Next enables

### Cheatsheet modal ("?"):
- Fixed to the top-right of the annotation card
- Opens a centred modal with a semi-transparent backdrop (fade-in animation)
- Two tabs: **Instructions** (rating scale explainer) and **Foundations** (all 6 definitions)
- Closes by clicking "?" again or clicking the backdrop

### Next/Submit button gating (`checkComplete()`):
- Disabled on dilemma load
- Enabled only when:
  1. Every foundation row has a rating selected
  2. Every open pill panel has at least one pill selected
- Called on every slider change, pill toggle, and pill panel open/close
- On the last dilemma, button reads "Submit ›"
- On Submit: button immediately disabled and shows "Submitting…" to prevent double-submission

---

## Data Model

### Dilemma object (from `RAW` array):
```javascript
{
  id: 33532,
  situation: "...",
  do_action: "Accept the Promotion",
  do_labels: "{'loyalty_betrayal'}",          // raw string from Excel
  notdo_action: "Reject the Promotion",
  notdo_labels: "{'authority_subversion', 'loyalty_betrayal'}",
  differentiator: "{'authority_subversion'}",
  // parsed versions added at runtime:
  do_parsed: ["loyalty_betrayal"],
  notdo_parsed: ["authority_subversion", "loyalty_betrayal"],
  diff_parsed: ["authority_subversion"],
}
```

`parseSet(s)` converts the raw string format `"{'care_harm', 'loyalty_betrayal'}"` into a JS array, filtering out any keys not in `FOUNDATION_META`.

Some dilemmas have `set()` for one side — these render "No foundation labels assigned" but still show the "+ Add foundation" button.

### Response state:
```javascript
responses = {
  _pillState: {
    "col-pills-do-33532": ["care_harm", "equality"],  // selected pills per column
  },
  [dilemma_id]: {
    do: {
      [foundationKey]: { rating: "agree"|"neutral"|"disagree" },
      _pills: "care_harm,equality",   // comma-separated selected pill keys, or null
    },
    notdo: { ... }
  }
}
```

Saved by `saveCurrentResponse()` on every Next/Back navigation.
Restored by `restoreResponse()` + `restoreColumnPills()` when navigating back.

---

## Foundation Keys

Internal keys used in the data and `FOUNDATION_META`:

| Key | Display Label |
|---|---|
| `care_harm` | Care |
| `equality` | Equality |
| `proportionality` | Proportionality |
| `loyalty_betrayal` | Loyalty |
| `authority_subversion` | Authority |
| `purity_degradation` | Purity |

---

## Google Sheets Integration

**Webhook URL:**
```
https://script.google.com/macros/s/AKfycbx0a-aKW-xIMHBMzbHGOMC5lPX3JKqkyNNIeYqAXL1fh0il4KqmNeEPU1K9JoIVe4IH8w/exec
```

On submit, `submitAll()` flattens `responses` into one row per foundation and POSTs to the webhook.

### Row types:

**Pre-labeled foundation ratings:**
```javascript
{
  prolific_pid: "abc123",
  dilemma_id:   "33532",
  side:         "do",         // "do" or "notdo"
  foundation:   "care_harm",
  rating:       "agree",      // "agree", "neutral", or "disagree"
}
```

**Annotator-added foundations:**
```javascript
{
  prolific_pid: "abc123",
  dilemma_id:   "33532",
  side:         "do",
  foundation:   "equality",
  rating:       "added",      // always "added" for pill selections
}
```

Uses `mode: "no-cors"` (required for Google Apps Script — no response body returned but data writes correctly).

**Sheet columns:** `prolific_pid | dilemma_id | side | foundation | rating | timestamp`

### Apps Script (`doPost`):
```javascript
function doPost(e) {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const data = JSON.parse(e.postData.contents);
  data.rows.forEach(row => {
    sheet.appendRow([
      row.prolific_pid,
      row.dilemma_id,
      row.side,
      row.foundation,
      row.rating,
      new Date().toISOString()
    ]);
  });
  return ContentService
    .createTextOutput(JSON.stringify({ status: "ok" }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

---

## Prolific Integration

Participant ID is read from the URL on load:
```javascript
const PROLIFIC_PID = new URLSearchParams(window.location.search).get("PROLIFIC_PID") || "UNKNOWN";
```

Study URL format to paste into Prolific:
```
https://your-site.vercel.app?PROLIFIC_PID={{%PROLIFIC_PID%}}
```

**To update before going live:**
```javascript
const PROLIFIC_CODE = "C1B2-A3D4";  // ← replace with real Prolific completion code
```
The completion URL is auto-constructed from this code.

---

## Key Constants to Update Before Deployment

| Constant | Location | What to change |
|---|---|---|
| `PROLIFIC_CODE` | Top of `<script>` | Your actual Prolific study completion code |
| `SHEET_URL` | Inside `submitAll()` | Already set — only change if you redeploy the Apps Script |

---

## Key Functions Reference

| Function | Purpose |
|---|---|
| `goTo(screenId)` | Navigate between screens |
| `startTask()` | Shuffle dilemmas and show screen 4 |
| `renderDilemma()` | Render current dilemma, update progress, restore state |
| `renderFoundations(containerId, foundations, differentiators, d, side)` | Build foundation rows + pill button for one column |
| `saveCurrentResponse()` | Save all ratings and pill state for current dilemma |
| `restoreResponse(d, saved)` | Restore slider positions from saved state |
| `restoreColumnPills(d)` | Restore pill selections and panel open state |
| `checkComplete()` | Enable/disable Next based on completeness |
| `togglePills(colId)` | Open/close pill panel for a column |
| `togglePill(el)` | Toggle a single pill on/off |
| `toggleCheatsheet()` | Open/close the cheatsheet modal |
| `switchTab(tab)` | Switch between Instructions/Foundations tabs |
| `submitAll()` | Flatten responses and POST to Google Sheets |

---

## Design System

CSS variables (defined on `:root`):

| Variable | Use |
|---|---|
| `--bg` | Page background (`#f5f0e8`) |
| `--card` | Card background (`#fffdf8`) |
| `--ink` | Primary text |
| `--ink-light` | Muted text / labels |
| `--accent` | Red accent (`#c84b2f`) — progress bar, left borders |
| `--agree` | Green (`#2a7a4b`) |
| `--disagree` | Red (`#c84b2f`) |
| `--neutral` | Amber (`#b08a50`) |
| `--border` | Subtle border colour |

Fonts: `Playfair Display` (headings) + `DM Sans` (body) via Google Fonts.

---

## What Has Been Deliberately Kept Simple

- No React, no bundler, no npm — single file, drop and deploy
- No user accounts or sessions — Prolific PID is the only identifier
- No backend server — Google Apps Script handles all persistence
- Dilemma order is shuffled on load (`shuffle()`) to mitigate ordering effects
- Back navigation supported — restores all ratings and pill state correctly
- Double-submission prevented by disabling Submit button on first click
