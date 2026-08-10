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

- **Look**: Mac OS 9 Platinum. Copy the CSS blocks from an existing app —
  root color vars, beveled buttons, sunken text fields, and the pixel-sampled
  OS 9 scrollbar block. Only the Return-triggered default button gets the
  black ring; others stay plain. Bordered scroll containers share 1px edges
  with the scrollbar — reuse the existing block verbatim, don't restyle.
- **Desktop bridge** (postMessage, same-origin): send `{exe:"focus"}` on
  pointerdown so clicks raise the window; if `window.grow` is true, include
  the 15px grow-box SVG and stream `{exe:"grow", dx, dy}` / `grow-start` /
  `grow-end` so the desktop resizes the window. Copy both blocks as-is.
- App name comes from `location.pathname.split("/")[2]`, token from
  `?token=` — don't hardcode either.
- Vanilla JS + inline CSS in one index.html; no frameworks, no CDNs.
- `<canvas>` gotcha: `inset:0` doesn't stretch a replaced element — set
  `width/height:100%` too, and scale the backing store by devicePixelRatio.

## Apps

- **Notes** — two-column Note Pad (Chat-window layout: 190px list + document).
  All notes live in one `notes.json` through the app-data API — that API has
  no list endpoint, so one doc beats per-note files; auto-save is a debounced
  (400ms) whole-doc PUT, serialized so a slow write is never overtaken, with
  a `keepalive` flush on pagehide because the desktop tears the iframe down
  when the window closes. If the initial GET fails, saving stays disabled —
  otherwise an empty in-memory doc would clobber the stored notes. Titles are
  the first non-empty line; the edited note bubbles to the top like Chat
  sessions; deletes use the desktop's two-click armed × pattern. Its always-on
  editor bar carries the OS 9 disabled-flat scrollbar block (flat #eee strip,
  #888 ghost arrows) that Tides doesn't need — copy from here for any
  overflow:scroll bar. Bordered boxes keep border-right: the app's 15px bar
  spec has no trailing line, so the box border is the bar's right rail
  (has-vbar toggling is a desktop-16px-spec thing).
- **Tides** — NOAA CO-OPS tide charts (fetches NOAA directly; CORS is open).
  Gotchas encoded there: only type-R stations serve 6-minute curves (type-S
  fall back to cosine fit between hi/lo); fetch `time_zone=gmt`, render in
  viewer-local time; station list cached slim in localStorage for a week.
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
  app-data API, re-thresholded on load.
