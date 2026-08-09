# What's in the Box

A phone-first inventory app. Photograph each object as it goes into a storage
box, identify it with the Claude API, search later to find which box it is in.
Personal project, two people, two genuinely separate physical houses synced
into one catalogue. Target device is a phone browser (iOS Safari first,
Android Chrome also supported for install/camera/PWA).

## Hard constraints

- **No build step.** Vanilla JS, no bundler, no npm at runtime, no framework.
  The app must run by opening `index.html` from a static host.
- **No server the user administers.** Static hosting plus third-party APIs
  (Anthropic, Microsoft Graph, Open Library) is the whole architecture. Do
  not propose a backend.
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
| Both phones share one dedicated Microsoft account for sync, not two personal accounts | First built against two separate personal accounts, needing the broad `Files.ReadWrite` scope so the non-owner phone could reach a folder shared from the other's account. That scope reaches the *entire* OneDrive, including anything unrelated already stored there - reconsidered once that was fully understood. One dedicated account (`frozenfas.apps@outlook.com`), used for nothing else, lets the app request the narrow `Files.ReadWrite.AppFolder` scope instead. Traded a shared login for meaningfully less access. |
| Houses are the top-level entity, above boxes, and are themselves synced | Not a hypothetical: the two people using this app pack at two genuinely different physical addresses. Houses sync as their own small list (`houses.json`, merged by `uid`), and every box carries the real house it belongs to through sync (via that `uid`, translated to each device's own local house id) rather than being force-reassigned to whatever house happens to exist locally. Chosen over keeping the two houses' catalogues fully separate: both are visible and searchable from either phone, clearly labelled which house each box is in — useful for "is this at ours or at hers?" A switcher on the Boxes tab creates/renames/selects the active house. |
| `DB_VER` only ever increases; migrations are additive | The catalogue is the only copy of months of packing data on someone's phone. See "Data model versioning" below — this is the pattern every future schema change must follow. |
| Open Library over Google Books for book lookup | Google Books' unauthenticated quota is zero (`quota_limit_value: "0"`, confirmed by an actual 429) - not rate-limited under load, categorically blocked without an API key. Adding a Google Cloud project/key wasn't worth the setup cost this session already paid once for Azure. Open Library needs no key and was confirmed working, including on a Spanish-language title, from this deployment's real origin. |
| Rarity/value flagging is one extra field on the existing Claude call, not a second call | Keeps the cost impact marginal (a few dozen tokens) and spread across every category, not doubled for books specifically. Combined for free with Open Library's `edition_count` where available. Never a real appraisal - see roadmap item 5. |
| "Box" renamed to "location" in the UI only; every internal name stays `box` | A location doesn't have to be a literal box - a shelf or cupboard works the same way. Renaming every function, variable, DB store name, and sync file naming convention (`{box.uid}.box.json`) to match would have meant re-touching most of an already-tested codebase for zero user-visible benefit. `CODE_PREFIX` (`'LOCATION-'`, 4-digit) controls the printed/displayed code; `boxes`/`createBox`/`item.box`/etc. are unchanged and mean the same thing they always did. Don't "fix" this inconsistency by renaming internals later without a real reason - it was a deliberate scope cut, not an oversight. |
| Numista over Colnect for coin lookup | Numista has a real, documented, keyed API with CORS support, confirmed live from this deployment's origin. Colnect was checked directly against its own docs page and explicitly states `CORS: No` - a non-starter for a browser-only app with no proxy. |
| Coin image search is a manual, cost-gated batch action, not automatic | Numista's `search_by_image` endpoint needs a paid plan (€100/month minimum), unlike the free text search used automatically on every coin. Automatic per-photo image search would be an ongoing €100/month for a personal project. Instead `runImageSearchBatch()` lets the paid plan be activated for one calendar month (Numista waives the minimum for that first month per their terms §6.7), run once against whatever backlog has built up, then cancelled - real cost becomes the one-time activation plus ~€0.03/coin, not a standing subscription. `confirm()` shows the real estimated cost before it runs. |
| Scale card uses literal `mm` CSS units, same as the label sheet | Already proven physically accurate for printed labels at 96dpi-equivalent CSS mm. Reusing it for the ruler means the same `@page`/print pipeline and the same "print at Actual size" caveat, rather than a second unit system to get right. |
| Scale card is two L-shaped corner rulers (one per coin side), not one linear ruler | A linear ruler laid next to an object only reads one axis; an L-shaped corner ruler, with the object placed flush into the corner, reads width and height from a single photo. Two of them, each framing a dotted "OBVERSE"/"REVERSE" placement circle, doubles as the obverse/reverse side-tracking the coin module needed - the printed label is what tells Claude which side it's looking at, since it's in the photo itself, not a separate data field. The alternating black/grey segment bar above each ruler's tick marks is a coarse, at-a-glance scale that doesn't require reading a number, sitting above the precise numbered ticks rather than replacing them. |
| A second photo is opt-in, added after the fact, not a required second capture step | A coin's date or mint mark can end up on either face, so one photo is routinely not enough to be sure - but making every capture a two-photo step would slow down the fast one-tap-per-object flow the whole app is built around, for every category, not just coins. `full2`/`thumb2` stay `undefined` unless the user deliberately taps "+ Add other side" afterwards. |
| "+ Add another photo" also lives on the Capture tab, not just Find | Originally only reachable from the Find tab, which meant leaving the tab you're actively packing from just to flip a coin over - real friction for something meant to save a step, reported directly. `showLast()` now offers the same action immediately after a capture, and `$('shot').onchange` gives the initial identification a 1.5s local timer (not a network wait) before firing, specifically so tapping this within that window results in one two-photo `classify()` call instead of two racing single-photo ones. |
| `classify()` re-fetches the item immediately before its final write, rather than reusing its start-of-call snapshot | The same class of bug as the `DB_VER`/`walkCursor` issue above: if a second photo is added to an item while its first identification call is still in flight, writing back a stale snapshot at the end would silently erase that photo. A `classifying` `Set` also skips a second identification call for an item already mid-identification, so two photos added in quick succession don't fire two overlapping (and double-cost) Claude calls - if that skip ever means the second photo isn't factored in immediately, the photo itself is never lost, just needs one more tap to force a re-run. |
| Item detail view opens on tap, not a separate route/tab | A modal overlay (`#detailOverlay`) keeps this to one file with no router, consistent with the rest of the app. It exists specifically because the compact result card doesn't have room for every field, and because a wrong Numista match (Claude's title genuinely leading it astray - a real 20-cent-euro-coin case, not hypothetical) needed a place to fix the search text and retry without a full re-identification. |
| Numista search never runs off a single photo; only ~1.5s after both sides are known, or on an explicit tap | Originally every coin identification triggered a search immediately - meaning any coin that later got a second photo (a routine sequence) was searched twice, once on partial information about to be superseded. Splitting `classify()` (identification) from `runCoinSearch()`/`searchNumista()` (matching) entirely removes that waste, at the cost of a single-photo coin needing one manual "Retry coin lookup" tap to get a match at all, rather than getting one for free - a deliberate tradeoff, chosen directly over the alternatives (a longer debounce, or leaving the double-search as-is). |
| Item detail's Numista search shows up to 6 candidates to pick from, not just the top result | `q` is a free-text search with no guaranteed-correct top hit - confirmed a real case where it wasn't (a 20-cent-euro-coin match landed on the wrong design). `lookupCoin()` (first-result-only) still backs the *automatic* and quick-retry paths, since nobody's present there to choose; the deliberate, manual detail-view search is where a wrong top pick is actually correctable. |

