# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Static HTML mockups for the SMC/PMSP *emenda parlamentar* (parliamentary amendment) flow — creation, review/approval, and tracking screens. Purpose is validating flow and layout with stakeholders before implementation, so the mockups fake their data instead of calling a backend.

**Zero dependencies, zero build step.** No `package.json`, no bundler, no tests, no linter. Every page is one self-contained `.html` file with inline `<script>` — no external JS, fonts, or images anywhere in the repo. The one exception is `styles/global.css`, shared by the promoted `pages/<slug>/index.html` pages (see below); flat `pages/*.html` snapshots keep their styles inline. Keep it that way otherwise: a page must render correctly when opened directly from disk.

## Running

```bash
open index.html                      # good enough for most pages
python3 -m http.server 8000          # then http://localhost:8000 — use when testing relative links / localStorage
```

## Page layout and naming

- `index.html` — the gallery. Links **only** promoted pages. Its app-bar pill hardcodes the screen count (`2 telas`) — update it when adding or removing a card.
- `pages/<slug>/index.html` — **promoted** page: the current, canonical version of a screen (`criacao-emenda/`, `acompanhamento/`).
- `pages/*.html` (flat) — earlier explorations kept for reference, not linked from the index. Don't update these when changing a promoted page; they are 
- `pages/<slug>/versions/v{version_number}.html` the old version for updated page to preserve history

