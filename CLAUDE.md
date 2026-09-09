# exe-apps

Experimental web apps for the exe desktop (~/Developer/exe — a personal VM
cloud whose embedded web UI is a Mac OS 9 Platinum desktop). This repo is
deliberately separate from exe: apps here are mounted via `apps_dirs` in
`~/.exe/config.json` and show up as desktop icons next to the built-in ones.

## Dev loop

- Apps are served **live from this checkout** at `http://127.0.0.1:7777/apps/<Name>/`
  — edit, reload the desktop window, done. No build step, no daemon restart.
- Only changes to the exe daemon itself need `make build` + POST
  /v1/daemon/restart (in ~/Developer/exe; restarting kills running VMs).
- API token: `api_token` in `~/.exe/config.json`. The desktop passes it to
  apps as `?token=`; for curl use `Authorization: Bearer <token>`.
- Visual checks: headless Chrome + CDP (`--headless=new
  --remote-debugging-port`, PUT /json/new, Page.captureScreenshot) against
  `/apps/<Name>/?token=…`. Screenshot and LOOK at every UI change.

## Bundle layout

```
AppName/            folder name = app identity (must match ^[A-Za-z0-9][A-Za-z0-9 ._-]*$)
  app.json          { "title", "icon": "icon.svg", "window": { "width", "height", "grow": true } }
  index.html        the whole app, opened in an iframe by the desktop
  icon.svg          32×32 pixel-art desktop icon (render at exactly 32 CSS px, crispEdges)
```

- Per-app persistent state: `GET/PUT/DELETE /v1/apps/<Name>/data/<file>`
  (lives in `~/.exe/appdata/<Name>/`, outside the served tree; 10 MB/file cap).
- `~/.exe/apps` wins name collisions with this repo.

## Conventions (see Tides/index.html for a worked example)

- **Look**: Mac OS 9 Platinum — the full guide (numbers, colours, rules) is
  `/www/exe/docs/platinum.md`. Copy the CSS blocks from an existing app —
  root color vars, beveled buttons, sunken text fields, and the pixel-sampled
  OS 9 scrollbar block. Only the Return-triggered default button gets the
  black ring; others stay plain. Bordered scroll containers share 1px edges
  with the scrollbar — reuse the existing block verbatim, don't restyle.
- **Desktop bridge** (postMessage, same-origin): send `{exe:"focus"}` on
  pointerdown so clicks raise the window; if `window.grow` is true, include
  the 15px grow-box SVG and stream `{exe:"grow", dx, dy}` / `grow-start` /
  `grow-end` so the desktop resizes the window. Copy both blocks as-is.
  A closed window is hidden, not destroyed: the desktop posts `{exe:"hide"}`
  to the app's iframe on close and `{exe:"show"}` when it is reopened. An
  app with loops, timers or a simulation pauses on hide (and flushes any
  pending save) and resumes on show; a plain document app can ignore both.
  The frame allows fullscreen, so an app may call
  `document.documentElement.requestFullscreen()` from a user gesture (City
  binds it to F) and `document.exitFullscreen()` to come back.
- App name comes from `location.pathname.split("/")[2]`, token from
  `?token=` — don't hardcode either.
- Vanilla JS + inline CSS in one index.html; no frameworks, no CDNs.
- `<canvas>` gotcha: `inset:0` doesn't stretch a replaced element — set
  `width/height:100%` too, and scale the backing store by devicePixelRatio.

## Sync-capable storage (all apps)

Every app follows the same persistence contract so the daemon's node-to-node
sync can merge or LWW its files: debounced whole-doc PUT, **serialized**
(`saving`/`again` gate) so a slow write is never overtaken, `keepalive` flush
on pagehide/visibilitychange, and a `loaded` guard — if the initial GET
fails, saving stays disabled so an empty in-memory doc can never clobber the
stored one. Record-bearing apps (Todo, Notes) give every item an `id`,
`created`/`updated` epoch-ms stamps bumped on every change, and deletions
leave a `{id, deleted: <ms>, updated}` tombstone (text dropped) GC'd on save
after 30 days — that's what lets two nodes' edits merge item-by-item.