## Files

- `index.html` — the entire app. Styles, markup and logic in one file, in that
  order. Sections are marked with `/* ---- name ---- */` comments.
- `qrcode.js` — vendored QR encoder (MIT, Kazuhiko Arase). Do not edit.
- `sw.js` — caches the app shell; passes all cross-origin requests through.
- `manifest.json`, `icon-*.png` — home screen install.

## Data model

```
houses   { id, uid, name, created, updated }                 keyPath: id (auto)
boxes    { code: "BOX-001", uid, name, created, updated,      keyPath: code
           sealed, house, chip, enteredBy }                   index: house
items    { id, uid, box, full, thumb, full2, thumb2, state,   keyPath: id (auto)
           created, updated, title, category, material,       indexes: box, state
           colour, approx_size_cm, visible_text, date_on_item,
           condition, rarity_notes, book, coin, tags[], confidence }
settings { k, v }                                              keyPath: k
```

`state` is `pending` (awaiting identification), `working`, or `done`.
`full` and `thumb` are JPEG data URLs at 1024 px and 320 px.
`box.house` points at a `houses.id`. `box.chip` is a hex colour from the
8-swatch `PALETTE` in `index.html`, used for the printed label's colour bar
and the in-app chip — kept separate from `item.colour` (the object's own
dominant colour, set by the vision model) even though the names are similar.
`box.enteredBy` is a free-text name, typed once per phone under Setup and
stamped onto each box created there — no way to read an actual device name
from a web page, so this is manual, not inferred. Items don't carry `house`
directly; they're scoped to a house through their box, so moving a box
between houses (not yet exposed in the UI) would only need one write.

