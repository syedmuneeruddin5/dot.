# dot. — Product Requirements Document

*Version 1.0 · Audience: AI coding agent · First target: Windows 11 (macOS and Linux later)*

## Overview

**dot.** is a tiny to-do list that lives in the background and appears at the press of a shortcut. It shows only today's tasks in a small, dark, frameless window, and disappears the moment you are done.

**What it is.** A desktop utility, not a productivity suite. One list per day, edited entirely from the keyboard, saved as you type, and reset each day at 4:00 AM. Older days are kept as history.

**Who it is for.** First, its owner: someone who plans the day as a short list, works keyboard-first, sometimes works past midnight, and wants a tool that never competes for attention. More broadly, anyone on Windows 11 who finds full task managers heavy and wants something that gets out of the way. macOS and Linux follow later.

**What it aims to do.**

- Open instantly and stay out of the way, with very low idle memory.
- Remove every distraction, so the only thing on screen is the list.
- Feel polished enough to be a pleasure to use: calm, dark, sleek and smoothly animated.
- Never lose a task.

**Inspiration.**

- **Notion** for the editing philosophy: `Enter` starts the next item, `Backspace` on an empty item removes it, the arrow keys move around, and there are no buttons where context or the keyboard already makes the action clear.
- **Linear and Raycast** for the dark, minimal, fast, summon-with-a-shortcut feel.
- **macOS apps** for translucency (acrylic blur on Windows) and attention to detail.

**Technology in one line.** Tauri v2 (Rust) with a React frontend, chosen for fast startup and low idle RAM.

**What this document covers.** Goals and features (§1–4), exact behavior (§5), technical decisions (§6), UI/UX guidelines (§7), risks (§8), and the phased build plan (§9), which the coding agent follows one phase at a time.

## 0. How to use this document

Rules for the coding agent:

1. Build **one phase at a time** (§9). At the end of each phase: stop, summarize what was built, explain how to run it, list anything unresolved, and **wait for the owner's approval**. Never start the next phase on your own.
2. Do not add features, settings or dependencies outside the current phase.
3. Before using any library, read its current official docs and install the latest stable version. APIs have changed recently (Tauri v2, shadcn/ui now defaulting to Base UI, Phosphor icon components now ending in `Icon`). Do not rely on memory.
4. Prefer existing, well-maintained libraries over hand-written equivalents (§6.1). Do not rebuild what a chosen library already provides.
5. The visual design is **decided together with the owner** in Phase 3. Do not invent a design system silently. This document gives goals and guidelines only (§7).
6. If something is ambiguous, ask. Items marked **\[Confirm\]** must be confirmed with the owner when you reach them.
7. Keep the repo runnable at the end of every phase, with one commit per phase. Log meaningfully (§6.9).

## 1. Summary

**dot.** is a minimal, frameless desktop to-do list. Press a global shortcut (default `Alt+Shift+N`) and a small dark window appears showing today's tasks. Type, check things off, press `Esc`, and it's gone. It is a utility, not a productivity suite: no accounts, no sync, no notifications, no clutter. The only thing on screen is today's list.

Each day starts fresh at **4:00 AM** (late-night work counts toward the previous day). Older lists are kept as history. Everything is saved as you type.

Built with **Tauri v2 (Rust) + React** for fast startup and low idle memory. The editing feel follows Notion: the keyboard does everything, and there are no buttons for things the keyboard or context can make obvious.

## 2. Goals

**User goals**

- Capture and check off tasks in seconds, entirely from the keyboard.
- Zero distraction. Nothing but the list, so attention stays on the tasks and not the app.
- A fresh list each day, with late nights belonging to the previous day.
- Trust: everything saves as I type, and a task is never lost.
- Peek at the last week and pull a past list into today.

**Product goals** (numeric targets confirmed after Phase 0 measurements)

- Hotkey-to-visible feels instant. Aspiration: under \~50 ms warm, under \~300 ms cold.
- Minimal idle memory. In cold mode, only a tiny Rust process stays resident.
- A premium feel: calm, polished, animated, comparable to Raycast, Linear and Notion.

## 3. Features

**3.1 Core (v0.1, Phases 1–6)**

- Frameless, fixed-size dark window opened and closed by a global shortcut; `Esc` or clicking outside dismisses it.
- "Today" heading above the list.
- Notion-style editing: `Enter` splits and moves down, `Backspace` on an empty item removes it, arrow keys move between items, wrapped text.
- Check/uncheck, move item up/down, undo/redo.
- Autosave as you type; daily reset at 4:00 AM into history storage (30 days kept).
- Config file, terminal logging flag, tray icon toggle, launch-at-login toggle, run modes.
- Empty state: one unchecked box with the caret next to it.

