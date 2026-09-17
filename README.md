# Car Soccer — Web Port (single-file loader)

A self-contained web port of [car-soccer.com](https://car-soccer.com) (Rocket League–style
car soccer). The original game bundle is hard-wired to load its ~65 MB of assets from
`car-soccer.com` itself, so it cannot run from any other host. This repo makes it run from
anywhere:

- **`index.html`** — a single loader file. It boots the original game bundle and shims every
  asset request (`fetch`, `<img>`, Web Workers, and requests made *inside* workers) to an
  asset base URL. It also stubs the game's offline service-worker handshake when the real
  worker is unavailable.
- **`assets/`** — the full original game (three.js bundle, CSS, stadium/ball/car models, audio,
  ONNX bot policies, WASM runtime), copied from car-soccer.com.
- **`game-sw.js`** — a service worker that precaches all 156 game files so the game also works
  offline when hosted at an origin root (e.g. GitHub Pages user site, localhost).

> All game assets and code belong to car-soccer.com. This is for personal/educational use.

## Run it

### 1. Locally (easiest)

Serve the folder with any static server from this directory, then open the printed URL:

```
npx serve .
```
or
```
python -m http.server 8000
```

No configuration needed — assets load from the same folder. First load precaches the whole
game (~65 MB) for offline use.

### 2. One HTML file + CDN

Push this repo to a **public** GitHub repository, then use jsDelivr as the asset CDN:

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@main
```

Then either:

- edit the `CDN` variable at the top of the script in `index.html`, or
- just open `index.html` with a query parameter:

```
index.html?cdn=https://cdn.jsdelivr.net/gh/<user>/<repo>@main
```

The value is remembered in `localStorage`, so you only need the parameter once.

### 2b. Double-click `index.html` (file://)

Opening the file directly also works — no server needed. The loader falls back to
XMLHttpRequest for `fetch()` (browsers block `fetch` on `file://`), rewrites the bundle's
drive-rooted asset URLs, and stubs the service-worker handshake (real offline caching is
http-only). Verified in Chromium-based browsers; Firefox/Safari may still block module
scripts or workers on `file://` pages — use a local server if so.

### 3. GitHub Pages

Push to GitHub, enable Pages (deploy from branch `main`, root). If the site is served at a
domain/subdomain root, the real `game-sw.js` activates and the game becomes fully playable
offline. On project pages (`user.github.io/<repo>/`), the loader automatically falls back to
online mode.

## How the loader works (summary)

The original bundle (`assets/game-DskACDFO.js`) references assets with origin-rooted paths
(`fetch("/assets/ball/ball.gltf")`), spawns its bot AI worker via
`new URL("/assets/worker-iFqqV1m9.js", import.meta.url)`, and the worker itself embeds ONNX
Runtime which locates `/assets/ort-wasm-simd-threaded-*.wasm` relative to its own URL.
`index.html` rewrites all of these to the configured asset base and wraps the cross-origin
worker in a same-origin Blob so the browser allows it. The `/api/sponsors` endpoint is left
alone — the game already degrades gracefully without it.

## Troubleshooting

- **"Game assets were not found next to index.html"** — you opened the file directly or the
  folder is incomplete. Serve the folder, or pass `?cdn=...`.
- **Stuck on the loading screen** — open DevTools → Network and check the first failing
  request; if you previously used a `?cdn=` value that no longer exists, clear it with
  `localStorage.removeItem("car-soccer:cdn")`.
- **Assets 404 on jsDelivr** — make sure the repo is public and the branch/commit in the URL
  is correct; jsDelivr caches `@main` for up to 12 hours (pin `@<commit-sha>` for a frozen
  version).
- **Bots not moving** — bot AI runs in a Web Worker with ONNX Runtime; it needs http(s)
  hosting (not `file://`).
