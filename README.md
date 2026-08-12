# Stockholm Housing

A single-file, dependency-free map view of my Stockholm rental search — every listing I've
contacted on one map instead of scrolling a message list. It started from my Qasa inbox and is
built to take listings from other sites alongside them.

**Live: https://0xxy-gen.github.io/Stockholm-Housing/**

Or open `index.html` in any browser. There is no build step and no server; map tiles are
fetched from OpenStreetMap, so an internet connection is required.

## Today

The map is the home screen; **Today** is one tab across and is where the daily work happens.
Four sections, in the order they need you:
**Needs your reply** (every listing where the landlord answered last), **Today**, **Coming up**,
and **Waiting on a time** (proposals with no hour agreed). Each row carries the listing, why
it's there, and one primary action — Open chat for a call or a reply, Directions for a visit.

## What it does

- **Map** — pan/zoom slippy map (OSM tiles) with one marker per listing, coloured by
  conversation status. Overlapping markers collapse into a count bubble whose ring shows the
  status mix inside it; click to zoom in, or to fan the pins out when they're stacked on the
  same spot.
- **Sidebar** — the same listings as a scrollable list with thumbnail, address, rent, size,
  price per m², status and source. Clicking a row flies the map to its marker, and vice versa.
- **Search** — free-text over address, area, contact and source.
- **Sort** — rent, size, price per m², status, or address.
- **Filters** — status, source, favourites only, max total rent, min size. The Filters button
  shows how many are active; "Clear all filters" resets them.
- **Price labels** — rents appear on the markers once you zoom in far enough.
- **Distance to Stockholm C** — a rough transit-time band (never a false-precision kilometre;
  hover for the straight-line figure it came from), sortable, plus a button that opens Google
  Maps transit directions for the real journey.
- **Dark mode** — follows your OS setting, map tiles included.

## Favourites and notes

The **Favourites** tab (top-left, next to Map) is a two-pane view: a scrolling list of starred
listings on the left, and the selected one in full on the right — large photo with thumbnails,
every field, links to the original ad and chat, a transit-route button, and the notes box.

Notes are written only when you press **Save note**. The button stays disabled until there is
something unsaved; the label next to NOTES reads "Unsaved" while you're editing and "Saved" once
written, and the left-hand list marks which listings carry a note. Switching to another listing
mid-edit keeps your unsaved text — it comes back when you return, still marked unsaved. The same
notes box appears on the map's detail card once a listing is starred.

Favourites and notes live in the browser's `localStorage`, so they're per-device and per-origin:
what you star on the live site is separate from what you star opening the file locally, and
neither is part of the repo. Un-starring keeps the note, so re-starring brings it back.

### Getting them onto another device

Two routes, because `localStorage` doesn't sync and Chrome won't do it for you:

- **Seeded from the data.** A listing carrying `seedFav` / `seedNote` / `vw` applies them once
  on first load, then the device owns them. This is how the laptop's favourites reach the phone,
  and the daily sweep keeps it current by writing browser state back into the data.
- **The Sync button** in the top bar. It shows a code containing everything this browser holds —
  favourites, notes, viewings, removals, dismissed to-dos. Copy it, paste it into the same box on
  the other device, and Apply. Importing **merges**: it only ever adds, never deletes, so pasting
  a stale code cannot wipe newer work.

### Why travel time is a link, not a number

Showing a real door-to-door transit time in the page needs a routing API (Google Directions,
or Trafiklab/ResRobot for SL), and any key would have to be embedded in this file — which is
public. So the app shows straight-line distance, which needs no key and is enough to sort by,
and hands the real journey off to Google Maps. Wiring up a live API is doable if the key is
kept off the public page (private repo, or a small proxy).

## Scheduled viewings

The **Viewings** tab lists booked viewings grouped by day, with "Today" / "Tomorrow" / "in N
days" labels and a count on the tab. Each entry shows the listing, its status and source, your
note for the visit, and buttons for transit directions to the address, the listing page, and
showing it on the map. Add or edit one from the tab's **+ Add viewing** button, or from the
viewings strip on any listing's detail card.

Viewings live in `localStorage` like favourites and notes. Listings can also carry a `vw:[]`
array in the data — each entry is merged in once and marked "from your messages", which is how
the daily inbox sweep records a time you agreed with a landlord without ever overwriting an
entry you edited by hand.

## Removing listings

Removing lives in the detail views only — **Remove this listing** on the map card and on the
favourite, behind a confirmation dialog. There is deliberately no × in the list rows: it sat a
few pixels from the favourite star, so a mis-click removed a listing instead of starring it.

Removing hides a listing from the map, the list, the counts and favourites — it does not touch
the data in the file. The keys of what you removed are kept in `localStorage`, and a
**"N listings removed · Restore all"** bar appears under the listing count to bring them back.

## Status colours

| Colour | Meaning |
| --- | --- |
| 🟠 Orange | Waiting on me (they replied last) — your move |
| 🟢 Green | Waiting on them — ball in their court |
| ⚪ Mauve | No reply yet |
| 🟡 Yellow | Favourite |

Pink carries the interface itself — buttons, focus rings, selection — so it never collides with
a listing status.

Closed and rented conversations are filtered out — they're dead ends for an active search.
They're still in the data; the line that drops them is at the top of the script block.

## Adding listings from other sites

Every listing carries a `src` field that drives the coloured source badge in the list and on
the detail card. Anything without one is treated as `"Qasa"`. Once more than one source is
present, a **Source** filter group appears automatically.

To add listings from elsewhere, fill in the `EXTRA` array near the top of the script block —
it's kept separate from the generated Qasa blob so re-exporting the inbox won't clobber it:

```js
var EXTRA = [
  {
    src:"Blocket", a:"Exempelgatan 1", ar:"Södermalm",
    lat:59.3145, lng:18.0723, tot:9500, rent:9500, fee:0,
    sq:"32", rm:"1", ty:"Apartment", c:"Anna", st:"No reply",
    lu:"https://www.blocket.se/annons/000000", im:[]
  }
];
```

Required: `a` (address), `ar` (area), `tot` (total monthly SEK), `st` (status), `src`.
Everything else is optional — omit `lat`/`lng` and the listing stays off the map but still
appears in a note under the list. Photo URLs in `im` may be full `https://` URLs, a path
beginning `img/` for a file committed next to the page, or a bare key resolved against Qasa's
image CDN. Images that fail to load hide themselves rather than showing a broken-image glyph. Known sources get a fixed badge colour, and unknown
ones are assigned a stable colour from a fallback palette.

## Data

Listing data is embedded directly in the HTML — a point-in-time snapshot, not a live feed.
Nothing is fetched from Qasa at runtime.

## On a phone

Below 760px the layout stacks: the map takes the top ~44% of the screen with the toolbar and
listing list scrolling beneath it, the detail card becomes a bottom sheet, favourites go
single-column, and the nav compacts (the wordmark drops below 420px).

The map takes touch: one finger drags, two fingers pinch to zoom, and `touch-action: none`
stops the browser hijacking the gesture to scroll the page. It is all one set of pointer
handlers, so mouse, trackpad and touch go through the same code.

## Not done yet

Markers aren't keyboard-reachable and carry no screen-reader labels; status is conveyed on the
map by colour alone.

## Credits

Map data © [OpenStreetMap](https://www.openstreetmap.org/copyright) contributors.