- **Todo** — `todos.json` is `{version:2, items:[{id,text,done,created,
  updated,order?,deleted?}]}`; v1 bare arrays migrate on load with content-derived
  ids (`v1Id`) so two nodes migrating independently agree on them. Rows
  drag-reorder via a fractional `order` rank (falls back to `created`, so
  unranked docs keep creation order): a drop writes order+updated on the
  dragged item only — midpoint of its new neighbors' keys — so a reorder
  merges like any single-item edit. The daemon's merge struct must carry
  every field the apps write (internal/peer/merge.go strips unknown keys).

- **Notes** — two-column Note Pad (Chat-window layout: 190px list + document).
  All notes live in one `notes.json` through the app-data API — one doc beats
  per-note files for whole-doc auto-save (400ms debounce). Titles are
  the first non-empty line; the edited note bubbles to the top like Chat
  sessions; deletes use the desktop's two-click armed × pattern (tombstoned,
  see above). The open-note selection is per-viewer UI state and lives in
  localStorage (`exe-notes-sel:<App>`), NOT in the synced doc — clicking
  around never PUTs and two machines can't fight over it. Enter
  continues markdown list items (`- * +`, `1.`/`1)` incrementing, `- [ ]`
  resetting), Enter on an empty item turns its marker into a blank line and
  drops to a fresh one, Shift+Enter is the plain-newline escape — all
  through execCommand so native undo survives. Its always-on
  editor bar carries the OS 9 disabled-flat scrollbar block (flat #eee strip,
  #888 ghost arrows) that Tides doesn't need — copy from here for any
  overflow:scroll bar. Bordered boxes keep border-right: the app's 15px bar
  spec has no trailing line, so the box border is the bar's right rail
  (has-vbar toggling is a desktop-16px-spec thing).
- **Tides** — NOAA CO-OPS tide charts (fetches NOAA directly; CORS is open).
  Gotchas encoded there: only type-R stations serve 6-minute curves (type-S
  fall back to cosine fit between hi/lo); fetch `time_zone=gmt`, render in
  viewer-local time; station list cached slim in localStorage for a week.
- **World Clock** — the desktop's menubar clock opens it (the shell calls
  `openAppWin("World Clock")` when `/v1/apps` lists the folder). A list of
  cities with a 15px pixel dial each (the phone menubar's face at 2x, hands
  rasterised with Bresenham), the time in the viewer's own format, and
  "Today / Tomorrow, +16h" against the viewer's zone (¼ ½ ¾ for the odd
  offsets). Search is local: `CITIES` embeds every IANA zone's principal
  city from tzdata's zone1970.tab (US states / Canadian provinces, other
  countries by name) plus well-known cities sharing a zone and a few
  aliases (NYC, LA, SF, Saigon…) — regenerate it from tzdata rather than
  editing by hand. `clocks.json` is `{version:1, items:[{id,name,region,tz,
  created,updated,deleted?}]}` with content-derived ids (`name|tz`, so two
  nodes adding the same city agree); tombstones drop name/region/tz; the
  daemon merges it item-by-item (internal/peer/merge.go matches the full key
  `World Clock/clocks.json` — the City app has an unrelated cities.json).
  Default is Los Angeles, CA when the doc is absent; removing uses the
  two-click armed ×; the minute tick sleeps on hide and resumes on show.

