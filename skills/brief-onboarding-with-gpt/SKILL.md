---
name: brief-onboarding-with-gpt
description: >
  ChatGPT Work version of the xTiles onboarding preview — use this variant in
  ChatGPT; in Claude use `brief-onboarding` instead.

  Use immediately after a user finishes the xTiles onboarding questionnaire —
  builds their first Daily planner **preview** from their role and the tools
  they said they use, so they see the value of the service before doing
  anything themselves. This same skill also serves every **recurring** run
  once the user schedules it.

  Entry data on a first run (already known, never re-asked with a form):
  `role:` — the role from the questionnaire; `used_connectors:` — the tools
  the user said they use there (may include a custom name, or `other`).
  **Connection status is never handed to this skill as data — it determines
  that itself**, with a lightweight live probe per named connector. Gmail and
  Calendar are probed first, as the highest-value connectors.

  Detect the surface, not the request: inline `ask_user_input` / `genui` forms
  mean ChatGPT Work — this variant. `show_widget` or `AskUserQuestion` mean
  Claude / Cowork — use `brief-onboarding` instead.

  Environment triggers: "Brief Onboarding for GPT", "Brief Onboarding in
  ChatGPT", "the GPT version".

  Triggers: "start onboarding preview", "show me what my Daily could look
  like", "onboarding welcome digest", "first-run preview",
  "run brief-onboarding-with-gpt".

  Only the Daily period is supported.
---

# xTiles Onboarding — First & Recurring Daily Preview (GPT)

One Daily planner page that turns the user's role and the tools they said
they use — already captured by the onboarding questionnaire — into a real
preview of their morning brief. **Period is always Daily** — never ask which
period. This same skill also serves every recurring run once scheduled —
there is no separate daily-digest skill to hand off to.

## Seven principles

1. **Never re-ask what onboarding already answered.** Role and the tools the
   user said they use come in as `role:` and `used_connectors:` — never show
   a role/tools form.
2. **Never trust a name as proof of connection.** Whether something is
   actually connected is never handed to this skill as data — it's
   determined live, per connector, with a lightweight probe (Stage 1). A
   tool the user *said* they use may not be connected yet, or may have been
   connected since.
3. **Never block on a connector.** The one form this skill shows for
   connectors always has a way to proceed with nothing resolved at all.
4. **Never end a run with nothing to show.** If zero connectors end up
   usable, don't ship an empty digest — research and build a News tile
   instead (Stage 2), so the user gets real value from the very first run.
5. **Forms first, xTiles last; real data only.** Nothing is written until
   the user has seen a real preview and approved it. Never invent a person,
   message, meeting, link, or count.
6. **Personal planner only.** The single allowed write is
   `xtiles_create_tiles_from_markdown_in_my_planner` with `period: "day"`.
   Never create a project, a view, or a standalone page.
7. **Every write is followed by the layout pass**, then the CTA link. A run
   that ends with text in chat instead of tiles in xTiles is a failed run.

Match the language of the incoming onboarding message, and adapt if the user
switches. Every label in this file (`Needs action`, `FYI`, `Noise`, `Open
email`, `Tasks`…) is an English placeholder, not literal output.

---

**Tool names** in this file are capability names, not host-specific ones. Call
whatever the current surface exposes for that capability. If a required
xTiles capability is genuinely missing, say so plainly and stop — never
substitute a different write path.

---

## Form protocol

Every question is delivered through the host's `ask_user_input` surface as a
`genui` directive emitted **directly in your assistant message** — but it
only renders as an interactive form when wrapped in the host's three
**invisible sentinel characters** (Private-Use-Area code points). Without
them the host prints the raw JSON as text, which the user must never see.

The wrapper (sentinels shown here by code point — they are invisible in the
file and in output):

- **Prefix:** `U+E200`, then the literal `genui`, then `U+E202`
- **Payload:** the JSON object `{"ask_user_input":{"questions":[ … ]}}`
- **Suffix:** `U+E201`

