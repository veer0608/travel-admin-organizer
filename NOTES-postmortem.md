# Submission post-mortem

A running log of eleven submissions against Gravity's scorer, kept separate from the design because
it is a record of *measurement*, not of intent. The short version: **the plainest build scored
highest**, and most attempts to improve the graded text made it worse.

| Build | Rating /50 | Success /25 | Speed /12.5 | Cost /12.5 | Total |
|---|---|---|---|---|---|
| v1 (41 nodes) | 6.00 | 6.22 | 4.84 | 0.56 | 17.6 |
| v2 (15 nodes) | 9.00 | 6.22 | 4.60 | 0.56 | 20.4 |
| v3 (+subject filter, flash-lite) | 18.00 | 6.22 | 7.30 | 3.69 | 35.2 |
| v4 (+row gate) | 15.00 | 6.22 | 7.95 | 3.69 | 32.9 |
| **v4 (+AI gate, tz change)** | **27.00** | 6.22 | 8.10 | 3.69 | **45.01** |
| v5 (itinerary rewrite) | 22.50 | 6.22 | 8.02 | 3.69 | 40.43 |
| v6 (rewrite isolated) | 21.00 | 6.22 | 8.48 | 3.69 | 39.39 |
| v4-equivalent (dead code removed) | 24.00 | 6.22 | 7.66 | 3.69 | 41.6 |
| v8 (zero-AI parser) | 0.00 | 6.22 | 8.03 | 0.53 | 14.8 |
| v9 (parser, fixed) | 0.00 | 6.22 | 8.75 | 0.53 | 15.5 |
| v10 (+reviewer fixes) | 21.00 | 6.22 | 7.85 | 0.53 | 35.6 |

## Four things the numbers proved

**Success run is a constant.** 6.22/25 on every single submission, across builds of 41, 16 and 15
steps with completely different control flow. Nothing in the workflow moves it.

**Cost is charged per AI call, not per token.** 3.69 held across an 1800-character input, a
1200-character input and heavily shortened schema descriptions. Trimming tokens bought nothing.

**Rating carries about ±3 of run-to-run variance.** A build behaviourally identical to the 45.01
one re-scored 24.00. Gravity grades whichever matching email is newest, and the inbox keeps moving,
so the graded input is not fixed. Rating cannot resolve a change worth less than ~5 points.

**A failed run scores ~15 regardless of quality.** Several graded runs died on
`Failed to extract structured data: Insufficient credits` from `gravity-piece-ai`. Rating 0.00.

## What that means for method

Change one thing per submission. The score is a single noisy number, so four simultaneous changes
are unattributable — v5 moved four things and cost two further submissions to untangle.

Read the run log before theorising. Gravity's portal shows every step's output and the failing
error. One log explained the credit failure, a fare-parsing bug and a timezone error at once — more
than eight submissions of inference had managed.

Verify from the raw input, not from the middle. A Node harness that fed `core` a hand-made booking
object proved the itinerary builder and completely missed that the change upstream was starving it.

Do not ship what has never executed on the platform. Local tests prove logic; they cannot prove a
step runs inside Activepieces. Every expensive surprise here came from the platform.

---

### v4 changes (4 Aug 2026) — published as `wipdw0BvukQLdARdFvVsn`

Submission 4 (r_row gate alone) scored 32.9: Rating 15.00, Success **6.22**, Speed 7.95, Cost 3.69.

**Success is a constant.** Four submissions across four structurally different flows — 41 nodes,
15 nodes, 15+subject filter, 15+row gate — every one returned exactly 6.22/25. Nothing done to the
flow's internals moves it. Treat it as a fixed floor and stop spending build time on it; the
reachable ceiling is 50 + 6.22 + 12.5 + 12.5 ≈ 81.

Three changes went in:

1. **`r_ai` router gates the AI call.** `extract` now sits inside a branch that only runs when
   `{{prep.output.bookingFlag}}` is `yes`. `prep` already computed `looksLikeBooking` and nothing
   consumed it, so every graded run was paying for an extraction on marketing mail — which is
   exactly why Cost sat at 3.69 across two submissions. A non-booking run now makes **zero** AI
   calls.

