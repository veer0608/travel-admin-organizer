# Travel Admin Organizer — design

One sentence: **a booking confirmation lands in Gmail; the agent parses it, files it into the
right trip by date and destination, and keeps one Google Doc itinerary, one buffered calendar
event, and one expense row current for that booking — modifying, never duplicating, when the
booking changes or is cancelled.**

Scoring is Quality 50 (an AI reads the final output) /
Reliability 25 / Speed 12.5 / Cost 12.5.

---

## 1. Trigger

**Gmail — New Email (polling), unfiltered.** One flow, one trigger. The brief says "detected
automatically within minutes, never forwarded by me", so polling must be tight and the user is
never in the loop.

**Corrected after checking the live piece.** The trigger has no free-form search-query field. Its
only props are `subject`, `from`, `to`, `label` (dropdown) and `category` (static dropdown:
Primary, Social, Promotions, Updates, Forums, **Reservations**, **Purchases**). A boolean keyword
query is not available.

Category looked like the elegant answer — Gmail's own classifier, free, vendor-agnostic, never
goes stale. It isn't, for two reasons found in the real inbox:

1. **The travel mail is split across two categories.** IRCTC confirmations (35) and Uber receipts
   (15) are in **Updates**. Accommodation — Stanza Living, goSTOPS, PerkUp, SPOT ON, Agoda,
   MakeMyTrip — is in **Reservations** (41). Purchases is shopping, and notably contains the IRCTC
   insurance decoys. One trigger takes one category, so any choice drops a whole booking type.
2. **Synthetic test mail lands in Primary.** A plain email sent from a second Gmail account is not
   classified as a reservation by Google. A category filter would mean the change, cancellation
   and ambiguity fixtures never fire at all — the test set would silently do nothing.

So: **no filter on the trigger, and the keyword gate moves into the Code step** that already
normalises the email. The funnel keeps its shape — a free deterministic filter before any AI —
it just runs one step later. The gate matches on subject and body keywords, not sender domains,
so it works on both real vendor mail and the fixtures.

Cost of this choice: the flow wakes on every incoming email. A non-match exits after two free
steps and never reaches an AI call, so the score is unaffected — but it does consume Activepieces
tasks while testing. If credit burn becomes a problem mid-build, set `category: Primary`
temporarily so only the fixtures fire, and clear it before publishing.

## 2. Questions — one

Run every candidate through the seven "don't ask" tests in NOTES.md and almost everything dies:

| Candidate | Verdict |
|---|---|
| Your email address | The Gmail connection knows it — test 1 |
| Which sheet to log expenses in | The agent creates it — test 2, and the brief demands it |
| Where to put the itinerary doc | The agent creates it — test 2 |
| Which trip a booking belongs to | The agent works it out — test 3, and the brief demands it |
| Flight buffer length | Sensible default, then learned from your corrections — test 7 |
| Home currency for totals | **Removed** — the brief says amounts stay in the currency charged |
| How often to check | The trigger's own polling — test 4 |
| Trip grouping window | Internal to the flow — never ask about your own flow |

One survivor:

> **"Which city do you travel from?"** — description: *"Your home base, so we know which leg is the
> outbound and how early to get you moving. e.g. Bengaluru"*

It is genuinely underivable on the very first booking, and it drives both the outbound/return
distinction and the leave-home buffer. One question against a limit of four is a design decision
worth stating out loud in the submission.

**Implementation trap:** the easy way to create questions is a Web Form trigger, but the trigger
here is Gmail and one flow gets one trigger. So the question is a **Storage Get step** with a key
nothing else writes, the answer in **Default Value**, "Ask user" toggled on, and the step named in
plain English. Their docs call this "a constant with an askable field". If the value is read in
several places, use **Bind** rather than a second question.

Clear the test value before publishing — whatever sits in the field becomes every user's
pre-filled answer.

## 3. What the agent creates for itself

Nothing is ever handed to the user to build.

**The expense sheet**, on first run, if `state.sheetId` is absent: create it, write the header row
and one clearly-marked example row, store the id, and tell the user the link with one line on what
it's for.

`Date · Trip · Trip type · Type · Vendor · Description · Category · Confirmation · Amount · Currency · Status · Refund · Booked on · Email link`

There is deliberately **no converted-amount column.** The brief is explicit: amounts stay in the
currency they were charged in. Inventing an FX rate would make the log dishonest and unusable for
a real claim.

