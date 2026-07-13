# Papan Proyek (FigJam-lite Desktop Board)

Welcome to the **Papan Proyek** developer guide. This document serves as the primary system context, architectural reference, and guidelines for AI coding assistants (such as Cursor, Windsurf, or Copilot) to continue development without violating constraints.

---

## 1. Project Context & Runtimes

This project is a single-file, offline interactive whiteboard dashboard containing sticky notes, task checklists (To-Do cards), and smart reminders with custom date/time alerts and recurrence.

The application operates in **two distinct runtime environments**:
1. **Wallpaper Engine / Lively Wallpaper Web Wallpaper**: Runs 24/7 as an active desktop background inside a headless Chromium Embedded Framework (CEF).
2. **Standard Web Browser**: Often pinned always-on-top using tools like PowerToys.

### Critical CEF (Wallpaper) Constraints
- **Unreliable Keyboard Focus**: Keyboard input is highly restricted in desktop wallpaper mode. Every single feature must be fully usable with mouse clicks/pointer events alone.
- **Native Popups Blocked**: Native elements like `<input type="datetime-local">` calendar popups or native `<select>` dropdowns fail to capture focus or render in CEF because they spawn OS-level sub-windows. **All dropdowns and pickers must be custom DOM-based overlays.**
- **No Background Idle Work**: This web page runs 24/7. Continuous timers (`setInterval`, persistent `requestAnimationFrame` loops) will drain laptop batteries and CPU cycles. Redrawing must occur **only** on direct user events (`pointerdown`, `wheel`, `input`, `resize`).

---

## 2. Hard Architectural Constraints

1. **Single File Deliverable**: The entire application lives inside [index.html](file:///c:/000%20DATA%20RZ/WEB%20DEVELOPMENT%20RZ/MY%20PROJECT/dekstop%20map%20project/index.html). No external CSS, no external JS scripts, no external fonts, and no CDN calls.
2. **Offline-First / Zero Network**: Must open and run perfectly via `file:///` protocol without any internet connection.
3. **CEF Click-Propagation Safeguard**: All interactive elements (inputs, contenteditable divs, checkboxes, custom dropdowns) must implement `.addEventListener("pointerdown", e => e.stopPropagation())` to prevent the parent card drag routine or board pan routine from swallowing clicks.
4. **UI Language**: Bahasa Indonesia. Code variables, comments, and commit messages: English.
5. **Sanitization Protocol**: User inputs and contenteditable text must be sanitized before saving using `sanitizeHTML()`. We allow only basic styling tags (`B`, `I`, `U`, `STRIKE`, `SPAN`, `STRONG`, `EM`, `BR`, `DIV`, `P`, `UL`, `OL`, `LI`) and preserve specific styling attributes like `type` for `<ol>`.

---

## 3. Data Schema & Persistence

State is saved to `localStorage` under the key `papan-proyek-v1`. If the schema changes, use a migration step on startup rather than changing the storage key.

```typescript
interface BoardState {
  pan: { x: number; y: number };
  zoom: number; // Clamped strictly between 0.35 and 2.4
  cards: Array<NoteCard | TodoCard | ReminderCard>;
  links: Array<{ id: string; from: string; to: string }>;
}

interface NoteCard {
  id: string;
  type: "note";
  x: number;
  y: number;
  w?: number;
  h?: number;
  color: string; // Hex color code from allowed palette
  text: string;  // HTML content (sanitized)
  rot?: number;  // Optional rotation angle
}

interface TodoCard {
  id: string;
  type: "todo";
  x: number;
  y: number;
  w?: number;
  h?: number;
  title: string;
  items: Array<{
    id: string;
    text: string;
    done: boolean;
    priority?: "urgent" | "medium" | "low" | "archive"; // urgent=red, medium=green, low=blue, archive=gray
  }>;
}

interface ReminderCard {
  id: string;
  type: "reminder";
  x: number;
  y: number;
  w?: number;
  h?: number;
  title: string;
  items: Array<{
    id: string;
    text: string;
    category: "Langganan" | "Jadwal" | "Umum";
    recurrence: "none" | "daily" | "weekly" | "monthly";
    recurDays: number[]; // Array of weekday index (0=Sunday, ..., 6=Saturday)
    datetime: string;    // ISO-8601 string (e.g. YYYY-MM-DDTHH:MM)
    done: boolean;
  }>;
}
```

---

## 4. Key Architectural Subsystems

### Coordinate Conversions (World vs. Screen Space)
The board `#board` is a viewport wrapper. The canvas elements reside in `#world`, which translates and scales via CSS transforms.
- **World coordinates**: Virtual placement of cards.
- **Screen coordinates**: Physical pixel locations relative to the browser window viewport.
- Conversion math helper:
  `screenToWorld(cx, cy)` computes coordinates based on the current pan offset and zoom scale factor.

### Connector Links Layer
- Links are drawn on a screen-space SVG layer (`#linksSvg`) positioned behind the cards.
- Coordinates for connector lines are calculated at runtime (`screenRect()` / `edgePoint()`) by scaling card positions with current zoom/pan factors.
- Redrawing (`redrawLinks()`) must trigger on card drag, zoom, pan, or window resizing.

---

## 5. Custom UI Components (CEF-Bypass Widgets)

### A. Custom Dropdown (`createCustomDropdown`)
Replaces native HTML `<select>`. Combines a display header with a position-absolute custom menu that opens on `pointerdown`.
- **Method**: `getValue()` returns the selected value.
- **Action**: Handles `pointerdown` capture & body clicks to dismiss automatically.

### B. Custom DateTime Picker (`createCustomDateTimePicker`)
Replaces `<input type="datetime-local">` with an interactive, dark-themed calendar grid and clock selection panel built purely out of standard DOM elements.
- Allows jumping months and typing hour/minute values.
- Triggers custom `onApply` callback once saved.

### C. To-Do Item Priority Dot
- Positioned inside each todo list row.
- Cycles priority colors on click: Red (`urgent`) $\rightarrow$ Green (`medium`) $\rightarrow$ Blue (`low`) $\rightarrow$ Gray (`archive`) $\rightarrow$ Red (`urgent`), and automatically sorts items.

### D. Automated Markdown Lists
- When typing inside a note card's body, the keydown handler automatically detects:
  - `- ` or `* ` followed by space $\rightarrow$ Converts block to a Bullet List (`<ul>`).
  - `1. ` followed by space $\rightarrow$ Converts block to a Numbered List (`<ol>`).
  - `a. ` or `A. ` followed by space $\rightarrow$ Converts block to an Alphabetical List (`<ol type="a">`).

---

## 6. How to Edit and Extend the Code

When implementing new features, remember these guidelines:
1. **Maintain Single-file Coherence**: Write new CSS styles in `<style>` blocks in `index.html` and logic within the main IIFE function.
2. **Verify User Event Constraints**: Ensure all interactions are single-click friendly. Test using pointer events.
3. **Avoid layout thrashing**: Perform reads of offset dimensions sequentially before writing styling updates.
4. **Sanitize Everything**: Never write user-provided string content using `.innerHTML` without running it through `sanitizeHTML()`. Use `.textContent` for safe plain text strings.
