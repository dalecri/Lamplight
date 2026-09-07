# Lamplight

A calm, single-file EPUB reader built around RSVP (Rapid Serial Visual Presentation) reading — one word (or a small chunk of words) flashed at a time — plus a conventional paginated/scroll reading mode for when you'd rather read normally. Everything runs client-side: no account, no server, no analytics. Your books and progress live only on your device.

## What it does

- **Import EPUBs** from the device's file picker. Cover art, metadata, chapter list, and word stream are all extracted client-side (via `epub.js` + `JSZip`) and stored locally.
- **RSVP reading mode** — flashes words at a configurable speed (words per minute), with adjustable chunk size (1–3 words), a focus-marker style (highlight / notch / underline) on the optimal recognition point of each word, scrub-to-rewind while paused, and double-tap-to-seek ±10 words while playing.
- **Normal reading mode** — paginated (swipe) or continuous scroll, with page-turn animations (fade / flip / slide), adjustable font, size, and text alignment, and a drop cap + chapter counter on chapter openings.
- **Four built-in reading themes** (Night, Lamplight, Day, Campfire) each pairing a color palette with an animated "sky" (stars, clouds, campfire, or a pull-cord lamp), plus nine additional flat-color palettes ("More colors"). Every theme's interactive fixture (moon/sun/lamp/fire) can be press-and-pulled like a real fixture for a bit of tactile delight.
- **Reading stats** — lifetime words read, a daily streak, and a 7-day words/wpm chart, all derived from a local activity log.
- **Sleep timer** with preset durations (15/30/45/60 min) that pauses playback when it elapses.
- **A one-time onboarding flow** that walks through typeface, text size, a real timed reading-speed calibration (not a guess), reading theme, and an optional set of curated starter books.
- **Cross-device progress sync** via manual JSON export/import (title + author + progress only — no book text leaves the device).
- **Small-screen landscape handling**: on a small phone turned sideways, the moon/sun/lamp/fire art hides (a small "tip" of moon/sun still peeks through; lamp/fire fixtures fully hide) while their light beam stays, and the speed slider collapses into a compact vertical rail on one edge instead of a wide horizontal bar.

## Architecture

Everything lives in a single `index.html`. No build step, no bundler.

- **UI**: React 18 (UMD build from cdnjs) written entirely as `React.createElement(...)` calls — there's no JSX transpilation step, so components are plain functions returning nested `createElement` trees. Styling is Tailwind via the CDN play script, plus a `<style>` block for anything Tailwind can't express (custom keyframes, `::first-letter` drop caps, slider thumbs, etc.).
- **EPUB parsing**: `epub.js` (via a global `window.ePub`) reads spine items, extracts plain text per chapter, strips `<style>`/`<script>` tags, splits it into a flat word array (with a special `PARA_BREAK` sentinel marking paragraph boundaries), and pulls chapter titles from the parsed navigation/TOC matched against spine hrefs.
- **State**: one big function component (`App`). No external state library — everything is `useState`/`useEffect`/`useRef`, with a handful of module-level pure helper functions above the component (word-timing math, color math, template/theme lookups, etc.).
- **Persistence**: two local stores, both scoped entirely to the browser/WebView:
  - **IndexedDB** (`lamplight_local_db`, store `books`, keyed by `id`) — the actual book records: title, author, cover (data URL), word array, chapters, and reading progress (a word index). This is the only place book text lives.
  - **localStorage** — three small JSON blobs: `lamplight_settings_v1` (all user preferences — theme, font, wpm, layout toggles, etc.), `lamplight_stats_v1` (lifetime words read + a per-day `{words, wpmSum}` map for the streak/chart), and one-off flags (`lamplight_intro_seen_v1`, `lamplight_local_seeded_v1`) that gate onboarding and demo-book seeding.
- **Native bridge (optional)**: a few hooks are written defensively for a Capacitor wrapper if one is ever added — haptics (`window.Capacitor?.Plugins?.Haptics`, falling back to `navigator.vibrate`) and a `lamplight-volume` custom window event a native shell could dispatch to bump reading speed from hardware volume buttons. Neither is required for the web app to work; they're just no-ops without a native layer.

## Key files/sections to know if you're editing this

- `THEMES` — the six font-family options (Serif/Mono/Sans/Lexend/Atkinson/Dyslexic) shown in the font picker.
- `READING_PRESETS` / `READING_EXTRAS` — the reading color themes. Presets pair a palette with a "sky" (animated background scene); extras are palette-only and reuse whatever sky is already active.
- `readingThemeTokens()` — turns one theme object into the full token set a component actually consumes (`bg`, `ink`, `pivot`, `chip`, `panel`, etc.). If you add a new theme, you only need `bg`/`ink`/`accent`/`sky` — this function derives the rest.
- `renderRichSegment()` — turns the flat word array back into paragraphs/headings for normal reading mode (drop caps, chapter counters, paragraph breaks).
- `getRsvpChunk()` / `rsvpWordDelay()` — the RSVP timing engine: how many words to flash together, and how long to hold each one (punctuation and word length both adjust the hold time).
- App-level `const`s defined just before the final `return` (e.g. `fontSection`, `sizeSection`, `speedTrackContent`, the `rsvp*Btn`s) — reusable pieces of the Customize panel and RSVP footer, shared between the RSVP and normal-reading layouts so the same JSX isn't duplicated per layout.

## Local development

There's nothing to install. Open `index.html` in a browser, or serve the folder with any static file server. `books-data.js` (referenced but not required to exist) can optionally define `window.CURATED_BOOKS` — an array of pre-bundled starter books offered during onboarding.
