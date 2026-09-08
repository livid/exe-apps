# exe-apps

Apps for the [exe](https://github.com/livid/exe) desktop — the Mac OS 9
Platinum desktop that exe, a personal VM cloud, serves from one Go binary.
Each top-level folder here is one app bundle, served live from this
checkout by the exe daemon: edit a file, reopen the window, no build step.
An app is a single `index.html` with vanilla JS and inline CSS, which makes
it the kind of thing to ask a coding agent to build for you.

## How it connects

The exe daemon scans `~/.exe/apps` plus every folder listed in `apps_dirs`
in `~/.exe/config.json`:

```json
"apps_dirs": ["~/Developer/exe-apps"]
```

(also editable in the desktop's Configuration window, under Daemon). Every
valid bundle shows up as a desktop icon and in the desktop menu's app list;
`~/.exe/apps` wins name collisions with this repo.

## Bundle layout

```
AppName/            folder name = app identity (^[A-Za-z0-9][A-Za-z0-9 ._-]*$)
  app.json          { "title", "icon": "icon.svg", "window": { "width", "height", "grow": true } }
  index.html        the whole app, opened in an iframe by the desktop
  icon.svg          32×32 pixel-art desktop icon (render at exactly 32 CSS px)
```

The desktop opens `index.html` in an iframe at `/apps/AppName/` with the
API token in `?token=`, and on a phone `?mobile=1` so the app can size its
chrome for one. `"grow": false` is for a window whose content dictates its
size; a phone shows every app fullscreen under the 20px menu bar.

## Talking to the desktop

Apps and the desktop are same-origin and talk over `postMessage`:

- `{exe:"focus"}` on pointerdown raises the window.
- `{exe:"grow-start"}` / `{exe:"grow", dx, dy}` / `{exe:"grow-end"}` from
  the app's own 15px grow box resize the window.
- A closed window is hidden, not destroyed: the desktop posts
  `{exe:"hide"}` on close and `{exe:"show"}` on reopen, so an app with
  timers or a simulation pauses and resumes; a plain document app can
  ignore both.
- `{exe:"reveal", id}` arrives when the menu bar's search names one of the
  app's items (Notes and Todo select and flash it, and ack).
- The frame allows fullscreen, so an app may call `requestFullscreen()`
  from a user gesture.

## Storage and sync

Per-app private state goes through `GET/PUT/DELETE
/v1/apps/<AppName>/data/<file>`, kept in `~/.exe/appdata/<AppName>/`
outside the served tree (10 MB per file). Files people should see go to the
shared Workspace (`/v1/workspace/<path>`), which is also a desktop Finder.

Joined exe desks sync app data between nodes, so every app here follows one
persistence contract: a debounced whole-document PUT, serialized so a slow
write is never overtaken, flushed on pagehide, and disabled until the first
GET succeeds so an empty document can never clobber the stored one. Apps
that hold records (Notes, Todo, World Clock) give every item an `id`,
`created` and `updated` stamps, and leave a tombstone on delete, which is
what lets two nodes' edits merge item by item; the daemon's merge schema for
each file lives in exe's `internal/peer/merge.go`.

## Apps

- **Notes** — a classic Note Pad in two columns: the note list on the left,
  the note on the right, titles from the first line, auto-saved as you
  type. Enter continues markdown lists (`-`, `1.`, `- [ ]`), Enter on an
  empty item ends the list, Shift+Enter is a plain newline. All notes live
  in one `notes.json`.
- **Todo** — a to-do list with a Return-default Add button. Drag rows to
  reorder (a fractional rank, so a reorder merges like any single edit); a
  newly added item scrolls into view and flashes. `todos.json`, version 2.
- **Paint** — a MacPaint homage: a 1-bit page (512×384 by default, Resize
  goes to 1152×1440), the tool and pattern palettes, the QuickDraw square
  pen, a width picker whose dotted top row is MacPaint's "no line". The
  page auto-saves as a PNG; **Save…** and **Load…** keep named paintings in
  the Workspace's `Paint` folder through a Standard File-style dialog. On a
  phone the tool bars swipe sideways.
- **Tides** — NOAA tide charts. Defaults to Newport Beach, CA (station
  9410580); Today / 3 Days / 7 Days; search any NOAA tide-prediction
  station. Reference stations draw the real 6-minute curve, subordinate
  stations get a cosine fit between highs and lows.
- **World Clock** — opens from the desktop's menubar clock. A list of
  cities, each with a pixel dial, the local time, and the day and offset
  against yours ("Tomorrow, +16h"). Search any city to add it (every IANA
  zone's principal city is built in); the × removes it. Defaults to Los
  Angeles, CA; `clocks.json`.

## Look

Everything is Mac OS 9 Platinum: 12px Charcoal type, beveled buttons,
sunken fields, the pixel-sampled scrollbar, one 1px line per seam, no hover
states. The guide with the numbers is exe's `docs/platinum.md`; copy the
shared CSS blocks from an existing app rather than restyling. `CLAUDE.md`
in this repo holds the conventions for coding agents working here.
