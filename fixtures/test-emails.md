# Test fixtures — Travel Admin Organizer

Target inbox: **veerarora06@gmail.com** (not the college account, not the Gravity login address).
That is the account to OAuth into Activepieces.

The backbone of the test set is **real mail already in this inbox** — IRCTC confirmations, Uber
receipts, Stanza Living. Only what the inbox cannot supply is synthesised: a change, a
cancellation, and a genuinely ambiguous booking. Synthetic emails are always tidier than real
ones, which is where parsing bugs hide, so use the real ones wherever possible.

---

## Part A — real mail, already in the inbox

Nothing to send. Point the trigger at these and let it run.

### A1. IRCTC confirmation — the backbone

Search: `subject:"Booking Confirmation on IRCTC"` · ~17 available.

Verified format from the 8 Jun 2026 booking:

```
Subject: Booking Confirmation on IRCTC, Train: 12628, 16-Jun-2026, 3A, BPL - SBC

PNR No. : 2953415913
Train No. / Name : 12628 / KARNATAKA EXP
Quota : GENERAL      Transaction ID : 100006632418293
Date & Time of Booking : 08-Jun-2026 09:49:44 pm HRS
Class : THIRD AC     From : BHOPAL JN (BPL)
Date of Journey : 16-Jun-2026    To : KSR BENGALURU (SBC)
Boarding At : BPL    Date Of Boarding : 16-Jun-2026
Scheduled Departure* : 16-Jun-2026 06:30
Reservation Up to : KSR BENGALURU (SBC)
Scheduled Arrival : 17-Jun-2026 12:00
Distance : 1705KM
Seat / Berth / WL No : RLWL 8
Fare Details (Inclusive of GST)
  Ticket Fare Rs. 1930.00 · Convenience Fee Rs. 23.60 · Total Fare Rs. 1953.60
```

What this one is worth testing against:

- **Overnight, next-day arrival.** Departs 06:30 on the 16th, arrives 12:00 on the **17th** — 29½
  hours. An extractor that assumes arrival shares the departure date produces a negative-length
  event.
- **Waitlisted.** `RLWL 8` is not a confirmed berth. The itinerary should say so rather than
  presenting it as settled.
- **Two fare numbers.** Ticket fare 1930.00 and total 1953.60. The expense log wants the total.
- **Boilerplate outweighs content.** The body is ~3,200 characters, of which maybe 900 matter; the
  rest is safety notices, a "Book Hotels" ad and cancellation instructions. This is the live test
  of the trimming step — feed the whole thing to a small model and you get exactly the failure the
  Gravity docs describe.

### A2. Uber receipts — the local-ride trap

Search: `from:uber.com subject:"trip with Uber"` · ~25 available.

Verified format from 11 Jul 2026:

```
Subject: Your Saturday evening trip with Uber

11 Jul 2026, 19:05
Total ₹63.39   Suggested fare ₹79.24   Promotion -₹15.85
Payments: Cash ₹63.39 · 11/07/2026 19:23
Trip details: Bike Saver · 5.68 kilometres, 13 minutes
License Plate: KA26W2005
19:10  77, Bellandur Main Rd, HSR Layout, Bengaluru 560102
19:23  WJ8C+J68, Outer Ring Rd, BTM 1st Stage, Bengaluru 560068
```

Two design-breaking properties, both worth verifying explicitly:

- **No confirmation number.** None, anywhere. Identity has to fall back to a fingerprint.
- **It is a receipt, not a reservation.** The ride already happened and the fare is already
  charged. No future calendar event should be created.
- **It is a commute, not travel.** Bellandur to Silk Board, ₹63, inside the home city. Most of the
  25 are like this. If the agent creates a trip for them, the local-ride rule isn't working.

Pick one Uber receipt dated **close to an IRCTC journey** for the positive case — that one *is*
trip-related and should attach.

### A3. Insurance mails — the decoy, with real data

Search: `subject:"PA Insurance"` · ~8, plus ~14 more from `noreply.irctc`.

`Congratulations your PA Insurance is issued for PNR No.2519967414 dtd 21/06/2026`

Carries a PNR, arrives moments after a genuine booking, and is **not a booking**. A better
classifier test than any marketing email, because it will fool a keyword filter and it is real.

### A4. Accommodation

Search: `from:stanzaliving` (2) and the PerkUp Hostel mails (2).
`Stanza Living Booking Confirmation | Veer Arora | Sao Paulo House`

Long-stay housing rather than a hotel, so check what the extractor does with a check-in that has
no check-out.

### A5. Rapido — a third identity format, and a same-day local pair

Checked: the fare and route **are in the body**, so this is not the unreadable case. B4 stays.

```
From: partner@rapido.bike        Sat, 11 Jul 2026, 17:34
Subject: Your trip with Rapido

Customer Name: veer arora
Ride ID: RD17837699447164964
Driver: Saharul Akter · Vehicle: KA53JC5071
Time of Ride: Jul 11th 2026, 5:15 PM
Selected Price: ₹100
  Silk Board, 100 Feet Ring Rd, BTM 1st Stage, Bengaluru 560068
  213, Bellandur, Bengaluru 560103
```

Two things this proves:

- **Three vendors, three identity schemes.** IRCTC has a PNR, Rapido has a Ride ID, Uber has
  nothing at all. The fallback fingerprint is not a hypothetical — it is required for one of the
  three cab vendors in this very inbox.
