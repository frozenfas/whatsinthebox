# What's in the Box

A phone-first inventory app. Photograph each object as it goes into a storage
box, identify it with the Claude API, search later to find which box it is in.
Single user, personal project. Target device is an iPhone in Safari.

## Hard constraints

- **No build step.** Vanilla JS, no bundler, no npm at runtime, no framework.
  The app must run by opening `index.html` from a static host.
- **No server the user administers.** Static hosting plus two third-party APIs
  is the whole architecture. Do not propose a backend.
- **Offline-tolerant.** Packing happens in cellars and lofts. Capture must
  never block on the network; unidentified photos queue and process later.
- **IndexedDB only** for persistence. No `localStorage`, no `sessionStorage`.
- **iOS Safari is the only target that matters.** Do not add APIs that Safari
  lacks (`BarcodeDetector`, background sync, File System Access) without a
  working fallback.
- **No secrets in the repo.** The Anthropic API key is entered on the device
  and lives in the `settings` IndexedDB store. Never hardcode or commit one.

## Decisions already settled, do not re-open

| Decision | Reason |
|---|---|
| PWA, not a native Swift app | Faster iteration, no €99/yr, works on any phone. Native remains the eventual target if this proves useful. |
| Hosted vision model, not a local one | Removes the GPU and the always-on machine. Roughly €1-2 per thousand objects. |
| Direct browser calls to `api.anthropic.com` | Needs `anthropic-dangerous-direct-browser-access: true`. Legitimate here: single user, own key, no proxy to run. |
| Location lives on the box, not the item | Moving a box relocates all its contents with one write. |
| No local classifier or embedding model | Structured JSON from the vision model plus substring search is enough at this scale. Revisit above ~2000 items. |
| iCloud rejected for sync | No usable API for a web app. CloudKit JS needs a developer container. OneDrive via Microsoft Graph is the intended path. |

## Files

- `index.html` — the entire app. Styles, markup and logic in one file, in that
  order. Sections are marked with `/* ---- name ---- */` comments.
- `qrcode.js` — vendored QR encoder (MIT, Kazuhiko Arase). Do not edit.
- `sw.js` — caches the app shell; passes all cross-origin requests through.
- `manifest.json`, `icon-*.png` — home screen install.

## Data model

```
boxes    { code: "BOX-001", name, created, sealed }        keyPath: code
items    { id, box, full, thumb, state, created,           keyPath: id (auto)
           title, category, material, colour,
           approx_size_cm, visible_text, condition,
           tags[], confidence }
settings { k, v }                                          keyPath: k
```

`state` is `pending` (awaiting identification), `working`, or `done`.
`full` and `thumb` are JPEG data URLs at 1024 px and 320 px.

## Running it

```bash
python3 -m http.server 8000    # then open http://localhost:8000
```

`localhost` counts as a secure context, so the service worker registers and
the install prompt works without any certificate.

## Style

- Two-space indent, single quotes in JS, semicolons.
- Small named functions over classes. No abstractions with one caller.
- User-facing copy: sentence case, active voice, no apologies in errors. Say
  what happened and what to do. Buttons name the action they perform.
- Design tokens are the CSS variables at the top of `index.html`. The yellow
  chip is the one loud element; everything else stays greyscale plus one blue.

## Roadmap, in order

1. OneDrive sync via Microsoft Graph and MSAL.js. One JSON per box plus photos
   as separate files. Never sync a live SQLite-style single file.
2. Scan a printed label to jump to that box's contents. Needs jsQR, since
   Safari has no `BarcodeDetector`.
3. Edit an item's fields after identification. Currently delete-only.
4. Store photos as Blobs rather than data URLs, about 33% space saving.
5. Category-specific enrichment behind one resolver interface: books via
   OpenLibrary, coins via Numista.
