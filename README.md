# What's in the Box

A storage register you run from your phone. Photograph an object as it goes
into a storage location — a box, a shelf, a cupboard, whatever holds it — and
the app identifies it, files it against that location, and lets you search
months later for where a thing ended up. The app's own name is the one place
"box" still literally means a box; everywhere in the UI it's a "location"
(`LOCATION-0001` on the printed labels), on purpose, so it fits a shelf or a
cupboard just as well.

Six static files. No server, no build step, no framework.

## How it fits together

```
Phone (Safari on iPhone, Chrome on Android; installed to home screen)
  |
  |-- static files ......... GitHub Pages or Cloudflare Pages
  |-- one call per photo ... api.anthropic.com
  |-- book lookups .......... openlibrary.org (books only, no key needed)
  |-- sync, if connected .... login.microsoftonline.com, graph.microsoft.com
  |-- catalogue + photos ... IndexedDB on the device
```

Everything this app talks to, and why, is also listed on the Setup tab
under "Connected services."

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
4. Locations tab: pick or create the **house** you're packing at (top of
   the tab) — a location always belongs to whichever house is selected
   there. Then create a location (a box, a shelf, a cupboard — whatever
   it actually is), pick a colour, print its label, tape it on. The
   "copies per label" choice controls how many identical copies print on
   one sheet — enough to tape on more than one side, or more than one
   shelf edge — sized to use the page well whether you pick 2, 4, or 6
   per sheet. Sheets print at A4 with zero page margin (the printable
   area is set by the label sheet's own layout, not your printer's
   default margin) — if your printer's dialog offers a margin or scale
   option, leave it at **Default margins / Actual size**, not
   "Fit to page" or "Shrink to fit", or the label sizes will be off.
   If a label's colour bar or the scale card's grey/black bar prints
   blank, look for a **"Background graphics"** toggle in the print
   dialog and turn it on — the app already asks for this in its CSS,
   but a few browsers still require it set manually too.
5. Capture tab: one object per photo as it goes in — only locations at
   the currently-selected house show up here, since you can't be
   physically standing at the other one.
6. Find tab: search by name, material, or text printed on the object —
   this searches across *every* house, each result labelled which one
   it's in.

## Book lookup

Anything identified as a book gets an automatic, free lookup against Open
Library (title/author, no account or key needed) — shown under the item as
supplementary info, never overwriting what Claude itself identified. Open
Library's data isn't always right (a real ISBN looked up while building
this returned a completely different, mismatched book), so treat it as a
second opinion to glance at, not a fact.

If the automatic lookup doesn't find a match, a **"Photograph copyright
page"** button appears on that item — take a photo of the page inside the
book with the ISBN and publisher details (more reliable than the cover),
and it looks up that exact edition instead of guessing from the title.

Anything that looks unusual — a book with very few editions on Open
Library, or Claude noticing a first-edition mark, signature, or similar
while identifying *any* object — shows as "Worth a look." That's a nudge
to look closer yourself, not a valuation; nothing here can tell you what
something is actually worth.

## Coin lookup