`Status` is `active` / `changed` / `cancelled`. A cancelled booking is marked, never deleted, so
the claim history stays auditable and totals can exclude it honestly. `Refund` starts empty and
becomes `refund pending` the moment a booking is cancelled — the brief asks for cancellations to
be flagged for refund tracking, and a reimbursement claim is exactly where an unrefunded
cancellation gets lost. `Trip type` is `personal` or `work`, which is what makes the sheet
filterable into a claim you can actually submit.

**The itinerary document**, one Google Doc per trip, created on that trip's first booking. The doc
id lives in trip state and every later booking rewrites that same doc. This is where "modify,
never duplicate" is easiest to get wrong — a second doc named "Delhi trip (1)" is the failure.

## 4. Memory — Storage piece, FLOW scope

RUN scope is wiped every run, so FLOW scope is not optional here. One key, one state object,
written once at the end of every run:

```json
{
  "version": 1,
  "sheetId": "...", "sheetUrl": "...",
  "processedMessageIds": ["..."],
  "unreadable": [
    { "messageId": "...", "subject": "...", "from": "...", "receivedAt": "...",
      "reason": "image-only attachment", "reportedAt": "..." }
  ],
  "pendingTripQuestion": {
    "bookingKey": "...", "threadId": "...", "askedAt": "...",
    "candidates": ["trip_2026_08_delhi", "trip_2026_08_mumbai"]
  },
  "preferences": {
    "buffers": { "domesticFlight": 120, "internationalFlight": 210, "train": 45, "transit": 60 },
    "learned": { "domesticFlight": { "minutes": 150, "observedAt": "..." } }
  },
  "bookings": {
    "<bookingKey>": {
      "type": "flight|hotel|train|cab|other",
      "vendor": "IndiGo", "confirmation": "ABC123", "agencyRefs": ["MMT7781234"],
      "fingerprint": "flight|2026-08-14T09:50|BLR|DEL",
      "start": "2026-08-14T09:50:00+05:30", "end": "2026-08-14T12:35:00+05:30",
      "originCity": "Bengaluru", "destinationCity": "Delhi",
      "address": "...", "terminal": "T2",
      "amount": 8420, "currency": "INR", "category": "Airfare",
      "tripId": "trip_2026_08_delhi",
      "calendarEventId": "...", "calendarStartWeSet": "2026-08-14T07:50:00+05:30",
      "sheetRow": 7, "status": "active", "refund": "", "revision": 2,
      "sourceMessageId": "...", "lastUpdated": "..."
    }
  },
  "trips": {
    "trip_2026_08_delhi": {
      "name": "Bengaluru → Delhi", "destination": "Delhi",
      "tripType": "work", "tripTypeConfidence": "high",
      "start": "2026-08-14", "end": "2026-08-18",
      "docId": "...", "docUrl": "...", "bookingKeys": ["..."]
    }
  }
}
```

**`bookingKey` is the identity that makes "modify, never duplicate" work:** normalised
`confirmation number + type`. The same key arriving again is an update, never an insert.

**Except that some bookings have no confirmation number at all.** A real Uber receipt carries a
date, a fare, two addresses and a licence plate — and no reference number of any kind. So the key
falls back to a fingerprint of `type + start datetime + pickup address` when no confirmation
exists. Designing around "every booking has a reference" would have looked fine until the first
cab receipt arrived. Every
booking also carries the ids of its sheet row, calendar event, and trip doc, so a change updates
three things in place instead of creating three new ones.

`calendarStartWeSet` is what makes buffer learning possible — see §6.

Cap `processedMessageIds` at the most recent ~200 so state doesn't grow without bound.

## 5. Flow