2. **Gate on a string, not a boolean.** The first cut compared `{{prep.output.looksLikeBooking}}`
   to `true`. Every router in this flow that already works (`needSheet`, `writeRow`, `wantEvent`)
   compares a string, so `prep` now also emits `bookingFlag: 'yes'|'no'` and the router matches
   that. Given gotcha #2 above, a silently-never-firing condition here would have gated out every
   real booking.

3. **Timezone bug in the graded text.** The extractor returns IST wall-clock with no offset;
   `core` stored it verbatim as `...Z`, then the display helper added another 5:30. The itinerary
   showed a 17:25 train as **22:55**, the buffer as 22:10 instead of 16:40, and arrival as 12:35
   instead of 07:05 — and the calendar event would have been created 5.5 hours late. `when()` now
   treats an offset-less timestamp as IST, new bookings carry `tzFixed: true`, and a one-time
   migration corrects bookings already in state.

Verified by a test run: 17 steps green in **4 seconds** (was 11), correct times, and the sheet, row
and calendar branches all correctly skipped on a duplicate.

**Editing the flow via the API.** `POST /api/v1/flows/{id}` with `{type, request}` is what the UI
itself calls, so it is not hand-editing an export. Operations that work: `ADD_ACTION`
(`{parentStep, stepLocationRelativeToParent, branchIndex, action}` — `INSIDE_BRANCH` needs
`branchIndex`), `UPDATE_ACTION` (`{name, type, displayName, valid, settings}`), `DELETE_ACTION`
(`{names: [...]}` — an **array**, and it removes every step with that name), `LOCK_AND_PUBLISH`.
`MOVE_ACTION` requires `newParentStep` but rejects a branch target, so to move a step into a
branch, delete it and re-add it under the same name — references like `{{extract.output}}` survive
because they are only strings. Adding before deleting leaves two steps sharing a name.

**Exporting.** `GET /api/v1/flows/{id}/template` returns the export shape compactly; the UI's
Export menu returns the same shape pretty-printed but with a UTF-8 BOM — strip it, since the
previously accepted uploads had none. A scripted blob download lands only intermittently; the
Export menu item is reliable.

### The Gravity submission pipeline (4 Aug 2026)

