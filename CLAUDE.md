# What's in the Box

A phone-first inventory app. Photograph each object as it goes into a storage
box, identify it with the Claude API, search later to find which box it is in.
Personal project, one install per household. Target device is a phone browser
(iOS Safari first, Android Chrome also supported for install/camera/PWA).

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
| No MSAL.js for OneDrive auth; hand-rolled OAuth2 Authorization Code + PKCE instead | MSAL's CDN build stopped at v2 - Microsoft's own docs say v3+ requires a package manager or bundler, which breaks the no-build-step rule. The hand-rolled flow is ~80 lines of `fetch` + Web Crypto, and keeps every byte of auth state in the `settings` IndexedDB store instead of needing `localStorage` for a token cache. |
| Sync stores the passphrase, never uploads it; OneDrive only ever sees encrypted bytes | Privacy was the explicit priority when this was designed. A lost passphrase makes synced data permanently unrecoverable - that's the deliberate cost, not a bug. |
| Houses are the top-level entity, above boxes | Two phones (iPhone + Android) already mean two isolated IndexedDBs with no sync. Houses give the eventual sync (OneDrive roadmap item) a natural partition, and let one install hold more than one household's boxes without redesigning the schema. No switcher UI yet — one house is created automatically, renamed under Setup. |
| `DB_VER` only ever increases; migrations are additive | The catalogue is the only copy of months of packing data on someone's phone. See "Data model versioning" below — this is the pattern every future schema change must follow. |

## Files

- `index.html` — the entire app. Styles, markup and logic in one file, in that
  order. Sections are marked with `/* ---- name ---- */` comments.
- `qrcode.js` — vendored QR encoder (MIT, Kazuhiko Arase). Do not edit.
- `sw.js` — caches the app shell; passes all cross-origin requests through.
- `manifest.json`, `icon-*.png` — home screen install.

## Data model

```
houses   { id, name, created }                              keyPath: id (auto)
boxes    { code: "BOX-001", name, created, updated, sealed,  keyPath: code
           house, chip }                                     index: house
items    { id, uid, box, full, thumb, state, created,        keyPath: id (auto)
           updated, title, category, material, colour,       indexes: box, state
           approx_size_cm, visible_text, condition,
           tags[], confidence }
settings { k, v }                                            keyPath: k
```

`state` is `pending` (awaiting identification), `working`, or `done`.
`full` and `thumb` are JPEG data URLs at 1024 px and 320 px.
`box.house` points at a `houses.id`. `box.chip` is a hex colour from the
8-swatch `PALETTE` in `index.html`, used for the printed label's colour bar
and the in-app chip — kept separate from `item.colour` (the object's own
dominant colour, set by the vision model) even though the names are similar.
Items don't carry `house` directly; they're scoped to a house through their
box, so a box move never needs to touch its items.

`updated` (on both boxes and items) and `item.uid` exist for sync, not for
the UI. `updated` decides which side wins when two phones both have a copy
of the same box - see Sync below. `uid` (`crypto.randomUUID()`) is what
sync matches items on across devices, because `id` is a per-device
auto-increment integer that two phones will happily generate the same
value for independently. Every code path that changes an item must call
`touchBox(item.box)` afterwards so the *box's* `updated` reflects the
change - sync only looks at the box's timestamp, not each item's.

## Data model versioning

`DB_VER` in `index.html` only ever goes up, and `openDB()`'s
`onupgradeneeded` is a stack of `if (from < N)` blocks, one per version, run
in order for whatever range the browser's stored version falls into. A
brand-new install runs every block; an existing phone only runs the ones
after its current version.

When a change needs a new store, index, or field:
1. Bump `DB_VER` by one.
2. Add one more `if (from < DB_VER) { ... }` block. Never edit a block that
   has already shipped — someone's phone may already be sitting at that
   version.
3. If existing records need the new field, backfill them with a cursor
   inside that same block (see the `from<2` block for the pattern) rather
   than leaving them `undefined` for the rest of the app to guard against.
4. Never remove or rename a field in place. Add the new one; leave the old
   one alone if anything might still read it.
5. Bump the `CACHE` constant in `sw.js` too, so installed phones actually
   fetch the new `index.html` instead of serving the cached shell forever.

This is the only way schema changes are safe to ship: nobody can run a
migration script against a phone that's face-down in a box in someone's
loft. The upgrade has to run itself, the next time the app happens to open.

## Sync (OneDrive)