```
1  Gmail: New Email (search query above)                    [trigger]
2  Code: normalise                                          [free]
       strip HTML, drop quoted replies and signatures, keep subject +
       sender + date + first ~2000 chars of plain text
3  Storage Get: travel_state (FLOW)                         [free]
4  Code: seen this message id already?                      [free]
5  Router: already seen → stop (no AI spend on a re-poll)
6  Storage Get: "Which city do you travel from?"            [the one question]
7  Router: is there a pendingTripQuestion?                   [§6, personal vs work]
       yes → Gmail: search that thread for the user's reply
           → Code: apply the answer, clear the pending question
8  Classify Text (small model): new booking / change /
       cancellation / not travel / unreadable
9  Router: "not travel" → save state, stop
10 Router: "unreadable" → Code: record it in state.unreadable
       → skip to the reporting steps (never silently dropped, §6)
11 Extract Structured Data (small model): typed fields
       type, vendor, confirmation, agency ref, start, end, origin,
       destination, address, terminal, amount, currency, category,
       isInternational, work/personal signals
12 Calendar: Get Event for this trip's existing events       [buffer learning, §6]
13 Code: the deterministic core                              [free — §6]
       duplicate detection · booking key · trip assignment ·
       personal/work inference · buffer calculation ·
       learned-preference update · transit gaps · conflicts ·
       chronological itinerary assembly
14 Router: first run and no sheet yet
       → Sheets: Create Spreadsheet + headers + example row
       → Code: store sheetId
--- Publish Gate: Continue Only If Published ---
15 Router on status:
       duplicate → do nothing at all (already captured)
       new       → Sheets insert row · Calendar create event
       changed   → Sheets update row · Calendar update event
       cancelled → Sheets mark cancelled + refund pending ·
                   Calendar delete event
16 Router: trip doc exists?
       no  → Docs: Create Document, store docId on the trip
       yes → Docs: replace the document body with the rebuilt itinerary
17 Router: trip type genuinely ambiguous?
       yes → Gmail: send the one-line question, store pendingTripQuestion
             (non-blocking — answered on a later run at step 7)
18 Storage Put: updated state
19 Ask AI (capped ~80 words): the "what to know" paragraph +
       conflicts phrased in plain language
20 Code: assemble the final itinerary markdown from state    [free]
21 Gmail: send the itinerary to the user
22 Code: OUTPUT — return the itinerary text                  [last step, main path]
```

Step 22 exists for one reason: **the grader reads the final output.** Ending on step 21 returns a
Gmail message id, and the 50-point quality score would be rating an id. Never end on Gmail,
Sheets, Docs, or Calendar.

**Why step 17 asks by email instead of pausing the run.** Gravity's docs bless a "reply
confirmation" pattern for exactly this — needing information rather than permission — and its
usual form pauses the run until the reply lands. Do not do that here. A paused run produces no
final output, and the reviewer sees exactly one run. Asking at the end and reading the answer on a
*later* run keeps every run complete, satisfies the brief's "asks me once", and costs nothing but
one line of state.

**No approval gate anywhere, and the brief confirms this is correct** — "it runs freely, it never
contacts an airline, hotel, or anyone else." Reading your mail and organising your own documents,
calendar, and sheet is internal work, and their docs say internal work runs free. An approval
email here would add a failure point and annoy the user for nothing.

## 6. The deterministic core (Code steps, no AI)

Everything here is exact, instant and free. Resist the urge to ask a model.

**Duplicate detection — two layers, because the confirmation number is not always the same.**
The brief calls out "a confirmation plus the travel agency's copy". MakeMyTrip's email carries
*its* booking reference, not the airline PNR, so matching on confirmation number alone would log
the same flight twice and put two events on the calendar. So:

1. **Exact:** normalised `confirmation + type` matches an existing `bookingKey`, or the incoming
   reference appears in that booking's `agencyRefs`. → update, never insert.
2. **Fingerprint:** `type + start datetime (±15 min) + origin + destination` matches an existing
   booking. → same booking arriving from a second sender. Merge: keep the first booking, add the
   new reference to `agencyRefs`, prefer the airline's own value as the displayed confirmation
   number, and do nothing to the calendar or the sheet.

Layer 2 is what actually satisfies the brief. Without it the agent looks correct in testing —
where you send each confirmation once — and double-logs the moment it meets a real inbox.

**Personal vs work, inferred first and asked only as a last resort.** Signals, in order of
strength: which of the user's addresses the confirmation was sent to; whether a company name or
GST/tax id appears in the booking; conference, summit, or client keywords in the subject;
weekday-only vs weekend-spanning dates; and the trip type of any booking already in that trip —
once a trip is typed, later bookings inherit it. Only when the signals genuinely conflict does
step 17 ask, in one line, and the answer is remembered for that trip forever.

