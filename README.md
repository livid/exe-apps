# exe-apps

Experimental web UI apps for the [exe](https://github.com/livid/exe) desktop.
Each top-level folder is one app bundle, served live from this checkout by the
exe daemon — edit a file, reload the desktop window, no build step.

## How it connects

The exe daemon scans `~/.exe/apps` plus every folder listed in `apps_dirs` in
`~/.exe/config.json`:

```json
"apps_dirs": ["~/Developer/exe-apps"]
```

(also editable in the desktop's Configuration window, Daemon tab). Every valid
bundle shows up as a desktop icon; `~/.exe/apps` wins name collisions.

## Bundle layout

```
AppName/
  app.json     { "title", "icon", "window": { "width", "height", "grow" } }
  index.html   the app; served at /apps/AppName/
  icon.svg     32×32 desktop icon (render at exactly 32 CSS px)
```

The desktop opens `index.html` in an iframe with the API token in `?token=`.
Per-app private state goes through `/v1/apps/<AppName>/data/<file>` — it lives
outside the served tree in `~/.exe/appdata/<AppName>/`.

## Apps

- **Tides** — NOAA tide charts. Defaults to Newport Beach, CA (station
  9410580); Today / 3 Days / 7 Days ranges; search any NOAA tide-prediction
  station. Reference stations draw the real 6-minute curve, subordinate
  stations get a cosine fit between highs and lows.
