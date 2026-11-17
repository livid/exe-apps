# exe-apps

Experimental web apps for the exe desktop (~/Developer/exe — a personal VM
cloud whose embedded web UI is a Mac OS 9 Platinum desktop). This repo is
deliberately separate from exe: apps here are mounted via `apps_dirs` in
`~/.exe/config.json` and show up as desktop icons next to the built-in ones.

## Dev loop

- Apps are served **live from this checkout** at `http://127.0.0.1:7777/apps/<Name>/`
  — edit, and every open window of the app reloads by itself once nobody
  is using it (the desk compares the `version` /v1/apps lists — a
  fingerprint of the bundle's files — with the one the window loaded; see
  docs/platinum.md, Apps). So an edit here reaches Livid's desks, phones
  and tablets too: an iPad left on an old Notes once wrote every note back
  without its colour. Flush on pagehide and mark as-you-type fields
  `data-autosave`, or the reload waits. No build step, no daemon restart.
- Only changes to the exe daemon itself need `make build` + POST
  /v1/daemon/restart (in ~/Developer/exe; restarting kills running VMs).
- API token: `api_token` in `~/.exe/config.json`. It is a password, so it
  goes on no address: the desk's session cookie lets an app in. An app's
  own address carries `?token=` only in a browser that keeps no cookies,
  and then the app sends it as before. For curl use `Authorization: Bearer
  <token>`.
- Visual checks: headless Chrome + CDP (`--headless=new
  --remote-debugging-port`, PUT /json/new, Page.captureScreenshot) against
  `/apps/<Name>/?token=…`. Screenshot and LOOK at every UI change.
- Projects, the issue tracker on the exe desk, holds this repo as the
  project `APPS` (`/www/exe-projects`; its README says how it is kept).
  An issue id such as `APPS-4` or `PROJ-5` is read whole with
  `/www/exe-projects/exe-projects show PROJ-5` (the binary is not on
  PATH; `help` lists the rest), and in this checkout a dot is its
  project: `exe-projects ls .`, `exe-projects new . "Title" -m "…"`.
  Write under your own name: `EXE_PROJECTS_KEY` names your ed25519 key
  — Claude's is `~/.claude/hub/claude_ed25519.pem`, Codex's its hub key
  `~/.codex/hub/codex_ed25519.pem` — and `exe-projects whoami -name "…"`,
  once, says what to call you. Without a key a record is signed as the
  node, which is Livid.

## Bundle layout

```
AppName/            folder name = app identity (must match ^[A-Za-z0-9][A-Za-z0-9 ._-]*$)
  app.json          { "title", "icon": "icon.svg", "window": { "width", "height", "grow": true } }
  index.html        the whole app, opened in an iframe by the desktop
  icon.svg          32×32 pixel-art desktop icon (render at exactly 32 CSS px, crispEdges)
```

- Per-app persistent state: `GET/PUT/DELETE /v1/apps/<Name>/data/<file>`
  (lives in `~/.exe/appdata/<Name>/`, outside the served tree; 100 MB/file cap).
- An app that can be read while exe does not answer adds `"offline"` to
  app.json: `true` keeps its own files on a device that has had the desk
  open over HTTPS, and `{ "keep": [addresses], "fixed": [prefixes],
  "wanted": address }` keeps data too. exe's service worker does the
  keeping and its alert offers the app; `/www/exe/internal/server/ui/sw.js`
  ("what an app keeps") is the contract, and a kept answer carries
  `X-Exe-Kept`. The app says it shows a copy and dims what writes
  (`/www/exe/docs/platinum.md`, Apps). Every shared sheet the app links
  (`/platinum/…`) is in `keep` too: the worker keeps only what is named,
  and a copy read without its sheet draws that block undressed (Notes'
  scroll bars and combo box, until 2026-11-17).
- `~/.exe/apps` wins name collisions with this repo.

## Conventions (see Tides/index.html for a worked example)