**A confirmation it cannot read.** An image-only attachment or a locked PDF is not something the
agent can parse — the platform does not support file upload or OCR, so pretending otherwise is
how you get a silently missing booking. Detect it (classifier returns `unreadable`, or the body
has almost no text and the mail carries an attachment), record subject, sender and a link in
`state.unreadable`, and name it in the output: *"Couldn't read: 'Your e-ticket' from
noreply@airindia.in — the details are in an image attachment. 2 min to add by hand."* The brief's
requirement is that it never skips one silently, and this is the whole of it.

**A local ride is not a trip.** This is the rule that keeps the agent from embarrassing itself.
Roughly 25 Uber receipts sit in this inbox and nearly all of them are ordinary Bengaluru commutes
— Bellandur to Silk Board, ₹63, thirteen minutes. Treating each as a booking would manufacture a
"trip" every few days and bury the real ones. So a cab joins a trip only when it is **within ±1
day of another booking in that trip**, or **its pickup or drop is outside the home city**.
Otherwise it is a local ride: not a trip, no calendar event, no itinerary line.

This is where the single question earns its place. "Which city do you travel from" looked like it
was for outbound/return direction; its real job is separating travel from commuting.

**A receipt is not a reservation.** Uber emails arrive *after* the ride, with the fare already
charged. Creating a future calendar event for something that already happened is nonsense. So if a
booking's start time is in the past at the moment it is ingested, log it to the expense sheet and
place it in the itinerary history, but do not create a calendar event. The brief wants the expense
sheet done *after* the trip and the calendar right *before* it — those are different jobs and the
same email should not do both.

**Trip assignment — by dates *and* destination**, as the brief specifies. For a new booking `B`:
join an existing trip `T` when `B.start` falls within `T`'s span widened by 2 days **and** `B`'s
city matches `T.destination`, `T`'s origin, or a city already in `T`. Date proximity alone would
merge two different trips in the same week; destination alone would merge January and December
Delhi trips. Requiring both is what makes grouping trustworthy. Otherwise create a trip named
`<destination> — <Mon YYYY>`.

**Buffers — sensible defaults, then learned.**

| Booking | Calendar event |
|---|---|
| International flight | starts departure − 3h30, ends arrival + 1h00 |
| Domestic flight | starts departure − 2h00, ends arrival + 0h30 |
| Train | starts departure − 0h45 |
| Hotel | check-in at stated time (default 14:00), checkout 11:00 |
| Cab / transfer | as booked |
| Between two items same day | flag if the gap is under 60 min |

**Learning a corrected buffer.** When we create an event we store `calendarStartWeSet`. On a later
run we read that event back; if its actual start no longer matches what we set, the user moved it
by hand. The delta becomes the learned preference for that booking type, saved in
`preferences.learned` and used for every future booking of that type. Clamp to 0–6 hours so one
stray drag doesn't poison the defaults, and say so in the output the first time it happens
("noted — you leave 2h30 for domestic flights, using that from now on"). This is the brief's "if I
correct one once, it remembers my preference", and it is the cheapest kind of memory to build:
no AI, one comparison.

**Conflict detection.** Chronologically sort the trip's bookings, then flag:
- hotel checkout later than the next departure event starts
- a connection gap under 90 min international / 60 min domestic
- a transit gap under 60 min between any two same-day items
- overlapping hotel stays
- a night between arrival and departure with no accommodation booked
- a booking whose start is in the past but was never marked complete

## 7. Final output

The shape the grader sees:

```
# Bengaluru → Delhi · 14–18 Aug 2026
Updated 31 Jul, after the IndiGo 6E-2134 confirmation · full itinerary: <doc link>

## What to know
<= 80 words, plain language, conflicts stated first

## Itinerary
Fri 14 Aug
  06:20  Leave home — traffic buffer
  09:50  6E-2134  BLR T2 → DEL T3   PNR ABC123
  14:00  Check in — The Claridges, 12 Aurangzeb Rd   Conf HTL9921
...

## Needs your attention (2)
- Hotel checkout is 11:00 on 18 Aug but your flight leaves 09:15.
- No accommodation booked for the night of 16 Aug.

## Couldn't read (1)
- "Your e-ticket" from noreply@airindia.in, 29 Jul — details are in an image
  attachment, so nothing was logged for it. <email link>

## Expenses — work trip
₹48,320 across 3 bookings · $220 across 1 booking · <sheet link>
1 cancelled booking excluded, refund pending.
```

