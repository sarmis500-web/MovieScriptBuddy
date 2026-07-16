# DEPENDENCY MAP — Movie Script Buddy

**Purpose:** the layer / dependency view of a project whose entire app is one file. Because there are
no separate source modules, the "graph" here is (1) the **region map** of `index.html`, (2) the
**storage-key contract** every save/load path must honor, (3) the **call graph** across the render /
save / AI functions, and (4) the **external dependencies** that reach outside the file.

**How to trust it:** the region line numbers are approximate (the file changes) — grep to confirm.
The storage-key schema and the external-dependency list are the load-bearing parts; keep them exact.
Read `START_HERE.md` first; architecture detail is in `ARCHITECTURE.md`.

## The one rule: everything flows through `storageManager` and `getPKey`

There is no import graph to violate, but there **is** one invariant that keeps data correct:

```
render/load functions ─getItem─▶ storageManager.cache ◀─setItem─ save functions
                                        │
                     all per-project keys go through getPKey(baseKey)
                                        │
                        Capacitor Preferences  (localStorage fallback)
```

- **Never** read or write a `scriptbuddy_v2_*` key by hand-building the string. Always go through
  `getPKey(baseKey)` so the `currentProjectId` namespace is correct.
- **Never** call `localStorage.*` directly for app data. Go through `storageManager` — it owns the
  Preferences/localStorage choice and the in-memory cache.
- **Never** read storage before `await storageManager.init()` has run (the cache is empty until then).

## Region map of `index.html`

| Region | Approx lines | Owns |
|---|---|---|
| `<head>` + CDN/font links | 1–12 | jsPDF (cdnjs), Google Fonts preconnect/link |
| `<style>` | 13–799 | All CSS: noir theme, layout, editor + beat-board styling |
| Markup: nav + views + modals | 800–1071 | The 5 `.view-section` panels, 3 modals, side panels, popups |
| JS: reference data | 1073–1097 | `acts` (4), `beatsData` (15 Save-the-Cat beats) |
| JS: `storageManager` | 1099–1166 | Persistence wrapper (Preferences + cache + fallback) |
| JS: project model + migration | 1168–1255 | `getPKey`, `ensureMigration`, `initApp` bootstrap |
| JS: views, editor, AI, export | 1256–2912 | All feature logic (see call graph below) |

Grep anchors that survive edits better than line numbers: `const storageManager`, `function getPKey`,
`async function initApp`, `function switchView`, `function renderBeatBoard`, `async function
runFullOutlineGeneration`.

## Storage-key contract

Every key begins `scriptbuddy`. Two classes:

| Key | Scope | Written by | Holds |
|---|---|---|---|
| `scriptbuddy_projects` | global | `saveProjects()` | JSON array of `{ id, name, lastEdited }` |
| `scriptbuddy_current_id` | global | `selectProject()` / `deleteProject()` | the active `currentProjectId` |
| `scriptbuddy_active_view` | global | `selectProject()` + view-switch | last-open view (e.g. `'write'`) restored on load |
| `scriptbuddy_api_keys` | global | AI key input `change` | JSON map `{ gemini: key, claude: key }` |
| `scriptbuddy_global_bucket` | global | `saveGlobalBucket()` | cross-project "global bucket" items |
| `scriptbuddy_beat_order` | **global** ⚠️ | `saveBeatOrder()` | array of beat-card ids (drag order) — see note |
| `scriptbuddy_v2_<projectId>_script` | per-project | editor autosave | Fountain/editor script HTML |
| `scriptbuddy_v2_<projectId>_beats` | per-project | `triggerBeatSave()` | beat-board content (per beat) |
| `scriptbuddy_v2_<projectId>_movie_idea` | per-project | Story Generator / idea box | the logline / movie idea |
| `scriptbuddy_v2_<projectId>_genre` | per-project | Story Generator / idea box | selected genre |
| `scriptbuddy_v2_<projectId>_vision` | per-project | `saveVision()` | `{ title, text }` |
| `scriptbuddy_v2_<projectId>_titlepage` | per-project | title-page modal | `{ title, author, contact }` |
| `scriptbuddy_v2_<projectId>_stash` | per-project | `saveStash()` | scrapbook "stash" items |

**Per-project keys are built ONLY via `getPKey(baseKey)`** → `scriptbuddy_v2_${currentProjectId}_${baseKey}`
(falls back to `scriptbuddy_${baseKey}` if no project is selected).

