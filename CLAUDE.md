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
`vw:[]` for viewings, merged per entry so adding one never disturbs existing entries.

A viewing entry is `{w, n, p, k}`:
- `k` — `"call"` for a video call, `"visit"` for going to the address. Calls hide directions,
  distance and the map button, since none of it applies. Leave unset when it is genuinely
  undecided.
- `w` — `"YYYY-MM-DDTHH:MM"` when a time is agreed, `"YYYY-MM-DD"` when only a day is,
  `""` when neither. `n` is the note. `p:1` marks it **proposed**.
- Booked (no `p`) entries appear under **Booked**, grouped by day. Proposed entries appear
  under **Proposed**, dated ones under their day and undated ones under "No date yet".
- Entries are keyed by date **and** proposed-ness, so a proposal can later be joined by a
  booked entry for the same day without either being lost.

`todo` is a one-line action the user owes a listing that is not simply "reply" — it surfaces in
a **To do** section at the top of Today. `todoWa` is a phone number that turns the row's primary
button into a WhatsApp link. Ticking a to-do off is stored per browser; never clear `todo`
during a sweep, and only add one when the user asks for it.

`st` is one of `"Waiting on me"`, `"Waiting on them"`, `"No reply"`. Closed/rented listings are
filtered out at runtime.

## Layout

One breakpoint at 760px (plus a 420px tweak) stacks the map above the list, turns the detail
card into a bottom sheet and makes favourites single-column. The map uses pointer events, not
mouse events, so touch and mouse share one path — keep it that way when adding interactions.

## Travel time to Stockholm C

`mins` on a listing is a **measured** door-to-door public-transport time to Stockholm C. Where it
exists the UI prints it exactly ("21 min"); where it does not, the UI falls back to a
straight-line estimate always written with a `~`, and the Ranking view's Travel factor uses the
same fallback. Never write a `~` estimate into `mins` — the tilde is the promise that a bare
number was actually measured.

Measuring one, in the user's Chrome:

1. Navigate to
   `https://www.google.com/maps/dir/?api=1&origin=<lat>,<lng>&destination=Stockholm%20Centralstation&travelmode=transit`
   and wait ~7s.
2. Take the left panel (the widest tall element with `left < 450`), find `"Copy link"`, and read
   the **first** duration AFTER it. That is the first transit itinerary. Match
   `/(?:(\d+)\s*hrs?\s*)?(\d{1,3})\s*min/` — Google writes "1 hr 3 min", so a pattern that only
   allows `h` silently records that as 3.
   - Do **not** take the smallest number on the page. The mode bar above `"Copy link"` shows
     driving, walking and cycling times too, and driving is always fastest — reading it would
     silently record a car journey as a transit time.
3. Store as an integer `mins`.

**Pin the departure time to 09:00.** The `?api=1` URL always means "leave now", which at night
returns night buses and roughly doubles the real figure. Use the long form instead, which does
carry a departure timestamp:

```
https://www.google.com/maps/dir/<lat>,<lng>/Stockholm+C,+Centralplan+15,+111+20+Stockholm/data=!3m2!1e3!4b1!4m13!4m12!1m0!1m5!1m1!1s0x465f9d60f602f26d:0x97c553dab672ba3d!2m2!1d18.0582377!2d59.3301476!2m3!6e0!7e2!8j<EPOCH>!3e3
```

`!8j<EPOCH>` is read as **naive local time**: pass the epoch of 09:00 UTC on the target weekday to
get 09:00 Stockholm. Confirm the panel reads "Depart at" and a 9-something departure before
trusting the number. Note `!2m3!6e0!7e2` must precede it — `!8j` appended after `!3e3` is dropped.

Do about 15 listings per sweep, preferring ones with a live conversation, a viewing or a
favourite, and re-measure anything older than a month.

## Browser state

Favourites, notes, viewings, removed listings and dismissed to-dos live in `localStorage`,
per-origin and per-device. Chrome does not sync it, so anything that must reach the phone
belongs in the data file instead, via the seed fields.

### Syncing browser state back into the data

Every sweep, after updating statuses, promote the live site's browser state into `DATA` so all
devices converge. Read `localStorage` on https://0xxy-gen.github.io/Stockholm-Housing/ and:

- `qasaFav` → set `seedFav:1` on every favourited listing, and **remove** `seedFav` from any
  listing no longer favourited. Otherwise un-starring on the laptop leaves the phone starred.
- `qasaNotes` → set `seedNote` to the current text; remove it when the note is gone.
- `qasaViewings` → merge any hand-added entry into that listing's `vw`, matching on date and
  proposed-ness so nothing duplicates.
- `qasaDeleted` → do **not** delete anything. Name the listings in the summary and let the user
  decide; removal from the data is theirs to ask for.

The laptop is the source of truth for this. Changes made on the phone are not visible to the
sweep and will be overwritten by it — say so if it ever matters.

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
4. Viewings come in two kinds and both belong in the data:
   - A specific date **and** time agreed → a booked entry (no `p`).
   - A viewing raised but not pinned down (a day without an hour, an open invitation, a
     landlord asking which day suits) → a proposed entry with `p:1`, `w` set to the day if
     one was named and `""` otherwise.
   When a previously proposed viewing gets a time, **update that entry in place**: set its
   `w` to the full timestamp and drop `p`. Do not leave a duplicate proposal behind. If the
   proposal falls through, remove it and say so in the summary.
   **Never rewrite `w` on an entry that has already been seeded into browsers.** Seeded entries
   are keyed by date and proposed-ness, so changing the date orphans the browser's copy and the
   viewing then shows twice on the user's device while looking correct in the data. A proposal
   whose day has passed keeps its date — the proposal really was for that day — and the note
   carries the fact that it lapsed.
5. Match threads to listings on address + contact. The `cu` chat URLs exported from Qasa do
   NOT match the real threads, so never navigate by them — the UI routes Qasa chat links to
   the inbox root instead of to a confidently wrong conversation.
6. Before committing, confirm the listing count is exactly unchanged. If it moved, something was
   added or dropped — undo it.
7. If Chrome is not running, the extension is unreachable, or the inbox shows a login wall: touch
   nothing and report why. Never attempt to log in.

## Scope of autonomy

The user has granted full autonomy for this project — edit, commit and push without asking —
with the single exception of sending messages to landlords, which always requires their explicit
approval per message.