Note the totals: **one subtotal per currency, never a merged number.** The brief calls for honest
sums of what was captured, so bookings with an unparseable amount are counted as
"2 bookings with no amount captured" rather than silently treated as zero.

## 8. Edge cases to build for

- **First run** — no sheet, no doc, no state, empty `bookings`.
- **Re-poll of the same email** — caught at step 4 before any AI spend.
- **Airline resends the same confirmation** — same `bookingKey`, treated as an update, no duplicate.
- **Travel agency's copy of a booking you already have** — different reference number, same flight;
  caught by the fingerprint layer, merged into one booking, one event, one row.
- **Modification** — new times under the same PNR; row updated, the *existing* calendar event moved
  rather than a second one created, doc rewritten.
- **Cancellation** — calendar event deleted, itinerary line struck through and marked cancelled,
  sheet row marked with refund pending, excluded from totals, kept in history.
- **Unreadable confirmation** — image-only attachment or locked file: recorded and named in the
  output, never silently skipped.
- **Personal and work trips in the same week** — separated by the signals in §6; asked once, in one
  line, only when the signals genuinely conflict, and remembered for that trip afterwards.
- **Marketing email that reads like a booking** ("Book now! Confirm your seat") — killed by the classifier.
- **Foreign currency** — stored and totalled in the currency charged, never converted.
- **A booking with no parseable amount** — logged with a blank amount and surfaced as uncaptured.
- **Extraction returns junk** — if `start` won't parse, log the booking as `needs review` and say so
  in the output rather than writing a garbage calendar event.
- **Booking for a trip whose doc was deleted by the user** — recreate and re-store the id.
- **Nothing to do** — save state and end with an honest short message. Never a blank output.

## 9. Cost and speed budget

**Revised after measuring the real thing.** Two AI calls per qualifying email, not three.

**The Classify Text step is gone.** Its job was to reject non-travel mail, and the free keyword
gate in the Code step already does that — it correctly rejects the IRCTC insurance mails, which
are the hardest real decoys in this inbox, at zero cost. Whether a booking is *new*, a *change* or
a *cancellation* is then just another field the extractor returns. Merging the two removes an
entire model round trip from every run, which is both cost and speed, and those are 25 of the 100
points.

**Boilerplate is cut before the AI sees anything.** A real IRCTC confirmation is ~3,200 characters
of which roughly 900 matter; the rest is safety notices, a "Book Hotels" advert and cancellation
instructions. The normalise step cuts at the first boilerplate marker (`must read`,
`passengers are advised`, `book hotels`, `this document is issued on request`, `unsubscribe`, …)
but never before character 200, so a short email is never truncated to nothing. Measured on the
reschedule fixture: 474 characters reach the model. URLs are stripped first — marketing mail is
mostly tracking links, and that is what actually blows a small model's context.

So the run cost is: one extraction on ~500–900 trimmed characters, and one capped 80-word summary.
Everything else — identity, grouping, buffers, conflicts, totals, itinerary assembly — is
deterministic Code at ~66 ms and zero credits. A non-booking email exits on the free gate. A
re-polled email exits on the seen-check.

Use a small model for extraction; it is mechanical. Cap output length on every AI step.

## 10. Test plan

1. Find 3–4 **real** past confirmations in your Gmail (flight, hotel, cab) and forward them to
   yourself so they arrive as new mail. Forwarding is a *testing* device only — the brief is
   explicit that a real user never forwards anything. Note the side effect: a forwarded mail comes
   from **you**, not from IndiGo, so a search query that leans on `from:` will miss your own test
   mail. Keep the subject-keyword half of the query doing the real work, and confirm the trigger
   also fires on at least one genuine unforwarded confirmation before you submit.
2. Watch every step go green, checking the **Input** tab, not just Output — most bugs are visible there.
3. **Run it twice.** Duplicate bugs only show up on the second run.
4. Send a modified confirmation under the same PNR — the row updates, the event moves, the doc
   rewrites, and no second doc appears.
5. Send a cancellation for that PNR — row marked, event gone, total adjusted.
6. Send an ordinary marketing email — the classifier stops it.
7. Send the *same* flight twice under two different reference numbers (mimic the airline copy and
   the agency copy) — one booking, one calendar event, one sheet row. This is the test most likely
   to fail, and the one the brief calls out first.
