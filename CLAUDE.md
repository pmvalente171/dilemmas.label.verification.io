\# CLAUDE.md — Daily Dilemmas Label Verification Task



\## Project Overview



A single-file web annotation app (`dilemmas\_annotation.html`) built for Prolific. Participants verify the quality of moral foundation labels assigned to everyday ethical dilemmas. The app collects ratings and submits them to a Google Sheet backend.



The theoretical framework is \*\*Moral Foundations Theory (MFQ-2)\*\* — Atari et al. (2023), \*Journal of Personality and Social Psychology\*. The six foundations are: Care, Equality, Proportionality, Loyalty, Authority, Purity.



\---



\## File Structure



Everything lives in a \*\*single HTML file\*\* — no build pipeline, no framework, no dependencies. It can be deployed by dropping it on Vercel, Netlify, or GitHub Pages.



```

dilemmas\_annotation.html

├── <style>          — All CSS (CSS variables, layout, components)

├── Screens 1–5      — HTML for each screen (hidden/shown via JS)

└── <script>

&#x20;   ├── Constants     — PROLIFIC\_CODE, PROLIFIC\_COMPLETION\_URL, SHEET\_URL

&#x20;   ├── FOUNDATION\_META  — Labels and definitions for the 6 foundations

&#x20;   ├── RAW           — The 22 dilemma objects (hardcoded data)

&#x20;   ├── State         — currentIdx, responses {}

&#x20;   └── Functions     — navigation, rendering, saving, submitting

```



\---



\## Screen Flow



```

Screen 1: Welcome

&#x20;   ↓ (Next button)

Screen 2: Consent

&#x20;   ↓ (checkbox must be ticked before Next unlocks)

Screen 3: Instructions

&#x20;   ↓ (Next button)

Screen 4: Annotation (×22 dilemmas, shuffled)

&#x20;   ↓ (Submit button — only enabled when all ratings + dropdowns are filled)

Screen 5: Completion (shows Prolific code, redirects to Prolific)

```



Navigation is handled by `goTo(screenId)` which shows/hides `.screen` divs.



\---



\## Annotation Screen Logic



Each dilemma has:

\- A situation text shown at the top

\- Two columns: \*\*Action A\*\* (internally `do`) and \*\*Action B\*\* (internally `notdo`)

\- Each column lists the moral foundations assigned to that action in the source data

\- Foundations are ordered: shared foundations first, differentiating foundation last (separated by a dashed line marked with an asterisk `\*`)



\### Per-foundation row:

1\. Foundation name (hovering shows a tooltip with definition + example)

2\. A 3-point dot slider: \*\*Disagree · Neutral · Agree\*\*

3\. A dropdown (hidden by default): appears only when rating is Disagree or Neutral, asking "Which foundation fits better?" — options are the 6 foundations + "None"



\### Next/Submit button gating:

\- Disabled on page load

\- Enabled only when \*\*every\*\* foundation row has a rating AND every visible dropdown has a selection

\- `checkComplete()` is called on every slider change and every dropdown change

\- On the last dilemma, button reads "Submit" instead of "Next"

\- On Submit: button is immediately disabled and shows "Submitting…" to prevent double-submission



\---



\## Data Model



\### Dilemma object (from `RAW` array):

```javascript

{

&#x20; id: 33532,

&#x20; situation: "...",

&#x20; do\_action: "Accept the Promotion",

&#x20; do\_labels: "{'loyalty\_betrayal'}",          // raw string from Excel

&#x20; notdo\_action: "Reject the Promotion",

&#x20; notdo\_labels: "{'authority\_subversion', 'loyalty\_betrayal'}",

&#x20; differentiator: "{'authority\_subversion'}",

&#x20; // parsed versions added at runtime:

&#x20; do\_parsed: \["loyalty\_betrayal"],

&#x20; notdo\_parsed: \["authority\_subversion", "loyalty\_betrayal"],

&#x20; diff\_parsed: \["authority\_subversion"],

}

```