**3.2 Post-core (added one at a time, Phases 7–10)**

1. Acrylic blur background.
2. Checked items move to the bottom (of their priority group once priority exists).
3. Three-level priority (red, yellow, green), shown as a faint tint and slightly larger gaps between groups.
4. History view (today plus 6 previous days) with copy-to-today.

**3.3 Future (not built now)** Settings window, drag-to-reorder, per-item timer. Architecture and data must not block them (stable item IDs, versioned file schema, configurable keybindings).

## 4. Out of scope

Cloud sync, accounts, mobile, sub-tasks or nesting, due dates, reminders, notifications, tags, search, rich text or markdown, multiple lists, light theme, telemetry or analytics, auto-update, installer signing and distribution, macOS and Linux builds (only keep the code ready for them), and anything listed under 3.3.

## 5. Behavior specification

**5.1 Keyboard**

| Input | Behavior |
| --- | --- |
| `Alt+Shift+N` (global, configurable) | Toggle the window |
| `Esc` or focus lost (click outside) | Hide the window |
| `Enter` | Split the item at the caret. Text after the caret moves into a new item below, caret at its start. At the end of an item, creates an empty item. New item inherits the priority of the item above (once priority exists) |
| `Backspace` on an empty item | Delete it, caret to the end of the previous item. If it is the first item, caret goes to the start of the next. The last remaining item is never deleted |
| `Backspace` at the start of a non-empty item | Merge into the previous item (Notion behavior) **\[Confirm\]** |
| `Up` / `Down` | Move the caret within wrapped lines first, then to the previous or next item |
| `Alt+Space` (configurable) | Check/uncheck the focused item. Owner-chosen. See risk R4 |
| `Alt+Up` / `Alt+Down` | Move the focused item up or down |
| `Ctrl+Z` / `Ctrl+Shift+Z` or `Ctrl+Y` | Undo / redo |
| Paste | Plain text only. Multi-line paste becomes multiple items **\[Confirm\]** |

**5.2 List rules**

- Items wrap onto multiple lines; no length limit enforced.
- On open, the caret goes to the end of the last item (or the lone empty item) **\[Confirm\]**.
- Empty state: a single unchecked box with the caret beside it. No "add" button, no placeholder illustration.
- Checked items show as checked and dimmed with a strikethrough, as in the mockup.

**5.3 Window rules**

- Frameless, fixed size from config, centered horizontally, vertically slightly above center (configurable anchor; tune by eye in Phase 1).
- Shown on the monitor containing the mouse cursor. Not in the taskbar, always on top while open.
- Clicking the space around the window, or `Esc`, dismisses it. The exit animation must not delay hiding noticeably.

**5.4 Logical day (4:00 AM rollover)**

- logical\_day = the calendar date of (local now − day\_start\_hour). Default `day_start_hour = 4`.
- It is computed **only when the window opens** (and at startup). No timers. Closed laptops and sleep need no handling.
- If the logical day changed since the last open, flush the old day, then load the new day (empty if no file). The old list simply becomes history. Nothing is deleted at rollover.
- If the window is open when 4:00 passes, it stays on the old list until it is next opened.
- Both checked and unchecked items are kept in history.

**5.5 Ordering (after Phases 8–9)** Groups top to bottom: red (high), yellow, green, then no priority. Within each group, manual order, with checked items at the bottom of that group. Groups are not labeled by headings. A slightly larger gap separates groups.

**5.6 History (Phase 10)**

- Shows today plus the 6 previous logical days as cards. Each card shows the day name (except today and yesterday) and a gradient-blurred peek at its items.
- Opening a card shows the same list UI, read-mostly. A copy icon (same style as the history icon) copies all items (checked and unchecked, with their state) to today's list, then switches to today **\[Confirm state handling and whether empty days appear\]**.
- 30 days are kept on disk. Older files are pruned at startup and on open.

## 6. Technical decisions and architecture

**6.1 Stack**