UTF-8 bytes — U+E200 = `EE 88 80` · U+E202 = `EE 88 82` · U+E201 = `EE 88 81`.

Every `genui{…}` block below already carries these sentinels around it.
Reproduce them **exactly**, including when you build a form dynamically —
never emit a bare `genui{…}` without the sentinels, and never render the
code points or JSON as visible text.

**Interactive-form contract — follow exactly whenever user input is required:**

1. Output exactly one short introductory sentence.
2. Immediately after it, emit the form directly in the assistant message:
   U+E200 + `genui` + U+E202 + valid JSON payload + U+E201.
3. Use the literal Private-Use-Area characters, **not** the string `U+E200`
   or its UTF-8 byte notation.
4. The directive must **not** be inside a Markdown code fence, blockquote,
   inline code, XML, or explanatory prose.
5. End the turn immediately after the closing U+E201 character.
6. Never call `functions.request_user_input`; it may be unavailable in
   Default mode.
7. Never print the JSON as visible text, and never replace the form with a
   numbered list or a plain-text question.
8. Every form must contain **1–3 questions**.
9. Every question must contain **2–10 options. Never exceed 10.**
10. If more than 10 options are needed, split them across two questions or
    consecutive forms. **Never silently drop an option.**
11. Every question must include: `question`, `options`, `type`
    (`single_select` or `multi_select`), and `free_text_placeholder`.
12. Nothing is preselected.
13. Parse the next user message as the form response and continue from the
    next workflow stage. Do not repeat an already-answered question.

Also: never substitute another surface (`show_widget`, `sendPrompt`,
`visualize`, HTML fragments, `window.openai.sendFollowUpMessage`,
`AskUserQuestion`, or Claude scheduling tools). An empty answer (`Не
вибрано`, `Не выбрано`, `Not selected`, blank) → one sentence saying what is
required, then re-emit **the same** form; never infer consent. **The one
exception is Stage 1's Connector check form** — an empty answer there means
"skip all" and must never be re-emitted (see Stage 1). Free text is kept
**verbatim** for any "Other …" value. Carry the accumulated config through
every stage.

The chain: **Entry + Connector check (if needed) → preview → Approval →
write + layout → CTA → Schedule → Related**.

---

## Run modes

Two ways this skill starts:

- **First run.** The incoming message carries `role:` and `used_connectors:`
  only — no resolved connector list yet. Always right after the onboarding
  questionnaire. **Before anything else, send one short, plain-language
  sentence of context** — e.g. "Setting up your first Daily preview for a
  {role}, based on the tools you said you use." — before any silent probing
  and before Stage 1's form, if one turns out to be needed. Never let the
  very first thing the user sees be a bare form with no context. Then start
  at Stage 1.
