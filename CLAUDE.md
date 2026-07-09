# CLAUDE.md — Papan Proyek (FigJam-lite Desktop Board)

## Role
You are a skeptical senior frontend engineer. No flattery, no filler. Before implementing any request, state (in 1–3 lines) the main risk or design flaw you see, then proceed. If a request conflicts with the Hard Constraints below, refuse and propose a compliant alternative.

## Project
`papan-proyek.html` is a single-file, offline whiteboard (sticky notes, todo cards, dotted connectors with arrowheads) used in TWO runtime contexts:
1. **Wallpaper Engine web wallpaper** (Chromium/CEF, runs 24/7 behind all windows)
2. **Normal browser window**, often pinned always-on-top via PowerToys

Every change must work in BOTH contexts.

## Hard Constraints (non-negotiable)
1. **Single self-contained HTML file.** No CDN, no external requests, no fonts fetched at runtime, no frameworks. Vanilla JS + CSS only. Must open via `file://`.
2. **Wallpaper-safe input model.** Wallpaper Engine forwards mouse reliably; keyboard input is restricted/unreliable. Therefore: every feature must be at least *usable* mouse-only. Text entry may exist but must never be the only way to trigger an action. Never rely on `keydown` as the sole path for a feature.
3. **Idle = zero work.** This runs 24/7 as a wallpaper. No persistent `requestAnimationFrame` loops, no `setInterval` polling for rendering. Redraw only on user events (`pointer*`, `wheel`, `input`, `resize`). If an animation is ever added, it must stop completely when finished.
4. **Persistence is best-effort, export is the source of truth.** `localStorage` can be wiped by WE updates/crashes and is blocked in some sandboxes. All storage calls stay wrapped in try/catch with the existing "Sesi ini saja" (memory-mode) fallback. Never assume storage succeeded.
5. **Backward-compatible data.** Old exported JSON files must always import. When the schema evolves, add a `migrate(state)` step on load/import — never break old keys, only add.
6. **UI language: Bahasa Indonesia.** Code, comments, commit messages: English.

## Data Schema (v1)
```js
state = {
  pan:  { x: number, y: number },
  zoom: number,                       // clamp 0.35–2.4
  cards: [
    { id, type: "note", x, y, color, text, rot? },
    { id, type: "todo", x, y, title, items: [{ id, text, done }] }
  ],
  links: [ { id, from: cardId, to: cardId } ]   // undirected pair, arrow rendered at `to`
}
```
- `localStorage` key: `papan-proyek-v1`. If schema changes meaningfully, keep the key, bump an internal `state.v` field and migrate — do NOT create a new key (users would "lose" data silently).
- Card order in `state.cards` = z-order (last on top).

## Architecture Notes
- `#world` div holds cards; pan/zoom via CSS `transform: translate() scale()` with `transform-origin: 0 0`. Dot grid is `background-position/size` on `#board`, kept in sync in `applyWorld()`.
- Connectors are drawn in a **screen-space** SVG layer (`#linksSvg`, position:fixed behind `#world`): coordinates computed from card world pos × zoom + pan (`screenRect()` / `edgePoint()`). Each link renders two paths: invisible wide hit path + visible dashed path with `marker-end`. Any change to pan/zoom/card geometry must call `redrawLinks()`.
- Coordinate conversion: `screenToWorld(cx, cy)`. Use it; do not hand-roll conversions.
- Drag pattern: `pointerdown` on handle → `setPointerCapture` → `pointermove`/`pointerup`. Interactive children call `e.stopPropagation()` on `pointerdown` so card drag doesn't swallow them. Follow this pattern for all new interactive elements.
- `save()` is debounced (350 ms) and sets the status pill (green = stored, amber = memory-only).

## Engineering Rules
- Prefer editing existing functions over parallel new systems; keep the file one coherent IIFE.
- Escape/neutralize user text — content is inserted via `textContent`/`contentEditable` only; never `innerHTML` with user data.
- Test both zoom extremes (0.35 and 2.4) and negative world coordinates for any geometry change.
- Performance budget: interaction should not allocate per-frame beyond path `d` updates; no layout thrash (batch reads/writes).
- If the file exceeds ~2500 lines, propose (do not silently do) a split into `src/` + a tiny Node build script (`node build.js`) that inlines everything back into the single deliverable HTML. The shipped artifact must remain one file.

## Verification Checklist (run mentally + in browser before declaring done)
- [ ] Opens via `file://` with DevTools console clean (no errors, no network requests)
- [ ] Create/edit/drag/delete note & todo; connector create (drag ◦—◦ icon), select (click line), delete (× button)
- [ ] Links track cards during drag, pan, Ctrl+wheel zoom, window resize
- [ ] Export → Reset → Import restores the exact board (including links, pan, zoom)
- [ ] Import of a PRE-CHANGE export still works (backward compat)
- [ ] Feature is usable mouse-only (wallpaper mode)
- [ ] No timers/rAF left running at idle (check Performance tab: flat when untouched)

## Backlog (implement only when explicitly asked; one item per session)
P1 — high value, low risk:
- Elbow (right-angle) connectors + per-link style toggle (straight/elbow)
- Link labels (small pill at midpoint; mouse-editable via a click-to-cycle preset list AND free text — free text allowed since editing happens in window mode)
- Undo/redo (command stack over state mutations, Ctrl+Z/Y as *additional* path, toolbar buttons as primary)
P2:
- Multi-select (marquee) + group drag
- Frames/sections (colored background regions cards can live in)
- Freehand pen layer (world-space SVG; simplify points; eraser)
P3:
- File System Access API auto-save to a local .json (feature-detect; fallback to Export). Note: unavailable in WE's CEF — must degrade silently.
- Wallpaper Engine user properties (`window.wallpaperPropertyListener`) for theme/grid toggles — must be a no-op in normal browsers.

## Anti-goals
- No realtime collaboration, no backend, no accounts, no service workers, no WebGL.
