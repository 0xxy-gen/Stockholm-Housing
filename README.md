# Stockholm Housing

A single-file, dependency-free map view of my Stockholm rental search — every listing I've
contacted on one map instead of scrolling a message list. It started from my Qasa inbox and is
built to take listings from other sites alongside them.

**Live: https://0xxy-gen.github.io/Stockholm-Housing/**

Or open `index.html` in any browser. There is no build step and no server; map tiles are
fetched from OpenStreetMap, so an internet connection is required.

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
- **Dark mode** — follows your OS setting, map tiles included.

Favourites are stored in the browser's `localStorage`, so they're per-device and survive
reloads but aren't part of the file.

## Status colours

| Colour | Meaning |
| --- | --- |
| 🟠 Orange | Waiting on me (they replied last) |
| 🔵 Blue | Waiting on them |
| ⚪ Grey | No reply yet |
| 🟡 Yellow | Favourite |

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
appears in a note under the list. Photo URLs in `im` may be full `https://` URLs; bare paths
are resolved against Qasa's image CDN. Known sources get a fixed badge colour, and unknown
ones are assigned a stable colour from a fallback palette.

## Data

Listing data is embedded directly in the HTML — a point-in-time snapshot, not a live feed.
Nothing is fetched from Qasa at runtime.

## Not done yet

The map is mouse-only: no touch panning or pinch-zoom, and the layout assumes a wide window,
so it isn't usable on a phone. Markers aren't keyboard-reachable and carry no screen-reader
labels.

## Credits

Map data © [OpenStreetMap](https://www.openstreetmap.org/copyright) contributors.