`parseSet(s)` converts the raw string format `"{'care\_harm', 'loyalty\_betrayal'}"` into a JS array, filtering out any keys not in `FOUNDATION\_META`.



\### Response state:

```javascript

responses = {

&#x20; \[dilemma\_id]: {

&#x20;   do: {

&#x20;     \[foundationKey]: { rating: "agree"|"neutral"|"disagree", correction: "care\_harm"|"none"|"" }

&#x20;   },

&#x20;   notdo: { ... }

&#x20; }

}

```



Saved by `saveCurrentResponse()` on every Next/Back navigation.



\---



\## Foundation Keys



Internal keys used in the data and `FOUNDATION\_META`:



| Key | Display Label |

|---|---|

| `care\_harm` | Care |

| `equality` | Equality |

| `proportionality` | Proportionality |

| `loyalty\_betrayal` | Loyalty |

| `authority\_subversion` | Authority |

| `purity\_degradation` | Purity |



\---



\## Google Sheets Integration



\*\*Webhook URL:\*\*

```

https://script.google.com/macros/s/AKfycbx0a-aKW-xIMHBMzbHGOMC5lPX3JKqkyNNIeYqAXL1fh0il4KqmNeEPU1K9JoIVe4IH8w/exec

```



On submit, `submitAll()` flattens `responses` into one row per foundation rating and POSTs:



```javascript

{

&#x20; rows: \[

&#x20;   {

&#x20;     prolific\_pid: "abc123",

&#x20;     dilemma\_id: "33532",

&#x20;     side: "do",               // "do" or "notdo"

&#x20;     foundation: "care\_harm",

&#x20;     rating: "agree",          // "agree", "neutral", or "disagree"

&#x20;     correction: "",           // foundation key, "none", or ""

&#x20;   },

&#x20;   ...

&#x20; ]

}

```



Uses `mode: "no-cors"` (required for Google Apps Script — no response body returned but data writes correctly).



The Apps Script (`doPost`) appends one sheet row per object in `rows`, adding a server-side timestamp.



\*\*Sheet columns:\*\* `prolific\_pid | dilemma\_id | side | foundation | rating | correction | timestamp`



\---



\## Prolific Integration



Participant ID is read from the URL on load:

```javascript

const PROLIFIC\_PID = new URLSearchParams(window.location.search).get("PROLIFIC\_PID") || "UNKNOWN";

```



Study URL format to paste into Prolific:

```

https://your-site.vercel.app?PROLIFIC\_PID={{%PROLIFIC\_PID%}}

```



\*\*To update before going live:\*\*

```javascript

const PROLIFIC\_CODE = "C1B2-A3D4";  // ← replace with real Prolific completion code

```

The completion URL is auto-constructed from this code.



\---



\## Key Constants to Update Before Deployment



| Constant | Location | What to change |

|---|---|---|

| `PROLIFIC\_CODE` | Top of `<script>` | Your actual Prolific study completion code |

| `SHEET\_URL` | Inside `submitAll()` | Already set — only change if you redeploy the Apps Script |



\---



\## Design System



CSS variables (defined on `:root`):



| Variable | Use |

|---|---|

| `--bg` | Page background (`#f5f0e8`) |

| `--card` | Card background (`#fffdf8`) |

| `--ink` | Primary text |

| `--ink-light` | Muted text / labels |

| `--accent` | Red accent (`#c84b2f`) — used for progress bar, foundation boxes |

| `--agree` | Green (`#2a7a4b`) |

| `--disagree` | Red (`#c84b2f`) |

| `--neutral` | Amber (`#b08a50`) |

| `--border` | Subtle border colour |



Fonts: `Playfair Display` (headings) + `DM Sans` (body) via Google Fonts.



\---



\## What Has Been Deliberately Kept Simple



\- No React, no bundler, no npm — single file, drop and deploy

\- No user accounts or sessions — Prolific PID is the only identifier

\- No backend server — Google Apps Script handles all persistence

\- Dilemma order is shuffled on load (client-side `shuffle()`) to mitigate ordering effects

\- Back navigation is supported and restores previous responses correctly via `restoreResponse()`

