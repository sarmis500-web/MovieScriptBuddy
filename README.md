# Movie Script Buddy

A personal screenwriting application inspired by Final Draft, Highland, and Fade In.

## Features

- **Fountain markup editor** — Write in plain text, get industry-standard formatting
- **Live preview** — See your formatted screenplay as you type
- **Scene navigator** — Jump between scenes instantly
- **Character tracker** — See all characters and their appearances
- **PDF export** — Generate print-ready screenplays
- **Auto-save** — Never lose your work
- **Multiple scripts** — Manage all your screenplays in one place
- **Distraction-free mode** — Full-screen writing

## Quick Start

```bash
npm install
npm run dev
```

Then open [http://localhost:5173](http://localhost:5173)

## Fountain Syntax Quick Reference

```
INT. LOCATION - DAY          → Scene heading
EXT. LOCATION - NIGHT        → Scene heading

CHARACTER NAME               → Character (must be UPPERCASE)
Dialogue goes here.          → Dialogue (follows character)
(whispering)                 → Parenthetical

CUT TO:                      → Transition
FADE IN:                     → Transition

*italics*                    → Emphasis
**bold**                     → Bold
_underline_                  → Underline
```

## Tech Stack

- Vite (dev server)
- Vanilla JavaScript
- Fountain markup format
