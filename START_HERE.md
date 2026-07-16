# START HERE — Movie Script Buddy Onboarding for a Fresh Agent

You are picking up **Movie Script Buddy** (ships as **ScriptBuddy**) — a distraction-free
screenwriting app: a Fountain-style screenplay editor, a Save-the-Cat **beat board** planner, an
AI **Story Generator**, a character tracker, and PDF / Fountain / FDX export. The entire web app is
**one self-contained file — `index.html`** (~2,900 lines, all HTML + CSS + JS inline) — wrapped in
**Capacitor** to run as a native **Android** app. This file is the single entry point: read it top
to bottom and you'll be oriented without reading the whole 2,900-line file.

**How to trust this file:** the *reference* sections (layout, build steps, how-systems-work, the
gotchas) are durable — trust them. Live repo state (branch, what's in flight, exact state of a
half-finished feature) is NOT hard-coded here; **STEP 1** tells you how to read it from git. Never
assume "what's done" from prose — verify against git and the code.

Movie Script Buddy follows the same START_HERE convention as the sibling "Buddy" apps (WeedBuddy,
ActionBuddy, CarSearchBuddy): one hub doc + a couple of durable companion docs (`ARCHITECTURE.md`,
`DEPENDENCY_MAP.md`), with live state read from git rather than written into prose.

---

## STEP 1 — Establish live state (do this first, every time)

```bash
git -C /Users/mike/Projects/MovieScriptBuddy status
git -C /Users/mike/Projects/MovieScriptBuddy log --oneline -10
```

Git is the source of truth for "what's done," not this doc. `index.html` at the repo **root** is the
source of truth for the app — everything else (`www/`, the Android project's bundled copy) is
generated from it.

## STEP 2 — Read these, based on your task

1. **`ARCHITECTURE.md`** — how the single-file app is put together: the five views + modals, the
   `initApp()` bootstrap, the `storageManager` persistence wrapper (Capacitor Preferences), the
   multi-project data model, the Fountain editor, the AI wiring (Gemini + Claude). Read before any
   coding task.
2. **`DEPENDENCY_MAP.md`** — the "modules inside one file" view: the region map of `index.html`, the
   **storage-key contract** (`scriptbuddy_v2_<projectId>_<key>`), who-calls-whom across the render /
   save / AI functions, and the external dependencies (jsPDF CDN, two AI APIs). Skim this to grasp
   the shape fast and to avoid breaking the key schema.
3. **This file's "How the key systems work"** (below) — deep reference on the subsystems that are
   DONE, so you don't reverse-engineer them.

## STEP 3 — How Mike works (read this — it is load-bearing)

- **Mike tests on a real Android phone**, not a browser. "Load it on my phone" means: build the APK
  and install it over USB (see **"Build, install & deploy"** below). A Samsung Galaxy (`SM-S931U`,
  adb serial `RFCY515DY3H`) is the usual target.
- **`index.html` at the repo root is the ONE source you edit.** ⚠️ Do **not** edit `www/index.html`
  or the copy under `android/app/src/main/assets/` — those are disposable build outputs, overwritten
  on every `npx cap sync`. Edits there are silently lost.
- **Keep it a single file.** The whole app is intentionally one `index.html` with inline CSS/JS. Do
  not "modularize" it into separate `.js`/`.css` files or introduce a bundler without Mike asking —
  that would break the zero-build-step deploy flow.
- **Don't trust the README.** `README.md` is aspirational leftover (it mentions Vite / `npm run dev`
  / port 5173 — none of that exists). The real app has **no dev build step**; you open `index.html`
  directly or run it on-device. `LANGUAGE_BUDDY_*.md` are cruft from an unrelated project — ignore.

## STEP 4 — Keep these docs current (your job)