- **The same-day local pair.** This Rapido ride is Silk Board → Bellandur at 17:15 on 11 Jul. The
  Uber receipt in A2 is Bellandur → Silk Board at 19:10 **the same day**. Two rides, opposite
  directions, both inside Bengaluru, both past-dated. An agent without the local-ride rule sees a
  journey out and a journey back and invents "Trip to Silk Board". This is the single best
  negative test in the whole set, and it is real.

---

### A6. Real cancellation — B2 is not needed

Search: `in:anywhere 2953415913` · `ticketadmin`, **Cancel Ticket**, 16 Jun 2026:
*"your ticket against PNR Number: 2953415913…"*

The real IRCTC booking from A1 was genuinely cancelled, and the cancellation email is sitting in
the inbox. So the whole lifecycle — book, reschedule, cancel — exists as real mail for one single
PNR. Use this instead of the synthetic B2.

---

## Part B — synthesise these

The inbox has no change and nothing genuinely ambiguous. Send these from a **second Gmail
account** — the college account `22cd10ve757@mitsgwl.ac.in` works and is already signed in. Do not
forward from yourself, which would make every sender identical.

> **Gmail will mark these as spam.** Confirmed on the first attempt: B1 was delivered straight to
> Spam, and the Gmail trigger only reads the inbox, so the flow never saw it. A "this is a system
> generated mail, do not reply" message arriving from a personal address looks exactly like
> phishing to Google's filter — which is a fair judgement, since it is impersonating IRCTC.
>
> After sending each fixture, open **Spam**, find it, and click **Report not spam**. It moves to
> the inbox and the trigger picks it up on the next poll. Check Spam before concluding a fixture
> "didn't arrive". This is also a good argument for leaning on the real mail in Part A wherever
> possible — real vendor mail is never filtered this way.

### B1. Journey change — modification test

Same PNR as the real A1 booking, so it must update that booking in place. A second calendar event
standing next to the old one is the failure the brief names.

**From:** ticketadmin@irctc.co.in
**Subject:** Train Rescheduled — PNR 2953415913 — Train 12628 on 16-Jun-2026

```
This is a system generated mail. Please do not reply to this email ID.

Dear veer arora(User Id: veer_0608),

Your train has been rescheduled by the Railway Administration.

PNR No. : 2953415913
Train No. / Name : 12628 / KARNATAKA EXP
From : BHOPAL JN (BPL)   To : KSR BENGALURU (SBC)
Date of Journey : 16-Jun-2026

Revised Scheduled Departure : 16-Jun-2026 09:45
Revised Scheduled Arrival   : 17-Jun-2026 15:10

Your booking status remains unchanged. No fare difference is payable.
```

### B2. Cancellation and refund — refund-tracking test

**From:** ticketadmin@irctc.co.in
**Subject:** Ticket Cancellation Confirmation — PNR 2953415913

```
This is a system generated mail. Please do not reply to this email ID.

Dear veer arora(User Id: veer_0608),

Your e-ticket has been cancelled.

PNR No. : 2953415913
Train No. / Name : 12628 / KARNATAKA EXP
Date of Journey : 16-Jun-2026
From : BHOPAL JN (BPL)   To : KSR BENGALURU (SBC)

Total Fare Paid   : Rs. 1953.60
Cancellation Charge : Rs. 240.00
Refund Amount     : Rs. 1713.60

Refund will be credited to your source account within 5-7 working days.
```

Note the refund is **less than the fare**. The sheet should track ₹1,713.60 as pending, not
₹1,953.60 — an honest expense log gets this right.

### B3. Ambiguous trip type — ask-once test

A single midweek hotel night in a city with no other booking and no signal either way. The agent
should ask once, in one line, and still finish the run with a complete itinerary.

**From:** noreply@agoda.com
**Subject:** Booking confirmed — Taj Santacruz Mumbai, 25 Aug

```
Booking confirmed
Booking ID: 892471336
Taj Santacruz, Off Western Express Highway, Santacruz East, Mumbai 400029

Check-in:  Tuesday, 25 August 2026, 15:00
Check-out: Wednesday, 26 August 2026, 12:00
1 night, 1 room, Superior King
Guest: Veer Arora

Total: INR 11,900
```

### B4. Image-only confirmation — only if A5 turns out readable

Send with an image attached and this body exactly:

**From:** noreply@airindia.in
**Subject:** Your e-ticket — Air India

```
Dear Guest,

Please find your e-ticket attached.

Thank you for choosing Air India.
```

---

## What each case proves

| Case | Source | Tests |
|---|---|---|
| A1 | real | Parsing under heavy boilerplate; overnight arrival; waitlist status; correct fare |
| A2 local | real | Local rides never become trips |
| A2 trip-adjacent | real | Cab with no reference number attaches to the right trip |
| A2 any | real | Past-dated receipt logs an expense but creates no calendar event |
| A3 | real | Classifier rejects a PNR-bearing non-booking |
| A4 | real | Accommodation with an open-ended stay |
| A5 | real | Attachment-only booking reported, never silently skipped |
| B1 | synthetic | Modification updates in place; conflicts recomputed |
| B2 | synthetic | Cancellation, refund amount net of charges |
| B3 | synthetic | Ambiguity asks once, non-blocking, remembered |
| B4 | synthetic | Unreadable path, if A5 doesn't already cover it |

Run the whole set twice. The second pass is where duplicate bugs surface.

## The submission run

The reviewer sees exactly one run. The strongest candidate is **B1 landing on top of the real A1
booking**: it exercises parsing of a real messy email, an in-place modification, a recomputed
schedule, and an itinerary with genuine substance — a 29-hour overnight journey, a waitlisted
berth, a linked cab, and a real fare total.