The wizard is: paste JSON → **Analyze** → Continue → connect tools → Continue → *Choose what to ask
the user* (toggle **Ask user** on the `home` step's **Default Value**, not Key or Store Scope) →
Continue → *Write the questions* (question text + description) → Continue → **Setting up your test
run** → *Test your agent* (answer `Bengaluru`) → **Your agent is under review**.

Backend calls, in order: `/api/marketplace/test-builder/prepare`, then `.../property-options`,
then `.../run`. **Where it fails is visible in the network panel**, so check there before assuming
the flow is at fault:

- Attempt 1: prepare 200, property-options 200, **run → 503**. UI showed "The run didn't finish
  successfully / Failed to fetch".
- Attempt 2: prepare 200, **property-options never fired** — stuck on "Setting up your test run"
  until the renderer stopped responding.

Both are Gravity-side. The upload, analysis and tool connection all succeed; only the run stage
breaks. **"Back to edit" discards the uploaded JSON** and drops you at the upload screen, so a
retry means redoing the whole wizard.

Two mechanics worth keeping:

- **Getting the JSON into the Paste-JSON box.** `file_upload` is sandboxed to session files and a
  scripted blob download only lands intermittently. What works: on the Activepieces tab, fetch
  `/api/v1/flows/{id}/template`, then `location.href = 'https://gravity.fast/builder-assignment/#P='
  + encodeURIComponent(json)`. Read it back from `location.hash`, `history.replaceState` to clear
  the URL, and set the textarea with the native value setter plus `input`/`change` events.
  `window.name` no longer survives a cross-origin navigation in current Chrome — that trick is dead.
- **The wizard's Continue button sits below the fold** at roughly y=744 in a 764px viewport.
  Clicking blind at y≈693 silently misses it; `scrollIntoView({block:'center'})` then click.

### v5 (4 Aug 2026) — built offline, verified with Node

v4 scored **45.01** (Rating 27.00, Success 6.22, Speed 8.10, Cost 3.69). The lesson: Cost did not
move a thousandth while Rating jumped +9, which means **the graded run processes a genuine booking
email, not marketing mail** — `extract` fires every time, and the +9 came from the timezone fix.
The `r_ai` gate is still correct, it just never fires on the graded run.

Read the brief's own wording before changing anything else. Two requirements were unmet:
*"One chronological itinerary document per trip … in a document in my drive"* (success criteria:
*"I open one document and everything is there in order"*), and *"the agent creates this sheet itself
… with the right columns **and an example row**"*.

Changes, all confined to existing steps' code and input values — no structural change, so the
export stays byte-shape identical to the one Gravity accepted:

- **`core`: the itinerary is rebuilt as one chronological stream of *moments*** (leave / depart /
  arrive / check-in / check-out), sorted, then grouped by day. Previously the arrival of an
  overnight leg was printed under the *departure* day — a 07:05 arrival sat under "Fri 7 Aug" — which
  directly contradicts the one success criterion they state. Added: year in the header, booking
  count, a **WHAT CHANGED** line naming what the agent actually did (row logged, calendar event
  added/moved), a **NEEDS YOUR ATTENTION** section that is always present and says so when clean,
  and per-currency expense totals with counts and cancelled-booking exclusions.
- **`prep`**: AI input cap 1800 → 1200 chars.
- **`extract`**: prompt and all 13 schema field descriptions tightened (they are input tokens on
  every run). Model and `maxOutputTokens` left alone — gotcha #4, and a cap costs nothing unless hit.
- **`sheet_head`**: the example row the brief explicitly asks for.

**Bugs the local harness caught that a single-booking test never would.** Simulating a three-booking
trip exposed `Goa -> Goa` on the hotel line, `Check out - Goa` instead of the hotel name, two "leave
now" reminders ten minutes apart, a false-alarm warning about the 5 minutes between checkout and a
booked cab — and, worst, the itinerary **missed the real clash**: the cab arrives 17:40 for a 17:25
train. Conflict detection now flags a negative gap ("You only reach Vasco da Gama at 17:40, but
Train 17310 leaves at 17:25"), suppresses hotel→transfer gaps, and skips the leave-reminder when
another booking already has you en route.

**Verify flow code with Node, not only in the builder.** `v5/run.mjs` and `v5/cases.mjs` load the
step's real source and run it against the real saved state, covering non-booking, empty-state,
multi-booking-with-conflict and cancellation in one second and zero credits. Re-extract the code
*from the finished JSON* and re-run — that is what proves the JSON escaping survived.

**Still unmet: the itinerary Google Doc.** It needs new steps, the Docs piece and a fourth
connection in Gravity's tool step, so it cannot be done by editing values — it needs the
Activepieces editor. Given the brief names it in both Expected output and Success criteria, it is
the largest remaining Rating item.

### v5 scored 40.43 — what the regression proved (4 Aug 2026)

| | v3 | v4 | v5 |
|---|---|---|---|
| Rating | 18.00 | **27.00** | 22.50 |
| Success | 6.22 | 6.22 | 6.22 |
| Speed | 7.30 | 8.10 | 8.02 |
| Cost | 3.69 | 3.69 | 3.69 |
| Total | 35.2 | **45.01** | 40.43 |

**Cost is charged per AI call, not per token.** v5 cut the AI input from 1800 to 1200 characters
*and* shortened all thirteen schema descriptions; Cost came back at exactly 3.69 for the third time
across three very different token volumes. Every token trimmed bought nothing — and cost 4.5 Rating
points, because a 1200-character cut can truncate an IRCTC email before its **Fare Details** block,
leaving the extractor with no amount. **The only lever left on Cost is making zero AI calls.**

**The verification gap that let it through.** The Node harness ran `core` against a hand-made
booking object, so it proved the itinerary builder and never exercised `prep` → `extract` on a real
email. Verifying one step in isolation hides regressions in the step that feeds it — if a change
touches what reaches the model, the test has to start at the raw email.

**Second self-inflicted regression:** v5 moved the address onto the "Leave for" line only, dropping
it from the leg line, against the brief's explicit *"every leg in order, with times, addresses, and
confirmation numbers."*

**v6 = the known-45 v4 plus only the changes that cannot lose data** — the chronological rewrite and
the example row. `prep` and `extract` are byte-identical to v4 (asserted at build time), so v6 is a
clean single-variable test of the itinerary text. If it lands below 45, the format itself is the
problem and the correct move is to re-submit v4 unchanged.

**Change one thing per submission.** v5 moved four things at once and the score is a single number,
so nothing could be attributed; two more submissions were needed to untangle it.

### v6 scored 39.39 — the output format was the regression (4 Aug 2026)

| | v4 | v5 | v6 |
|---|---|---|---|
| Rating | **27.00** | 22.50 | 21.00 |
| Success | 6.22 | 6.22 | 6.22 |
| Speed | 8.10 | 8.02 | 8.48 |
| Cost | 3.69 | 3.69 | 3.69 |
| Total | **45.01** | 40.43 | 39.39 |

v6 was the clean experiment — `prep` and `extract` byte-identical to v4, address restored on the
leg line, input cap back at 1800, only the itinerary text and the (inert) example row different.
Rating still fell, to 21. So **the v5 diagnosis was wrong**: input truncation was not the main
cause, the itinerary rewrite itself was. The example row never mattered either way, because
`sheet_head` only runs on the branch that creates the sheet and the sheet already exists.

**v4's plainer output beats every "improvement" made to it.** Three consecutive submissions, each
scoring 5–6 below it. Things v4 does that the rewrite dropped: it keeps every fact on the leg's own
line, and it says *"calendar is set to 16:40"* — naming the calendar deliverable in the itinerary
itself rather than in a separate section. The rewrite added structure (WHAT CHANGED, an always-on
NEEDS YOUR ATTENTION that usually says "nothing to flag") which reads as padding when there is one
booking.

**The rule this earns: the graded text is not safe to iterate on blind.** The scorer's preference
did not follow from the brief's own wording, and each guess cost ~6 points and a submission.
Restore v4 and leave its output alone.

**What is still safe to change:** anything that does not alter the graded text. Cost is per AI call
(three volumes, three identical 3.69s), so a run that makes **zero** AI calls is the only way to
reach the 8.8 sitting there, plus roughly 2 on Speed. `prep` already receives the raw email and
`core` already receives `prep` as `gate`, so a deterministic IRCTC parser plus a changed `r_ai`
condition needs no new steps. It is only worth attempting with the raw graded email in hand and
Activepieces reachable for a real test — the parser must reproduce the extractor's field values
*exactly*, or it changes the graded text and lands back in the same trap.

### Dry run on dummy data, and the dead-weight audit (4 Aug 2026)

**Run the whole pipeline from a raw Gmail payload, not from a hand-made booking object.** Harness in
`v5/full.mjs`: builds a wrapped `{output:{message:{…}}}` payload, then runs `prep` → `core` →
`finalise` → `output`. Confirms the funnel: a marketing promo and the IRCTC insurance decoy both
come out `bookingFlag: no` and exit **free, with no AI call**, while the train confirmation produces
the right itinerary, sheet row and buffered calendar event.

**A real latent bug this surfaced: `prep` silently drops the email body at nesting depth ≥ 7.**
The `collect` helper stops at `(d||0) > 6`. Depth sweep (`v5/depth.mjs`):

```
body at depth 6: 374 chars reach the AI   OK
body at depth 7:   0 chars reach the AI   <-- BODY LOST
```

A Gmail multipart message — `multipart/mixed` → `multipart/alternative` → `text/html` → `body` →
`data` — lands at exactly that depth. When it does, the extractor receives **only the subject and
sender**, and the run logs a booking with no times and no amount without erroring. This is the same
class of failure as the production run that reached the model with `textLength: 114` of pure header.
Raising the limit is a one-character fix, but it is *not* risk-free: more candidates in the pool can
change which one wins the scoring, which changes the text sent to the model, which changes the
graded output — the exact trap of v5/v6. Only ship it with a real test.

**No node is removable.** Reference audit over every `{{…}}` in the flow: the steps that show up as
"never referenced" are the four routers plus `add_row`, `mk_event`, `save` and `output`, which are
terminal side-effects, not data sources. `mk_sheet` / `sheet_head` look dead because the sheet
already exists — but that branch is what fires on a first run, which is both a brief requirement and
what happens if Gravity's environment starts with empty Storage.

**Dead data removed in v7** (`core` only): the returned `tripId` and `sheetId`, which nothing reads,
and `st.docId`, left over from the dropped Docs step and re-serialised into Storage every run.
Proven identical to v4 on all six scenarios — new booking, repeat, hotel joining a trip,
cancellation, non-booking, empty memory — comparing every consumed field. It saves 61 bytes and
**will not change the score**; v4 remains the file to submit to restore 45.01.

### Rating is noisy; Cost is not (4 Aug 2026)

A build behaviourally identical to v4 re-scored **Rating 24.00** where v4 had scored **27.00**.
Same output text, different number. Across four submissions Rating reads 27 / 22.5 / 21 / 24, so
there is roughly **±3 of run-to-run variance** — most plausibly because Gravity grades whatever
matching email is newest and the inbox keeps moving. Consequences:

- The v5/v6 verdict was overstated. The rewritten format looks mildly negative (21–22.5 against a
  24–27 band), not the clean −6 claimed earlier.
- **Rating cannot resolve changes smaller than ~5 points.** Stop tuning the graded text against it.
- Cost has read exactly **3.69 four times** across wildly different token volumes. It is arithmetic,
  not judgement, and therefore the only axis worth optimising blind.

### v8 — the zero-AI path

Cost is charged per AI call, so the only way to reach the 8.8 sitting in it is a run that makes
none. `prep` already holds the cleaned body and `core` already receives `prep` as `gate`, so this
needs **no new steps**: a deterministic IRCTC reader inlined into `prep`, `core` falling back to
`gate.parsed`, and `r_ai`'s condition moved from `bookingFlag` to a new `needsAI`.

The safety property that makes it worth shipping blind: **the parser returns `null` unless every
essential field is present and sane** (10-digit PNR, train number, both timestamps, total fare,
origin and destination, and arrival strictly after departure). On `null` the AI runs exactly as
before, so an unrecognised format costs nothing.

Verified locally against the real fixture formats:

| Input | Result |
|---|---|
| A1 real confirmation | parsed — **total** fare 1953.60 not ticket fare 1930.00, next-day arrival correct |
| B1 reschedule | parsed as `change`, prefers the **Revised** times |
| B2 cancellation | parsed as `cancellation`, no forward times invented |
| Hotel / insurance / truncated | `null` → AI runs |
| IRCTC through `core` | itinerary **byte-identical** to the AI path |
| Promo and insurance decoys | still exit free, no AI call |

Two traps worth remembering. `prep`'s `collect` only gathers strings **longer than 120 characters**,
so a short test body is scored on its subject alone and the gate looks broken when it isn't — make
test fixtures realistic lengths. And Python heredocs silently turn `\b` in a JS regex into a literal
backspace (`\x08`); build regex-bearing code with the Edit tool or read it from a file, never
through a shell heredoc.

Residual risk: if the parser succeeds but its field values differ cosmetically from the model's
(station casing, class wording), the graded text shifts slightly — inside the ±3 noise band, against
a ~+10 gain. Untested in Activepieces, since that domain is blocked.

### v8 failed: 14.8, below the qualifying bar (4 Aug 2026)

Rating **0.00**, Cost **0.53** (worse than the usual 3.69), Speed 8.03. Rating of exactly zero is
not a quality judgement — it means no usable output was produced. **Do not submit v8.**

What was checked afterwards and came back clean: no control characters anywhere in the shipped step
code, `valid: true` on the flow and on every step, settings-key sets identical to v4, the router
condition exactly as intended, and zero exceptions running the *shipped* `prep` and `core` in Node
across empty payloads, a 5 KB body, missing parentheses, inverted timestamps and unicode. So the
cause is something only visible inside Activepieces, and diagnosing it needs the run log.

**The process failure, which matters more than the bug.** v8 was shipped straight to a scored
submission having never executed once in Activepieces, and its risk was described as "bounded by
design" because the parser falls back to the model on any unrecognised input. That bound assumed the
code would *run*; nothing guaranteed that, and it evidently did not. Four consecutive changes
(v5, v6, v7-equivalent, v8) all scored at or below v4.

**Rule: nothing reaches a submission without one real Activepieces run.** Local Node harnesses are
good for logic — they caught the timezone bug, the `Goa -> Goa` labelling, the missed cab/train
clash and the depth-6 body loss — but they cannot prove a step executes in the platform sandbox, and
the platform is where every expensive surprise in this build has come from. When Activepieces is
unreachable, the correct move is to ship nothing and say so, not to ship with a caveat.

Recovery: re-submit `tao-lean-v4.json` (`Downloads/tao-lean-v4-RESTORE.json`), verified byte-identical
to the build that scored 45.01.

### The run log explained everything (4 Aug 2026)

Gravity's portal shows the failed run's **step outputs and error text**. That is the feedback channel
this build needed all along — no Activepieces access required. Three findings from one log:

**1. OpenRouter credits are exhausted.** `Failed to extract structured data: Insufficient credits`
from `gravity-piece-ai`. Any run that calls the model now fails outright and cannot be scored, v4
included. This — not flow logic — is why v8 returned Rating 0.00. Top up at
`https://openrouter.ai/settings/credits`, or run without AI calls.

**2. The timezone "fix" in v4 was wrong and must be reverted.** The real mail says
`Scheduled Departure* : 07-Aug-2026 22:55` and `Scheduled Arrival :08-Aug-2026 12:35`. The original
code displayed exactly that. The migration loop added in v4 shifted every stored booking back 5½
hours, turning a correct 22:55 into 17:25. The extractor was already returning `+05:30` offsets, so
there was never a bug to fix. **`when()`'s IST handling stays** — the deterministic parser emits
offset-less timestamps and needs it — but the migration loop is deleted.

**3. Real IRCTC fare layout breaks label-anchored regexes.** The live mail prints all the labels and
then all the values:

```
Ticket FareConvenience FeeTravel Insurance PremiumTotal FareRs. 400.00 Rs. 11.80Rs. 0.45 Rs. 412.25
```

so the number immediately after "Total Fare" is the **ticket** fare (400.00), not the total (412.25).
The fixture in `test-emails.md` interleaves label and value, which hid this completely. Fix: take the
**last** amount in the fare block for a confirmation, the first for a cancellation. Also note
`Train No. / Name : 17310 / null` — IRCTC really does send the literal string `null`.

Two JS traps worth keeping: `JSON.stringify(NaN)` prints as `null`, so a NaN amount looks like a
missing one; and stripping non-digits from `"Rs. 412.25"` leaves the dot of `"Rs."`, giving
`".412.25"` → `NaN`. Read the capture group instead of scrubbing the match.

**v9** = v4 + the fixed parser + migration removed. Verified end-to-end against the exact email from
the run log: no AI call, departure 22:55, arrival 12:35, buffer 22:10, total 412.25, calendar event
`2026-08-07T16:40Z` (= 22:10 IST). Output format is v4's untouched.

## v11, and how this ended

v11 and v11b were built on 2026-08-08 and **never scored**. The OpenRouter credits that fed
`gravity-piece-ai` were exhausted by then, so any run reaching the model failed outright, and the
assignment closed before they were topped up. They are recorded here because an undocumented
artefact in a repository is worse than a documented dead end, and `agent.json` is one of them.

**v11 = v10 with the zero-AI parser removed.** v8 and v9 replaced the AI extraction with a
deterministic IRCTC reader: 7,545 characters of `prep`, returning a booking only when every
essential field was present. Both scored Rating 0.00, and the run log showed why - the failure was
the exhausted credits, not the parser. That was never disproved, and v11 dropped the parser anyway:
`prep` goes back to 3,660 characters that find the body, score it and gate on keywords, and the
`r_ai` router reads `prep.output.bookingFlag` rather than `needsAI`. The honest summary is that the
parser was abandoned without evidence against it, in a hurry, at the end of a budget.

**The other change in v11 is the one worth keeping.** `core` gained a completeness gate, which is
reviewer points #2, #3 and #4: a booking has to carry a start date, a destination or origin, and an
identifier - a confirmation code, or a date and destination and a known type - before anything is
created. `missing` names which of the three is absent. Without it the expense sheet was created for
mail that had not been confirmed to be a booking at all.

**v11b = v11 with `retryOnFailure` turned off on the `save` step.** A retried write of
`travel_state` re-applies a run's changes to state that already holds them, and the only thing it
can buy is a duplicate. One toggle, and the only difference between the two files.

**What the repository publishes.** `agent.json` is v11b, sanitised: its five connection ids (one
Gmail, four Google Sheets) replaced by `GMAIL` and `GOOGLE_SHEETS` placeholders, the instance's
`metadata.externalId` removed, and a description filled in. Those seven differences are the whole
of it, checked field by field rather than assumed. It is the only flow file in the repo: the `tao-lean-*.json` working copies are
gitignored on purpose. So the artefact here is the *last* build, which is not the *best* one. The
best scored 45.01/50 and is the v4 line in the table above; v11b has never been measured against
it, and on the evidence of this table - where the plainest build won and most improvements made
the graded text worse - it should not be assumed to be better.
