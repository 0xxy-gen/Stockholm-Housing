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
4. Viewings come in two kinds and both belong in the data:
   - A specific date **and** time agreed → a booked entry (no `p`).
   - A viewing raised but not pinned down (a day without an hour, an open invitation, a
     landlord asking which day suits) → a proposed entry with `p:1`, `w` set to the day if
     one was named and `""` otherwise.
   When a previously proposed viewing gets a time, **update that entry in place**: set its
   `w` to the full timestamp and drop `p`. Do not leave a duplicate proposal behind. If the
   proposal falls through, remove it and say so in the summary.
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
