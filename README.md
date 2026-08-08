# What's in the Box

A storage register you run from your phone. Photograph an object as it goes
into a box, and the app identifies it, files it against that box, and lets you
search months later for where a thing ended up.

Six static files. No server, no build step, no framework.

## How it fits together

```
Phone (Safari on iPhone, Chrome on Android; installed to home screen)
  |
  |-- static files ......... GitHub Pages or Cloudflare Pages
  |-- one call per photo ... api.anthropic.com
  |-- catalogue + photos ... IndexedDB on the device
```

Nothing of yours listens for inbound connections. The phone makes outbound
calls only, the same way any app on it does. The static host holds no data and
runs no code of yours.

## Files

| File | What it is |
|---|---|
| `index.html` | The whole app: UI, database, camera, API calls, label printing |
| `qrcode.js` | QR encoder (MIT, Kazuhiko Arase), vendored so labels work offline |
| `manifest.json` | Makes it installable to the home screen |
| `sw.js` | Service worker; caches the app shell so it opens without signal |
| `icon-192.png`, `icon-512.png` | Home screen icons |
| `CLAUDE.md` | Project context for Claude Code |

## Hosting

The site has to be served over HTTPS, otherwise the browser refuses to register
the service worker and the app cannot be installed.

**Private repo, free:** Cloudflare Pages. It connects to a private GitHub repo
on the free tier and redeploys on every push. This is the recommended route.

**Public repo, free:** GitHub Pages. Settings, Pages, deploy from `main` at
root. Nothing sensitive is in the repo, so this is a legitimate option.

**Private repo on GitHub Pages** requires a paid plan, and even then the
published site is still publicly reachable. Only an organization on Enterprise
Cloud can serve Pages privately.

In every case the *site* is public; only the source can be private. That is
harmless here, because the page is an empty shell and all data stays on the
phone. If you want the URL itself gated, Cloudflare Access adds a login free.

## Setting it up

1. Push the repo, connect it to Cloudflare Pages, open the URL on the phone.
2. iPhone: Safari, Share, then Add to Home Screen — Chrome on iOS cannot
   install PWAs. Android: Chrome will offer to install it directly.
3. Setup tab: paste an Anthropic API key from console.anthropic.com and save.
   Set a monthly spend limit in the Console while you are there.
4. Boxes tab: create a box, pick a colour, print its label, tape it on. The
   "copies per label" choice controls how many identical copies print on one
   sheet — enough to tape on more than one side of the box — sized to use
   the page well whether you pick 2, 4, or 6 per sheet.
5. Capture tab: one object per photo as it goes in.
6. Find tab: search by name, material, or text printed on the object.

## Local development

```bash
python3 -m http.server 8000    # http://localhost:8000
```

`localhost` counts as a secure context, so the service worker and install
prompt both work with no certificate. After changing `sw.js`, bump the `CACHE`
constant or the old shell keeps being served.

## Cost

Photos are resized to 1024 px before sending, landing around 1,400 input
tokens each. On Haiku 4.5 that is roughly one to two euros per thousand
objects.

## Known limits of this prototype

- Data lives in this browser's IndexedDB. iOS can evict it after weeks of
  disuse. **Export from the Setup tab regularly.** No cloud sync yet, so two
  phones (say, one iPhone and one Android) each keep their own catalogue —
  they don't merge until sync ships.
- Each install is one household ("house" in the data model), renamed under
  Setup. The schema already supports more than one house per install; there
  just isn't a switcher UI yet, since nobody's needed it.
- Labels print but are not scanned back in; you pick the box from a list.
- Item fields are not editable after identification, only deletable.
- Photos are stored as data URLs, about a third larger than raw blobs. This
  matters more the bigger the catalogue gets — see the roadmap note in
  `CLAUDE.md` if you're aiming for a large one.
- Search is substring matching over the stored fields. Fine to a few thousand
  items, and the point at which that stops being true is the point to add
  embeddings.

## Background

Earlier designs, and why they were dropped:

- **FastAPI on a Mac, phone as a client over Tailscale.** Worked, but meant a
  machine to keep running, patched and certificated, holding the only copy of
  the database. Too much to own for a tool used while carrying boxes.
- **A local vision model on a GPU.** Same objection, plus the phone then only
  works when that machine is awake.
- **Native Swift with SwiftData and CloudKit.** Genuinely better: real sync,
  no data eviction, free on-device OCR and image feature prints through the
  Vision framework, background uploads that finish when the app is closed.
  The cost is Swift, Xcode, and €99/year. Still the right destination if this
  proves worth keeping.