When you change how a system works, update the doc that owns it — surgically, in the same session:
- New/renamed view, modal, or a change to the bootstrap/persistence flow → **`ARCHITECTURE.md`**.
- New storage key, a change to the key schema, or a new external dependency → **`DEPENDENCY_MAP.md`**.
- A new durable rule, a shipped feature, or the next-up work → **this file** ("How the key systems
  work" / "Current focus & open work").

Do **not** write volatile state (branch names, commit hashes, "currently debugging X") into the
reference sections. Date durable facts inline as `(YYYY-MM-DD)` when it helps — don't add a
top-of-file "last updated" stamp.

---

## Project layout

```
MovieScriptBuddy/
├── index.html              ⭐ THE APP — all HTML/CSS/JS inline (~2,900 lines). Edit ONLY this.
├── manifest.json           PWA manifest (name, icons, theme). Copied into www/ at build.
├── capacitor.config.json   appId com.scriptbuddy.app · appName ScriptBuddy · webDir "www"
├── package.json            Capacitor deps only (@capacitor/core|android|preferences ^8.3)
├── www/                    ⚠️ GENERATED + gitignored. Staged copy of index.html + manifest.json.
│                              Capacitor's webDir. Re-created by `npm run prep` (STEP: build) — never edit.
├── android/                Capacitor Android project (Gradle). Build the APK here.
│   ├── app/build.gradle    applicationId com.scriptbuddy.app · versionCode 1 · versionName "1.0"
│   ├── variables.gradle    minSdk 24 · compile/target 36 · cordova-android 14.0.1
│   └── app/src/main/assets/public/   bundled web copy (written by `cap sync` — never edit)
├── scripts/untitled.fountain   sample Fountain file (reference only)
├── README.md               ⚠️ STALE / aspirational — ignore (see STEP 3)
└── LANGUAGE_BUDDY_*.md      ⚠️ cruft from another project — ignore
```

## Build, install & deploy (Android, over USB)

There is **no web build step** — but Capacitor's `webDir` is `www/`, and `www/` is **gitignored and
not checked in**, so it has to be staged from `index.html` before every sync. That staging (and the
whole build→install→launch chain) is now wrapped in **`npm` scripts** (added 2026-07-16 —
`package.json`), so "load it on my phone" is one command:

```bash
cd /Users/mike/Projects/MovieScriptBuddy
npm install        # first time only (Capacitor CLI + platform)
npm run deploy     # stage www → cap sync → gradle build → adb install → launch on the phone
```

The scripts, smallest to largest (each builds on the previous):

| Script | Does | Use when |
|---|---|---|
| `npm run prep` | `mkdir -p www && cp index.html manifest.json www/` | just re-stage the web assets |
| `npm run sync` | `prep` → `npx cap sync android` | web changed, don't need an APK yet |
| `npm run apk` | `sync` → `./gradlew assembleDebug` | build the APK, no install |
| `npm run deploy` | `apk` → `adb install -r … && adb … monkey …` | **the "put it on my phone" button** |

⚠️ **Why the `prep` step exists:** editing `index.html` and running `cap sync` **without re-copying
it into `www/` first** ships the *old* UI. The scripts always re-stage `www/` for you, so as long as
you go through `npm run sync`/`apk`/`deploy` you can't hit that trap. Only reach for a bare
`npx cap sync` if you know `www/` is already fresh.

`npm run deploy` uses a bare `adb` (assumes exactly one device). With multiple devices attached, add
`-s <serial>` (Mike's phone is `RFCY515DY3H`) or run the steps by hand.

To test in a plain browser during development, just open `index.html` — `storageManager` falls back
to `localStorage` when Capacitor isn't present (see `ARCHITECTURE.md` "Persistence").

---

## How the key systems work (reference — these are DONE; here so you don't reverse-engineer them)

- **Five views + modals.** Top-nav tabs switch `.view-section` visibility via `switchView(target)`.
  The views are **Projects** (`view-projects`), **Vision** (`view-vision`), **Planner /
  beat board** (`view-planner`), **Write** (`view-write`), **Characters** (`view-characters`). Modals
  on top: **Title Page**, **Settings**, **Story Generator**. Details → `ARCHITECTURE.md` "Views & ownership".

- **Async bootstrap.** `initApp()` (async) runs on load: `await storageManager.init()` → load
  `projects` + `currentProjectId` → `ensureMigration()` → cache all DOM refs → wire listeners. Nothing
  touches storage before `init()` resolves. (This async refactor shipped in the Capacitor Preferences
  migration commit — see git log.)

- **Persistence = `storageManager` over Capacitor Preferences.** A thin wrapper that loads every
  `scriptbuddy*` key into an in-memory `cache` on `init()`, then serves synchronous `getItem` from
  cache and writes through async `setItem`/`removeItem`/`clear`. Two migrations run automatically:
  legacy `localStorage` → Preferences (in `init()`), and V1 single-script → V2 multi-project (in
  `ensureMigration()`). Falls back to `localStorage` when Capacitor is absent (browser dev). Full key
  schema → `DEPENDENCY_MAP.md` "Storage-key contract".

- **Multi-project model.** `projects[]` + `currentProjectId` are global. Per-project data lives under
  **`scriptbuddy_v2_<projectId>_<key>`**, built by `getPKey(baseKey)`; global data (the project list,
  the cross-project "global bucket") uses flat `scriptbuddy_*` keys. Selecting a project reloads its
  script, beats, vision, and title page.

- **Fountain editor (Write view).** A `contenteditable` screenplay page where each block is typed
  (scene heading / action / character / dialogue / parenthetical / transition); `getNextType()` cycles
  types, `updateIndicator()` shows the current one, autosave is debounced, and a scene **Navigator**
  + character autocomplete + a "Stash" scrapbook panel sit alongside. `generateFountainText()` +
  `jsPDF` drive export.

- **Beat board (Planner) + Story Generator.** 15 Save-the-Cat beats (`beatsData`) across 4 acts
  (`acts`) render as draggable cards; order persists per project. The **Story Generator** modal takes
  a logline and calls an AI to populate all 15 beats (`runFullOutlineGeneration()`), with per-beat
  **Regen** (`handleSingleBeatRegen()`).

- **AI providers.** Two, user-selectable, key entered in-app: **Google Gemini 1.5 Pro**
  (`generativelanguage.googleapis.com`) and **Anthropic Claude 3.5 Sonnet** (`api.anthropic.com`,
  called direct from the browser via the `anthropic-dangerous-direct-browser-access` header). Used by
  the AI Helper panel, the Vision seed-story, and the Story Generator. ⚠️ The model IDs are pinned to
  older versions in the fetch URLs/headers — check `DEPENDENCY_MAP.md` "External dependencies" before
  assuming they're current.

## Current focus & open work

Read the **git log (STEP 1)** for what actually just shipped and what's mid-flight — that's
authoritative. Durable, known-standing items:

- ✅ **`www/` staging + deploy are scripted** (2026-07-16). `npm run prep|sync|apk|deploy` handle the
  copy-into-`www/` step, so the "forgot to re-stage" trap is gone. Use `npm run deploy` to load onto
  the phone. (This is the one build wrinkle unique to this project — the sibling Buddy apps don't
  stage a `www/`, which is why the step never came up there.)
- ⚠️ **PDF export depends on a CDN** (`jspdf` from `cdnjs.cloudflare.com`, loaded in `<head>`). PDF
  export therefore needs internet even on-device. Bundling jsPDF locally would make export fully
  offline — see `DEPENDENCY_MAP.md` "External dependencies".
- ⚠️ **AI model IDs are dated** (Gemini 1.5 Pro, Claude 3.5 Sonnet, hard-coded in `index.html`). If
  Mike wants current models, these are the spots to bump.
- ✅ **Data layer is durable across reinstalls** — migrated to Capacitor Preferences (native
  key-value store), so screenplays survive app updates. Don't reintroduce raw `localStorage` as the
  primary store.