⚠️ **Scope quirk — `scriptbuddy_beat_order` is global, but beat _content_ (`…_beats`) is per-project.**
So beat card *ordering* is shared across all projects while the beats themselves are not. If you touch
the beat board, be aware of this split (unifying `beat_order` under `getPKey('beat_order')` would make
ordering per-project too — check with Mike before changing, it affects existing saved data).

### ⚠️ Migrations — read before touching keys
Two one-way migrations run at startup and must keep working:
1. **localStorage → Preferences** (`storageManager.init()`): first native launch copies any legacy
   `scriptbuddy*` values out of `localStorage` into Preferences.
2. **V1 → V2** (`ensureMigration()`): a legacy single-script install (flat `scriptbuddy_script`,
   `scriptbuddy_beats`, `scriptbuddy_vision`, `scriptbuddy_titlepage`) is wrapped into one auto-created
   project and rewritten under the `scriptbuddy_v2_<id>_*` schema. The old flat V1 keys are the
   migration *source* — don't repurpose those names for new data.

## Call graph (who drives whom)

- **Bootstrap:** `initApp` → `storageManager.init` → `ensureMigration` → `renderProjects` /
  `renderGlobalBucket` → wires `switchView` on nav tabs.
- **Project switch:** `selectProject` → loads per-project keys → `loadEditor`, `renderBeatBoard`,
  `loadVision`, `loadTitlePage`, `buildCharacterView`.
- **Write view:** editor input → debounced autosave (`getPKey('script')`) → `requestRebuildNavigator`
  → `buildNavigator`; `extractCharacterStats` feeds both autocomplete and `buildCharacterView`.
- **Planner:** `renderBeatBoard` → `attachPlannerListeners` → drag → `getDragAfterElement` →
  `saveBeatOrder` (`getPKey('beats')`).
- **Story Generator:** `openStoryGeneratorModal` → `runFullOutlineGeneration` → AI call →
  `parseAIResponse` → `populateBeatsWithAnimation`; per-card `handleSingleBeatRegen` → AI call.
- **Export:** editor → `generateFountainText` → `downloadString` (Fountain/FDX) or jsPDF +
  `checkPageBreak` (PDF).

## External dependencies (the only things outside `index.html`)

| Dependency | How it's loaded | Used by | Notes / risk |
|---|---|---|---|
| **jsPDF** 2.5.2 | `<script src>` from `cdnjs.cloudflare.com` | PDF export | ⚠️ CDN — PDF export needs internet. Bundle locally to go fully offline. |
| **Google Fonts** (Courier Prime, Inter) | `<link>` from `fonts.googleapis.com` | Whole UI | Cosmetic; degrades to system fonts offline. |
| **Google Gemini 1.5 Pro** | `fetch` `generativelanguage.googleapis.com/v1beta/models/gemini-1.5-pro:generateContent` | AI Helper, Vision seed, Story Generator | Model ID hard-coded in URL. User supplies API key. |
| **Anthropic Claude 3.5 Sonnet** | `fetch` `api.anthropic.com/v1/messages` (+ `anthropic-dangerous-direct-browser-access`) | same three | Model ID/version in headers. Direct-from-browser call. |
| **@capacitor/preferences** 8.0.1 | native plugin via `window.Capacitor.Plugins.Preferences` | `storageManager` | The durable data store. Absent in a plain browser → localStorage fallback. |

⚠️ **The AI provider branch (`if (model === 'gemini') … else …`) is duplicated in three places**
(`saveVision`/seed, AI Helper run, and the Story Generator). Changing a model ID, endpoint, or request
shape means editing **all three** call sites — grep `generativelanguage.googleapis.com` and
`api.anthropic.com` to find them all. Consider extracting a single `callAI(model, apiKey, prompt)`
helper before the next AI change.

## Dormant / intentional, do NOT delete as "unused"

- **`localStorage` fallback paths** in `storageManager` look dead on-device but are the browser-dev
  path — keep them.
- **V1 flat-key reads** in `ensureMigration()` look like dead references to keys nothing writes
  anymore — they're the migration source for old installs. Keep them.

## Regenerate this map

There's no generator — this is hand-maintained. When you add a storage key, a view, or an external
call, update the matching table here in the same change (see `START_HERE.md` STEP 4). Fastest audit:
```bash
grep -noE "scriptbuddy[a-z_]*|getPKey\('[a-z]+'\)" index.html | sort -u    # every storage key touched
grep -nE "fetch\(|cdnjs|googleapis|anthropic" index.html                    # every external call
```
