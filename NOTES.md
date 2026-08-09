# Gravity AI take-home — build notes

Source of truth: https://gravity.fast/docs (scraped 2026-07-31). The PDF guide is a summary;
where they disagree, docs win. Deadline: 5 days from the invite email.

## Scoring (this is what to optimise)

| Part | Points | What it measures |
|---|---|---|
| Quality | 50 | An AI reads the run's **final output** and rates it |
| Reliability | 25 | Did the run succeed |
| Speed | 12.5 | How long the run took |
| Cost | 12.5 | Credits used |

~40 = "decent". Unlimited resubmissions during the window. Score >35 + proof of a $1
Activepieces credit purchase = Gravity reimburses the $1.

**Biggest lever: the last step.** It must output the actual result as *text*, on the main
path. Never end on Gmail / Sheets / Slack — those return an id even on a perfect run, and the
grader sees an id. End on Ask AI, Code, Data Mapper, or Webhook Return Response. Make it a
finished deliverable: titled, structured, short summary + concrete results.

Speed and cost are almost entirely AI text volume: cap output length on every AI step, put word
limits in prompts, use a small model for mechanical work (classify/extract), skip AI calls when
there's nothing to process, batch writes, store instead of regenerate, HEAD not GET for link checks.

## Platform hard rules

- **One flow, one trigger.** Only the first flow in the uploaded file runs.
- Supported triggers: Web Form, Webhook, Schedule, Manual, app piece triggers (polling + webhook).
- **Blocked pieces (policy):** OpenAI, Azure OpenAI, Azure AI Foundry, Anthropic, Claude, AWS
  Bedrock, Google Gemini, Vertex AI, Mistral, Cohere, Perplexity, Groq, DeepSeek, Grok.
  Allowed exceptions: Hugging Face, Stability AI (open-weight hosts).
- **All AI goes through the universal AI piece** (`@activepieces/piece-ai`, "AI & Agents" tab).
  Six actions: Ask AI, Summarize Text, Generate Image, Classify Text, Extract Structured Data,
  Run Agent. No API key needed — routed through Gravity's managed OpenRouter. At publish the
  platform swaps steps to `gravity-piece-ai` (metered). Never add that by hand.
  Cloud AI slugs (text-ai, image-ai, utility-ai, agent) are accepted and auto-rewritten.
- **Not supported:** chat trigger (use Web Form), file upload, Activepieces Tables (use Google
  Sheets), Activepieces Database (use MongoDB/Oracle), audio, Cloud-only pieces, unknown slugs.
- **Connections must be one-click OAuth.** Never ask a user for an API key. If a source offers
  both OAuth and API key, take OAuth. If it only offers API keys, find a different source.
- **Never hand-edit the exported JSON.** Hand-edited exports lack internal fields and fail import.

## The 7 review guidelines

1. **One-click sign-ins only** — OAuth or no account at all (RSS, public pages, webhooks).
2. **Four questions or fewer** — depth belongs in the flow, not the form.
3. **Create everything the agent needs itself** — on first run, create the sheet/folder with the
   exact columns + one example row, then message the user the link. Users enter data, never structure.
4. **A real trigger** — schedule, inbox event, file drop, new sheet row, webhook. Set up once,
   never started again. The trigger is what makes it an agent instead of a tool.
5. **Make it remember** — Storage piece with **FLOW scope** (RUN scope is wiped every run). One
   state object, saved at the end of every run, plus a "have I already done this?" check near the
   start so re-runs don't duplicate.
6. **Approval gates in the right place** — anything that messages a third party or touches money
   waits for a one-tap yes; internal work (sheets, drafts, logs) runs free. Paused runs are free.
7. **Trim what you feed the AI** — the #1 failed run is a whole scraped page dumped into a small
   model. Summarize/Extract first, then prompt. Test on worst-case input.

Plus (from the PDF): never make people do homework — the data is already in their Gmail/Calendar/
Drive, go get it. Repeated user effort is only OK when they get something back immediately or are
replying to a question the agent asked.

## Approval patterns (pick one)