`item.rarity_notes` comes from the same Claude call as everything else -
one more field in `SCHEMA`, not a second API call, so it costs a few dozen
extra tokens on every photo rather than doubling the cost of any category.
`item.book` (`{title, author, year, isbn, cover, editionCount}` or `null`)
is additive metadata from Open Library, fetched only when `category` comes
back `"book"` - never merged into or overwriting Claude's own fields,
because Open Library's data isn't always reliable (a real ISBN looked up
during testing returned a completely mismatched, self-published record).
`worthLook(item)` in `index.html` combines `rarity_notes` with a free
heuristic (a book with very few known editions on Open Library, `editionCount
<= 5`) into a single "worth a second look" flag - neither signal is a real
appraisal, both are free, and genuine valuation stays a human step, same
answer as the paintings/antiques case in the roadmap below.

`item.coin` (`{n, title, issuer, minYear, maxYear, thumbnail,
reverseThumbnail, url}` or `null`) is the same additive pattern as
`item.book`, but unlike `item.book` it is **never** set as a side effect
of `classify()` - see "Numista search fields" below for the full
reasoning and for `q`/`year`/`size`. It's only ever written by one of:
- `runCoinSearch()`, automatically, ~1.5s after both sides of a coin are
  known (`coinSideShot`'s handler) - never on a single photo.
- The "Retry coin lookup" button (`renderResults()`, shown when
  `category==='coin' && !coin && state==='done'`) - a single quick pick
  of the top result, same as `runCoinSearch()`, for a single-photo coin
  the user wants a guess on without adding a second photo, or an item
  identified before a Numista key ever existed (`lookupCoin()` silently
  returns `null` with no key - a real thing that happened while building
  this, with no visible reason why until this button existed).
- The item detail view's own search, which shows up to 6 candidates to
  choose from (`searchNumista({...,count:6})`) rather than silently
  trusting Numista's top hit - a free-text search has no way to
  guarantee the first result is the right coin, and there was no way to
  tell if it wasn't before this existed.

Numista's Terms of Use require the N# catalogue number and "Source:
Numista" shown alongside any result derived from their data -
`renderResults()` always renders both, not optionally, and renders the
N# as an actual link to `item.coin.url` (`https://en.numista.com/{id}` -
confirmed live, not guessed: the API itself returns no URL field, and
the more obvious `/catalogue/pieces{id}.html` form 403s under a
non-browser user-agent and only reveals itself as a redirect to the
short form under a real one). `thumbnail`/`reverseThumbnail` (the API
returns both, unprompted) render next to that line specifically so the
user can visually confirm a match is actually the right coin before
trusting it, rather than having to click through to Numista's own site
to check. If nothing has matched yet, the item has no `coin` field and
sits in the pool `runImageSearchBatch()` (Setup tab, "Run image search
on unmatched coins") can search by photo instead, one paid batch call
per coin, cost shown and confirmed before it runs - see the decisions
table above for why this is manual and batched rather than automatic.
`item.full2`/`item.thumb2` are an optional second photo, same
shape as `full`/`thumb`, added after the fact via the "+ Add other side"
button (`renderResults()`, coin items only, shown once `state==='done'`
and no `full2` yet). One photo is always enough to save an item - this
is opt-in, not a second required step in the capture flow, which stays
exactly as fast as it was. Adding the second photo calls
`classify(id, true)` - the `force` param lets an already-`done` item be
re-sent, this time with both images in one message, so Claude can use
whichever face actually has the date/mint mark/condition detail rather
than guessing from one side. The response shape for `search_by_image` was not verified
against a real paid-plan account (the free plan returns 403
`NOT_ENTITLED`) - `searchCoinByImage()` in `index.html` is flagged
in-code as a best-effort guess at the shape, to be corrected the first
time the batch actually runs against a live paid account.

`updated` (on houses, boxes, and items) and `uid` (on houses, boxes, and
items) exist for sync, not the UI. `updated` decides which side wins when
two phones both have a copy of the same record. `uid` (`crypto.randomUUID()`)
is the identity sync actually matches on, because every store's local `id`
(or `code`, for boxes) is generated per-device and two phones will happily
produce the same one independently - `code` stays the human-readable label
and local primary key; `uid` is invisible to the user and is what the
OneDrive filename and cross-device matching are actually built on. Every
code path that changes an item must call `touchBox(item.box)` afterwards so
the *box's* `updated` reflects the change - sync only looks at the box's
timestamp, not each item's.

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

**One `walkCursor` pass per store, ever, per upgrade.** If two different
version blocks both need to touch e.g. `boxes`, do it as one combined
`walkCursor(t.objectStore('boxes'), b => { if(from<2){...} if(from<4){...} })`
covering everything relevant, not two separate calls in two separate
blocks. Two independent cursor passes over the same store within one
upgrade transaction race and the second one silently loses the first
one's writes - this actually happened (see the `from<2`/`from<4` box
block in `index.html` for the surviving pattern), was caught by testing a
real migration rather than assuming it worked, and cost real debugging
time to track down. Don't reintroduce it.

