Subject: Re: Travel Booking Extractor Agent — feedback

Hi Vivek,

Thanks for this — it's unusually detailed for a take-home and I've acted on several points already.

One thing that will help you re-check it: the build you reviewed is my **first** submission — 41 steps,
19 code steps, a Google Docs itinerary and a `shouldProcess` gate. My later submissions are a rebuilt
16-step version with 4 code steps and no Docs step, so a few of the numbered points describe code
that is no longer there. It cuts the other way too: two things you list under "done well" — the
single rewritten itinerary document, and tracking calendar event IDs for updates — were casualties
of that simplification. Reading your review, dropping the Doc was the wrong trade, and that's the
clearest thing I'd put back.

**Fixed since that submission**

- **#9 Calendar ID.** It was hardcoded to my own Google account, so events could never land in the
  consumer's calendar. Now `primary`.
- **#2 Resources created before validation.** The expense sheet was created regardless of whether a
  booking had been confirmed. It's now gated on a valid booking, so a non-travel email creates
  nothing — no sheet, no row, no event.
- **#14 Currency.** No longer defaults to INR when the email doesn't state one.

**Points I agree with and haven't addressed**

- **#12** Unreadable confirmations are recorded in state but never surfaced to the user. You're right
  that this doesn't meet the requirement — recording isn't notifying.
- **#10** There's no path for a user to correct a buffer and have it learned. The state field exists;
  the mechanism doesn't.
- **#13** The change and cancellation paths are implemented but I hadn't demonstrated them
  end-to-end, so your point stands — untested code isn't evidence of anything.

**One thing worth checking on your side**

Several of my graded runs failed with:

```
Failed to extract structured data: Insufficient credits.
Add more using https://openrouter.ai/settings/credits
    at gravity-piece-ai/src/lib/actions/utility/extract-structured-data.js:255
```

Those runs scored 0.00 on Rating regardless of the workflow, and I suspect it's why the update and
cancellation paths were never exercised during your testing — the run terminated at the AI step
before reaching them.

Happy to talk any of this through.

Best,
Veer