| Bounty says | Pattern |
|---|---|
| "queued for my one-tap approval", "waits for my yes" | Approval email (Gmail *Request Approval in Email*) — the default |
| "approve the whole batch in one tap" | One approval email covering the batch |
| "I tweak it before it sends", "it learns from my edits" | Editable draft in the user's Gmail drafts; watch sent mail |
| "ask me and I reply" | Reply confirmation — ask by email, read the reply in the thread |
| User lives in Slack | Slack approval buttons (Teams/Discord/Telegram equivalents exist) |

Rules for all: silence never means yes; time out gracefully; remind once at most.
**Do not use** the "Approval (Legacy)" piece (AI assistants reach for it first — don't let them)
or the Chat UI trigger.

## Publish Gate

`gravity-piece-publish-gate`, single action "Continue Only If Published". Optional today, but
recommended. Place it after the trigger + content-prep steps and before every externally visible
step. Add it from the step picker only. The pre-publish end-to-end test runs as a genuinely
published automation, so real actions fire exactly once there — intentional.

## Builder questions

- A question = one plain field on one step with "Ask user" toggled on. Text, number, dropdown,
  checkbox, date, list. JSON object fields can never be asked. File upload not supported.
- **Web Form trigger is the easiest way to create questions.** If you can't use it, add a Storage
  Get step per value with a key nothing else writes, put the value in Default Value, and name the
  step in plain English — it acts as a constant with an askable field.
- **Bind** fans one answer out to the same property name on several nodes (e.g. a Slack channel
  used by two steps, a sheet ID used by read + write). Badge shows the bound node numbers.
  Bind is fan-out only — it does not merge different properties.
- The field list shows *property* names, not step names — toggle "Ask user" and write the question
  one field at a time so they stay attached correctly.
- **Clear your test values before publishing.** Whatever sits in a field becomes every user's
  pre-filled answer.

### The seven "don't ask" tests
1. Can the agent get it from the connection? → don't ask
2. Can the agent create it itself? → don't ask, create and name it
3. Can it work it out from data it receives? → don't ask, let it decide
4. Does the platform already ask it (e.g. the schedule picker)? → don't ask
5. Same answer for nearly everyone? → default it
6. Is it a credential? → never ask, use OAuth
7. Just a preference where any value works? → default it

What survives is 2–3 questions, in one of three shapes: **target** (where output goes),
**subject** (what to work on), **limit** (a number that changes behaviour).

Wording: user's words not field names ("Where should we send the daily report?"), one thing per
question, always a description with a real example, dropdowns for fixed choices, mark required.
A stranger should answer everything in under a minute, and a wrong answer must not break the agent.

## Build practice

- Keep the main path flat, top to bottom. Router for decisions, shared steps after it (not copied
  into every branch).
- Math, dates, counting, sorting → **Code step** (exact, instant, free). AI only for judgement:
  writing, classifying, summarising.
- "Continue on failure" on for optional steps, off for the ones that matter. Add a fallback that
  still tells the user something honest. If the user's answers are missing, do nothing at all.
- When asking AI for JSON: "return raw JSON only, no code fences, no explanation", then parse with
  a JSON step.
- Silent breakers: mixed piece versions (same piece must use the same version everywhere or import
  fails); made-up condition names (there is no "does not exactly match" — put the skip case in the
  condition and the real work in "Otherwise"); use **Does Not Exist** to test for empty, not
  comparison with ""; never reference a loop's item outside the loop; verify an app actually has
  the action before designing around it.

## Testing before submit

- Test each step as you build; check the **Input** tab, not just Output — most bugs are visible there.
- **Run it twice** — the second run is where duplicate bugs show up.
- Test unhappy paths on purpose: missing answers, no data, dead link, failing step, worst realistic input.
- For agents that remember, seed the stored state by hand to test later runs without waiting days.
- The reviewer sees exactly one run — make the submitted run a good one.

## Credits

Every new Activepieces account gets 200 free credits. If you run out: export the workflow, make a
new account, import, continue. (Or $1 for 1,000 credits, reimbursable per above.)

## Useful links

- Portal: https://gravity.fast/builder-assignment/ (sign in with ca.itiarora@gmail.com)
- Docs: https://gravity.fast/docs
- Activepieces docs: https://www.activepieces.com/docs
- Activepieces MCP for AI assistants: https://www.activepieces.com/mcp/activepieces
- Support: vivek@gravity.fast
