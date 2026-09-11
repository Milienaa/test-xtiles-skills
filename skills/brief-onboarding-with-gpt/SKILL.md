---
name: brief-onboarding-with-gpt
description: >
  ChatGPT Work version of the xTiles onboarding brief — use this variant in
  ChatGPT; in Claude use `brief-onboarding` instead.

  Use immediately after a user finishes the xTiles onboarding questionnaire —
  builds and writes their first Daily planner brief directly from their role
  and the tools they said they use, so they see real value before doing
  anything themselves — no preview, no approval form. This same skill also
  serves every **recurring** run once the user schedules it.

  Entry data on a first run (already known, never re-asked with a form):
  `role:` — the role from the questionnaire; `used_connectors:` — the tools
  the user said they use there (may include a custom name, or `other`);
  `additional:` — optional, often absent; the only value recognized today is
  `News`/`News_yes` (case-insensitive), meaning the user explicitly asked for
  a news tile regardless of connector state. **A negative like `News_no`
  never counts, even though it contains "News"** — match the whole token,
  never a bare substring. These may arrive as labeled fields or as an equivalent
  natural bullet list (e.g. "Role: Marketing · My connectors: Notion, Google
  Calendar, Gmail, Other · Additional: News") — parse whichever shape carries
  these semantic slots, never re-derive them from a form.
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
  "run brief-onboarding-with-gpt", "Set workflow of Onboarding Brief
  (brief-onboarding) on xTiles MCP" — this handoff phrase names the shared
  workflow id used for **both** surface variants; which one actually runs is
  resolved by "Detect the surface, not the request" above, never by the id
  in the phrase itself.

  Only the Daily period is supported.
---

# xTiles Onboarding — First & Recurring Daily Brief (GPT)

One Daily planner page that turns the user's role and the tools they said
they use — already captured by the onboarding questionnaire — into a real
brief, written directly, no preview or approval step in between. **Period is
always Daily** — never ask which period. This same skill also serves every
recurring run once scheduled — there is no separate daily-digest skill to
hand off to.

## Principles

1. **Never re-ask what onboarding already answered.** Role and the tools the
   user said they use come in as `role:` and `used_connectors:` (plus an
   optional `additional:`) — never show a role/tools form.
2. **Never trust a name as proof of connection.** Whether something is
   actually connected is never handed to this skill as data — it's
   determined live, per connector, with a lightweight probe (Stage 1). A
   tool the user *said* they use may not be connected yet, or may have been
   connected since.
3. **Never block on a connector.** The forms this skill shows for
   connectors always have a way to proceed with nothing resolved at all.
4. **Never end a run with nothing to show.** An empty resolved set, or an
   explicit `additional: news` request, always produces a real Today News
   tile (Stage 2) — never an empty digest.
5. **No preview, no approval form.** Nothing is shown for sign-off — once
   Stage 3's fetch completes, write directly. Never invent a person,
   message, meeting, link, or count.
6. **A distinct tool is always a distinct tile, never merged by category.**
   Two calendars, or Gmail and Reclaim, never share a tile just because
   they're both "scheduling" or "inbox" — see Stage 3.
7. **Personal planner only.** The single allowed write is
   `xtiles_create_tiles_from_markdown_in_my_planner` with `period: "day"`.
   Never create a project, a view, or a standalone page.
8. **Never surface a third-party connector's own rendered preview or embed
   in chat — absolute, no exceptions, not even "just this once."** Read
   what Todoist, Calendly, Reclaim, or any other connector returns purely as
   data to build your own xTiles tile. **Todoist and Calendly in particular
   are known to auto-render their own rich preview/card the instant their
   link or reference appears in a message — never let that happen.** Never
   let a raw response render as its own
   card, never paste a bare URL that triggers a link-unfurl preview, never
   call a capability whose only purpose is to produce an embed, and never
   forward or echo a connector's own UI element into the conversation. If a
   connector's output would only be useful to the user as its own rendered
   card, that's a signal it belongs described in your own tile, never shown
   as-is.
9. **Never recreate a task that's already open.** Check `xtiles_list_tasks`
   before writing any `<task>` (Stage 4) and drop anything that duplicates
   an already-open task from yesterday or today.
10. **Every write is followed by the layout pass**, then the CTA link. A run
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

1. Output exactly one short introductory message (may be a few short labeled
   lines, e.g. a ✅/🔌/📰 status block — see Stage 1 — but never a long
   preamble).
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
required, then re-emit **the same** form; never infer consent. **The
exceptions are Stage 1's Connector check form and Stage 6's "Add anything
to your Daily?" question** — an empty answer on either means "skip
all"/"no, that's enough" and must never trigger a re-emit; Stage 6's other
question ("Run this automatically every morning?") still always needs a
real answer (see Stage 1 and Stage 6). Free text is kept
**verbatim** for any "Other …" value. Carry the accumulated config through
every stage.

**The chain: Entry + Connector check (if needed) → Today News (if
triggered) → Silent fetch → write + layout → CTA → Schedule → Related.**
There is no preview form and no approval form anywhere in this chain.

---

## Run modes

Two ways this skill starts:

- **First run.** The incoming message carries `role:` and `used_connectors:`,
  and optionally `additional:` — no resolved connector list yet. Always
  right after the onboarding questionnaire. **Before anything else, send one
  short, plain-language sentence of context** — e.g. "Building your first
  Daily Brief for a {role} — pulling it together from the tools you already
  use, so you open something real, not an empty page." — before any
  silent probing and before Stage 1's form, if one turns out to be needed.
  Never let the very first thing the user sees be a bare form with no
  context. Then start at Stage 1.
- **Recurring run.** The incoming message instead carries the **full config
  this skill itself wrote at the end of a previous run** (Stage 6) —
  `role:`, `tools:` (the already-resolved set),
  `skipped:` (connectors that weren't connected last time, if any),
  `news_categories:` and `additional:` (present together, and only when
  `news` was explicitly requested, so a future recurring run keeps building
  the tile even once other connectors are resolved — a pure fallback tile
  never persists either field), and `notify:`. Its
  presence (specifically `tools:` alongside `role:`) is the
  signal — there is no separate scheduled-run skill to hand off to. This
  runs silently — no intro sentence, nobody is watching chat. **First,
  silently re-probe every connector in `skipped:`, and also run the same
  connected-but-unused discovery pass as Stage 1 point 4** (check this
  session's own available connector capabilities for anything that responds
  successfully and isn't already in `tools:` or `skipped:`) — **this is the
  only place that discovery ever repeats automatically after the first
  run**, since Stage 6's own version of it (the "Add anything?" question)
  only ever runs once, at initial setup, and never fires again on a
  recurring run. If either check finds something new: fold it into today's
  resolved set, mention it once, briefly, in today's notification ("Gmail
  just connected — added to today's brief"), and **silently re-issue the
  same automation-creation call Stage 6 uses**, with the prompt's
  `tools:`/`skipped:` refreshed to match, so tomorrow's recurring run
  already reflects it too — without ever asking again. **Never show a form
  on a recurring run** — this entire step is silent whether or not it finds
  anything. Then go to **Stage 2**
  (a mandatory checkpoint — it triggers Today News only if the resolved set
  is still empty or `additional: news` carried forward, otherwise it's a
  no-op) **and then Stage 3 (Silent fetch)**, using the resulting resolved
  set.

  **What actually runs after the write on a recurring run — spelled out
  exactly, nothing implied:** the layout pass always runs; Gmail
  follow-through (Stage 5's item 2) always runs if Gmail is in the resolved
  set, whether or not anyone is watching chat — it's inbox hygiene, not a
  chat-visible action; and — only if `notify:true` — the notification
  (Stage 6) fires. **The Schedule form and the related-workflows question
  never fire on a recurring run** — those, and only those, are what
  "silent" excludes.

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
3. For an unfamiliar connector name, **check the known-bundle table below
   first** — only once it's confirmed the name isn't a known alias do you
   fall back to looking for a capability whose name matches it directly, and
   use the least invasive read call available. If no matching capability
   exists at all *and* no bundle row covers it, treat it as not connected —
   a candidate to connect natively or to skip. **Never report "no connector
   available" for a name that appears in the table below** — that's a wrong
   answer, not a missing one.

   **Known connector bundles — probe, offer, and connect once per bundle,
   never once per name:**

   | Named tool(s) the user may list | Underlying connector |
   |---|---|
   | Jira, Confluence | Atlassian (Rovo) |

   If two or more of the user's named connectors fall in the same row, they
   share **one** probe, **one** connect option in the form, **one** line in
   the status block, and **one** `Add`/`Connect` option — never a separate
   one per named tool. Connecting or skipping the bundle resolves every
   named tool listed in that row at once. **This applies just as much when
   the user names only one tool from a row** (e.g. just "Jira," with no
   "Confluence") — it still resolves through that row's underlying
   connector, never treated as its own unmatched name. **Label it by the
   bundle's own name in every user-facing line** (status block, form
   option) — e.g. `Atlassian`, never the raw name the user happened to
   type — so the label is consistent whether one or several names from the
   row were mentioned. **Record it in `tools:`/`skipped:` (Stage 6) the same
   way** — that's what a future recurring run's silent re-probe (Run modes)
   keys off of. **This table is illustrative, not exhaustive** — if the
   session has a matching capability under a different name for something
   not listed here, that still counts as a known connector, not an
   unsupported one.
4. **If `other` is in `used_connectors`** — it's a signal the user's real
   stack is bigger than what they listed. Beyond probing the named
   connectors, check what other connector capabilities this session
   actually has available and quickly probe any that weren't named.
   Anything that responds successfully becomes an **extra candidate** below
   — distinct from a missing named connector, since it's already usable and
   just needs opting in, no auth flow required.
5. **If `used_connectors` is non-empty and every named connector's probe
   succeeds, and point 4 above found no extra candidates to offer** — there
   is nothing to ask, but **still send the bold-badge status block** (point
   6's format below, ✅ line only, plus a 📰 line if Today News will run
   this pass) so the user sees confirmation of what's already connected
   before the flow moves on. **No form accompanies it** — a status line is
   not a question and needs no answer. Then go straight to Stage 2 (a no-op
   there, unless `additional: news` was requested) **and then Stage 3.**
   **If `used_connectors` was empty from the very start, this is not that
   case** — zero probes is not zero failures, treat it exactly like an
   empty resolved set and go to Stage 2 (skip the status line too — there's
   nothing to confirm).
6. **Otherwise** (something failed to connect, or there's an extra
   candidate to offer):
   - **Offer every named connector whose probe failed**, no cap, no
     deferral. The form protocol already supports up to 10 options per
     question and splitting overflow into a follow-up form (rule 10), so
     use that whenever there are more than 9.
   - The introductory message before the form below must be a bold-badge
     status block naming what's already fine, what's being offered, and
     whether Today News will run — e.g.:
     ```
     **🔍 Checking what's connected…**
     ✅ **Connected:** Gmail, Slack
     🔌 **One click away:** Google Calendar, Todoist
     🚫 **No native connector yet:** {name} — I'll skip that for this brief
     📰 **Bonus:** News will be added as a separate tile
     ```
     Bold both the intro line and every label, one status per line so it
     scans as a list of badges, not a paragraph. The ✅ **Connected:** line
     lists everything already connected. The 🔌 **One click away:** line
     names **every** named connector whose probe failed but that maps to a
     real capability (checked against the bundle table in point 3) — all of
     them get an option in the form below. A 🚫
     **No native connector yet:** line is reserved only for a name that,
     after checking the bundle table, genuinely has no matching capability
     at all — that one is **not** offered a form option, since there's
     nothing to connect. The 📰 **Bonus:** line appears only if Today News
     (Stage 2) will run this pass. Omit any line with nothing to say.
     Translate into the user's language, keeping the same bold-label
     structure.
   - **Then emit exactly ONE form**, never two in the same turn (Form
     protocol rule 5 forbids continuing after a form's closing sentinel, so
     a second standalone `genui` block right after the first would either
     break the protocol or never render). Build it with up to 2 questions
     in the single call, one per applicable case below — **include only the
     questions that apply**, never a question with just its one fixed
     "nothing to do" option:
     - **If the 🔌 line is non-empty** — question 1: `"Want to connect
       anything before I build your brief?"`, one `Connect {name}` option
       per connector on the 🔌 line, plus the fixed final option `"Skip
       all — show my brief now"`.
     - **If point 4 found extra candidates** — question 2: `"I also see
       {names} connected. What should I add to the brief?"`, one `Add
       {name}` option per extra candidate, plus the fixed final option
       `"Add nothing"`.
     - **If neither applies** (every failure was a 🚫 with no matching
       capability, and there are no extra candidates) — there is nothing
       left to offer: skip the form entirely, the status block above is
       the whole message, and continue straight to Stage 2.

```
genui{"ask_user_input":{"questions":[
  {"question":"Want to connect anything before I build your brief?","options":["Connect Gmail","Connect Calendar","Skip all — show my brief now"],"type":"multi_select","free_text_placeholder":"Name another connector"},
  {"question":"I also see Notion connected. What should I add to the brief?","options":["Add Notion","Add nothing"],"type":"multi_select","free_text_placeholder":"Something else"}
]}}
```

   **Cap each question at 9 dynamic options plus its fixed one = 10** (Form
   protocol rule 9) — if either exceeds 9, keep the highest-priority ones in
   this form and follow up with a further form for that question's overflow
   alone, per rule 10.

**This form is an exception to the Form protocol's empty-answer rule.**
An empty or all-unselected answer means the same as picking every included
question's fixed "skip"/"nothing" option — proceed straight to Stage 3 with
whatever probed successfully, or with no extras added. **Never re-emit it to
demand a selection.**

For every `Connect {name}` picked, run the native connect flow for that
connector (say its name explicitly first, then use whatever connect
capability the current surface exposes; confirm once finished) before
continuing. Every `Add {name}` picked needs no connect flow at all — it's
already usable, just fold it straight into the resolved set. If a connect
attempt fails or stalls, drop that connector and continue — never block the
run over it. Any name on the 🚫 **No native connector yet** line is still
tracked in `skipped:` below, so a future recurring run can quietly re-probe
it (see Run modes) the moment a matching capability ships, without ever
asking again.

**xTiles itself is required, not optional** — if it is not connected, this
skill is not reachable at all; connect it first, outside this flow.

**Resolved set** = every connector whose probe succeeded, plus any just
connected or added, minus anything skipped. **Track the skipped list too** —
carried forward as `skipped:` into the schedule config in Stage 6, so a
future recurring run can quietly notice if one of them gets connected later
(see Run modes) without ever asking again. Every later stage reads this set.

**When to ask — and when never to ask again automatically.** These forms
fire **at most once per run**, right after the intro sentence, and only on
a first run. It is not a recurring nag:
- On a **recurring run**, this stage never appears at all — Run modes handles
  it with a silent re-probe of `skipped:` instead.
- The user can always trigger a fresh check by asking directly at any time
  ("connect my Slack now") — that re-enters this stage for just the named
  connector, regardless of run mode.

**No content-preference questions, ever.** Every connector in the resolved
set contributes its own default content (Stage 3) — the user can still ask
to change anything after the write.

**Two worked examples of the probe — not an exhaustive list, the pattern is
the same for anything else the user names:**
- **Gmail** — call `list_labels` (cheap, no query needed). Success = connected.
- **Calendar** — call `list_events` with `maxResults:1`. Success = connected.

---

## Stage 2 — Today News: a standalone tile, by request or fallback

**This is a mandatory checkpoint, not an optional detour — every path from
Stage 1 or Run modes passes through here, on a first run and on a recurring
run alike. It is never valid to route straight from Stage 1 or Run modes to
Stage 3.**

This stage produces a `### 📰 Today News` tile in **two independent
situations** — check both, every run:

- **Fallback.** The resolved set is empty — whether because `used_connectors`
  was empty from the very start, every named connector's probe failed, or
  the user skipped everything in Stage 1. A run must never ship nothing.
- **Explicit request.** The incoming config's `additional:` field includes
  `News` — an optional field, often absent. When requested, build this tile
  **even if other connectors are also resolved** — unlike the fallback
  case, it does not step aside the moment real per-connector tiles exist.

If **neither** applies, skip this stage entirely and continue to Stage 3.

When either applies:

1. **If the incoming config already carries `news_categories:`** (a
   recurring run that already built this tile before) — use those exact
   categories, don't re-derive them; they were chosen deliberately for this
   person and should stay stable run to run. **Otherwise** (first time this
   tile is built), infer 2–4 topic categories from `role:` that this person
   would plausibly care about right now (the same judgment
   `today-news-with-gpt` uses — e.g. a Product Manager cares about
   product/UX trends, competitor moves, and AI tooling news). If the
   role doesn't narrow it down, default to broadly useful categories:
   industry news, productivity/tools, and a general "worth knowing"
   pick. **Whichever way they were obtained, this exact category list is
   what carries forward into `news_categories:` in Stage 6** if the user
   schedules a recurring run.
2. **Sourcing — mail first, then web.** If Gmail is in the resolved set,
   first check there: search recent (last 24–48h) mail that itself carries
   real news — subscribed news digests/newsletters, alert-style mail,
   publications — relevant to the chosen categories. If genuine, current
   items turn up this way, use them. **Only if Gmail isn't in the resolved
   set, or that search turns up nothing usable**, fall back to whatever
   web-search and page-fetch capability this surface exposes: find real,
   current items from the last 24–48 hours per category, verified against
   reputable sources. **Never invent an item, a date, or a link.** If a
   category genuinely yields nothing either way, drop that category rather
   than force it; if literally every category comes back empty, say so
   plainly rather than writing an empty tile.
3. Build **one** tile, `### 📰 Today News`, with one labeled sub-section per
   category and 2–3 real items each — one line per item, source linked
   inline.
4. **Persistence.** The category list carries forward into `news_categories:`
   in Stage 6, regardless of which trigger produced it. If this run's tile
   came from an explicit `additional: news` request, also carry
   `additional: news` forward — that's what tells a future recurring run to
   keep building it even once other connectors are resolved. **The fallback
   case has no such persistence flag** — the moment even one connector is
   usable in a later run, the fallback tile stops appearing on its own.

---

## Stage 3 — Silent fetch

No messages while fetching. Record a connector **error** separately from an
empty **result** — they render differently. Pull fresh data from every
connector in the resolved set, and — if Stage 2 triggered — research (or
read mail for) the Today News tile too.

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

**A distinct tool is always a distinct tile, never merged by category.**
Two calendars (e.g. a work Google Calendar and a personal one, or Calendar
alongside Reclaim) each get their own `### 📅` tile — never combined into
one "Calendar" or "Scheduling" section just because they're the same kind
of tool. Gmail and Reclaim never share a tile either, even though both are
"inbox-adjacent." The only place multiple sources ever land in one tile is
when they're genuinely the *same* connector's own data split by volume
(Email's three buckets, Slack's two) — never across two different
connectors.

**Never surface a third-party connector's own rendered preview or embed in
chat — absolute, no exceptions.** Read what Todoist, Calendly, Reclaim, or
any other connector returns purely as data to build your own tile from.
**Todoist and Calendly in particular are known to auto-render their own
rich preview/card the instant their link or reference appears — never let
that happen.** Never let a raw response render as its own card, never paste
a bare URL that triggers a link-unfurl preview, never call a capability
whose only purpose is to produce an embed, and never forward or echo a
connector's own UI element into the conversation — no matter how convenient
it seems in the moment.

- **Email arrives as a firehose that needs triage — the natural question is
  "do I have to act on this."** That's why it splits by urgency: 🔴 needs a
  concrete next step, 🟡 informational only, ⚪ automated noise. **With
  real volume in each bucket**, this becomes three tiles (Stage 4):
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
  `### 📅 Workload` tile per calendar connector — never a second copy of
  the schedule the user can already see, and never split by volume the way
  Email or Slack might be. Compute event count, hours occupied, and the
  longest free focus window; write one concrete 🎯 focus-recommendation
  sentence; for each event, a one-sentence agenda (from the event
  description or the most recent related email/meeting note — never
  invented) and, only where genuinely implied, one prep `<task>`. Collect
  anomalies at the bottom, never inline.

**Apply the same kind of thinking to anything else in the resolved set** —
Linear, Google Drive, Reclaim, or a connector this file has never heard of.
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
in the write (Stage 4) — never silently written as "no data."

---

## Stage 4 — Write + layout

**Write immediately once Stage 3's fetch completes — first run or recurring
alike. There is no preview and no approval form anywhere in this skill.**
**This skill always runs manually, once per morning — a recurring run is
this same skill invoked again, not a different one** (see Stage 6).

1. `xtiles_get_user_timezone` → today's local date as `yyyy-MM-dd`.
2. **Before finalizing the task list — dedup against already-open tasks.**
   Call `xtiles_list_tasks` for tasks that are **not completed**, due
   yesterday or today, and drop any action item about to be written whose
   text duplicates one that's already open. Never recreate the same
   unfinished task twice across two runs.
3. **Resolve the assignee once per run.** Call `xtiles_get_current_user` and
   reuse that email for the `assignee` attribute on **every** `<task>`
   written this run — first or recurring, no exceptions.
4. **This skill has no way to update an existing tile's content, so every
   run creates a fresh set of tiles.** Write all the approved sections in
   the create call below regardless of what's already on the page — a
   re-run is expected to produce a fresh, current brief.
5. **One** call to `xtiles_create_tiles_from_markdown_in_my_planner` with
   `period: "day"`, today's `date`, and all sections in a single markdown
   string. Inspect the schema first: it must accept `date`, `period`,
   `markdown` without `projectId`/`viewId`. If it demands either, report
   that the personal Daily write is unavailable and stop.
6. **Layout pass — mandatory, silent, never asked about.** Take `view_id`
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

<task dueDate="2026-08-11" assignee="user@example.com">Restore the Google ad account</task>

<task priority="high" dueDate="2026-08-10" assignee="user@example.com">Sign the Acme contract</task>
```

One `<task>` per line, blank line between, never nested in a list item.
`dueDate` always set, defaulting to today; a later real deadline from the
source overrides it. `priority` only when the source signals it — at most a
third of a morning's tasks should be `high`. `assignee` — always the current
xTiles user's own email, resolved via `xtiles_get_current_user`, on every
task, every run. Never `completed="true"`. Never a task that duplicates an
already-open one — checked against `xtiles_list_tasks` before this call.

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

<task dueDate="2026-08-11" assignee="user@example.com">Restore the Google ad account</task>
```

```
### 📅 Workload
@colorSize: LIGHTER
@color: BERMUDA

**N events · ~X h occupied · longest focus window HH:MM–HH:MM (X h)**

🎯 [Focus recommendation — one concrete sentence]

**HH:MM–HH:MM · Meeting name** — Participant · [Google Meet](url)

📋 [Agenda — one sentence, from a real source]

<task dueDate="2026-08-11" assignee="user@example.com">[Prep task, only if genuinely implied]</task>
```

`### 📩 Email — Key People` (🟡, grouped by sender) and `### 📩 Email —
Noise` (⚪, one rollup line) follow the same pattern as Action Points,
scoped to their own bucket. `### 📧 Newsletters` (if any found) is one line
per publication: `**[Publication](url)** — one-line summary.` `### 📰 Today
News` (Stage 2, whether by fallback or explicit request) is one labeled
sub-section per category with 2–3 linked items each.

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

<task dueDate="2026-08-11" assignee="user@example.com">Restore the Google ad account</task>

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

## Stage 5 — CTA and Gmail follow-through

1. **CTA** — one line, confirmation and link together, never split into two
   messages: link to the **first created tile**, not the page, using the
   `resource_url` of the **first** entry in the write response's `tiles`
   array **byte-for-byte**. Never rebuild it from `view_id`. Only if the
   first tile has no `resource_url` fall back to `parent_resource_url`; if
   neither exists, say so and stop before the schedule offer.
   `✅ Your Daily Brief is ready — built just now from real data. [Open your Daily Brief in xTiles]({first tile resource_url})`
   (matches the button label used by the Claude/Cowork variant's CTA
   widget — same product, same wording, just a markdown link here instead
   of a native button.) Translate into the user's language.
2. **If Gmail is in the resolved set — mandatory, silent, every run:** mark
   every ⚪ Noise and newsletter thread as read (remove the unread label).
   Never touch 🔴 or 🟡 threads, and never draft or send anything on the
   user's behalf.

---

## Stage 6 — Schedule

**A schedule set up here re-invokes this same skill,
`brief-onboarding-with-gpt`, every morning — there is no separate
daily-digest skill.** The whole point of resolving everything once is that
the recurring run can skip straight past Stages 1–2.

**First, check whether the scheduling capability is even available in this
environment — before sending any Stage 6 message or form.** If it genuinely
isn't, say so in one plain line ("I can't set up automatic scheduling here
— but your Daily Brief is ready, and I'll build a fresh one anytime you
ask.") and **skip straight to Stage 7, exactly as after `No schedule`
below** — never show the schedule form, the cadence form, or the
"Add anything" question only to discover at the end that none of it could
be used. A missing scheduling capability skips the schedule itself, never
the mandatory closing stage.

Otherwise, **run the connected-but-unused discovery pass now** (same
technique as Stage 1 point 4: check this session's own available connector
capabilities for anything that responds successfully and isn't already in
the resolved set) — **excluding any connector Stage 1 already offered this
run** (whether the user added it or explicitly left it out via "Add
nothing") — never re-surface something the user already decided on minutes
ago. **If the pass finds nothing**, propose 1–2 connectors that fit the
user's role/interests from `role:` instead (e.g. a designer → Figma, a
support lead → Intercom, someone tracking metrics → Amplitude), offered as
`Connect {name}` — genuine suggestions, not confirmed-connected.

Then send one line adapting this to the actual resolved connectors,
translated into the user's language:

> Every morning I can check {resolved connectors, e.g. Gmail and Calendar}
> myself and have your Daily Brief waiting in xTiles — no need to ask each
> time.

Immediately after, emit **one form with both questions together** — never
two separate forms for this decision (Form protocol rule 5 forbids
continuing after a form's closing sentinel, and splitting "should I
schedule" from "what should the schedule include" only adds a redundant
round-trip right at the moment of commitment):

```
genui{"ask_user_input":{"questions":[
  {"question":"Run this automatically every morning?","options":["Schedule it","No schedule"],"type":"single_select","free_text_placeholder":"Another cadence"},
  {"question":"Add anything to your Daily?","options":["Add {name}","No, that's enough"],"type":"multi_select","free_text_placeholder":"Another connector"}
]}}
```

Build question 2's options from the discovery pass (or the role-based
fallback) above, plus the fixed final option `"No, that's enough"`.

- **If question 1 = "Schedule it"** — fold every `Add {name}`/`Connect
  {name}` pick from question 2 into the resolved set first (running the
  native connect flow for any `Connect {name}` suggestion; drop it and
  continue if the attempt fails or stalls), **then** emit one follow-up form
  for cadence, time, and notification:

```
genui{"ask_user_input":{"questions":[
  {"question":"Which days?","options":["Weekdays","Every day"],"type":"single_select","free_text_placeholder":"Specific days"},
  {"question":"What time?","options":["08:00","09:00","10:00"],"type":"single_select","free_text_placeholder":"Another local time"},
  {"question":"Notify me in xTiles each time it runs?","options":["Yes, notify me","No notification"],"type":"single_select","free_text_placeholder":"Something else"}
]}}
```

  **Fold every addition from question 2 into the resolved set before the
  config below is assembled** — the recurring config gets created once,
  already final, never scheduled first and patched afterward.

- **If question 1 = "No schedule"** — discard question 2's answer entirely
  (there is no recurring config to fold it into, and this run's brief is
  already written) and **continue to Stage 7 in the same turn.**

**The combined form above is an exception to the Form protocol's
empty-answer rule.** An empty or all-unselected answer for question 2 means
the same as picking `"No, that's enough"` — never re-emit the form to
demand a selection on that question alone; question 1 still always needs a
real answer to know which path to take.

Resolve the timezone with `xtiles_get_user_timezone`, then create the
automation with an exact schedule whose prompt calls **this same skill**
(`brief-onboarding-with-gpt`) and embeds the **full recurring-run config**,
assembled from this run's resolved state:

```
Run brief-onboarding-with-gpt — role: {role} · tools: {resolved connector set} · skipped: {connectors whose probe failed or that the user skipped, or "none"} · news_categories: {the exact categories Stage 2 used this run — only include this field at all if Stage 2 ran from an explicit `additional: news` request} · additional: {"news" if this run's Today News tile came from an explicit request, otherwise omit} · notify: {true/false} (if true, call xtiles_create_notification at the very end of that run — mandatory, do not skip) · schedule: daily-{HH:MM} days:{days}
```

**`news_categories:` and `additional:` are conditional, and only ever
travel together.** Include both only when this run's Today News tile came
from an **explicit** `additional: news` request — that's the only case a
future recurring run needs to keep rebuilding the tile with stable
categories once other connectors are resolved. **Never persist
`news_categories:` for a pure fallback tile** — the fallback tile stops
appearing the moment any connector is usable, so a category list from that
case has nothing left to describe and would just sit in the config unused.
When there's no Today News tile this run, omit both entirely.

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
the same tile-focused deep link resolved in Stage 5, `text` is the fixed
string "Your Daily Brief is ready — see what matters today in 2 min."
translated into the user's language (no dynamic part), `agent_source` is
"ChatGPT".

Confirm with the fixed template "Done — your Daily Brief will be ready in
xTiles every morning at [time]." (append ", and I'll notify you in xTiles
each time" when `notify:true`), translated into the user's language.

---

## Stage 7 — Related workflows

**Branches on Stage 6's outcome — never the same treatment regardless of the
answer:**

- **If Stage 6 ended in a schedule actually being created** — the user is in
  a "yes" mood; offer the other three workflows, each with a one-line
  description of what it does — never a bare list of names. Never offer
  `brief-onboarding-with-gpt` itself here — its own digest and schedule are
  already handled in Stage 6. **Omit the Today News option entirely if
  Stage 2 already built a Today News tile this run** — offering it again
  reads as duplicating what the user just got.

```
genui{"ask_user_input":{"questions":[
  {"question":"Want to set up anything else on xTiles?","options":["🌙 Evening Reflection — an end-of-day synthesis and a seed for tomorrow","📰 Today News — a daily topic-based news digest from the live web","📊 Weekly Review — what actually moved forward this week","Nothing else"],"type":"single_select","free_text_placeholder":"Something else"}
]}}
```

  Drop the Today News option from the options array above when it doesn't
  apply — never send it as a disabled or dead choice.

- **If Stage 6 ended in "No schedule", or the scheduling capability was
  unavailable** — the user just declined one piece of automation; don't
  immediately ask them to consider another. No form here. Send one short,
  low-pressure line instead and stop: "You can also set up Evening
  Reflection, Today News, or Weekly Review anytime — just ask."

**When the form above was shown**, treat the selection as a direct
invocation: **in the same turn**, call `xtiles_get_workflow` with the
matching id and continue from its first applicable stage:

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
after the layout pass and Gmail follow-through (Stage 5's item 2, if
applicable) — and, only if the config's `notify:true`, after the
notification in Stage 6.