- **Look**: Mac OS 9 Platinum — the full guide (numbers, colours, rules) is
  `/www/exe/docs/platinum.md`. Copy the CSS blocks from an existing app —
  root color vars, beveled buttons, sunken text fields. Two blocks are the
  exception, shared stylesheets the daemon serves, linked in the head and
  never copied: the pop-up menu button, `<link rel="stylesheet"
  href="/platinum/popup.css">` (Weather's units menu), with the
  markup `<span class="popup"><select>…</select><span class="well"><i><b></b></i></span></span>`
  and a width from the app; and the pixel-sampled OS 9 scroll bar,
  `<link rel="stylesheet" href="/platinum/scrollbar.css">`, which draws
  both its states (something to scroll, nothing to scroll: an
  `overflow: scroll` bar is in the second whenever its content fits).
  Every app here links it; none carries a copy. Only the Return-triggered
  default button gets the black ring; others stay plain. Bordered scroll
  containers share 1px edges with the scrollbar — link the block, don't
  restyle.
- **Desktop bridge** (postMessage, same-origin): send `{exe:"focus"}` on
  pointerdown so clicks raise the window; if `window.grow` is true, include
  the 15px grow-box SVG and stream `{exe:"grow", dx, dy}` / `grow-start` /
  `grow-end` so the desktop resizes the window. Copy both blocks as-is.
  On a phone the desktop adds `?mobile=1` and the window fills the screen:
  hide the grow tile there (and give the status line its 8px right padding
  back; Todo also drops the horizontal bar that only reserved the corner).
  A closed window is hidden, not destroyed: the desktop posts `{exe:"hide"}`
  to the app's iframe on close and `{exe:"show"}` when it is reopened. An
  app with loops, timers or a simulation pauses on hide (and flushes any
  pending save) and resumes on show; a plain document app can ignore both.
  `{exe:"close"}` puts the app's own window away, for an app with an OK
  or Done button of its own.
  The frame allows fullscreen, so an app may call
  `document.documentElement.requestFullscreen()` from a user gesture (City
  binds it to F) and `document.exitFullscreen()` to come back.
- **Opening another app**: `{exe:"open-app", app:"Writer", path:"Planet/x/posts/y/index.md"}`
  asks the desk for that app's window on a Workspace path (relative, no
  `..`): a first open gets it as `?path=` in the frame's query, an open
  window gets `{exe:"open", path}` posted into it. The desk also forwards
  every Workspace change as `{exe:"workspace-changed", path, deleted,
  client}` to every app frame; an app showing a file reloads it when it
  holds nothing unsaved (the Writer) or refreshes its listing (Planet).
- **Opening a picture**: `{exe:"view-image", url, name}` opens it in a
  Picture viewer window of the desk's (the Hub app's embeds, a Projects
  attachment, Easel's canvas). `url` is the picture's whole http(s)
  address, which the desk fetches itself with its cookie — so never put
  the token on it — and `name` is the window's title. Out of the desk
  (`window.parent === window`) open the address in a new tab instead.
- **A coding-agent session in a folder**: `{exe:"agent", app:"claude", dir:"/www/exe"}`
  (or `app:"codex"`) opens the desk's Claude Code or Codex window, or
  brings it forward, and starts a new session in that folder, at the top
  of its session column (Projects' **Open in Claude Code**, with the
  project's working directory). `dir` is a whole path; the daemon refuses
  a folder that is not on this machine, and the window says so. Without
  the agent's CLI on the host the desk says so too; without tmux a second
  session cannot start. Out of the desk there is no window to ask, so dim
  the control. `prompt` is the session's first message (Projects' Send to
  Claude Code: the issue; at most 64 KiB). With a `prompt` and no `dir`
  (Notes' Send to Claude Code: a note has no folder) the window's New
  Session dialog asks where, and its Start sends the prompt with the
  folder chosen. Which agents there are, and what the desk takes, is
  `/v1/hostinfo`'s `agents`: `{claude: {dir, sessions, tasks, ask}}` —
  `sessions` is tmux for a second session, `tasks` a desk that takes a
  `prompt`, `ask` one that asks where when no `dir` comes with it.
- **Naming the window**: `{exe:"title", title:"EXE-12"}` titles the app's
  window for what it shows, after the app's name — "Projects — EXE-12"
  while that issue is open — and `{exe:"title", title:""}` is the app's
  name alone again. Send it when what the window shows changes, not on a
  timer; out of the desk set `document.title` instead.
- App name comes from `location.pathname.split("/")[2]`, token from
  `?token=` — don't hardcode either. The token is usually absent (the
  cookie does the work); put it on an address you build only when the
  app's own address came with one.
