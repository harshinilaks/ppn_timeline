# PPN TimelineJS Modifications — Developer Documentation

This document describes every custom modification made to the upstream
[TimelineJS3](https://github.com/NUKnightLab/TimelineJS3) source code for the
PPN (Potential Lamp / Palestinian Primary Sources) project. It is intended to
help future PPN developers quickly understand what was changed, why, and how to
maintain or extend it.

---

## Overview of What Was Changed

The upstream TimelineJS is a general-purpose interactive timeline library. The
PPN fork extends it with:

1. **Predefined contextual events** — key Palestine-related dates that always
   appear on the timeline regardless of what data source the user loads.
2. **Collection-based filtering** — a UI for showing only events tagged to one
   of four PPN collection categories.
3. **Keyword filtering** — a chip-based UI for filtering events by
   semicolon-delimited keyword/subject tags.
4. **Title pin** — a search-to-pin UI letting users surface specific items by
   their source title.
5. **Date range filtering** — a date picker bar for narrowing the displayed
   events by start/end date, independent of the timeline navigation.
6. **CSV column mapping** — extended CSV parsing to accept the PPN spreadsheet
   column headers alongside the default TimelineJS headers.
7. **Styling for all new UI** — new LESS rules for every custom filter
   component, added to the existing MenuBar stylesheet.

---

## Modified Files

### 1. `src/js/timeline/Timeline.js`

This is the primary modified file. All PPN-specific logic lives here.

#### 1a. `PREDEFINED_EVENTS` constant (lines 19–132)

```js
const PREDEFINED_EVENTS = [ ... ]
```

**What it does:**
Defines an array of 14 hardcoded Palestine-related historical events that are
injected into every timeline at load time, regardless of the user's data source.
These events always appear and are never filtered out by any of the collection,
keyword, or title filters.

**Current events (in chronological order):**
- Balfour Declaration (1917)
- British Mandate (1923)
- Great Revolt (1936)
- The Nakba (1948)
- Intilaqa (1965)
- The Naksa / 6 Day War (1967)
- Battle of Karameh (1968)
- Sabra and Shatila Massacres (1982)
- The First Intifada (1987)
- Madrid Conference (1991)
- Oslo Accords (1993)
- The Second Intifada (2000)
- Siege of Gaza (2007)
- March of Return (2018)

**How to update events:**
Edit the `PREDEFINED_EVENTS` array directly in this file. Each entry must have:
```js
{
    unique_id: "a-unique-string-id",   // used internally; must be unique
    start_date: { year: "YYYY", month: "MM", day: "DD" },
    text: {
        headline: "Event Title",
        text: "Event description."
    }
}
```
After editing, run `npm run dist` and redeploy.

---

#### 1b. `COLLECTION_CATEGORIES` constant (lines 134–139)

```js
const COLLECTION_CATEGORIES = [
    "Heritage, Belonging, and Belief",
    "Violence, War, and Displacement",
    "Space, Architecture, and Environment",
    "Music, Dance, and Literature"
]
```

**What it does:**
Defines the four collection category names that appear as filter buttons in the
UI. These must exactly match the values in the `assigned collection` column of
the PPN Google Sheet (case and quote insensitive — see normalization below).

**How to update:**
If the collection categories in the Google Sheet change, update this array to
match. Values are normalized (whitespace + quote-stripped) before comparison, so
minor formatting differences in the sheet are tolerated.

---

#### 1c. Filter helper functions (lines 141–252)

These pure functions support the filter system:

| Function | Purpose |
|---|---|
| `normalizeCollectionValue(value)` | Strips whitespace and straight/curly quotes from a collection value for reliable comparison |
| `isPredefinedEvent(event)` | Returns `true` for events in `PREDEFINED_EVENTS`; these are exempt from all filters |
| `matchesAssignedCollection(event, selectedCollections)` | Checks if an event's `assigned_collection` field matches the active filter |
| `normalizeKeywordValue(value)` | Lowercases and trims a keyword string |
| `getEventKeywords(event)` | Extracts the `keywords` array or string from an event |
| `matchesKeywordFilter(event, selectedKeywords, mode)` | Checks if an event matches selected keywords; supports `'any'` (OR) and `'all'` (AND) modes |
| `normalizeTitleValue(value)` | Lowercases and trims a title string for comparison |
| `getEventSourceTitle(event)` | Returns `event.source_title` or falls back to `event.text.headline` |
| `markPredefinedEvent(event)` | Stamps `event.is_predefined_event = true` for internal tracking |
| `addPredefinedEvents(data)` | Injects predefined events into a raw JSON data object before parsing |

---

#### 1d. `_initData()` — predefined event injection (lines 531–554)

**What it does:**
Modified from the original to inject `PREDEFINED_EVENTS` into the config
regardless of whether the data source is a URL string, a `TimelineConfig`
object, or a raw JSON object. Predefined events are added via `config.addEvent()`
so that they go through TimelineConfig's normal date parsing pipeline.

**Why it matters:**
This is what ensures the predefined contextual events always appear on the
timeline even when the user loads a completely different Google Sheet.

---

#### 1e. State properties on `Timeline` constructor (lines 339–355)

Three new state objects were added to the `Timeline` class constructor:

```js
this._collection_filter = { selected: new Set() };
this._keyword_filter = { selected: new Set(), mode: 'any', all_keywords: [] };
this._title_pin = { selected_ids: new Set(), all_titles: [] };
this._original_all_events = null;
this._original_all_eras = null;
```

These maintain the current filter state across user interactions. The
`_original_all_events` / `_original_all_eras` snapshots are taken when the
config first loads and are never mutated — all filtering works by deriving
a filtered subset from these.

---

#### 1f. `_applyCollectionFilter()` (lines 556–602)

**What it does:**
The central filter dispatch function. Called any time the user changes any
filter selection. It:

1. Reads the current collection, keyword, and title pin selections.
2. Always keeps predefined events visible.
3. Hides all non-predefined events when no filter is active (default state:
   only predefined events are shown until the user selects a filter).
4. Shows events that match all active filters.
5. Resets `_original_events` for the date filter to always operate on the
   currently-filtered set (not the full set).

**Default behavior:**
When the timeline first loads, no collection is selected, so only the
predefined events are visible. Users must select a collection to see
items from the data source.

---

#### 1g. Filter UI methods (lines 604–984)

All of these are new — they do not exist in the upstream TimelineJS.

| Method | What it renders |
|---|---|
| `_initCollectionFilterUI()` | Renders "Filter by collection" pill buttons (All + 4 categories) |
| `_syncCollectionFilterUI()` | Updates active/inactive visual state of collection buttons |
| `_rebuildKeywordUniverse()` | Scans all non-predefined events to build the list of available keywords |
| `_initKeywordFilterUI()` | Renders keyword autocomplete input, Match any/all toggles, chip display |
| `_syncKeywordFilterUI()` | Updates chips and mode button active states |
| `_rebuildTitleUniverse()` | Scans all non-predefined events to build the list of available source titles |
| `_initTitlePinUI()` | Renders title autocomplete input and pinned-title chips |
| `_syncTitlePinUI()` | Updates title pin chips |
| `_initDateFilterBarUI()` | Renders "Filter by date" bar with start/end date pickers and Filter/Clear buttons |

---

#### 1h. `_initLayout()` — new UI insertion (lines 1136–1217)

**What changed:**
Four new calls were added at the start of `_initLayout()`, before the original
storyslider/timenav/menubar creation:

```js
this._initCollectionFilterUI();
this._rebuildKeywordUniverse();
this._initKeywordFilterUI();
this._rebuildTitleUniverse();
this._initTitlePinUI();
this._initDateFilterBarUI();
```

The height calculation was also updated to subtract the heights of the new
filter UI bars from the available height before computing timenav and
storyslider dimensions.

---

### 2. `src/js/core/ConfigFactory.js`

#### 2a. `getRowValueCaseInsensitive()` (lines 9–25)

**New helper function.** Looks up a key in a CSV row object using a
case-insensitive string match. Used to read columns like
`assigned collection` or `Keywords/Subjects/Tags` regardless of how they are
capitalized in the sheet.

---

#### 2b. `extractEventFromCSVObject()` — extended column mapping (lines 80–174)

**What changed:**
The original function only recognized standard TimelineJS column headers.
The PPN version adds fallback mappings for the PPN spreadsheet's custom
column names:

| TimelineJS default | PPN fallback |
|---|---|
| `Headline` | `Title` |
| `Text` | `Object Description (Abstract)` |
| `Display Date` | `Year Published` |

Two new fields are extracted from the CSV row and stored on each event:

- `source_title` — set to `row['Title'] || headline`; used by the title pin
  feature to identify events by their archival/source title.
- `assigned_collection` — read via `getRowValueCaseInsensitive(row, 'assigned collection')`;
  used by the collection filter.
- `keywords` — read via `getRowValueCaseInsensitive(row, 'Keywords/Subjects/Tags')`,
  then split on `;` into an array; used by the keyword filter.

The date parsing branch now also accepts `row['Year Published']` as a fallback
for `row['Year']` when constructing the `start_date`.

---

#### 2c. `processCSVRows()` — shared CSV processing (lines 183–206)

**New function** (extracted from what was previously inline in the Google Sheets
handler). Applies `extractEventFromCSVObject` and `handleRow` to an array of
parsed CSV rows. This shared logic is used by both `readGoogleAsCSV` and the
new `readCSVFromURL`.

---

#### 2d. `readCSVFromURL()` — direct CSV file support (lines 250–269)

**New function.** Reads a CSV file directly from a URL (not via Google Sheets
proxy) and processes it through `processCSVRows`. This was added to support
loading `.csv` files hosted directly alongside the Jekyll site.

---

#### 2e. `isCSVURL()` (lines 277–281)

**New helper.** Returns `true` if a URL ends in `.csv` (case-insensitive). Used
in `makeConfig()` to route `.csv` URLs through `readCSVFromURL` instead of the
JSON fetch path.

---

#### 2f. `makeConfig()` — CSV URL routing (lines 388–446)

**What changed:**
Added a new branch before the JSON fetch fallback:

```js
} else if (isCSVURL(url)) {
    const json = await readCSVFromURL(url);
    finalizeConfig(json, callback);
}
```

This means users can now pass a `.csv` file URL directly to `TL.Timeline()`
as a data source, in addition to Google Sheets URLs and JSON URLs.

---

### 3. `src/less/ui/TL.MenuBar.less`

**What changed:**
All styles for the new filter UI components were added here. None of the
original `.tl-menubar` rules were removed. New CSS classes added:

| Class | Component |
|---|---|
| `.tl-collection-filter` | Wrapper row for collection filter buttons |
| `.tl-filter-label` | "Filter by collection" / "Filter by date" label text |
| `.tl-collection-filter-button` | Individual pill-shaped collection category button |
| `.tl-collection-filter-button.tl-is-active` | Active/selected state (amber tint) |
| `.tl-keyword-filter` | Wrapper row for keyword filter |
| `.tl-keyword-filter-inputwrap` | Wraps the keyword text input |
| `.tl-keyword-filter-input` | Keyword autocomplete text input |
| `.tl-keyword-filter-mode` | Wraps Match any / Match all buttons |
| `.tl-keyword-filter-mode-button` | Pill button for match mode |
| `.tl-keyword-filter-mode-button.tl-is-active` | Active mode button (grey tint) |
| `.tl-keyword-filter-chips` | Flex container for selected keyword chips |
| `.tl-keyword-filter-chip` | Individual chip (blue tint) |
| `.tl-keyword-filter-clear` | Clear button for keyword filter |
| `.tl-title-pin` | Wrapper row for title pin |
| `.tl-title-pin-inputwrap` | Wraps the title text input |
| `.tl-title-pin-input` | Title autocomplete text input |
| `.tl-title-pin-chips` | Flex container for pinned title chips |
| `.tl-title-pin-chip` | Individual chip (green tint) |
| `.tl-title-pin-clear` | Clear button for title pin |
| `.tl-date-filter-bar` | Wrapper row for the above-timeline date filter bar |
| `.tl-date-filter-bar-input` | Date picker inputs in the bar |
| `.tl-date-filter-bar-button` | Filter / Clear buttons in the bar |
| `.tl-date-filter` (updated) | The original in-menubar date filter; hidden via `.tl-menubar .tl-date-filter { display: none }` since the new bar replaces it above the timeline |

---

## Google Sheet Column Requirements

For PPN data to work correctly with the filters, the source Google Sheet must
include the following column headers (exact names, case-insensitive):

| Column | Purpose | Required? |
|---|---|---|
| `Title` | Source/archival title, used by title pin | Recommended |
| `Object Description (Abstract)` | Slide body text (fallback for `Text`) | Recommended |
| `Year Published` | Year for date (fallback for `Year`) | One of `Year` / `Year Published` required |
| `Assigned Collection` | Collection category for collection filter | Required for collection filter |
| `Keywords/Subjects/Tags` | Semicolon-delimited keywords for keyword filter | Required for keyword filter |

Standard TimelineJS columns (`Headline`, `Text`, `Year`, `Month`, `Day`,
`Media`, `Media Caption`, `Media Credit`, `Background`, `Group`, `Type`) also
continue to work as before.

---

## Build & Deployment

After any change to source files:

```bash
# From the TimelineJS3 repo root:
npm run dist
```

This runs `npm run clean` then `npm run build`, producing:

```
dist/
  js/
    timeline.js        ← the compiled JS bundle
    locale/            ← language files
  css/
    timeline.css       ← compiled CSS
    fonts/
    icons/
    themes/
  embed/
    index.html
```

To deploy to the Jekyll (potential-lamp) site, copy:

```
dist/css  →  potential-lamp/assets/timelinejs/css
dist/js   →  potential-lamp/assets/timelinejs/js
```

The Jekyll timeline page should reference these files as:

```html
<link rel="stylesheet" href="{{ '/assets/timelinejs/css/timeline.css' | relative_url }}">
<script src="{{ '/assets/timelinejs/js/timeline.js' | relative_url }}"></script>
```

---

## Architecture Summary

```
User loads page
    └─ new TL.Timeline('timeline-embed', googleSheetsURL, options)
           ├─ makeConfig(url)              [ConfigFactory.js]
           │     ├─ parseGoogleSpreadsheetURL() → reads sheet via proxy
           │     ├─ readCSVFromURL()       → reads .csv files directly
           │     └─ ajax()                → reads .json files directly
           │
           ├─ _initData(data)             [Timeline.js]
           │     └─ injects PREDEFINED_EVENTS into config
           │
           ├─ setConfig(config)
           │     ├─ saves _original_all_events snapshot
           │     └─ _applyCollectionFilter() → shows only predefined events by default
           │
           └─ _initLayout()
                 ├─ _initCollectionFilterUI()   → collection pill buttons
                 ├─ _initKeywordFilterUI()       → keyword autocomplete + chips
                 ├─ _initTitlePinUI()            → title search + chips
                 ├─ _initDateFilterBarUI()       → date range pickers
                 └─ [original storyslider + timenav + menubar]
```

---

## Key Design Decisions

- **Predefined events are always visible** — they are never removed by any
  filter, so the Palestine historical context is always present regardless of
  what data the user loads.
- **Default state shows only predefined events** — when the timeline first loads
  and no collection is selected, non-predefined events are hidden. This forces
  users to actively choose a collection, preventing an overwhelming default view.
- **Filters are compositing** — collection, keyword, and title pin filters
  combine with AND logic between types (an event must pass all active filters).
  The keyword filter internally supports both ANY and ALL mode for its keywords.
- **Date filter operates on the already-filtered set** — so date narrowing
  works within the currently selected collection, not across all events.
- **`source_title` field** — the title pin searches by `source_title` (the
  archival title of the item), not by `text.headline` (the TimelineJS slide
  headline). This distinction matters because the slide headline may be
  abbreviated.