## Sync (OneDrive)

Three things sync, each its own file(s) in the app's OneDrive folder:
`salt.txt` (unencrypted, bootstraps the shared encryption key),
`houses.json` (one small encrypted file, every house, merged by `uid` with
last-write-wins on rename), and one `{box.uid}.box.json` per box (box
fields plus its items, photos included). Box files are named by `uid`,
not the human `code` - two boxes independently created with the same code
(a real, tested scenario, not hypothetical, now that there are two people
actively packing at two different houses) get two different files and
never overwrite each other. If a box arrives from sync with a code that
collides with a *different* local box (different `uid`, same `code`), the
incoming one gets renamed to the next free code on this device rather
than silently overwriting the box that happened to share it - see
`mergeRemoteBox()`. `syncHouses()` runs before the box loop each sync, so
the box loop always has an up-to-date `uid -> local house id` map to
resolve each box's real house through, rather than guessing.

Both phones sign into **the same dedicated Microsoft account**
(`frozenfas.apps@outlook.com`), not two separate personal accounts. That
one decision is what makes everything else about this simple:

- The OAuth scope is the narrow `Files.ReadWrite.AppFolder`, not full
  `Files.ReadWrite`. The app can only ever touch its own dedicated
  OneDrive folder (`/me/drive/special/approot` in Graph terms) - nothing
  else in the account, and there's nothing else in the account anyway
  since it holds nothing but this app's data.
- No manual "share this folder with the other person" step. Both phones
  are "the owner" of the same account's app folder, so there's no
  cross-account sharing to set up or get the ordering wrong on - the
  whole owner-vs-shared-with-me problem an earlier version of this had
  to solve doesn't exist here.
