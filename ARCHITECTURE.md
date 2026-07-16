# ARCHITECTURE — Movie Script Buddy

How the app is put together. Read `START_HERE.md` first. This doc is durable reference — verify
counts and specifics against `index.html` if they matter; live state lives in git.

## The one big fact: it's a single self-contained HTML file wrapped in Capacitor

The entire web app is **`index.html`** at the repo root — one file, ~2,900 lines, with all CSS in a
single `<style>` block (lines ~13–799) and all JavaScript in a single `<script>` block (lines
~1072–2913). No framework, no bundler, no modules, no build step. **Capacitor** wraps that file as a
native Android WebView app (`appId com.scriptbuddy.app`, `appName ScriptBuddy`). The only runtime
dependencies pulled from outside the file are: the **jsPDF** library (CDN, in `<head>`), two web
fonts (Google Fonts), the **@capacitor/preferences** native plugin, and two **AI HTTP APIs**.

Because it's one file, "modules" are **regions of `index.html`**, not files — mapped in
`DEPENDENCY_MAP.md` "Region map".

### Portability / platform notes
- Runs identically in a desktop browser (for dev) and in the Android WebView (production). The only
  branch is persistence: `storageManager` uses Capacitor Preferences on-device and falls back to
  `localStorage` in a plain browser — see "Persistence" below.
- `minSdk 24`, `compile/target SDK 36`, Capacitor `^8.3`, cordova-android `14.0.1`
  (`android/variables.gradle`).

## Bootstrap & lifecycle

`initApp()` (async, near line 1197) is the single entry point, invoked on load. Order matters:

```
initApp()
  → await storageManager.init()        // load all scriptbuddy* keys into cache; run localStorage→Preferences migration
  → projects = JSON.parse(getItem('scriptbuddy_projects') || '[]')
  → currentProjectId = getItem('scriptbuddy_current_id')
  → await ensureMigration()            // V1 single-script → V2 multi-project, if legacy data found
  → cache all DOM element references
  → load per-project state (stash, etc.) via getPKey(...)
  → wire event listeners, render initial view
```

⚠️ Nothing may read or write storage before `storageManager.init()` resolves — the cache is empty
until then. This async-first shape is deliberate (it replaced a synchronous `localStorage` bootstrap
so native Preferences reads could be awaited).

## Persistence

`storageManager` (lines ~1100–1166) is a thin wrapper over **Capacitor Preferences** with an
in-memory cache:

- **`init()`** — reads every key beginning `scriptbuddy` from Preferences into `cache`. If Preferences
  is empty, it migrates any legacy `scriptbuddy*` values from `localStorage` into Preferences. If
  Preferences isn't available at all (browser dev), it loads `cache` from `localStorage` instead.
- **`getItem(key)`** — synchronous, reads from `cache` (so render code stays synchronous).
- **`setItem` / `removeItem` / `clear`** — async, write through to Preferences (or `localStorage` on
  the catch path) and update `cache`.

This is why the app survives reinstalls: Preferences is a native key-value store, unlike WebView
`localStorage` which can be cleared by the OS. **Do not** reintroduce raw `localStorage` as the
primary store. Full key list → `DEPENDENCY_MAP.md` "Storage-key contract".

## Data model (multi-project)

- Globals: `projects[]` (each `{ id, name, lastEdited }`) and `currentProjectId`.
- **Per-project keys** are namespaced: `getPKey(baseKey)` → `scriptbuddy_v2_<currentProjectId>_<baseKey>`.
  Per-project base keys: `script`, `beats`, `movie_idea`, `genre`, `vision`, `titlepage`, `stash`.
- **Global keys** (not project-scoped): `scriptbuddy_projects`, `scriptbuddy_current_id`,
  `scriptbuddy_active_view`, `scriptbuddy_api_keys`, `scriptbuddy_global_bucket`, and
  `scriptbuddy_beat_order`. ⚠️ Note `beat_order` is global while beat *content* is per-project — see
  `DEPENDENCY_MAP.md` "Storage-key contract".
