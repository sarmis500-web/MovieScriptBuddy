# Handoff: Language Buddy Registration (V2)

**Mission**: Register "Language Buddy" in the Antigravity "Select to open" menu.

## Current State & History

### 1. CLI Registry (Verified Working)
- **File**: `~/.antigravity/antigravity-launch.sh`
- **Status**: The project **is** in the `projects` array.
- **Action**: Running `antigravity` from the terminal *does* show the menu option.

### 2. Project Markers (Verified Present)
- **Path**: `/Users/mike/Projects/LanguageBuddy`
- **Status**: Standard markers (`android/`, `node_modules/`, `www/`) are present.
- **Action**: Initialized via `npm install` and `npx cap add android`.

### 3. IDE Internal Registry (Just Modified)
- **File**: `~/Library/Application Support/Antigravity/User/globalStorage/storage.json`
- **Action**: I manually added the project to:
    - `"profileAssociations": { "workspaces": { ... } }`
    - `"lastKnownMenubarData": { ... "openRecentFolder": [ ... ] }`
- **Status**: **EXPERIMENTAL**. This was my attempt to bridge the gap between the CLI registration and the IDE's GUI menus.

---

## If the Menu is STILL Missing

If you are reading this, it means the GUI "Select to open" or "Recent" menu is still not showing "Language Buddy" even after an IDE restart.

### Deep-Dive Suggestions for You:

1. **Check for DB-backed State**:
   - Inspect `~/Library/Application Support/Antigravity/User/globalStorage/state.vscdb`.
   - This is an SQLite database where VS Code-based apps often store project history. You may need to use `sqlite3` to query/edit it if `storage.json` wasn't enough.

2. **Search for "Recent Projects" Plugins**:
   - Check `~/.antigravity/extensions/` for any custom extension that might be managing the project list.
   - Some Antigravity-specific extensions might have their own JSON files in `~/.gemini/antigravity/`.

3. **Check Workspace Files**:
   - Check if there is a `.code-workspace` file that needs to be created or registered (like `PantryBuddy.code-workspace` seen in the launch script).

4. **Registry Cache**:
   - Investigate if there is a cache in `~/Library/Application Support/Antigravity/Cache` or `~/Library/Caches/Antigravity` that needs clearing.

---

**DON'T RE-DO THESE**:
- Do not re-edit `antigravity-launch.sh` (it's already correct).
- Do not re-initialize `node_modules` or `android/` folders.