- The tradeoff, already made deliberately: a genuinely shared login
  between two people, rather than each keeping their own account. See
  the decisions table above for why that was chosen over the broader
  `Files.ReadWrite` scope on two separate accounts - the narrower scope
  was judged worth a shared password once it became clear that scope
  would otherwise reach the *entire* OneDrive, including anything
  unrelated already stored there.

Everything that reaches OneDrive is encrypted client-side first
regardless - see the `/* ---------------- encryption ---------------- */`
and `/* ---------------- OneDrive sync ---------------- */` sections in
`index.html` for the actual code and their own inline reasoning.

**Status: configured, not yet tested live.** `MS_CLIENT_ID` in `index.html`
is a real Azure app registration (personal Microsoft accounts only,
`consumers` authority, SPA redirect URI). The Azure app's API permissions
need `Files.ReadWrite.AppFolder`, not the broader `Files.ReadWrite` used
in an earlier version of this - if this is ever revisited, check the
Azure portal's API permissions blade matches `MS_SCOPES` in `index.html`.
The authorize request was confirmed to redirect cleanly to Microsoft's
real sign-in page with correct parameters carried through - as far as
this can be checked without completing an actual login, which needs a
real device and account. Everything that doesn't require live OAuth was
verified directly: encryption round-trips, a wrong passphrase correctly
fails closed instead of overwriting good remote data, and a full
push-then-pull cycle was tested against a mocked Graph API using the
`special/approot` addressing.

**Known limitations, deliberately scoped this way for v1, not oversights:**
- **Deletions don't sync.** Pulling a box only adds or updates items by
  `uid`; it never removes a local item just because the remote copy lacks
  it. That's what stops a pull from wiping out an item you added on this
  phone before it had a chance to sync - the alternative (treat the
  remote item list as authoritative) risks real data loss. The cost:
  delete an item on one phone, and it can reappear from the other phone's
  next push until real tombstones are built.
- **Whole-box last-write-wins, not per-field merge.** If both phones edit
  the *same* box while both are offline, the newer `updated` wins entirely;
  the older edit is just gone. Fine for how this app is actually used
  (pack into a box on one phone at a time); would need real conflict
  resolution to be safe for genuinely concurrent editing.

`getFile`/`putFile`/`listFiles` (in the OneDrive sync section of
`index.html`) surface the real Graph error body (`error.message` or
`error.code`) on failure, not just "Could not upload salt.txt" with no
detail - added after an intermittent, unreproduced real failure on that
exact call. A bare "Could not upload X" gives nothing to diagnose an
intermittent failure with; a 403 for the wrong scope and a 429 for
throttling need different fixes, and only the Graph body tells you
which one happened. If this fires again, the toast text itself now
says why.

## Numista search fields

Confirmed directly against Numista's real API docs (`en.numista.com/api/doc`
- bot-walled, needs a real browser, not `curl`, to load), not assumed. The
`/types` search endpoint's actual parameters: `q` (free text), `issuer` (an
internal *code*, not a free country name - would need a separate `/issuers`
lookup to resolve one to the other, not built), `catalogue`+`number`
(cross-reference lookup, not useful here), `ruler`/`material` (internal
IDs, same problem as issuer), `year` (as written on the item, or a range
like `"1800-1850"`), `date` (Gregorian year/range), `size` (mm, or a
range), `weight` (grams, or a range). At least one of
`q`/`issuer`/`catalogue`/`date`/`year` is required. **There is no
parameter to search by obverse/reverse lettering, script, or
denomination** - those exist only in the full type detail response
(`/types/{id}`), not as search filters, so "search by what's written on
the reverse" isn't something this API can do, only `q` free text can get
you there indirectly.