| Layer | Choice |
| --- | --- |
| Shell | Tauri v2, Rust. Built-in window effects (acrylic) and official plugins: global-shortcut, single-instance, autostart, log |
| Frontend | React + TypeScript + Vite (SPA, no Next.js), pnpm |
| Styling / primitives | Tailwind CSS v4, shadcn/ui on its Base UI flavor. Only for pieces we need (checkbox, tooltip, scroll area; later switch, select, tabs, kbd). Heavily restyled to the agreed design |
| Animation | Motion (formerly Framer Motion) |
| Icons | `@phosphor-icons/react`. Import each icon individually; in dev import from per-icon file paths to avoid transpiling thousands of icons |
| Font | Inter, bundled locally (no network requests), final choice agreed in Phase 3 |
| Editor | No editor framework (no Tiptap, Lexical, ProseMirror, BlockNote). One auto-growing `<textarea>` per item with custom focus management. Use CSS `field-sizing: content` where supported, with a small fallback for other webviews later |
| State | Zustand. App-level undo stack (own implementation, small) |
| Validation | Zod for file schemas |
| Drag (Phase 11+) | dnd-kit or Pragmatic drag-and-drop, decided at that time |
| Tooling | Vitest for frontend logic, Rust unit tests for day logic and storage, Biome for lint and format |

**6.2 Responsibilities**

- **Rust owns:** window lifecycle, global shortcut, tray, config, storage, logical-day logic, logging, single-instance, autostart, CLI flags.
- **Frontend owns:** rendering, keyboard interaction, animations, undo/redo.
- **Communication:** Tauri commands (`get_today`, `save_day`, `get_history`, `copy_day_to_today`) and events (`window-shown`, `request-hide`). Hide handshake: Rust emits `request-hide`, the frontend plays the exit animation, then calls `confirm_hide`. Rust hides anyway after a short timeout (\~250 ms) if no confirm arrives.

**6.3 Window setup** Frameless, transparent, non-resizable, skip-taskbar, always-on-top, created hidden. Create windows from async Rust code, not from synchronous commands (deadlock risk on Windows). When acrylic is added, html and body must both be transparent. Fixed window size is deliberate: window resizing lags with acrylic, so the list scrolls inside a fixed window.

**6.4 Run modes** (config `memory_mode`)

- `warm`: webview created at startup and kept hidden. Fastest show; highest idle RAM.
- `cold`: only the Rust process, tray and shortcut listener stay alive. The window and webview are created on each hotkey press and destroyed on dismiss. Lowest idle RAM; slowest show.
- `hybrid`: warm, but while hidden WebView2's memory usage target is set to low (reduces memory, best effort), with an optional destroy-after-N-idle-minutes.
- The default mode is chosen from Phase 0 measurements. Closing the window never exits the app.

**6.5 Global shortcut** Default `Alt+Shift+N`, configurable. Registration can fail silently when another app owns the combination, so detect and log failure and surface it (tray notification when a tray exists). Alt+Shift is also Windows' keyboard-layout switch shortcut; Phase 0 tests for side effects. Launching `dot.exe` a second time always shows the window.

**6.6 Storage and data model**

- Location: `%APPDATA%\dot\days\YYYY-MM-DD.json`, one file per logical day.
- Writes: debounced (\~250 ms), atomic (write temp file, then rename), flushed on hide and on exit. A corrupt file is renamed `.corrupt-<timestamp>`, logged, and the app continues with an empty list. The app never crashes on bad data.
- Single-instance protection prevents two processes writing the same files.

```json
{
  "schemaVersion": 1,
  "day": "2026-11-19",
  "items": [
    { "id": "uuid", "text": "...", "done": false, "priority": 0, "createdAt": "ISO", "updatedAt": "ISO" }
  ]
}
```

Array order is the manual order. `priority` is 0 (none) to 3 and exists from v1 so Phase 9 needs no migration.

**6.7 Undo** Native textarea undo cannot span items. Implement one app-level history of list snapshots covering edits, splits, merges, deletes, checks and moves. Typing is grouped into chunks (pause or boundary based). Undo history clears when the day changes.

**6.8 Config file** (`%APPDATA%\dot\config.toml`, created with commented defaults if missing, read at startup; restart to apply during development)

| Key | Default |
| --- | --- |
| `window.width`, `window.height` | set in Phase 1 |
| `window.vertical_anchor` | tuned in Phase 1 |
| `shortcut.toggle` | `Alt+Shift+N` |
| `keybindings.toggle_done`, `keybindings.move_up`, `keybindings.move_down` | `Alt+Space`, `Alt+Up`, `Alt+Down` |
| `behavior.memory_mode` | from Phase 0 |
| `behavior.hybrid_idle_destroy_minutes` | e.g. 10, `0` = never |
| `behavior.day_start_hour` | `4` |
| `behavior.history_retention_days` | `30` |
| `ui.show_tray_icon` | `true` |
| `startup.launch_at_login` | `true` |
| `logging.level` | `info` |

**6.9 Logging and CLI**

- `tauri-plugin-log` with terminal and file targets.
- `dot.exe --debug` or `--log <trace|debug|info|warn|error>` prints logs to the terminal that launched it. On Windows, release builds are windowed with no console, so when these flags are present the app must attach to the parent console. Verify this on a release build, not only in dev.
- `dot.exe --quit` stops the running instance (via single-instance).
- Log: startup, config values used, shortcut registration result, show/hide with timings, mode changes, day rollover, save/load results, and errors.