Anything identified as a coin gets an automatic, free lookup against
Numista by title, shown under the item with its Numista catalogue number
(N#, a clickable link back to the real Numista page) as their terms
require — plus Numista's own photo of that catalogue entry, shown right
there, so a text-only match can be visually checked at a glance rather
than trusted blind or clicked through to confirm. If it doesn't look
right, that's what the image search batch (below) is for. Add a free
Numista API key under Setup, "Coin lookup", to turn this on — no key,
no automatic lookup. If a coin was identified *before* the key was
added, it won't have a match yet either — a **"Retry coin lookup"**
button appears on any coin item missing one, and just re-runs the free
search using the title Claude already gave it, no new photo needed.

If the text search doesn't find a match, it just stays unmatched — there's
no automatic image fallback, because Numista's image-search endpoint needs
their paid plan (€100/month minimum). Setup has a **"Run image search on
unmatched coins"** button instead: it shows you the real estimated cost
first, and is meant to be run as an occasional batch once a backlog of
unmatched coins has built up, not per-coin. Numista waives the €100
monthly minimum for the first calendar month a paid plan is active, so
the intended pattern is: activate the plan, run the batch once, then
cancel or downgrade before the next billing month.

A coin's date or mint mark can end up on either face, so one photo is
routinely not enough — but capture stays one tap, one photo, exactly as
fast as any other object. Right after taking a photo, the Capture tab
itself offers **"+ Add another photo"** — flip the coin over and
photograph the other face there and then, no need to switch to Find
first. The same button also appears on any coin item afterwards, on
the Find tab. Either way, Claude re-identifies the item using both
photos together rather than guessing from just the one it already had.
Entirely optional — an item with just one photo works fine and nothing
prompts you to add the second. Sending a second photo roughly doubles
that item's identification cost, same as any extra image in the request.

## Item detail

Tap any item on the Find tab (not its photo-strip's delete button) to
open a full detail view: every field Claude filled in, both photos if
there are two, and — for a coin — the exact text that was sent to
Numista, in an editable box with a **"Search again"** button next to
it. If Claude's title sends the search astray (a real 20-cent-euro-coin
test during development matched the wrong design entirely), correct it
there and re-search — free, no new photo, no re-identification needed.
Nothing else on this screen is editable yet; deleting and re-capturing
is still the way to fix a wrong category, material, or title.

## Photo scale card

Setup, "Photo scale card" — prints two L-shaped corner rulers side by
side, one labelled **OBVERSE** (front/"heads") and one **REVERSE**
(back/"tails"), each framing a dotted circle to place a coin in. A
corner ruler reads width *and* height from one photo, unlike a single
ruler laid alongside an object which only gives one axis, and the
alternating black/grey bar above each ruler's numbers gives a
size at a glance without reading them. Photograph the coin in the
correctly labelled patch for each side — the printed label itself is
what tells Claude which side it's looking at, since it's right there in
the photo. Print it once and reuse it. Same "Actual size, not Fit to
page" rule as the labels applies.

## Syncing two phones (OneDrive)

Setup is a one-time thing, already done for this deployment: an Azure app
registration (personal Microsoft accounts, SPA redirect URI, the narrow
`Files.ReadWrite.AppFolder` Graph permission — not the broader
`Files.ReadWrite`) with its client ID wired into `MS_CLIENT_ID` in
`index.html`. No cost, no client secret.

On each phone, under Setup:
1. Set a **sync passphrase** — the same one on both phones. It encrypts
   everything before it ever reaches OneDrive, so OneDrive only ever holds
   unreadable bytes. Put it in a password manager: if it's lost, whatever
   synced under it is unrecoverable, by design.
2. **Connect OneDrive** — sign into the **same dedicated Microsoft
   account** on both phones (not your own personal one). That's what
   makes the narrow `Files.ReadWrite.AppFolder` scope possible — the app
   can only ever reach its own folder, and there's nothing else in that
   account to reach anyway. No folder-sharing step needed; both phones
   are automatically looking at the same app folder.
3. **Sync now** whenever you want the two catalogues to merge. It also
   fetches on its own each time the app is opened with a connection.

## Resetting the catalogue

Setup, "Danger zone", "Reset all data" — wipes every house, location, and
item on this phone. It does **not** touch your saved API keys, sync
passphrase, or OneDrive connection, so you won't need to re-enter those
afterwards. Before anything is deleted it downloads a full backup
(catalogue + photos) automatically, and asks you to confirm twice — the
second time by typing `DELETE`. If the other phone is synced, a reset here
is recoverable by syncing again; if it isn't, the downloaded backup is the
only copy, so keep it somewhere safe.

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
objects. The rarity/"worth a look" check rides along on that same call
(a few dozen extra tokens, not a second call), so it doesn't meaningfully
change this. Book lookups and the automatic coin text lookup are both
free and don't touch the Anthropic bill at all. The optional coin image
search is the one paid extra in this app, entirely separate from the
Anthropic bill and never run without a confirmation showing the real
estimated cost first — see "Coin lookup" above.

## Known limits of this prototype

- Data lives in this browser's IndexedDB. iOS can evict it after weeks of
  disuse. **Export from the Setup tab regularly** even with sync configured.
- Sync doesn't propagate deletions yet — deleting an item on one phone can
  have it reappear from the other phone's next sync. Documented, scoped
  limitation in `CLAUDE.md`, not a bug waiting to be found.
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
