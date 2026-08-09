# Travel Admin Organizer

An Activepieces workflow that watches Gmail for booking confirmations and keeps one chronological
itinerary, buffered calendar events, and a reimbursement-ready expense sheet current per trip —
modifying, never duplicating, when a booking changes or is cancelled.

Built for a Gravity AI take-home. The design write-up in [DESIGN.md](DESIGN.md) is the substantive
part of this repo; `agent.json` is the artefact it describes.

## The problem

> Every trip scatters itself across my inbox — flight confirmation here, hotel there, cab receipt
> somewhere else. The night before I travel I'm searching my email at midnight for a confirmation
> number, and after the trip I lose money because I never compile the expenses.

## How it works

```
Gmail (subject: "Booking Confirmation")
  └─ prep        strip HTML, score candidate bodies, cut boilerplate, keyword gate   [free]
  └─ state_in    load saved trips from Storage (FLOW scope)                          [free]
  └─ home        the one question: "Which city do you travel from?"                  [free]
  └─ r_ai        ── is this a booking? ──▶ extract   structured extraction           [1 AI call]
  └─ core        identity · trip grouping · buffers · conflicts · itinerary          [free]
  └─ r_sheet     ── no sheet yet? ──▶ create it + headers
  └─ finalise    merge the new sheet id into state                                   [free]
  └─ r_row       ── something to log? ──▶ append the expense row
  └─ r_cal       ── future booking? ──▶ create the calendar event with its buffer
  └─ save        persist state
  └─ output      return the itinerary as text
```

16 steps, 4 of them code, exactly one AI call — and none at all on mail that fails the free gate.

## Design decisions worth naming

**A local ride is not a trip.** Around 25 Uber receipts in the target inbox are ordinary
Bengaluru commutes. A cab joins a trip only if it falls within ±1 day of another booking or starts
or ends outside the home city. Without this rule the agent manufactures a "trip" every few days and
buries the real ones. This is what the single setup question actually earns its place for.

**A receipt is not a reservation.** Uber mail arrives *after* the ride with the fare already
charged, so a past-dated booking is logged as an expense but never becomes a calendar event.

**Identity survives missing reference numbers.** A real Uber receipt carries a date, a fare, two
addresses and a licence plate — and no reference of any kind. Booking identity is
`confirmation + type`, falling back to a fingerprint of `type + start + origin + destination`.
A second fingerprint layer catches the travel agency's copy of a flight you already have, which is
the duplicate the brief calls out first.

**Nothing is created until a booking is real.** A booking needs a date, a destination and an
identifier before any trip, sheet, row or event is created — so a newsletter or a social
notification produces nothing at all, and there are no placeholder trips.

**Boilerplate is cut before the model sees anything.** A real IRCTC confirmation is ~3,200
characters of which roughly 900 matter; the rest is safety notices and adverts. The normalise step
cuts at the first boilerplate marker but never before character 200, so short mail is never
truncated to nothing.

## Using it

1. Import `agent.json` into Activepieces.
2. Connect Gmail, Google Sheets and Google Calendar — the placeholders `GMAIL`, `GOOGLE_SHEETS`
   and `GOOGLE_CALENDAR` will resolve to your own connections.
3. Publish. The expense sheet is created on the first booking; nothing needs building by hand.

## What I'd do next

- **Restore the per-trip Google Doc.** The lean rebuild dropped it for step count, and that was the
  wrong trade — the brief asks for "one document in my drive" and it was the reviewers' favourite
  part of the earlier build. `piece-google-docs` has no update action, so it needs
  `custom_api_call` → `documents.batchUpdate` (see [DESIGN.md](DESIGN.md) §11).
- **Notify on unreadable confirmations.** They're surfaced in the output; they should also reach the
  user directly.
- **Learn corrected buffers.** State stores `calendarStartWeSet` so a hand-moved event can be
  detected and the delta remembered — the mechanism to read the event back isn't built.
- **Exercise change and cancellation end-to-end.** The paths exist; they haven't been demonstrated.

## Notes

- [DESIGN.md](DESIGN.md) — the design, and the Activepieces behaviours that cost the most time to
  find (every step reference is wrapped in `{output: …}`; `maxOutputTokens` truncates structured
  output mid-JSON rather than erroring; a step is invalid unless `propertySettings` names every
  property, including empty ones).
- [NOTES-postmortem.md](NOTES-postmortem.md) — eleven scored submissions and what the numbers
  actually proved, including the ones where I was wrong.
- [fixtures/test-emails.md](fixtures/test-emails.md) — the test set, built from real mail wherever
  possible, because synthetic fixtures are tidier than real ones and that is where parsing bugs hide.