- `selectProject(id)` sets `currentProjectId`, persists it, and reloads that project's data.
  `createNewProject()` / `deleteProject(id)` manage the list.
- Static reference data lives as top-of-script constants: `acts` (4 acts) and `beatsData` (15
  Save-the-Cat beats with name/target/description/tip).

## Views & ownership

Five `.view-section` panels; `switchView(target)` toggles the `active` class on both the nav tab and
`#view-<target>`. Modals overlay independently.

| View (`#view-…`) | data-target | Owns / renders | Key functions |
|---|---|---|---|
| Projects | `projects` | Project cards + the cross-project "global bucket" | `renderProjects()`, `renderGlobalBucket()`, `selectProject()`, `createNewProject()`, `deleteProject()` |
| Vision | `vision` | Logline/vision text + AI "seed story" | `loadVision()`, `saveVision()` (calls AI to seed) |
| Planner (beat board) | `planner` | 15 draggable Save-the-Cat beat cards across 4 acts | `renderBeatBoard()`, `attachPlannerListeners()`, `triggerBeatSave()`, `saveBeatOrder()`, `getDragAfterElement()` |
| Write | `write` | Fountain `contenteditable` editor + Navigator + Stash + AI Helper | `loadEditor()`, `getCurrentBlock()`, `updateIndicator()`, `setType()`, `getNextType()`, `buildNavigator()`, `updatePageCount()` |
| Characters | `characters` | Character grid + the Dialogue "Tuner" | `extractCharacterStats()`, `buildCharacterView()`, `activateTuner()`, `deactivateTuner()` |

Modals: **Title Page** (`loadTitlePage()`, save via `#btn-tp-save`), **Settings** (`#btn-wipe-data`
→ `storageManager.clear()`), **Story Generator** (`openStoryGeneratorModal()`,
`runFullOutlineGeneration()`, `handleSingleBeatRegen()`, `parseAIResponse()`,
`populateBeatsWithAnimation()`).

## The Fountain editor (Write view)

A `contenteditable` "screenplay page" (`#screenplay-editor`) where each block carries a **type class**
(scene heading, action, character, dialogue, parenthetical, transition) that controls formatting.
`getCurrentBlock()` finds the caret's block; `setType()` / `getNextType()` cycle the type (Tab-style
progression); `updateIndicator()` reflects it in the floating `#element-indicator`. Autosave is
debounced (`scriptSaveTimer`). Alongside the editor: a scene **Navigator** (`buildNavigator()`,
rebuilt via a debounced `requestRebuildNavigator()`), character **autocomplete** (`#autocomplete-popup`),
and a **Stash** scrapbook panel (`renderScrapbook()`, `saveStash()`). `togglePanel()` manages the
mutually-exclusive Stash / AI side panels.

## Export

- `generateFountainText()` serializes the editor blocks to Fountain plain text.
- `downloadString(text, fileType, fileName)` triggers a file download.
- **PDF**: `checkPageBreak()` + the **jsPDF** UMD build (loaded from `cdnjs.cloudflare.com` in
  `<head>`) lay out a print-ready screenplay. ⚠️ External CDN dependency — PDF export needs internet.
- Formats offered in the export dropdown: **PDF**, **Fountain** (`.fountain`), **FDX** (Final Draft).

## AI integration

Two providers, selected in-app (`#ai-model-select`), API key entered by the user (`#ai-api-key`):

- **`gemini`** → `POST https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-pro:generateContent?key=<apiKey>`
- **`claude`** → `POST https://api.anthropic.com/v1/messages` with headers `anthropic-version: 2023-06-01`
  and `anthropic-dangerous-direct-browser-access: true` (lets the browser call Anthropic directly,
  no proxy).

The same provider branch (`if (model === 'gemini') … else …`) is duplicated in **three** call sites:
the AI Helper panel (polish dialogue / new scene from beat), the Vision seed-story, and the Story
Generator (full outline + per-beat regen). ⚠️ Because the branch is copy-pasted, changing a model ID
or request shape means editing **all three** places — see `DEPENDENCY_MAP.md` "External dependencies".
