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

1. **Single File Deliverable**: The entire application lives inside `index.html`. No external CSS, no external JS scripts, no external fonts, and no CDN calls.
2. **Offline-First / Zero Network**: Must open and run perfectly via `file:///` protocol without any internet connection.
3. **CEF Click-Propagation Safeguard**: All interactive elements (inputs, contenteditable divs, checkboxes, custom dropdowns) must implement `.addEventListener("pointerdown", e => e.stopPropagation())` to prevent the parent card drag routine or board pan routine from swallowing clicks.
4. **UI Language**: Bahasa Indonesia. Code variables, comments, and commit messages: English.
5. **Sanitization Protocol**: User inputs and contenteditable text must be sanitized before saving using `sanitizeHTML()`. We allow only basic styling tags (`B`, `I`, `U`, `STRIKE`, `SPAN`, `STRONG`, `EM`, `BR`, `DIV`, `P`, `UL`, `OL`, `LI`) and preserve specific styling attributes like `type` for `<ol>`.

---

## 3. Data Schema & Persistence

Workspace state is saved to `localStorage` under the key `papan-proyek-v1`. The key stays the same for backward compatibility; schema evolution is handled by `migrate()` on startup/import.

- Current schema version: `v: 4`.
- Legacy exports with root-level `cards`, `links`, `pan`, and `zoom` are migrated into one board.
- Export/Import now includes all boards in the workspace.
- `Reset` clears only the active board, not the entire workspace.
- Each board owns its own `pan`, `zoom`, cards, links, lock state, and blur state. Switching boards must never reuse another board's viewport transform.

```typescript
interface WorkspaceState {
  v: 4;
  activeBoardId: string;
  boards: BoardState[];
}

interface BoardState {
  id: string;
  name: string;
  v?: number;
  pan: { x: number; y: number };
  zoom: number; // Clamped strictly between 0.35 and 2.4
  locked: boolean;
  blurred: boolean;
  cards: Array<NoteCard | TodoCard | ReminderCard | ImageCard>;
  links: Array<{ id: string; from: string; to: string; style?: "straight" | "elbow" }>;
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
    recurDay?: number;   // Monthly anchor day (1-31), clamped to the last day of shorter months
    datetime: string;    // ISO-8601 string (e.g. YYYY-MM-DDTHH:MM)
    done: boolean;
  }>;
}

interface ImageCard {
  id: string;
  type: "image";
  x: number;
  y: number;
  w: number;
  h: number;
  name: string;
  src: string; // Sanitized offline data URL: PNG, JPEG, WebP, GIF, or BMP
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

### C. Board Switcher
Provides mouse-only controls for multiple local boards:
- Previous board (`‹`) and next board (`›`) switch the active board.
- `+ Papan` appends a blank board and switches to it.
- The active board label shows only its position, for example `1/2`.
- Board switching must call `renderAll()` after updating `activeBoardId`, so each board restores its own pan/zoom and card positions.

### D. Offline Images
- Clicking `Gambar` inserts uploads into the currently focused note; without an active note it creates a separate image card on the board.
- Pasting an image inside a note inserts it inline. Pasting while no editable field is active creates an image card.
- Click an inline note image to reveal four corner handles, then drag a corner to resize it while keeping its aspect ratio. The width is persisted with the note.
- Images are downscaled and converted locally before storage. SVG is rejected because it can contain active or external content.
- Images still increase the JSON and `localStorage` size. Automatic storage remains best-effort; regular Export backups are mandatory.

### E. To-Do Item Priority Dot
- Positioned inside each todo list row.
- Opens a mouse-only popover for direct priority selection: Red (`urgent`), Green (`medium`), Blue (`low`), or Gray (`archive`), then automatically sorts items.
- Completed items are automatically moved below active items and remain crossed out.

### F. Automated Markdown Lists
- When typing inside a note card's body, the keydown handler automatically detects:
  - `- ` or `* ` followed by space $\rightarrow$ Converts block to a Bullet List (`<ul>`).
  - `1. ` followed by space $\rightarrow$ Converts block to a Numbered List (`<ol>`).
  - `a. ` or `A. ` followed by space $\rightarrow$ Converts block to an Alphabetical List (`<ol type="a">`).

---

## 6. Update Wallpaper Tanpa Kehilangan Data

GitHub menyimpan kode, bukan isi board yang tersimpan di `localStorage`. Sebelum mengganti wallpaper atau memasang versi baru:

1. Buka versi lama, klik `Export`, lalu simpan file JSON di luar folder wallpaper.
2. Jangan hapus folder lama sebelum file JSON dapat dibuka dan ukurannya masuk akal.
3. Pasang versi baru. Jika dijalankan dari lokasi/origin yang sama, aplikasi akan membaca key `papan-proyek-v1` dan melakukan migrasi otomatis.
4. Jika board kosong karena lokasi/origin berubah, klik `Import` dan pilih file JSON tadi.
5. Setelah semua board, reminder, To-do, dan gambar diperiksa, lakukan `Export` lagi sebagai backup baru.

## 7. How to Edit and Extend the Code

When implementing new features, remember these guidelines:
1. **Maintain Single-file Coherence**: Write new CSS styles in `<style>` blocks in `index.html` and logic within the main IIFE function.
2. **Verify User Event Constraints**: Ensure all interactions are single-click friendly. Test using pointer events.
3. **Avoid layout thrashing**: Perform reads of offset dimensions sequentially before writing styling updates.
4. **Sanitize Everything**: Never write user-provided string content using `.innerHTML` without running it through `sanitizeHTML()`. Use `.textContent` for safe plain text strings.