- **Weather** — the World Clock's list shape, each row a 32-grid condition
  icon (`ART` in index.html, generated pixel art: sun, moon, cloud, rain,
  snow, sleet, thunder, fog; icon.svg is the sun-behind-cloud one), the
  temperature and today's high/low, unfolding on click — the Finder list
  view's disclosure triangle, sampled from a lossless OS 9 capture — into
  every field Open-Meteo's forecast endpoint serves, in four folding
  sections: Now (the 15 current variables plus the reading's time and
  interval), Place (the geocoder's record and the model's grid point,
  elevation, zone, offset, generation time), Daily (7 days × 59
  aggregates) and Hourly (48 hours × 65 surface variables), the tables in
  bordered boxes that scroll sideways under the OS 9 horizontal bar with
  a sticky label column, the current hour tinted. A `.popup` units menu
  (°C · km/h · mm or °F · mph · in, defaulting from the browser locale)
  refetches, since the API converts server-side. Search is Open-Meteo's
  geocoder (`geocoding-api.open-meteo.com/v1/search`, ≥2 chars, 250 ms
  debounce, a sequence number drops stale answers); one forecast call per
  city, three at a time, with `timezone=auto&forecast_days=7&
  forecast_hours=48`, the API's local ISO times shown as the city's wall
  clock in the viewer's format. Forecasts are cached in localStorage
  (`exe-weather-cache:<App>`, with the units they came in) so a reopen
  paints at once, and re-asked when older than 10 minutes on a one-minute
  tick that sleeps on hide; which cities and sections are unfolded is
  per-viewer UI state in localStorage too. `places.json` is `{version:1,
  items:[{id,name,region,country,cc,lat,lon,elev?,tz,pop?,feature,created,
  updated,deleted?}]}` — the geocoder's record keyed by its GeoNames id
  (`geo:5368361`) so two nodes adding the same result agree; tombstones
  keep only id and stamps; the daemon merges it item by item (merge.go
  matches the full key `Weather/places.json`, numbers as pointers so a
  sea-level elevation survives). Default is Los Angeles when the doc is
  absent. The three variable lists are what the API accepted on
  2026-09-09 — one unknown name fails the whole request, so regenerate
  from the docs rather than guess; the daily aggregates the default model
  leaves null (updraft, growing degree days, the layered soil means), the
  pressure-level columns and `minutely_15` are left out on purpose.
  Headless check: `~/tools/playwright/exe-weather-test.js` (mocks the
  document, calls Open-Meteo for real; `frame.click` in that harness can
  take seconds, so a timed state like the 3-second armed × is driven with
  DOM clicks).

- **Paint** — MacPaint homage: 1-bit page (512×384 default; Resize dialog
  goes up to 1152×1440, the saved PNG carries the size), tool + pattern
  palettes, QuickDraw square pen. The truth is one ImageData; every mark goes through
  it (shapes/text rasterize on a scratch canvas, thresholded, then stamped)
  so no antialiased gray ever lands — that's what keeps the bucket's flood
  fill exact. Pattern ink is opaque (set bits black, clear bits white) and
  origin-aligned so overlaps tile seamlessly; spray stamps only set bits.
  The width picker drives pencil, brush, line and shape borders (MacPaint
  kept pencil at 1px — users read that as broken); its dotted top row is
  MacPaint's "no line": shape borders vanish (filled shapes commit fill-only,
  previewed with a dashed guide), the line tool draws nothing, and pencil/
  brush fall back to the thinnest pen. Auto-saves the page as PNG to the
  app-data API, re-thresholded on load (`pageFromBitmap` always starts from
  a white page, so a sync reload never ORs stale ink in). Save… / Load…
  keep named copies in the Workspace's `Paint` folder (`/v1/workspace/
  Paint/<name>.png`, listed with `?dir=Paint`; the folder appears with the
  first save) through one Standard File-style dialog: a sunken list of the
  folder's PNGs, a "Save as:" field whose name turns the button into
  Replace when it already exists, "Painting N.png" as the free default;
  Load is one undoable op and scales anything past the 1152×1440 page
  limit to fit before thresholding. The button row and the pattern bar
  scroll sideways with no scrollbar (`scrollbar-width: none`), which is
  how they fit a phone; a phone (`?mobile=1`, body.mobile) also hides the
  grow tile.