`searchNumista({q, year, size, count})` in `index.html` uses the three
search params that are actually practical here:
- `q` - `item.title`. `SCHEMA`'s `title` field instructs Claude to phrase
  a coin's title in Numista's own catalogue convention specifically
  (`"{denomination} - {design/ruler} - {issuer}"`, e.g. `"1 Cent -
  Victoria - Canada"`) rather than a natural description, on the theory
  that matching their own indexing style matters more than readability
  for this one field - confirmed that's genuinely their convention by
  reading a real catalogue entry (numista.com/7984), not guessed.
- `year` - `item.date_on_item`, a `SCHEMA` field for any date/year
  literally visible on the object (not coin-specific in the schema
  itself, in case it's ever useful for books or maker's marks too, but
  currently only coins do anything with it).
- `size` - `item.approx_size_cm`, already collected on every item for
  the catalogue regardless of Numista, converted to mm via `cmToMm()`.
  Free: no extra Claude field, no extra API call, just using data that
  already existed.

`lookupCoin(params)` is a thin wrapper over `searchNumista` that returns
only the first result (`count:1`) - used exactly where nobody is present
to choose one: `runCoinSearch()` (the automatic path, below) and the
quick "Retry coin lookup" button. The item detail view's own search
(Find tab, tap any coin, "Search again") calls `searchNumista` directly
with `count:6` and renders every candidate as a pick list instead -
Numista's top hit is not guaranteed to be the right coin for a free-text
search, and before this there was no way to even see whether it was
right, let alone correct it. All three params are user-editable there
too, since Claude can and does get the title wrong (a real
20-cent-euro-coin test matched the wrong design entirely).

**The search never runs automatically off a single photo.** `classify()`
identifies a coin (title, material, etc.) same as any other category,
but deliberately does not call any Numista function itself - searching
on one side's information only to search again once a second photo
shows up (a routine sequence, given a date or mint mark can be on either
face) wastes a free-tier request on a result about to be superseded.
Instead, `coinSideShot`'s handler calls `runCoinSearch()` ~1.5s after
the two-photo re-identification finishes - same "small local pause
before firing an external call" pattern as the initial capture debounce,
just applied a step later. A single-photo coin gets no automatic search
at all; "Retry coin lookup" or the detail view is how its match, if any,
gets found.

Once a search matches, opening that item's detail view separately
fetches the full type detail (`/types/{id}`, via `fetchCoinDetail()`) to
show the parts of the catalogue that aren't in the lean search result:
denomination (`value.text`), composition, and each side's own
`lettering`/`description` - lazily, only when a coin's detail view is
actually opened, not on every automatic match, so routine capture
doesn't pay for a second free-tier request per coin nobody asked to
inspect. `openDetailId` guards against that lazy fetch (or a stale
`searchNumista` candidate list) writing into the panel if the user
closes it or opens a different item before it resolves - a real race,
not hypothetical, given how long a network round trip can take against a
fast local re-render.

## Printing (labels and the scale card)