**6.10 Tray, autostart, single instance** Tray icon (toggle `ui.show_tray_icon`): Open, Quit, Settings (disabled until later). With the tray off the app has no visible exit; that is accepted, and `--quit` and the config file remain. Autostart via the official plugin, toggled by `startup.launch_at_login`.

**6.11 Cross-platform readiness** Keep Windows-specific code in a `platform` module behind a small interface (window effects, focus quirks, console attach). Core logic must not depend on Windows.

**6.12 Performance rules** Release profile tuned for size and speed (LTO, single codegen unit, strip, panic abort). No network requests at all. No background timers or polling. Minimal dependencies. Fonts and icons bundled and tree-shaken. Measure RAM across all processes (the app plus the msedgewebview2 processes), not just the main one.

## 7. UI/UX guidelines

These are intent, not a spec. Concrete values are decided with the owner in Phase 3 and recorded in a living `DESIGN.md`.

- **Feel:** clean, dark, quiet and sleek, in the family of Notion, Linear, Raycast and well-made macOS apps. Premium comes from restraint and consistency, not decoration.
- **Reference:** the attached mockup (dark rounded card, thin-outline checkboxes, generous row spacing, dimmed strikethrough for done items). It is a starting point; the "Today" heading is new.
- **No chrome:** nothing exists on screen that isn't the list. Actions appear through context (hover, caret) and keyboard. Hidden affordances are dim until hovered, then reach full contrast.
- **Motion:** purposeful, quick, and natural (spring-like where it fits). Covers show/hide, item add/remove, check, move, priority tint. Never blocks typing, never feels slow, and respects reduced-motion settings.
- **Consistency:** one source of truth for colors, spacing, radius, type and motion. No one-off values scattered in components.
- **Accessibility basics:** readable contrast, visible keyboard focus where it matters, proper labels on controls.
- **Process:** in Phase 3 the agent proposes 2–3 visual directions as quick previews, the owner picks and refines, and decisions are written to `DESIGN.md`. Later phases follow it.

## 8. Risks and how phases address them

| ID | Risk | Plan |
| --- | --- | --- |
| R1 | Acrylic with rounded corners may show a square blurred backdrop. Acrylic can lag on move/resize in some Windows builds | Test in Phase 0; fixed size; fallback options decided with owner |
| R2 | Window may fail to take keyboard focus on hotkey | Test in Phase 0 across foreground apps; fix in Phase 1 |
| R3 | `Alt+Shift+N` may fail to register or trigger a layout switch | Phase 0 test; failure surfaced; configurable |
| R4 | `Alt+Space` normally opens the Windows window menu, and another app (e.g. a launcher) may own it globally and fire instead | Phase 0 test; suppress the system menu natively if needed; the binding is configurable. The earlier candidates were `Shift+Space` (misfires when typing capitals) and `Ctrl+Enter` |
| R5 | Cold start may be too slow, warm may use too much RAM | Phase 0 numbers; three modes; owner decides default |
| R6 | Release builds show no terminal logs | Console attach for debug flags, verified on release build |
| R7 | Data loss | Atomic writes, flush on hide/exit, corrupt-file handling, tests |

## 9. Build phases

Each phase ends with an owner checkpoint. Do not continue until approved.

### Phase 0 — Feasibility spike (throwaway)

**Goal:** replace assumptions with numbers before building. **Build:** a minimal Tauri v2 + React app in `spike/`, release build, testing:

1. Transparent frameless window with Tauri's built-in acrylic. Report how rounded corners look, and behavior on this machine.
2. Global shortcut: `Alt+Shift+N` registration, failure detection, any keyboard-layout side effects. Also check whether `Alt+Space` inside the window opens the system menu.
3. Keyboard focus after hotkey, from several foreground apps.
4. Warm, warm with low WebView2 memory target, and cold (create/destroy) modes: hotkey-to-first-paint time and total idle RAM across all processes.
5. Auto-growing wrapped `<textarea>` using `field-sizing: content`; caret behavior.
6. Release-build terminal logging with `--debug`. **Deliver:** `SPIKE-REPORT.md` with a results table, problems found, and recommended defaults (memory mode, window position). The spike code may be discarded. **Owner checks:** the numbers; decides mode default and any shortcut change.

### Phase 1 — Foundation and the shortcut window