8. Send a confirmation whose details exist only in an image attachment — it should appear under
   "Couldn't read" in the output, not vanish.
9. Send a booking that could plausibly be either personal or work — it should ask once, and the
   answer should stick to that trip on the following run.
10. Drag a calendar event by hand, then send another booking of the same type — the learned buffer
   should apply and be mentioned in the output.
11. Seed the state object by hand to test a trip that already has three bookings, rather than
   waiting to accumulate them.
12. Make the run you submit a good one — the reviewer sees exactly one run. The strongest submission
   run is the arrival of a booking that *completes* a trip and surfaces a real conflict, because
   the output then has genuine substance to be graded on.

## 11. Verified against the live piece catalogue (31 Jul 2026)

Checked via the Activepieces API on cloud.activepieces.com, not guessed.

| Need | Piece | Result |
|---|---|---|
| Trigger on new mail | `piece-gmail` v0.12.7 | ✅ `gmail_new_email_received` (also `new_attachment`) |
| Send mail, read a reply | `piece-gmail` | ✅ `send_email`, `gmail_search_mail`, `gmail_get_mail` |
| Expense sheet | `piece-google-sheets` v0.16.4 | ✅ `create-spreadsheet`, `insert_row`, `update_row`, `find_rows`, `find-or-create-row` |
| Calendar create/move/remove | `piece-google-calendar` v0.9.5 | ✅ `create_google_calendar_event`, `update_event`, `delete_event` |
| Read an event back (buffer learning) | `piece-google-calendar` | ✅ `google_calendar_get_event_by_id` |
| State between runs | `piece-store` v0.6.19 | ✅ `get` / `put`, scope dropdown offers Project / **Flow** / Run |
| AI | `piece-ai` v0.4.9 | ✅ `askAi`, `summarizeText`, `classifyText`, `extractStructuredData`, `generateImage`, `run_agent` |
| **Replace a Google Doc's body** | `piece-google-docs` v0.4.5 | ❌ **does not exist** |

### The Google Docs problem

The piece offers exactly six actions: `create_document`, `create_document_based_on_template`,
`read_document`, `google-docs-find-document`, `append_text`, `custom_api_call`. There is no update,
no replace, no clear. `create_document_based_on_template` makes a *new* document every time, which
is the duplication the brief forbids.

**Solution: `custom_api_call`.** It runs on the same OAuth connection — no API key, so it stays
inside Gravity's one-click sign-in rule — and reaches the Docs API directly:

1. `read_document` on the trip's stored `docId`, to get the body's end index.
2. `custom_api_call` → `POST /v1/documents/{docId}:batchUpdate` with two requests in order:
   `deleteContentRange` over `[1, endIndex-1]`, then `insertText` at index 1 with the rebuilt
   itinerary.

One document per trip, rewritten in place, exactly as the brief requires. Costs one extra step and
one extra call per run.

**Fallback if `custom_api_call` misbehaves:** `append_text` a new dated revision each run. It is
strictly worse — the document grows without bound and the newest itinerary sits at the bottom,
which fails "I open one document and everything is there in order". Use it only if the API call
cannot be made to work.

### One thing to watch

Storage `put` takes its value as `SHORT_TEXT`, so the state object must be JSON-stringified going
in and parsed coming out. As `bookings` and `processedMessageIds` grow, that string grows with
them. Cap `processedMessageIds` at ~200, drop trips that ended more than 90 days ago, and if it
still gets unwieldy, split into one key per trip plus one index key.

### Activepieces gotchas found the hard way (1 Aug 2026)

Four undocumented behaviours, each of which failed silently rather than erroring:

1. **A step is invalid unless `propertySettings` names every property** — including ones left
   empty. Omitting `files` on the AI step made it permanently "Incomplete" with nothing in the UI
   pointing at the cause.
