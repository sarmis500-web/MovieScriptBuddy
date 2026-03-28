# Fallback Prompt for Next Agent: Language Buddy Registration

Deliver this prompt to the next agent if the "Language Buddy" project is still missing from the Antigravity "Select to open" list.

---

**PASTE THE FOLLOWING TO THE NEXT AGENT:**

"The previous agent attempted to register the 'Language Buddy' project in the Antigravity launch menu but I cannot find it. 

**Deep-Dive Findings provided for your context:**
1. **The Registry Mechanism**: In this environment, `antigravity` is an alias to a custom shell script at `~/.antigravity/antigravity-launch.sh`.
2. **Hard-coded Projects**: This script contains a manual array named `projects` that lists the display names and paths for the launch menu. It is NOT an automated filesystem scanner.
3. **The Current Fix State**: 
   - Registry modification was attempted at: `/Users/mike/.antigravity/antigravity-launch.sh`.
   - The line `"Language Buddy|/Users/mike/Projects/LanguageBuddy"` should be in the `projects` array.
   - Project marker initialization was performed in: `/Users/mike/Projects/LanguageBuddy` (including `node_modules`, `android/`, and `www/`).

**Your Mission**:
1. Check `which antigravity` and `alias antigravity` to confirm it points to `~/.antigravity/antigravity-launch.sh`.
2. Inspect `~/.antigravity/antigravity-launch.sh`. If 'Language Buddy' is missing from the `projects` array, add it immediately.
3. If it is already there but the menu is not showing it, research if there is a secondary registry in `~/.gemini/antigravity/` or a caching mechanism for the launch script."