**Goal:** the real project skeleton, and a window that appears and disappears correctly. **Build:** project scaffold (Tauri v2, Vite, React, TS, Tailwind, shadcn/Base UI, Biome, Vitest); platform module; config loader with commented defaults; logging with `--debug` and `--log`; Rust-created hidden window; global shortcut toggle; `Esc` and blur dismissal; fixed size, position and monitor logic from config; focus fix from Phase 0 findings. The UI is just a dark rounded rectangle with "Today". Warm mode only. **Out of scope:** list, persistence, tray, other modes. **Owner checks:** hotkey opens a centered dark window on the right monitor; `Esc` and click-outside close it; changing config size and anchor and restarting works; logs appear in the terminal from a release build.

### Phase 2 — Run modes and system integration

**Goal:** the lightweight-runtime decisions become real. **Build:** `cold` and `hybrid` modes alongside `warm` (config switch); single-instance (second launch shows the window); `--quit`; tray icon with Open and Quit, toggleable; launch-at-login toggle; shortcut registration failure handling; log timing for every show/hide. **Owner checks:** switch modes and compare feel and RAM against the spike; tray on/off; second launch behaves; logs show timings.

### Phase 3 — Design direction and visual shell

**Goal:** the owner approves the look before any logic exists. **Build:** a design discussion (§7), then the static UI with hardcoded sample items matching the mockup, the "Today" heading, empty state, and the show/hide animations. `DESIGN.md` records decisions. No editing, no persistence. **Owner checks:** look and feel, typography, motion, empty state. Expect refinement rounds here; this is the moment to iterate.

### Phase 4 — Editing engine (in memory)

**Goal:** Notion-style editing feels right. **Build:** Zustand list state; one textarea per item with auto-grow wrapping; `Enter` split, `Backspace` rules, arrow navigation across wrapped lines and items, focus on open, check toggle via configurable keybinding, paste handling, empty-state behavior. Data is lost on app restart at this stage. **Owner checks:** type, split, delete, navigate, wrap, check. Confirm the **\[Confirm\]** items in §5.

### Phase 5 — Persistence and logical days

**Goal:** nothing is ever lost, and days roll over correctly. **Build:** Rust storage per §6.6 (atomic debounced saves, flush on hide/exit, corrupt-file handling); commands and events; logical-day computation on open and startup; history files kept 30 days with pruning; Zod validation and `schemaVersion`; unit tests for day logic (boundaries at 03:59, 04:00, midnight, month and year ends) and storage. **Owner checks:** type, kill the app, restart, data is there. Change the system clock or the `day_start_hour` setting to verify rollover. Files appear as expected.

### Phase 6 — Undo, redo and move

**Goal:** safe, fluid editing. Finishes v0.1. **Build:** app-level undo/redo (§6.7); `Alt+Up/Down` item move; animations for add, remove and move; edge-case pass on focus and caret. **Owner checks:** undo across deletes and splits; moving items; animation polish. **v0.1 milestone: daily-usable.**

### Phase 7 — Acrylic background

**Goal:** the translucent look, with no performance regression. **Build:** Tauri's window effects (acrylic) with a transparent webview; handle rounded-corner outcome as decided in Phase 0 and the owner's preference; graceful fallback when unsupported. Re-measure show latency and RAM. **Owner checks:** look on varied wallpapers; no lag on open; numbers still acceptable.

### Phase 8 — Checked items move to the bottom

**Goal:** done items sink out of the way. **Build:** a checked item animates to the bottom of the list; unchecking returns it to a sensible position **\[Confirm\]**; undo works with it; manual move interacts sensibly with it. **Owner checks:** the motion, and the edge cases of check, uncheck and undo.

### Phase 9 — Priority

**Goal:** minimal three-level priority with no extra chrome. **Build:** a colored circle at the right of the item's line, hidden until the row is hovered or has the caret, dim until the circle itself is hovered; click cycles none, red, yellow, green (cycle order **\[Confirm\]**); faint row tint; larger gaps between groups; ordering per §5.5 (checked items now sink within their group); `Enter` inheritance; undo support. **Owner checks:** discoverability without clutter, tint strength, group spacing, behavior with checked items.

### Phase 10 — History

**Goal:** look back a week and reuse a past list. **Build:** a history icon near the top right, dim and appearing only when the mouse is near the top; history view of today plus 6 days as cards with day names and blurred gradient peeks; open a card to see its list; copy-to-today icon; keyboard navigation and `Esc` behavior. Resolve the **\[Confirm\]** items in §5.6. **Owner checks:** card visuals, copying, switching back to today, performance of opening history.

### Phase 11+ — Future (separate planning)

Settings window, drag-to-reorder, per-item timer, and cross-platform builds. Each gets its own short PRD after v1 is in use.