Two phones, two separate personal Microsoft accounts, one shared OneDrive
folder (`WhatsInTheBox`) that one of them creates and shares with the
other the normal OneDrive way. Everything that reaches OneDrive is
encrypted client-side first - see the `/* ---------------- encryption
---------------- */` and `/* ---------------- OneDrive sync
---------------- */` sections in `index.html` for the actual code and
their own inline reasoning.

**Status: built, not yet configured or tested live.** `MS_CLIENT_ID` in
`index.html` is a placeholder - sync cannot work until it's replaced with
a real Azure app registration's client ID (SPA redirect URI, `Files.ReadWrite`
delegated scope, "personal Microsoft accounts" supported). Everything
that doesn't require live OAuth was verified directly: encryption
round-trips, a wrong passphrase correctly fails closed instead of
overwriting good remote data, and a full push-then-pull cycle was tested
against a mocked Graph API. The actual OAuth redirect flow needs a real
device and a real Microsoft login to verify - untested until that happens.

**Known limitations, deliberately scoped this way for v1, not oversights:**
- **Deletions don't sync.** Pulling a box only adds or updates items by
  `uid`; it never removes a local item just because the remote copy lacks
  it. That's what stops a pull from wiping out an item you added on this
  phone before it had a chance to sync - the alternative (treat the
  remote item list as authoritative) risks real data loss. The cost:
  delete an item on one phone, and it can reappear from the other phone's
  next push until real tombstones are built.
- **Box codes can theoretically collide.** `nextCode()` only looks at
  boxes already known to *this* device. If both phones create a new box
  before either has synced even once, they could pick the same code, and
  the later sync silently lets one overwrite the other. Narrow window in
  practice (sync before a fresh install starts creating boxes), but real -
  a merge-time collision detector would close it properly.
- **Whole-box last-write-wins, not per-field merge.** If both phones edit
  the *same* box while both are offline, the newer `updated` wins entirely;
  the older edit is just gone. Fine for how this app is actually used
  (pack into a box on one phone at a time); would need real conflict
  resolution to be safe for genuinely concurrent editing.

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
- The brand mark (`LOGO_SVG` in `index.html`) is an isometric open crate with
  a small yellow tag on its corner — reused as-is for the header, the home
  screen icons (`icon-192.png`, `icon-512.png`), and the printed label.
  Regenerate the PNGs from the same SVG if it ever changes, so all three
  stay in sync; don't hand-edit the PNGs.
- Box colours (`PALETTE` in `index.html`) are a fixed 8-swatch set, not a
  free colour picker — keeps every box's colour visually distinct at a
  glance across a shelf, and keeps the printed label's colour bar readable
  at a small physical size.

## Roadmap, in order

1. Configure and live-test OneDrive sync (see Sync above) - it's built, it
   just hasn't been pointed at a real Azure app registration or tried on
   real devices yet. After that: proper deletion tombstones and box-code
   collision handling, the two known gaps called out above.
2. Scan a printed label to jump to that box's contents. Needs jsQR, since
   Safari has no `BarcodeDetector`.
3. Edit an item's fields after identification. Currently delete-only.
4. Store photos as Blobs rather than data URLs, about 33% space saving and
   much cheaper to list/search since `getAll('items')` stops dragging full
   photo bytes through memory for every row. **Priority note:** the stated
   target is thousands of boxes and 1M+ photos. IndexedDB itself (already in
   use, not a flat JSON file) handles thousands of *records* fine, but a
   million photos' worth of bytes sitting in browser storage is a real wall
   — quota limits, iOS eviction, and this item's cost all get worse well
   before that scale. Do this before item count gets much past a few
   thousand, not after. Past that, the honest ceiling for a no-server
   browser PWA is reached; the documented native Swift + CloudKit path
   (see Background in README) is what actually scales to a million photos,
   because it gets real sync and storage outside the browser sandbox.
5. Category-specific enrichment behind one resolver interface: books via
   OpenLibrary, coins via Numista, stamps and other catalogued collectibles
   via similar structured lookups where a free/affordable API exists —
   including flagging items that look valuable or rare (e.g. a coin or
   stamp the API marks as a scarce date/variant) rather than just filling
   in fields. Paintings, sculpture, and furniture don't have an equivalent
   of an ISBN or catalogue number to look up, and no affordable structured
   database of maker's marks or auction records exists to query. For that
   category the realistic scope is not automated identification — it's
   flagging the item (e.g. low classifier confidence, or a visible
   signature/mark worth a closer look) so the human decides whether it's
   worth a real appraisal, not this app guessing at attribution or value.