2. **Every step reference is wrapped in `{ output: … }`.** This is the single most expensive thing
   in the build; it caused four separate failures before Google's Sheets API named it outright
   (`Unknown name "output" at 'data.values'`). So:
   - `{{step_1}}` → `{ output: <value> }`, which stringifies to `[object Object]`
   - `{{step_1.text}}` → **empty**, because there is no `text` at that level
   - `{{step_1.output.text}}` → the actual value
   Use `.output` in **every** reference: piece inputs, URLs, JSON bodies, Storage values, and
   router conditions. A router condition comparing `{{need_sheet}}` to a literal silently never
   matches, so the branch never fires and nothing errors.
   Code steps hide this, because the `unwrap` helper in each one strips the wrapper — which is
   exactly why the bug stayed invisible for so long.
   Beware also that `String({})` is the truthy `"[object Object]"`: a check like
   `id ? 'have' : 'need'` returns `have` for an empty object. Validate the shape of what you
   expect (`/^[A-Za-z0-9_-]{25,}$/` for a Google id) rather than testing truthiness.
3. **`{{trigger}}` is wrapped** as `{ output: { message, thread } }`, and the Gmail trigger has no
   plain `from` field — the sender lives in `headerLines` as `{ key: 'from', line: 'From: …' }`.
   Both are handled by resolving fields recursively rather than by fixed path.
4. **`maxOutputTokens` silently truncates structured output.** Capping the extractor at 400 cut
   the JSON mid-object; `JSON.parse` threw and the downstream step saw nothing at all. Now 900,
   with a parser that salvages a truncated object by closing it at the last complete field. The
   standard cost lever corrupts extraction rather than failing loudly — cap prose, not JSON.

### Account-level findings (3 Aug 2026)

Three things that cost points and credits without appearing anywhere in the flow's own logic.

1. **Two flows were ENABLED at once.** The old 40-step `Travel Admin Organizer`
   (`v9KFWjF7DQ40kBWenrj15`) stayed enabled alongside the lean submission flow, and its trigger had
   `subject: ""` — no filter — so it woke on *every* email in the inbox, spent an AI call on each,
   and wrote rows into a spreadsheet. Its last run ingested a Loom marketing email and emitted
   "ITINERARY — --:-- Google security alert email, not a travel booking". Publishing a new flow does
   not disable the old one; check the flow list, not just the flow you are editing.

2. **The free plan is 100 credits *per day*, not 200 once.** `platform-billing/info` reports
   `includedCredits: 100` with `creditsNextResetAt` 24 hours out (reset at 16:58 UTC here). Runs
   that die instantly with status `Out of credits` and 0 ms duration are the budget being exhausted
   before the daily reset, not a flow bug. An unfiltered flow polling a busy inbox will eat the
   whole day's budget on marketing mail.

3. **Runs record the flow *version* they used — check it before drawing conclusions.** Five lean-flow
   runs appeared to prove the `subject: "Booking Confirmation"` filter does not filter, because they
   fired on Reddit, Swiggy and HDFC mail. They had all run version `FmgtZbgJ9iIk4YCZlNqKR`, whose
   trigger had `subject: ""`. The version carrying the filter had been published 35 minutes later
   and had zero runs. `GET /api/v1/flows/{id}` returns the **draft**; pass `?versionId=` with
   `publishedVersionId` to see what is actually running, and compare `flowVersionId` on each run.

Useful read-only probes, all via `fetch` with `Authorization: Bearer <localStorage token>`:
`/api/v1/flows?projectId=…`, `/api/v1/flows/{id}?versionId=…`,
`/api/v1/flow-runs?projectId=…&flowId=…`, `/api/v1/flow-runs/{runId}` (gives every step's status and
output, including the final text the grader reads), `/api/v1/platform-billing/info`.
Reading a run's `state_in` output is the cheapest way to inspect saved Storage state.

### Original checklist

- Google Sheets: **Create Spreadsheet**, **Insert Row**, **Update Row**, **Find Row**
- Google Calendar: **Create Event**, **Update Event**, **Delete Event**, and crucially **Get Event**
  (buffer learning depends on reading an event back)
- **Google Docs: an action that replaces or clears-and-writes an existing document's body.**
  This is the highest-risk unknown in the build. If the piece can only *create* and *append*, then
  "one itinerary per trip, modified never duplicated" needs a different mechanism — appending a
  revision each time is not the same thing.
- Gmail trigger: **New Email** with a search/query field
- Gmail actions: **Send Email**, and a **search / find email** action capable of reading a reply in
  a known thread (the personal-vs-work question depends on it)
- Storage: **Get** / **Put** with a **FLOW** scope option
- AI piece: **Classify Text**, **Extract Structured Data**, **Ask AI**

If an action turns out not to exist, tell me and the design changes — do not work around it by
hand-editing exported JSON, which fails to import.