- **Recurring run.** The incoming message instead carries the **full config
  this skill itself wrote at the end of a previous run** (Stage 8) —
  `role:`, `tools:` (the already-resolved set),
  `skipped:` (connectors that weren't connected last time, if any),
  `news_categories:` (only present if the resolved set was empty last time
  — see Stage 2), and `notify:`. Its presence (specifically `tools:`
  alongside `role:`) is the
  signal — there is no separate scheduled-run skill to hand off to. This
  runs silently — no intro sentence, nobody is watching chat. **First,
  silently re-probe every connector in `skipped:`** (same lightweight probe
  as Stage 1, no form): if one now succeeds, fold it into today's resolved
  set and mention it once, briefly, in today's preview or notification
  ("Gmail just connected — added to today's brief"). This is the only
  re-check that ever happens on a recurring run. Then go to **Stage 2**
  (a mandatory checkpoint — it triggers the News fallback only if the
  resolved set is still empty, otherwise it's a no-op) **and then Stage 3
  (Silent fetch)**, using the resulting resolved set.

  **What actually runs after the write on a recurring run — spelled out
  exactly, nothing implied:** the layout pass always runs; Gmail
  follow-through (Stage 7's item 3) always runs if Gmail is in the resolved
  set, whether or not anyone is watching chat — it's inbox hygiene, not a
  chat-visible action; and — only if `notify:true` — the notification
  (Stage 8) fires. **The approval form, the CTA, the schedule form, and the
  related-workflows question never fire on a recurring run** — those, and
  only those, are what "silent" excludes.

---

## Stage 1 — Connector check

**There is no fixed catalog of connectors in this skill.** `used_connectors`
can name anything — Gmail, Calendar, Slack, a tool invented after
this file was written, or `other` with a name typed by the user. Treat every
name the same way:

1. For each connector in `used_connectors`, make one lightweight, read-only
   probe call using whatever capability that connector exposes (a minimal
   list/search call, never a write). A response with no auth error means
   it's connected right now; an auth error or a missing capability means it
   isn't. **This probe result — not the questionnaire answer — is the only
   source of truth for "connected."**
2. **Gmail and Calendar are probed first**, since they tend to carry the
   richest everyday signal.
3. For an unfamiliar connector name, look for a capability whose name
   matches it and use the least invasive read call available. If none
   exists, treat it as not connected — a candidate to connect natively or to
   skip.
4. **If `other` is in `used_connectors`** — it's a signal the user's real
   stack is bigger than what they listed. Beyond probing the named
   connectors, check what other connector capabilities this session
   actually has available and quickly probe any that weren't named.
   Anything that responds successfully becomes an **Add {name}** candidate
   in the form below — distinct from **Connect {name}**, since it's already
   usable and just needs opting in, no auth flow required.
5. **If `used_connectors` is non-empty and every named connector's probe
   succeeds, and point 4 above found no extra candidates to offer — skip
   straight to Stage 2** (a no-op there, since the resolved set isn't
   empty) **and then Stage 3.** No form at all. **If `used_connectors` was
   empty from the very start, this is not that case** — zero probes is not
   zero failures, treat it exactly like an empty resolved set and go to
   Stage 2.
6. **Otherwise** (something failed to connect, or there's an extra
   candidate to offer), the one introductory sentence before this form
   (per the Form protocol, rule 1) must **name what's already connected**
   — e.g. "Gmail and Slack are already connected — want to connect
   Calendar too, add anything else, or just skip and see your preview?"
   Never let the form appear with no context about what's already fine.
   Then emit one form:

```
genui{"ask_user_input":{"questions":[
  {"question":"Want to connect or add anything before I build your preview?","options":["Connect Gmail","Connect Calendar","Add {extra candidate}","Skip all — show my preview now"],"type":"multi_select","free_text_placeholder":"Name another connector"}
]}}
```

Build the options dynamically: one `Connect {name}` per connector whose probe
failed (Gmail and Calendar first), one `Add {name}` per extra candidate found
in point 4, and always the fixed final option `Skip all — show my preview
now`. **Cap at 9 dynamic options plus the fixed one = 10** (rule 9). **If
more than 9 connectors need offering, keep the highest-priority ones here
(Gmail and Calendar first, then the rest in the order named) and follow up
immediately with a second `multi_select` form for the overflow** — per the
Form protocol's own rule 10, never silently drop one.

**This form is the one exception to the Form protocol's empty-answer rule.**
An empty or all-unselected answer means the same as picking `Skip all` —
proceed straight to Stage 3 with whatever probed successfully. **Never
re-emit this form to demand a selection.**

For every `Connect {name}` picked, run the native connect flow for that
connector (say its name explicitly first, then use whatever connect
capability the current surface exposes; confirm once finished) before
continuing. Every `Add {name}` picked needs no connect flow at all — it's
already usable, just fold it straight into the resolved set. If a connect
attempt fails or stalls, drop that connector and continue — never block the
run over it.

**xTiles itself is required, not optional** — if it is not connected, this
skill is not reachable at all; connect it first, outside this flow.

**Resolved set** = every connector whose probe succeeded, plus any just
connected or added, minus anything skipped. **Track the skipped list too** —
carried forward as `skipped:` into the schedule config in Stage 8, so a
future recurring run can quietly notice if one of them gets connected later
(see Run modes) without ever asking again. Every later stage reads this set.

**When to ask — and when never to ask again automatically.** This form
fires **at most once per run**, right after the intro sentence, and only on
a first run. It is not a recurring nag:
- On a **recurring run**, this form never appears at all — Run modes handles
  it with a silent re-probe of `skipped:` instead.
- The user can always trigger a fresh check by asking directly at any time
  ("connect my Slack now") — that re-enters this stage for just the named
  connector, regardless of run mode.

**No content-preference questions, ever.** Every connector in the resolved
set contributes its own default content (Stage 3) — the user can still ask
to change anything at the preview/approval step.

**Two worked examples of the probe — not an exhaustive list, the pattern is
the same for anything else the user names:**
- **Gmail** — call `list_labels` (cheap, no query needed). Success = connected.
- **Calendar** — call `list_events` with `maxResults:1`. Success = connected.

---

## Stage 2 — Fallback check: a News tile when nothing is usable

**This is a mandatory checkpoint, not an optional detour — every path from
Stage 1 or Run modes passes through here, on a first run and on a recurring
run alike. It is never valid to route straight from Stage 1 or Run modes to
Stage 3.**

Check the resolved set:
- **Non-empty** — nothing to do here. Continue straight to Stage 3.
- **Empty** — whether because `used_connectors` was empty from the very
  start, every named connector's probe failed, or the user skipped
  everything in the form — don't ship an empty run. Build a News tile
  instead, right here, before Stage 3:
  1. **If the incoming config already carries `news_categories:`** (a
     recurring run that fell back before) — use those exact categories,
     don't re-derive them; they were chosen deliberately for this person
     and should stay stable run to run. **Otherwise** (first time this
     fallback triggers), infer 2–4 topic categories from `role:` that this
     person would plausibly care about right now (the same judgment
     `today-news-with-gpt` uses — e.g. a Product Manager cares about
     product/UX trends, competitor moves, and AI tooling news). If the
     role doesn't narrow it down, default to broadly useful categories:
     industry news, productivity/tools, and a general "worth knowing"
     pick. **Whichever way they were obtained, this exact category list is
     what carries forward into `news_categories:` in Stage 8** if the user
     schedules a recurring run — spell it out explicitly there, don't
     leave the next run to silently reinvent it.
  2. Use whatever web-search and page-fetch capability this surface
     exposes to find real, current items from the last 24–48 hours per
     category, verified against reputable sources. **Never invent an
     item, a date, or a link.** If a category genuinely yields nothing,
     drop that category rather than force it; if literally every category
     comes back empty, say so plainly in the preview instead of writing an
     empty tile.
  3. Build **one** tile, `### 📰 News for you`, with one labeled
     sub-section per category and 2–3 real items each — one line per item,
     source linked inline.
  4. This is a fallback, not a permanent feature — the moment even one
     connector is usable (this run or a future one), skip it entirely and
     go back to normal per-connector tiles.

---

## Stage 3 — Silent fetch

No messages while fetching. Record a connector **error** separately from an
empty **result** — they render differently. Pull fresh data from every
connector in the resolved set, and — if Stage 2 triggered — research the
News tile instead.

**There is no single grouping that fits every connector — the right shape
follows the nature of the data itself, never a template repeated for each
one.** Before building a tile, ask what *this specific kind of data*
actually needs, not "which of the usual three buckets does this go in."

**And separately — whether that shape becomes one tile or several is a
question of volume, not a fixed rule per connector.** A handful of items
reads fine inside one tile with labeled internal sections; it's genuine
volume in one of those sections that earns it a tile of its own. Never
split into several thin, mostly-empty tiles just because a connector
"usually" gets split, and never cram a genuinely large volume into one
dense tile either — let what was actually pulled decide, each time.

- **Email arrives as a firehose that needs triage — the natural question is
  "do I have to act on this."** That's why it splits by urgency: 🔴 needs a
  concrete next step, 🟡 informational only, ⚪ automated noise. **With
  real volume in each bucket**, this becomes three tiles (Stage 6):
  `### 📩 Email — Action Points` (🔴, plus real `<task>`s), `### 📩 Email —
  Key People` (🟡, grouped by *sender* — a completely different axis from
  urgency), `### 📩 Email — Noise` (⚪, one rollup line, never itemized).
  **With only a handful of relevant emails**, keep it all in one
  `### 📩 Email` tile instead, with the same three labeled blocks inside
  it. Tone either way: retell the email in second person, action +
  consequence, don't copy the subject line — "Google shut down your ad
  account yesterday — log in and appeal, the window is limited," not "Your
  account closed."
- **Slack already arrives grouped — by channel and by thread — so urgency
  isn't the useful axis there.** The real question is "was this addressed
  to me, or is it ambient discussion I can skim." **With enough real
  activity**, that becomes two tiles: mentions/DMs that need a reply as
  real `<task>`s, and a short topics rollup. **On a quiet Slack day**, fold
  both into one `### 💬 Slack` tile instead, with the same two labeled
  sections inside it. Either way, never reshuffle it into an
  urgency/FYI/noise split — that throws away the channel structure that's
  the whole point of Slack, regardless of tile count.
- **Calendar isn't a triage problem at all — there's no "noise" in
  someone's schedule, and a single day only ever needs one tile.** The
  natural shape is chronological and forward-looking: what does today look
  like, where's the free time, what's worth preparing for. One
  `### 📅 Workload` tile — never a second copy of the schedule the user
  can already see, and never split by volume the way Email or Slack might
  be. Compute event count, hours occupied, and the longest free focus
  window; write one concrete 🎯 focus-recommendation sentence; for each
  event, a one-sentence agenda (from the event description or the most
  recent related email/meeting note — never invented) and, only where
  genuinely implied, one prep `<task>`. Collect anomalies at the bottom,
  never inline.

**Apply the same kind of thinking to anything else in the resolved set** —
Linear, Google Drive, or a connector this file has never heard of.
Read what it actually returns, decide the natural axis (urgency, person,
project, time, or something else entirely), and only split it into more
than one tile when there's genuinely enough volume on that axis to earn
separate scanning — never reuse Email's or Calendar's exact shape, or their
tile count, for a different kind of data just because it's already written
down here.

Never fabricate a category that has nothing in it — omit an empty block
rather than write "none" three times over. A tile from a connector that
returned genuinely nothing still gets written, with one line saying so.

**If Slack is in the resolved set and the user hasn't named channels**, pick
a small, sensible set yourself before reading: favor channels the user has
posted or been mentioned in recently, plus any obviously company-wide ones,
capped at around five. A judgment call, not a fixed algorithm.

**If Gmail is in the resolved set**, also check for unread newsletters and —
if any are found — collect them for a single `### 📧 Newsletters` tile (one
line per publication, title as the hyperlink). Skip the tile if none found.

**Cross-source overlap.** If the same real ask clearly shows up in two
different connectors, don't produce two separate action items — treat one as
primary, and add a short cross-reference on the other's line. Only merge
when it's genuinely the same ask.

**Derive action items.** Every 🔴 email, every genuinely-needed meeting
prep, and anything else that clearly implies a next step yields one
verb-first `<task>`. A stated deadline overrides the default date; genuine
urgency sets `priority`. Absent either, the task takes the page's day as its
`dueDate` and gets no `priority`.

Use only real data. Never invent names, events, messages, or links. If a
connector call fails outright, record that as a failure, surfaced explicitly
in the preview — never silently written as "no data."

---

## Stage 4 — Preview

Show the real preview in chat, one section per tile that will actually be
written, with real names, counts and links. Render empty reads explicitly
("No unread emails", "No updates today") and failures explicitly ("Could not
fetch Calendar — connector error"). Blank line between items.

Before the approval form, call `xtiles_get_current_user` and show the
destination email on the line above it — approval confirms that account.

Then **stop and wait**. Nothing is written yet.

---

## Stage 5 — Approval

```
genui{"ask_user_input":{"questions":[
  {"question":"Create this Daily in xTiles?","options":["Create it","Change something","Cancel"],"type":"single_select","free_text_placeholder":"Say what to change"}
]}}
```

`Change something` → ask what, update **only** that part, re-show the
preview, emit this form again. `Cancel` → acknowledge and stop.

---

## Stage 6 — Write + layout

Only after `Create it` (or immediately, on a recurring run — see Run modes).
**This skill always runs manually, once per morning — a recurring run is
this same skill invoked again, not a different one** (see Stage 8).

1. `xtiles_get_user_timezone` → today's local date as `yyyy-MM-dd`.
2. **This skill has no way to update an existing tile's content, so every
   run creates a fresh set of tiles.** Write all the approved sections in
   the create call below regardless of what's already on the page — a
   re-run is expected to produce a fresh, current brief.
3. **One** call to `xtiles_create_tiles_from_markdown_in_my_planner` with
   `period: "day"`, today's `date`, and all sections in a single markdown
   string. Inspect the schema first: it must accept `date`, `period`,
   `markdown` without `projectId`/`viewId`. If it demands either, report
   that the personal Daily write is unavailable and stop.
4. **Layout pass — mandatory, silent, never asked about.** Take `view_id`
   and the ordered `tile_ids` from the write response (never re-derive
   them), call `xtiles_get_workflow` with id `tile-layout` and follow it
   exactly, **passing `tile_ids` as its "added tiles" and the markdown just
   written as their content** — those are required inputs the workflow
   itself expects, not optional context. Hints: default 2 tiles per row;
   give a heavy tile its own full-width row. This workflow is the one that
   actually calls `get_page_layout`/`set_page_layout` — skipping the input
   handoff here is why it can silently do nothing.

### Tile format

Each `###` section carries its colour annotations immediately under the
heading, no blank line:

```
### 📩 Email — Action Points
@colorSize: LIGHTER
@color: SAIL
```

`@colorSize` is always `LIGHTER`. `@color` is picked from `GHOST, CUMULUS,
GOSSIP, COLDTURKEY, BLUE_CHALK, MILK_PUNCH, HAWKES_BLUE, PATTENS_BLUE, SAIL,
ATHENS_GRAY, BERMUDA, PERFUME, SELAGO, RICE_FLOWER, WHITE_LINEN, POLAR` —
never a semantic name, never the same colour twice in a row.

- **No date or title-only tile.** Start with content.
- The heading emoji names the *subject*, never a status. Status markers
  (🔴 🟡 ⚪ ⏳ ✅ ❓) belong on item lines, never in a heading.
- **Links are inline hyperlinks inside a sentence** — `… → [Open email](url)`.
  A line containing only a link renders as a block card. Never a bare URL,
  never a link on its own line, never a link inside a `<task>`.
- Blank line between every item.

**Action items are tasks, not checkboxes:**

```
**Action items**

<task dueDate="2026-08-11">Restore the Google ad account</task>

<task priority="high" dueDate="2026-08-10">Sign the Acme contract</task>
```

One `<task>` per line, blank line between, never nested in a list item.
`dueDate` always set, defaulting to today; a later real deadline from the
source overrides it. `priority` only when the source signals it — at most a
third of a morning's tasks should be `high`. Never `completed="true"`.

**Worked tile examples, in full markdown — the same two connectors as
Stage 3's Email and Calendar, each shown in its split (high-volume) case.
Everything else follows Stage 3's reasoning (the shape *and the tile count*
follow the data), never a layout copied from these:**

```
### 📩 Email — Action Points
@colorSize: LIGHTER
@color: SAIL

- [Poke-style description — action + consequence, second person] → [Open email](url)

---

**Action items**

<task dueDate="2026-08-11">Restore the Google ad account</task>
```

```
### 📅 Workload
@colorSize: LIGHTER
@color: BERMUDA

**N events · ~X h occupied · longest focus window HH:MM–HH:MM (X h)**

🎯 [Focus recommendation — one concrete sentence]

**HH:MM–HH:MM · Meeting name** — Participant · [Google Meet](url)

📋 [Agenda — one sentence, from a real source]

<task dueDate="2026-08-11">[Prep task, only if genuinely implied]</task>
```

`### 📩 Email — Key People` (🟡, grouped by sender) and `### 📩 Email —
Noise` (⚪, one rollup line) follow the same pattern as Action Points,
scoped to their own bucket. `### 📧 Newsletters` (if any found) is one line
per publication: `**[Publication](url)** — one-line summary.` `### 📰 News
for you` (the Stage 2 fallback, if it ran) is one labeled sub-section per
category with 2–3 linked items each.

**The combined (low-volume) case** uses the exact same blocks, just inside
one tile instead of several:

```
### 📩 Email
@colorSize: LIGHTER
@color: SAIL

🔴 **Needs action**

- [Poke-style description] → [Open email](url)

---

**Action items**

<task dueDate="2026-08-11">Restore the Google ad account</task>

🟡 **Key people**

**Name (context)**

- [One-line item — no link]

⚪ **Noise**

- N notifications — nothing urgent
```

Same idea for Slack — a quiet day gets one `### 💬 Slack` tile with a
`**Mentions**` block (real `<task>`s) and a `**Topics**` block underneath,
instead of two separate tiles.

---

## Stage 7 — CTA and Gmail follow-through

1. Confirm in one line: `✅ Daily created.`
2. **CTA** — link to the **first created tile**, not the page: use the
   `resource_url` of the **first** entry in the write response's `tiles`
   array **byte-for-byte**. Never rebuild it from `view_id`. Only if the
   first tile has no `resource_url` fall back to `parent_resource_url`; if
   neither exists, say so and stop before the schedule offer.
   `✅ Daily created. [Open in xTiles →]({first tile resource_url})`
3. **If Gmail is in the resolved set — mandatory, silent, every run:** mark
   every ⚪ Noise and newsletter thread as read (remove the unread label).
   Never touch 🔴 or 🟡 threads, and never draft or send anything on the
   user's behalf.

---

## Stage 8 — Schedule

**A schedule set up here re-invokes this same skill,
`brief-onboarding-with-gpt`, every morning — there is no separate
daily-digest skill.** The whole point of resolving everything once is that
the recurring run can skip straight past Stages 1–2.

```
genui{"ask_user_input":{"questions":[
  {"question":"Run this automatically every morning?","options":["Schedule it","No schedule"],"type":"single_select","free_text_placeholder":"Another cadence"}
]}}
```

On `Schedule it`, one follow-up form for cadence, time, and notification:

```
genui{"ask_user_input":{"questions":[
  {"question":"Which days?","options":["Weekdays","Every day"],"type":"single_select","free_text_placeholder":"Specific days"},
  {"question":"What time?","options":["08:00","09:00","10:00"],"type":"single_select","free_text_placeholder":"Another local time"},
  {"question":"Notify me in xTiles each time it runs?","options":["Yes, notify me","No notification"],"type":"single_select","free_text_placeholder":"Something else"}
]}}
```

Resolve the timezone with `xtiles_get_user_timezone`, then create the
automation with an exact schedule whose prompt calls **this same skill**
(`brief-onboarding-with-gpt`) and embeds the **full recurring-run config**,
assembled from this run's resolved state:

```
Run brief-onboarding-with-gpt — role: {role} · tools: {resolved connector set} · skipped: {connectors whose probe failed or that the user skipped, or "none"} · news_categories: {the exact categories Stage 2 used this run — only include this field at all if the resolved set was empty} · notify: {true/false} (if true, call xtiles_create_notification at the very end of that run — mandatory, do not skip) · schedule: daily-{HH:MM} days:{days}
```

**`news_categories:` is the one field that's conditional — include it only
when the resolved set is empty this run (the News fallback fired).** When
there's at least one real connector, omit it entirely. When it does apply,
this is important enough to spell out explicitly, not leave for tomorrow's
run to silently reinvent — it's what makes a connector-less recurring
digest actually consistent day to day.

**No `daily_content:` field** — Stage 3 always re-derives each connector's
content fresh from what it actually returns that day, so a snapshot from
setup time would only ever be stale; there's nothing to gain from
persisting one. This is exactly the config the Run modes section looks for
to detect a recurring run — assembling it correctly here is what lets
tomorrow's run skip the connector check entirely, while `skipped:` still
lets it quietly notice a later connection (see Run modes). Never leave a
placeholder unresolved.

**If `notify:true`** — the digest already on the page right now is worth
notifying about too. Call `xtiles_create_notification` immediately: `url` is
the same tile-focused deep link resolved in Stage 7, `text` is the fixed
string "Your Daily Digest is ready — see what matters today in 2 min."
translated into the user's language (no dynamic part), `agent_source` is
"ChatGPT".

Confirm the chosen time and timezone afterwards (append ", and I'll notify
you in xTiles each time" when `notify:true`).

**If the scheduling capability genuinely isn't available in this
environment — this is not a dead end.** Say so in one plain line ("I can't
set up automatic scheduling here — but your Daily is ready, and I'll build
a fresh one anytime you ask.") and, if `notify:true` was requested, still
send today's notification per the paragraph above — that part never
depended on the schedule actually being created. **Then continue to
Stage 9 in the same turn, exactly as after `No schedule`.** A missing
scheduling capability skips the schedule itself, never the mandatory
closing stage.

---

## Stage 9 — Related workflows

Offer the other three workflows, each with a one-line description of what it
does — never a bare list of names. Never offer `brief-onboarding-with-gpt`
itself here — its own digest and schedule are already handled in Stage 8.

```
genui{"ask_user_input":{"questions":[
  {"question":"Want to set up anything else on xTiles?","options":["🌙 Evening Reflection — an end-of-day synthesis and a seed for tomorrow","📰 Today News — a daily topic-based news digest from the live web","📊 Weekly Review — what actually moved forward this week","Nothing else"],"type":"single_select","free_text_placeholder":"Something else"}
]}}
```

Treat the selection as a direct invocation: **in the same turn**, call
`xtiles_get_workflow` with the matching id and continue from its first
applicable stage:

| Option | Workflow id |
| --- | --- |
| Evening Reflection | `evening-reflection-with-gpt` |
| Today News | `today-news-with-gpt` |
| Weekly Review | `weekly-review-with-gpt` |

Never print a handoff command, a `workflow_id`, or `Use $...` as user-facing
text, and never make the user repeat the choice. `Nothing else` → acknowledge
and stop.

---

## Closing rule

After a successful write, **every** terminal response repeats the same
labelled CTA link as its final line — after `No schedule`, after `Nothing
else`, after a later correction, after a connector clarification. A
successful manual run never ends without it. A recurring run ends silently
after the layout pass and Gmail follow-through (Stage 7's item 3, if
applicable) — and, only if the config's `notify:true`, after the
notification in Stage 8.
