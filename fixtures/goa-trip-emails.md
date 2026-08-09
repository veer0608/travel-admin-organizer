# Goa trip enrichment — fixture emails (3 Aug 2026)

Purpose: the graded itinerary currently shows **one train and nothing else**. These two emails turn
`trip_2026-08-07_vascodagama` into a real three-item day with a genuine conflict warning, for zero
code change.

## What's already in state

```
trip_2026-08-07_vascodagama  "Bengaluru to Vasco da Gama"
  train:4242297176 — IRCTC 17310 Sleeper, VSG → YPR
  dep 2026-08-07 17:25  ·  arr 2026-08-08 07:05  ·  ₹412.25  ·  buffer 45 min
```

Home city stored as `Bengaluru`. Expense sheet `1cD4X4IcktchcA2qYhCCP8LgYGplZXRir834Ty7JkSEM`.

## Rules these are built against

- **Subject must contain `Booking Confirmation`** — the published trigger filters on that string.
  Both subjects below do.
- **The conflict rule in `core`** fires when a *hotel checkout* is less than 60 minutes before the
  next booking starts. The late-checkout timing below is what makes it fire, and it fires
  *correctly*: you cannot check out at 17:00 and catch a 17:25 train from a station 35 minutes away.
- **Send order matters.** Hotel first, cab last — the cab is the email that completes the trip and
  surfaces the conflict, and the grader runs against the newest mail.
- **Gmail will spam-filter these.** Per `test-emails.md`, vendor-impersonating mail from a personal
  account lands in Spam and the trigger only reads the inbox. After sending each one, open **Spam**,
  find it, click **Report not spam**, and wait for the next poll.

Send from the college account `22cd10ve757@mitsgwl.ac.in` to `veerarora06@gmail.com`.

---

## Email 1 — hotel (send first)

**Subject:** `Booking Confirmation — Treebo Trend Sea Breeze, Goa`

```
Booking Confirmation

Booking ID: TRB8841207
Hotel: Treebo Trend Sea Breeze
Address: 217 Beach Road, Colva, Salcete, Goa 403708

Guest: Veer Arora
Check-in:  Wednesday, 05 August 2026, 14:00
Check-out: Friday, 07 August 2026, 17:00  (late checkout, prepaid)
2 nights, 1 room, Deluxe Double

Room charge   Rs. 4,400.00
Late checkout Rs.   600.00
Taxes (GST)   Rs.   600.00
Total paid    Rs. 5,600.00

Payment: Prepaid via UPI
```

Why 17:00: an 11:00 checkout leaves a 6-hour gap and the conflict rule stays silent. A prepaid late
checkout is a realistic thing to buy and a realistic thing to forget about — and it puts checkout
25 minutes before the train leaves.

## Email 2 — cab (send last)

**Subject:** `Booking Confirmation — Station transfer, 07 Aug 2026`

```
Booking Confirmation

Booking Reference: GC-2026-55817
Service: Private cab transfer (Sedan, AC)

Passenger: Veer Arora
Pickup:  07 August 2026, 17:05
         Treebo Trend Sea Breeze, 217 Beach Road, Colva, Salcete, Goa
Drop:    Vasco da Gama Railway Station (VSG), Goa
Estimated journey time: 35 minutes

Fare: Rs. 850.00, payable to the driver
```

## What a correct run should then produce

```
Bengaluru to Vasco da Gama · 5–8 Aug 2026

Wed 5 Aug
  14:00  Check in — Treebo Trend Sea Breeze, Colva, Goa   Conf TRB8841207
Fri 7 Aug
  17:00  Check out — Treebo Trend Sea Breeze
  17:05  Cab — Colva → Vasco da Gama Railway Station        Conf GC-2026-55817
  16:40  Leave for the station — 45 min before departure
  17:25  Train 17310 Sleeper — VSG → YPR                    PNR 4242297176
Sat 8 Aug
  07:05  Arrive Yesvantpur

Needs your attention
- Checkout from Treebo Trend Sea Breeze is 25 minutes before your train departs.
- The cab is booked for 17:05 and the drive is 35 minutes; the train leaves at 17:25.

Expenses — ₹6,862.25 across 3 bookings
```

Three items, a real chronology across three days, and two honest warnings. That is a substantially
stronger thing for an AI grader to read than one train.

## Optional third email — the outbound leg

Only if the first two land cleanly. It makes the trip a complete round trip rather than a return
leg, but it also gives the trip-assignment logic a second chance to get the grouping wrong.

**Subject:** `Booking Confirmation on IRCTC, Train: 17309, 04-Aug-2026, SL, YPR - VSG`

```
PNR No. : 4242118902
Train No. / Name : 17309 / YPR VSG EXPRESS
Class : SLEEPER      From : YESVANTPUR JN (YPR)
Date of Journey : 04-Aug-2026    To : VASCO DA GAMA (VSG)
Scheduled Departure : 04-Aug-2026 19:15
Scheduled Arrival   : 05-Aug-2026 09:40
Total Fare Rs. 405.00
```

---

## Open question this does not settle

If Gravity's grading run executes the flow in **their** environment with **empty Storage**, none of
this seeded state carries over — the itinerary would rebuild from the single triggering email and
still show one booking. The seeding is still worth doing (it is what makes local testing real), but
if the graded output keeps showing one item after this, the cause is a fresh-state environment, and
the fix is for the flow to back-fill from a Gmail search rather than trusting Storage alone.