- **Appearance (day and night)**: write the sheet by day, as Platinum says it;
  never write a night. The daemon gives the page its night on the way out
  (`/www/exe/docs/platinum.md`, "Appearance: day and night"): every colour
  in a `<style>` rule or a `style=""` attribute gets one. It cannot read
  what a script writes, so:
  - a colour a script would set goes in the sheet instead: a class on the
    element and a rule for the class (Notes' tints are `tint-yellow` and so
    on, not `el.style.background`);
  - a `<canvas>`, or SVG a script builds in more than one colour, keeps its
    colours twice, a day set and a night set from the guide's palette, takes
    the one `exeDark()` names and draws again when the root hears
    `exe-scheme` (Tides' `INK`); guard it, `typeof exeDark === "function"`,
    for a page opened without the daemon;
  - a line that must be seen on a list or a chart is ink, `color:
    var(--black); border: 1px solid currentColor`, for an outline is black
    by night; a colour that must keep the day (ink on a painting's white
    paper) is said through a variable of the rule's own, `--ink: #262626;
    color: var(--ink)`.
  Check both: `node ~/tools/playwright/exe-app-look.js <App>` loads the page
  by day and by night, takes a screenshot of each and lists what stayed
  light or reads badly. By day nothing may change.
- Vanilla JS + inline CSS in one index.html; no frameworks, no CDNs.
- `<canvas>` gotcha: `inset:0` doesn't stretch a replaced element — set
  `width/height:100%` too, and scale the backing store by devicePixelRatio.

## Sync-capable storage (all apps)

Every app follows the same persistence contract so the daemon's node-to-node
sync can merge or LWW its files: debounced whole-doc PUT, **serialized**
(`saving`/`again` gate) so a slow write is never overtaken, `keepalive` flush
on pagehide/visibilitychange, and a `loaded` guard — if the initial GET
fails, saving stays disabled so an empty in-memory doc can never clobber the
stored one. Every PUT carries `X-Exe-Seq` — `nextSeq()`, which is
`lastSeq = Math.max(Date.now(), lastSeq + 1)`, taken with the snapshot it
stamps, so a save and the flush behind it in one millisecond, or a clock set
back, still count upward — and
`X-Exe-Client: <a random id made at load>`: the daemon drops a PUT stamped
older than one it already took **from that client** (an in-flight save must
not land over the pagehide flush) and answers `{"status":"stale"}`, and the
client id lets a window skip the `data-changed` echo of its own write. The
seq is never held against another window's — two devices' clocks differ,
and a save must not vanish for being stamped off the slower one; without
the client id the daemon can only judge it file-wide, so send both. Which
window's content wins is the app's job: reload on `data-changed` and keep a
local record only when it is strictly newer. Record-bearing apps (Todo,
Notes) give every item an `id`, `created`/`updated` epoch-ms stamps bumped
on every change — an edit, never a background write — and deletions
leave a `{id, deleted: <ms>, updated}` tombstone (text dropped) GC'd on save
after 30 days — that's what lets two nodes' edits merge item-by-item.

The daemon also judges each save of a record-bearing document against the
copy on disk, record by record (`peer.GuardLocalWrite`, since 2026-10-09),
so a window running older code cannot take back what newer ones wrote: a
record older than disk's keeps disk's; a record the save leaves out is
kept (tombstones are how things go; one past the 30-day TTL is GC); at an
equal stamp the save's fields win, but a field it does not carry at all
comes from disk. When it kept anything, the change event goes out with no
client tag, so the writer rereads too. What that asks of an app: bump
`updated` on every change (removing a field included), tombstone
deletions, and keep fields you do not know (`Object.assign({}, n, …)` in
the normalizer) so an edit made by your code never drops a newer one's.
It was an iPad still running the Notes from before colours, woken by a
turn-end push, that wrote every note back without its colour.

Reads never come out of the browser's cache: the daemon answers app data
`Cache-Control: no-store` (since 2026-09-23), and an app can say
`fetch(url, { cache: "no-store" })` as well. With only a Last-Modified,
Chrome kept a file that had sat idle for hours "fresh" for a tenth of that
age: a desk that had not saved since reread its old copy after every
`data-changed`, and typing there took words written on another desk. When
the desktop's change stream reopens after a gap (a daemon restart, a
sleep) every app window gets `{exe: "data-resync"}` — nothing written
meanwhile was announced, so read every file again as if each had changed
(Blue Pencil also rereads when its page becomes visible). A field whose
text the app saves as it is typed carries `data-autosave`: the desktop
holds its update reload while any other typed field still has text, and a
field that is never empty (Blue Pencil's) held a desk on an old build for
days.

- **Todo** — `todos.json` is `{version:2, items:[{id,text,done,created,
  updated,order?,deleted?}]}`; v1 bare arrays migrate on load with content-derived
  ids (`v1Id`) so two nodes migrating independently agree on them. Rows
  drag-reorder via a fractional `order` rank (falls back to `created`, so
  unranked docs keep creation order): a drop writes order+updated on the
  dragged item only — midpoint of its new neighbors' keys — so a reorder
  merges like any single-item edit. A mouse picks a row up after 4px; a
  finger pans the list (rows keep the default `touch-action` — `none`
  there locked a phone's list) and picks a row up only after resting
  500ms within 8px, the long press, when a non-passive `touchmove` on the
  list stops the pan for the rest of the drag
  (`~/tools/playwright/exe-apps-touch-reorder-test.js`). The daemon's
  merge struct must carry every field the apps write
  (internal/peer/merge.go strips unknown keys). Both the list's bars are
  always on (the linked `/platinum/scrollbar.css`), so they sit empty
  while the items fit; rows end in an ellipsis, so the sideways one
  never has anything to scroll and the scroll listener sets only the y
  classes.

- **Notes** — Mac OS's Note Pad (Apple Extras in the Mac OS 9 guest; Sherlock
  finds it), sampled from the real one: one white sheet under a toolbar
  (its writing in Geneva 12 on 16px lines, set in by the toolbar's own 10px
  at the top and sides — Livid's ask; Note Pad's was 3px and 5px), the
  reader's choice of size and face — the toolbar, a note open, carries −
  + (10 11 12 14 16 18 20 24 on 14…32px lines, never 13; 12 by default)
  and a Sans Serif | Serif | Mono segment (Tides' joined buttons, `.on`
  pressed): the serif is Charter (built into macOS and iOS), then Georgia;
  Mono uses the desktop's Monaco / system monospace stack.
  The choice is the same on every desk (Livid expected it saved and
  synced, 2026-10-09): `settings.json` `{size, face}`, a file of its own
  under the storage contract with its own save gate, last writer wins
  between windows and nodes, so a press never rewrites the notes; fields
  this Notes does not know go back as they came; opening never writes it
  (a browser's old choice is not pushed, the next press anywhere is);
  `data-changed`/`data-resync` read it again, and a board that is up is
  laid out again in the new face. localStorage (`exe-notes-type:<App>`)
  is only the first paint's copy, and the notes are held until the file
  is read, so the writing never changes size as a window opens. The size
  is the pad's, the face the board's sheets' too; where the words will not fit the
  toolbar gives way a step at a time (`fitBar`: the faces as "Aa"
  samples, then Corkboard as its pushpin, then tighter sides and gaps,
  then Delete and New Note as 30px glyph buttons, a trash can and a sheet
  with a plus: Livid's ask for a really narrow window, 2026-11-09) —
  the sheet's foot left to right: the dog-ear, the note's date (`created`) in
  11px Geneva `#777` 6px past it, the page number, the colour squares 5px
  short of the scroll bar (Livid moved the date out of the toolbar's
  middle and the squares to the right). The date takes the longest form
  (full, long, medium) that keeps 16px from the page number, else none,
  refitted by a ResizeObserver on the sheet (the desk can size the window
  after a resize event has passed); blank on a blank sheet; a failed save
  says so in its place (`#msg`). The scroll bar runs down to the grow box, two page edges (`#eee`/`#444`) under the
  sheet's bottom line, the page number centred on the window's width with its
  baseline on the grow box's top line, and the dog-ear at the bottom left —
  the flap's pixels are the real ones (a vector copy at fractional scales),
  its right line stops a row short of the sheet's bottom line. A click on
  the flap (above the fold) turns forward, below the fold back, wrapping at
  both ends. No list of notes anywhere (Livid asked for none): every note
  is a sheet pinned to a **corkboard**, the toolbar's Corkboard button
  (`.open`, the pressed look, while the board is up) switches between the
  board and the pad, and a click on a sheet opens it on the pad. Mac OS 9
  shipped no cork among its desktop patterns (checked in the guest's
  Appearance panel), so the cork is a 64px tile drawn the way those were,
  in the system palette's cube colours, `image-rendering: pixelated`,
  scrolling with the board (`local`). Sheets are 120×100 on a grid
  (`auto-fill`, centred), each nudged up to 3px off it and given one of
  four 13px pixel-art pushpins (a vector copy between 1x and 2x) by a hash
  of its id, so a sheet hangs the same way on every desk — whole pixels,
  never rotated. A sheet shows its first line in bold (two lines, four when
  the note has nothing else), the lines under it (four, two under a
  two-line title, whole lines only via line-clamp, measured after the
  board is built), the date and the page number. No sheet is marked as
  the open one (Livid: not needed); the board only scrolls it into view
  when put up. The board has the 15px status
  line (the count) with the grow box on its end. Checks for the whole app
  live in `~/tools/playwright/exe-notes/` (README there; the harness stubs
  every write and keeps Livid's desk layout out). The views stack in
  `#main`, the hidden one `visibility: hidden` so each keeps its scroll;
  between them a zoom and fade (Livid's ask, `swapViews`, Web Animations):
  opening, the sheet grows from its card's rectangle (measured inside
  `#main`'s top line) as it fades in over the first half, the board fading
  out leaning 8% in toward that card; returning runs backwards, the sheet
  solid until halfway, landing on its card. 240ms ease-in-out; no card
  (New Note, a blank sheet) grows from the middle at 92%; a newer switch
  cancels the running one and only the newest tidies (`swapSeq`); nothing
  is left transformed; none of it under reduced motion;
  which view is up is per viewer (`exe-notes-view:<App>`), the board at
  first when there are notes. Delete (the two-click armed button) is on
  the toolbar only with a note open (Livid: no disabled Delete on the
  board); New Note opens a blank sheet on the pad from either view. Pages
  are notes in `created` order, newest on top as page 1, so a page keeps
  its number while it is written on. New Note lays a blank sheet on top
  that becomes a note at the first keystroke and is dropped if turned away
  from or the board is put up, so empty notes never reach the document; a
  pad with no notes opens such a sheet. Each note may carry a `color`, the
  name of a Stickies colour (`yellow blue green pink purple gray`, sampled
  from Mac OS 9's Stickies in the guest; none is Note Pad's white),
  chosen not in the toolbar (Livid: not there) but from a row of small
  squares at the sheet's foot on the right (`#tints`: 14px boxes 2px
  apart, 20px from the window's right, 5px short of the bar; 18px boxes
  8px in on a phone): each its sheet's tint inside Stickies' own edge
  colour for it, the chosen one ringed 2px in `--hl`, every line a border
  on its own box. A press there keeps the caret. Where the window's
  middle would meet the row (narrower than 280px, a phone narrower than
  324px) the page number centres in the room left of it
  (`~/tools/playwright/exe-notes/footcheck.js` checks the foot at seven
  widths). The sheet, its band and its board card take the tint, under a paper
  grain (`--grain`, Livid's ask: SVG `feTurbulence` fractal noise as a dark
  speckle at most 6% deep, drawn at the screen's own resolution — about
  2.5 levels of variation, white paper under 2 levels darker on average,
  the same in Chromium and WebKit; the dog-ear's flap stays clean). Stickies' pastels
  are kept but for its two loud ones (`#ffff99`, `#99ffff`), a step
  lighter (`#ffffcc`, `#ccffff`) so a whole sheet stays subtle — all in
  the Mac's 256-colour cube. The dog-ear's flap is the same paper turned
  over: `FLAP`, the sampled pixels as grey levels, is multiplied by the
  tint into `--flap-px` / `--flap-vec`, and under the fold the next page's
  tint shows times `#eee`. A colour is an edit (`updated` bumps); a blank
  sheet keeps one until it is written on; a tombstone drops it; a name the
  app does not know is kept and shown white. The daemon's merge struct
  carries `color` (internal/peer/merge.go). The icon is three Stickies sheets
  fanned (blue, pink, yellow — Stickies' purple is the desk's own
  lavender), the front one with the dog-ear over the pink page beneath and
  the corkboard's pin. Note Pad's text stops above the dog-ear band
  while its bar runs on: `#foot` is laid over the field's foot, the field's
  30px bottom padding lets the last line scroll clear, and
  `scroll-padding-bottom` keeps the caret clear of it (Chromium honours it
  for typing and arrowing). Saves are silent; only a failure shows, at the
  sheet's foot and on the board's status line. All notes live in one `notes.json` through the app-data
  API — one doc beats per-note files for whole-doc auto-save (400ms
  debounce); deletes are tombstoned (see above); the field carries
  `data-autosave`. The open page is per-viewer UI state and lives in
  localStorage (`exe-notes-sel:<App>`), NOT in the synced doc — turning
  pages never PUTs and two machines can't fight over it. Enter
  continues markdown list items (`- * +`, `1.`/`1)` incrementing, `- [ ]`
  resetting), Enter on an empty item turns its marker into a blank line and
  drops to a fresh one, Shift+Enter is the plain-newline escape — all
  through execCommand so native undo survives. Its always-on
  page bar and the corkboard's link `/platinum/scrollbar.css` and sit in
  its disabled-flat state (flat #eee strip, #888 ghost arrows) whenever
  the note or the board fits. The app's 15px bar spec has no trailing line: here
  the window frame's line is the bar's right rail, as a bordered box's
  border-right would be (has-vbar toggling is a desktop-16px-spec thing).
  On a phone the sheet hides its bar (a finger scrolls it) and the band and
  page edges run the full width; New Note raises the keyboard, a page turn
  does not. **Send** (2026-11-09, Livid's ask): with a note open the
  toolbar has a glyph button left of Delete, a tray with an arrow leaving
  it, whose menu (Projects' contextual block, 2px under the button) says
  Send to Claude Code and Send to Codex, each only where
  `/v1/hostinfo`'s `agents` names that agent with `tasks`; with neither,
  in a Sticky and out of the desk there is no button. An item posts
  `{exe:"agent", app, prompt: <the note's text>}`, and the desk's New
  Session dialog asks for the folder (a desk without `ask` gets the
  entry's `dir` too and starts at once). The answer is kept in
  localStorage (`exe-notes-agents:<App>`) so the button is there from the
  first paint, and asked again at open, at `{exe:"show"}` and at each
  menu. Dimmed on a blank sheet and offline; an item is dimmed, with the
  reason as its title, without tmux and for a note past 64 KiB. Checks:
  `~/tools/playwright/exe-notes/send.js`. **Stickies** (2026-10-14): the pushpin at the sheet's foot, left
  of the colours, sticks the open note to the desk — the note gets
  `place: {x, y, w, h, shaded?}` (desk pixels: the window's corner and
  width, the sheet's height) and `placed`, a stamp of its own — and the
  desk (exe's index.html, "Stickies") opens this page a second time on
  that note, `?sticky=<id>`: `body.sticky` shows the field alone, fixed to
  the one note, remembering no page or view. The desk tells that page
  `{exe:"placed", x, y, w, h, shaded}` after a drag, a resize or a
  windowshade and `{exe:"unstick"}` from its close box, and the page
  writes the note; a drag never bumps `updated`, so the text and the place
  merge apart, each by its own stamp — in `reloadFromDisk` (told keeps
  both stamps), in the daemon's `mergeNotes` and in `GuardLocalWrite`
  (Codex's two-desk test, guard_test.go). A fresh sticky cascades from
  the desk's top left; the desk fits it to its own screen and writes that
  back only when something is moved there. A tombstone drops the place.
  The pin is hidden on a phone and on the page by itself (`body.alone`),
  which have no desk. The sticky page keeps Stickies' own inset (2px
  under the bar, 4px at the sides) and shows no scroll bar, as its notes
  had none; the window's chrome is the desk's, sampled from the guest
  (exe docs/platinum.md, Stickies).
  **Boards** (2026-11-09, Livid's ask): Corkboard is a combo button, the
  first one (`/platinum/combo.css`, linked; exe docs/platinum.md, Combo
  button). Its body puts the board up and down as before, pressed in
  (`.open`) while the board is up, and names the board that is up, in
  whole pixels (`nameBoard`, 132px at most); its arrow part has the
  boards' menu: every board in the order made, the one that is up
  checked, then New Board…, Rename Board… and Delete Board…, the last two
  dimmed on the first board. A press shows the menu at once, so a drag to
  an item and a release choose it. The boards are a file of their own,
  `boards.json` `{version: 1, items: [{id, name, created, updated,
  deleted?}]}`, with its own save gate like `settings.json`, so the notes'
  saves never carry it and a window on an older Notes never writes it;
  the daemon merges it board by board (`Notes/boards.json` in
  internal/peer/merge.go). The first board, Corkboard, has no record. A
  note hangs on one board: it keeps the board's id as `board` (none: the
  first board; an id no live board has: the first board too, so deleting
  a board writes no note). A move is an edit (`updated` bumps); the
  daemon's merge struct carries `board`, and at an equal stamp a copy
  without it (an older daemon's) does not take it away. Which board is up
  is per viewer (`exe-notes-board:<App>`): the pad's pages, their numbers
  and the status line's count are that board's, New Note pins its sheet
  there, and a note found from elsewhere (the desk's search) brings its
  board up. New and Rename ask for the name in the window's own dialog
  (`ask`: 12px inside, a sunken field, Cancel and the Return default 58px
  wide, Escape cancels), and Delete Board… and a sheet's Delete ask
  through it too. On the board a right-click on a sheet (a finger held
  500ms) shows the sheet's menu: Send to Claude Code and Send to Codex
  (the Send button's items and reasons), Move to each other board, and
  Delete. One menu at a time (`showMenu`), the Send button's included.
  Checks: `~/tools/playwright/exe-notes/boards.js`.
- **Tides** — NOAA CO-OPS tide charts (fetches NOAA directly; CORS is open).
  Gotchas encoded there: only type-R stations serve 6-minute curves (type-S
  fall back to cosine fit between hi/lo); fetch `time_zone=gmt`, render in
  viewer-local time; station list cached slim in localStorage for a week
  (`tides-stations-v2`, with each station's lat/lon). Weather rides each
  high/low marker: the temperature joins the clock row after a middot
  ("8:33AM · 71°"; alone at 7 days, which has no clocks), and the
  condition icon sits outermost (over a high, under a low). The icons are
  the Weather app's conditions drawn again on the 16 grid, OS 9's small
  icon size (`ICONS`, one character a pixel, in the 32-grid art's `PAL`;
  `WMO` is copied from Weather/index.html — keep the two in step), painted
  per DPR into a sprite with every art pixel snapped to device pixels. One
  Open-Meteo call per station and day covers the week from local midnight
  (hourly temperature_2m, weather_code, is_day plus daily sunrise,sunset;
  °C, GMT unixtime; whole `start_date`/`end_date` days, since an hour range
  leaves the daily block empty), so the range buttons never refetch. Sun or
  moon follows the tide's own time against sunrise/sunset (Codex's catch:
  the nearest hour's is_day drew a sun before dawn), is_day only where no
  real pair brackets it (polar sunrise = sunset). Re-asked after 30
  minutes on the minute tick. `plan()` shows the weather while neighbouring highs (or lows)
  stand their rows' half widths apart (measured with a "00°" stand-in),
  the icons too while the plot keeps 50px of water after their 42/46px
  reserves; the y scale keeps that room past the highest high and lowest
  low before the forecast lands, so nothing moves when it does. The now
  label lives in the margin over the plot, where no stack reaches. The
  station's position is saved with
  it in settings.json when picked; a settings file from before is looked
  up once in NOAA's mdapi (`stations/<id>.json`) and kept in memory — no
  background write. The station list's scroll bar is the linked
  `/platinum/scrollbar.css` (`overflow-y: auto`: it never sits empty).
  °C/°F is the shared `.popup`, per viewer in localStorage
  (`exe-tides-units:<App>`), converted client-side. Checks:
  `~/tools/playwright/exe-tides-now-label-test.js` (label text with the
  temperature, fit, no overlap with the markers' rows) and
  `exe-tides-tick-test.js`.
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
  The clock list's bar is always on (a Finder list's), so it sits empty
  until the cities outgrow the window; it and the search list's bar are
  the linked `/platinum/scrollbar.css`.

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
  a sticky label column, the current hour tinted. The bars are the linked
  `/platinum/scrollbar.css`: the list's and each table's are always on, so
  they sit empty while the content fits, and the scroll listener sets the
  x classes as well as the y ones. A `.popup` units menu
  (°C · km/h · mm or °F · mph · in) refetches, since the API converts
  server-side; the choice is the same on every desk (Livid, 2026-10-18):
  `settings.json` `{units: "metric" | "us"}`, a file of its own under the
  storage contract with its own save gate like Notes', last writer wins,
  read before the first forecasts are asked for and again on
  `data-changed`/`data-resync`; opening never writes it, fields the app
  does not know go back as they came, and localStorage
  (`exe-weather-units:<App>`) is only the first paint's copy — a browser
  with neither goes by its language (US, Liberia, Myanmar read °F).
  Search is Open-Meteo's
  geocoder (`geocoding-api.open-meteo.com/v1/search`, ≥2 chars, 250 ms
  debounce, a sequence number drops stale answers); one forecast call per
  city, three at a time, with `timezone=auto&forecast_days=7&
  forecast_hours=48`, the API's local ISO times shown as the city's wall
  clock in the viewer's format. Forecasts are cached in localStorage
  (`exe-weather-cache:<App>`, with the units they came in) so a reopen
  paints at once, and re-asked when older than 10 minutes on a one-minute
  tick that sleeps on hide; which cities and sections are unfolded is
  per-viewer UI state in localStorage too. Rows drag-reorder exactly like
  Todo's (press, move 4px — a finger holds 500ms — the black drop line,
  edge autoscroll): a fractional `order` rank falling back to `created`, written with
  `updated` on the dragged city only so a reorder merges as one item's
  edit; a drop is not a click, so the row does not toggle. `places.json`
  is `{version:1, items:[{id,name,region,country,cc,lat,lon,elev?,tz,pop?,
  feature,order?,created,updated,deleted?}]}` — the geocoder's record keyed by its GeoNames id
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
  document and the units file, calls Open-Meteo for real; `frame.click`
  in that harness can take seconds, so a timed state like the 3-second
  armed × is driven with DOM clicks). The units going between a desk, a
  phone and a new browser through the daemon's real change stream:
  `exe-weather-sync-test.js` beside it, on a throwaway daemon
  (`SCRATCH=<dir>`, its home `<dir>/exehome` on 127.0.0.1:7796 with
  `apps_dirs: ["/www/exe-apps"]`, places.json seeded).

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
  grow tile. The page's two bars and the file list's are the linked
  `/platinum/scrollbar.css`; the page scrolls both ways, so the scroll
  listener sets `scrolled-x` / `at-x-end` beside the y pair, or the
  sideways thumb would keep its start merge all the way along.

- **Ping** — SmokePing in a window, with its measuring in the exe daemon
  (`/www/exe/internal/ping`, routes in `internal/server/ping.go`): the only
  app here with a daemon side of its own, so a change to what it is told
  is a change to exe, and `docs.md` there ("Ping for an agent") lists the
  calls. The list of addresses is the daemon's (`~/.exe/ping/targets.json`,
  beside one `<id>.rrd` each), not app data: it does not sync, since each
  node pings from where it stands. Every picture is `rrdtool graph` asked
  for at the box's exact CSS size with `zoom` = devicePixelRatio, so it is
  sharp at 1, 1.5 and 2; the box has that size before the picture comes
  (no jump). The box is the window's width, a phone's too: under the
  daemon's `wide_width` (680, where SmokePing's legend stops fitting) the
  daemon draws its figures two to a line a size smaller and the window asks
  for taller pictures (380 and 150, not 300 and 120); only under
  `min_width` (300) does a box scroll sideways (its bar, like the list's,
  is the linked `/platinum/scrollbar.css`). `X-Ping-Plot` on a graph says
  where its plotting area is and the times at its edges; the drag-to-zoom
  (SmokePing's navigator) maps the pointer through it and asks for
  `start`/`end`. The window polls `GET /v1/ping` every 15 s (3 s while a
  round is out) and redraws a destination's graphs when its last round
  changes or the window's width does. Test: `node
  ~/tools/playwright/exe-ping-test.js http://127.0.0.1:7797` on a scratch
  daemon (it adds and removes an address).
