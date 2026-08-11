# Stockholm Housing — working notes

Single-file rental map. Everything lives in `index.html`: styles, markup, the `DATA` array of
listings, and the script. No build step, no dependencies. Published to GitHub Pages from `main`,
so any push to `main` redeploys https://0xxy-gen.github.io/Stockholm-Housing/.

## Editing index.html

- `DATA` is one very long line. Edit it by parsing the JSON out, changing objects, and writing
  it back with `json.dumps(..., ensure_ascii=False, separators=(',',':'))`. Before rewriting,
  assert the round-trip reproduces the original byte-for-byte, so unrelated listings can't drift.
- Listings from sites other than Qasa go in the `EXTRA` array near the top of the script, not in
  the generated `DATA` blob — re-exporting the Qasa inbox would otherwise wipe them.
- After any script change run `node --check` on the script portion before committing.
- Verify in a browser via a local server (`python3 -m http.server`), not `file://` — the Chrome
  extension cannot open `file://` URLs.

## Per-listing fields

Required: `a` address, `ar` area, `tot` total monthly SEK, `st` status, `src` source site.
Optional: `lat`/`lng` (omit to keep off the map), `c` contact, `sq`, `rm`, `ty`, `rent`, `fee`,
`lu` listing URL, `cu` chat URL, `no` listing note, `im[]` photos, `el`, `net`, `pk`, `mi`,
`le`, `ext`.

Seed fields, applied once per listing then owned by the user: `seedFav`, `seedNote`, and
`vw:[{w:"YYYY-MM-DDTHH:MM", n:"note"}]` for viewings (merged per entry, so adding a new one
never disturbs existing entries).

`st` is one of `"Waiting on me"`, `"Waiting on them"`, `"No reply"`. Closed/rented listings are
filtered out at runtime.

## Browser state

Favourites, notes, viewings and removed listings live in `localStorage`, per-origin. Anything
that must reach every device belongs in the data file instead, via the seed fields.

When testing in the user's browser, clear any `qasaFav` / `qasaNotes` / `qasaViewings` /
`qasaSeeded` / `qasaDeleted` keys you wrote afterwards — that is the user's real data.

## Daily Qasa inbox sweep

A scheduled job reads https://qasa.com/se/en/messages and updates the map. Rules, in priority
order:

1. **Never send, reply to, or draft-send anything to a landlord.** Read only. Anything needing a
   reply goes in the summary for the user to handle.
2. **Never add new listings.** The sweep only updates listings already in `DATA`. A conversation
   with no matching entry gets named in the summary, not created. Listings are added only when
   the user explicitly asks.
3. **Never delete a listing** unless the landlord states the place is gone, and then say so
   clearly in the summary.
4. Only record a viewing when a specific date **and** time were actually agreed. Proposed but
   unconfirmed times go in the summary.
5. Match threads to listings on the `cu` chat URL where possible, else address + contact.
6. Before committing, confirm the listing count is exactly unchanged. If it moved, something was
   added or dropped — undo it.
7. If Chrome is not running, the extension is unreachable, or the inbox shows a login wall: touch
   nothing and report why. Never attempt to log in.

## Scope of autonomy

The user has granted full autonomy for this project — edit, commit and push without asking —
with the single exception of sending messages to landlords, which always requires their explicit
approval per message.