Both the label sheet and the scale card render into the same `#sheet`
element and share one `@media print` block. `@page` sets `size:A4
portrait;margin:0` deliberately - without an explicit `margin:0`, the
browser's own default print margin stacks on top of `#sheet`'s own `8mm`
padding, and that doubled-up margin was enough to push even a normal
4-label, 2-row sheet onto a second page (a real bug, found and fixed, not
hypothetical). `#sheet`'s `8mm` padding is the only margin that should
exist; A4 is forced explicitly rather than left to the browser/printer
default (which can be US Letter, a few mm shorter, on a US-configured
device) so the row-height math in the `.copies-N` rules stays correct
regardless of local printer defaults. If the row heights for `copies-2` /
`copies-4` / `copies-6` (or the scale card's ruler) ever change, re-check
that `rows * row-height <= 281mm` (297mm page - 16mm sheet padding) before
shipping - that's the actual budget, not 297mm.

The scale card's width budget is easy to get wrong the same way, because
of the page's global `*{box-sizing:border-box}` reset: `#sheet` has no
explicit `width` (so its content box is genuinely `210 - 2*8mm = 194mm`,
box-sizing doesn't affect an `auto` width), but `.scalecard` also fills
that 194mm as its own *outer* box and then loses its own `8mm*2` padding
from that before anything inside it gets to use the rest - the real
budget for `.cornerrow`'s contents is `194 - 16 = 178mm`, not 194mm. The
two `.lblock` corner rulers (`82mm` each, `10mm` gap = `174mm`) fit that
178mm budget with 4mm to spare - if `SCALE_LEN`/`SCALE_STRIP` in
`index.html` or the `.lblock` width in CSS ever change, re-derive this
the same way (subtract *every* nested element's own padding once, in
order, from the outside in) rather than assuming one padding subtraction
covers it - this was miscounted more than once while building it and
only caught by measuring the rendered layout, not by re-reading the CSS.

`#sheet, #sheet *{print-color-adjust:exact}` (plus the `-webkit-` prefix
for Safari) is there because browsers silently drop `background-color`
from printed pages by default, to save ink, regardless of what the
print dialog's own "Background graphics" toggle happens to be set to -
with no warning and no visual difference on screen. Without it, the
label's colour bar and the scale card's alternating segments both print
blank. This was a real bug (found by actually printing, not by
inspecting the page on screen - the two look identical) that predates
the scale card; the label's colour bar had almost certainly always had
it. Anything added to `#sheet` in future that depends on a background
colour to be legible needs this rule in scope, not a fresh one per
element.

## Reset & backup

Setup tab, "Danger zone": `resetAll` clears the `houses`, `boxes`, and
`items` stores only - never `settings`, so API keys, the sync passphrase,
and the OneDrive connection all survive a reset. Two gates before
anything is deleted: a `confirm()` describing the scope, then a forced
`exportJson()` + `exportPhotos()` (the same functions the manual "Export
catalogue"/"Export photos" buttons call) before a second `prompt()`
requires typing `DELETE` exactly. If sync is connected, a reset is
recoverable via the next `syncNow()` pull; if it isn't, the downloaded
backup is the only copy - the forced-backup step exists specifically for
that case, not as a formality.

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

1. Live-test OneDrive sync on both phones (see Sync above) - it's built
   and configured, just not yet tried with a real login on a real device.
   After that: proper deletion tombstones, the remaining known gap called
   out above (box-code collisions were found and fixed during this build,
   not left open).
2. Scan a printed label to jump to that box's contents. Needs jsQR, since
   Safari has no `BarcodeDetector`.
3. Edit an item's fields after identification. Still delete-only for
   most fields - the item detail view (tap any item on the Find tab)
   added one narrow exception: the text sent to Numista is editable and
   re-searchable there, because a wrong Claude title is otherwise a
   dead end for finding the right coin. General field editing (title,
   category, material, etc.) is still not built.
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
5. Category-specific enrichment behind one resolver interface. **Books:
   done** - Open Library lookup by title (auto) or ISBN (manual retry when
   the title search doesn't match), plus the `rarity_notes` /
   `worthLook()` flag described above. **Coins: done** - Numista lookup by
   title (auto, free plan), plus an optional paid batch image search for
   anything the text search missed (see "Coin lookup" in the Data model
   section above). Stamps and other catalogued collectibles via similar
   structured lookups where a free/affordable API exists. Paintings,
   sculpture, and furniture don't have an equivalent of an ISBN or
   catalogue number to look up, and no affordable structured database of
   maker's marks or auction records exists to query. For that category
   the realistic scope is not automated identification — it's flagging
   the item (e.g. low classifier confidence, or a visible signature/mark
   worth a closer look) so the human decides whether it's worth a real
   appraisal, not this app guessing at attribution or value.
6. Live-test the Numista batch image search against a real paid-plan
   account - the response shape in `searchCoinByImage()` is currently a
   best-effort guess (see the "Coin lookup" note above), since the free
   plan can only confirm the endpoint returns 403 `NOT_ENTITLED`, not what
   a successful response actually looks like.
7. Revisit the QR label design - deferred on purpose until the coin
   module, batch image search, scale card, and reset/backup features
   (all above) were done first.
