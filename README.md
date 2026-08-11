# Stockholm Housing

A single-file, dependency-free map view of my Stockholm rental search — built from my Qasa
inbox so I can see every listing I've contacted on one map instead of scrolling a message list.

**Live: https://0xxy-gen.github.io/Stockholm-Housing/**

Or open `index.html` in any browser. There is no build step and no server; map tiles are
fetched from OpenStreetMap, so an internet connection is required.

## What it does

- **Map** — pan/zoom slippy map (OSM tiles) with one marker per listing, coloured by
  conversation status.
- **Sidebar** — the same listings as a scrollable list with thumbnail, address, rent, size and
  the landlord's first name. Clicking a row highlights its marker, and vice versa.
- **Filters** — toggle each status on/off, show favourites only, and cap results with a
  max-total-rent slider.
- **Price labels** — rents appear on the markers once you zoom in far enough.

## Status colours

| Colour | Meaning |
| --- | --- |
| 🟠 Orange | Waiting on me (they replied last) |
| 🔵 Blue | Waiting on them |
| ⚪ Grey | No reply yet |
| 🔴 Red | Closed / rented |
| 🟡 Yellow | Favourite |

Listings whose page has been removed have no coordinates and can't be placed on the map;
they're listed in a note at the bottom of the sidebar.

## Data

Listing data is embedded directly in the HTML — it's a point-in-time snapshot of my own
inbox, not a live feed. Nothing is fetched from Qasa at runtime.

## Credits

Map data © [OpenStreetMap](https://www.openstreetmap.org/copyright) contributors.
