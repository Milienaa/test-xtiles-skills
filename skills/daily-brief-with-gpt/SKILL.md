---
name: daily-brief-with-gpt
description: >
  ChatGPT Work version of the xTiles Daily Brief — use this variant in ChatGPT;
  in Claude use `daily-brief` instead.

  Set up or run a Daily planner page that works as a live morning brief from the
  user's connected work tools — Slack, Gmail, Calendar and others — written to
  their personal Daily planner. Use when the user wants a morning digest of what
  happened in the last 24 hours, wants to choose which sources feed it, or wants
  it produced automatically every morning.

  Detect the surface, not the request: inline `ask_user_input` / `genui` forms
  mean ChatGPT Work — this variant. `show_widget` or `AskUserQuestion` mean
  Claude / Cowork — use `daily-brief` instead.

  Environment triggers: "Daily Brief for GPT", "Daily Brief in ChatGPT",
  "the GPT version".

  Triggers: "morning brief", "run my digest", "what do I need to know today",
  "set up my daily", "create my Daily Brief", "run daily-brief-with-gpt".

  Only the Daily period is supported. For the end-of-day version use
  `evening-reflection-with-gpt`; for personal rather than work signals use
  `life-brief-with-gpt`.
---

# xTiles Daily Brief — GPT

One Daily planner page that turns the user's last 24 hours of work signals into
a morning brief. **Period is always Daily** — never ask which period.

## Five principles

1. **Ask in forms, always.** Every decision goes through an inline
   `ask_user_input` form — see the protocol below. Never prose, never a
   numbered list.
2. **Forms first, xTiles last.** Nothing is written until the user has seen a
   real preview and approved it.
3. **Real data only.** Fetch before preview. Never invent a person, message,
   meeting, link, or count.
4. **Personal planner only.** The single allowed write is
   `xtiles_create_tiles_from_markdown_in_my_planner` with `period: "day"`.
   Never create a project, a view, or a standalone page — even if the user says
   just "create it" or "add everything".
5. **Every write is followed by the layout pass**, then the CTA link. A run that
   ends with text in chat instead of tiles in xTiles is a failed run.

Match the language of the user's latest message — translate every question,
option, label and confirmation. Every label in this file (`Needs action`, `FYI`,
`Noise`, `Open email`, `Tasks`…) is an English placeholder, not literal output.

---

**Tool names** in this file are capability names, not host-specific ones
(`xtiles_…`, `slack_…`, `search_threads`, `list_events`). Call whatever the current
surface exposes for that capability. If a required xTiles capability is genuinely
missing, say so plainly and stop — never substitute a different write path.

---

## Form protocol

Every question is delivered through the host's `ask_user_input` surface as a
`genui` directive emitted **directly in your assistant message** — but it only
renders as an interactive form when wrapped in the host's three **invisible
sentinel characters** (Private-Use-Area code points). Without them the host
prints the raw JSON as text, which the user must never see.

The wrapper (sentinels shown here by code point — they are invisible in the file
and in output):

- **Prefix:** `U+E200`, then the literal `genui`, then `U+E202`
- **Payload:** the JSON object `{"ask_user_input":{"questions":[ … ]}}`
- **Suffix:** `U+E201`

UTF-8 bytes — U+E200 = `EE 88 80` · U+E202 = `EE 88 82` · U+E201 = `EE 88 81`.

Every `genui{…}` block below already carries these sentinels around it.
Reproduce them **exactly**, including when you build a form dynamically — never
emit a bare `genui{…}` without the sentinels, and never render the code points
or JSON as visible text.

**Interactive-form contract — follow exactly whenever user input is required:**

1. Output exactly one short introductory sentence.
2. Immediately after it, emit the form directly in the assistant message:
   U+E200 + `genui` + U+E202 + valid JSON payload + U+E201.
3. Use the literal Private-Use-Area characters, **not** the strings `U+E200`,
   `\uE200`, or their UTF-8 byte notation.
4. The directive must **not** be inside a Markdown code fence, blockquote,
   inline code, XML, or explanatory prose.
5. End the turn immediately after the closing U+E201 character.
6. Never call `functions.request_user_input`; it may be unavailable in Default
   mode.
7. Never print the JSON as visible text, and never replace the form with a
   numbered list or a plain-text question.
8. Every form must contain **1–3 questions**.
9. Every question must contain **2–10 options. Never exceed 10.**
10. If more than 10 options are needed, split them across two questions or
    consecutive forms. **Never silently drop an option.**
11. Every question must include: `question`, `options`, `type`
    (`single_select` or `multi_select`), and `free_text_placeholder`.
12. Nothing is preselected.
13. Parse the next user message as the form response and continue from the next
    workflow stage. Do not repeat an already-answered question.

Also: never substitute another surface (`show_widget`, `sendPrompt`,
`visualize`, HTML fragments, `window.openai.sendFollowUpMessage`,
`AskUserQuestion`, or Claude scheduling tools). An empty answer (`Не вибрано`,
`Не выбрано`, `Not selected`, blank) → one sentence saying what is required,
then re-emit **the same** form; never infer consent. Free text is kept
**verbatim** for any "Other …" value. Carry the accumulated config through every
stage.

The chain: **Setup → Content + Slack consent → Channels (if needed) →
Newsletters (if needed) → preview → Approval → write + layout → CTA → Schedule →
Related**.

---

## Run modes

Classify before opening any form.

- **Scheduled run** — the incoming message contains a config (`role:`, `tools:`,
  `daily_content:` …) or states it is an automated run. **No forms, no preview,
  no approval, no CTA, no schedule, no related offer.** Parse the config, fetch,
  write, run the layout pass, stop.
- **Fast-track** — the user names a source and a specific ask ("show me today's
  Slack"). Infer only what is explicit, verify that connector, fetch, preview.
  If something essential is missing, show the **smallest** relevant form, not
  the full setup.
- **Full setup** — everything else. Start at Stage 1.

---

## Stage 1 — Setup

One sentence, then:

```
genui{"ask_user_input":{"questions":[
  {"question":"What is your role?","options":["Product Manager","Designer","Engineer","Growth & Marketing","Founder / CEO","Support & Success"],"type":"single_select","free_text_placeholder":"Add another role"},
  {"question":"Which sources should feed your Daily Brief?","options":["Slack","Gmail","Calendar","Calendar (xTiles)","Granola","Linear"],"type":"multi_select","free_text_placeholder":"Add another source"},
  {"question":"Any other sources to include?","options":["GitHub","Google Drive","Figma","Gamma","LinkedIn"],"type":"multi_select","free_text_placeholder":"Add another source"}
]}}
```

Translate questions and placeholders; keep role and source names stable. The
source list is split across two questions because `ask_user_input` allows at most
10 options per question — **keep every source; never drop one to fit**.
**Require one role and at least one source across the two source questions**; the
second ("Any other sources") question may be left empty — do not re-emit the form
once a role and at least one total source have been given. Selecting a source is
permission to read it for the brief.

**Calendar (xTiles) is a distinct, optional source — no special-casing, but no
auth check either.** It aggregates whatever Google/Outlook calendars the user
connected inside xTiles itself, via `xtiles_list_calendar_events`. Unlike the
other sources, that tool has no error path for "no calendar linked" — a
successful call proves nothing, since an empty result is indistinguishable
from a genuinely empty day. Selecting it in the form is the only signal there
is; don't run a read-only check on it expecting an auth failure (see below).
The `Calendar` option is unchanged from before — it stays the optional,
directly-connected Google Calendar, and it does get the normal auth check.
Either, both, or neither may be selected; when both are selected, their events
merge into the same Workload tile (Stage 4).

**The free-text row is mandatory** — the catalog is finite, the user's stack is
not. A source typed there is treated exactly like a listed one: detect its MCP
tools, ask what it should contribute (Stage 2), carry it through fetch and
write. Silently dropping a user-named source is the most common setup failure.

---

## Stage 2 — Content + Slack consent

Show only content options belonging to the sources chosen in Stage 1. Include
the consent question **only if Slack was selected**.

```
genui{"ask_user_input":{"questions":[
  {"question":"What should appear in your Daily?","options":["<only options for selected sources>"],"type":"multi_select","free_text_placeholder":"Add something else"},
  {"question":"May I search your public and private Slack conversations for mentions?","options":["Yes, allow","No, public channels only"],"type":"single_select","free_text_placeholder":"Add a condition"}
]}}
```

| Source | Content options | Keys |
| --- | --- | --- |
| Slack | Channel updates; Mentions and DMs | `channel_updates`, `mentions_and_dms` |
| Gmail | Important unread email; Newsletters; Emails by topic | `important_unread`, `newsletters`, `by_topic` |
| Calendar | Day shape and focus time; Meetings and conflicts | `day_shape`, `meetings_and_conflicts` |
| Calendar (xTiles) | Day shape and focus time; Meetings and conflicts | `day_shape`, `meetings_and_conflicts` |
| Granola | Relevant meeting notes | `meeting_notes` |
| Linear | New or updated issues | `updated_issues` |
| GitHub | Pull requests; Review requests | `pull_requests`, `review_requests` |
| Google Drive | Recently shared or updated files | `recent_files` |
| Figma | Design updates and comments | `design_updates_and_comments` |
| Gamma | Presentation updates | `presentation_updates` |
| LinkedIn | Messages and mentions; Post engagement | `messages_and_mentions`, `post_engagement` |
| Other source | Recent updates from the named source | `recent_updates` |

- Require at least one content option. If none come back, re-emit with one-tap
  bundles derived from the selected sources — **Everything from selected
  sources** and **Only important signals** — alongside the individual options.
- Require the Slack consent answer separately whenever Slack is selected. Do not
  call `slack_search_public_and_private` without an affirmative answer.
- **Email is often the main signal.** If `Emails by topic` is chosen, ask one
  more form for the topics (`multi_select` of topics visible in the inbox + free
  text). Each topic becomes its own `### 📩 [Topic]` tile.
- For every "Other" source named in Stage 1, ask what it should contribute — one
  question per source, never assumed.

After valid answers, run a harmless read-only check on each selected connector
that can actually fail auth (e.g. `list_events` with `maxResults:1` for
Calendar). If one fails auth, say so in one line and offer the native connect
flow — keep the full config and resume from the connector check, never from
Stage 1. **Skip this check for Calendar (xTiles)** — it has no auth-error path
to test; its "maybe not linked" ambiguity is handled later, in Stage 4, if the
merged event list comes back empty. **xTiles is required**: if it is not
connected, stop and connect it first.

---

## Stage 3 — Channels and Newsletters

**Channels** — only if `channel_updates` is selected and the user has not
already named channels. Discover first, then ask:

1. Search `general`, `all`, `team`, `company`, `announcements`, `product`.
2. With consent, `slack_search_public_and_private` for `from:me` and `to:me`.
   Per channel record **recency** of the user's last post/mention (primary
   signal) and **frequency** (tiebreaker).
3. Derive 2–3 role-relevant terms and search them, plus any interests the user
   named.
4. Drop duplicates and low-signal names (`random`, `fun`, `off-topic`, `bots`,
   `test`, `hiring`, `onboarding`).
5. Rank: recent personal activity → frequency → role/interest match → bare
   presence in a universal channel. Put the strongest 5 first (max 2 general),
   then list the rest.

```
genui{"ask_user_input":{"questions":[
  {"question":"Which channels do you open first each morning?","options":["<discovered channels, strongest first>"],"type":"multi_select","free_text_placeholder":"Add another channel"}
]}}
```

**Cap the options at 10 per question** (interactive-form contract, rule 9). If
more than 10 channels remain after ranking, keep the strongest 10 in this
question and offer the rest in a second `multi_select` form — never drop a
discovered channel silently. Without consent, discover public channels only.
Free-text channel names are kept exactly as typed.

**Newsletters** — only if `newsletters` is selected. Search Gmail
`from:(@substack.com OR @beehiiv.com OR @convertkit.com OR @mailchimp.com) newer_than:30d`,
extract unique publication names, then:

```
genui{"ask_user_input":{"questions":[
  {"question":"Which newsletters do you want in your Daily?","options":["<discovered publications>"],"type":"multi_select","free_text_placeholder":"Add another publication"}
]}}
```

List at most **10 publications per question** (interactive-form contract, rule
9); if more were discovered, split them across two `multi_select` forms rather
than dropping any. If nothing is found, offer `Search again` / `Continue without
newsletters` with the same free-text row. Neither form is shown twice unless the
user asks to change it.

---

## Stage 4 — Silent fetch

No messages while fetching. Record a connector **error** separately from an
empty **result** — they render differently.

**Gmail — important unread**: `search_threads` with
`is:important in:inbox newer_than:1d`, then `get_thread` per hit for sender,
subject and `threadId` (link:
`https://mail.google.com/mail/u/0/#inbox/{threadId}`). Exclude newsletters here
entirely. Classify into three buckets:

- 🔴 **Needs action** — the user must reply, decide, act, or log in.
- 🟡 **FYI** — confirmations, payments, signed docs, status updates. One-liners,
  no links.
- ⚪ **Noise** — automated notifications. Counted, never listed individually.

Tone for 🔴/🟡: retell the email, don't copy the subject. Subject → action →
consequence, second person, people's names not addresses ("Google shut down your
ad account yesterday — log in and appeal, the window is limited").

**Gmail — newsletters**: `search_threads` on the selected senders plus the
common newsletter domains, `is:unread newer_than:1d`. One factual line each plus
its thread link.

**Slack**: `slack_read_channel` (top 50) per chosen channel, filtered to the
last 24 h; and, with consent, `slack_search_public_and_private` with `to:me`.
Group into Mentions, Action points, Topics, Decisions, Open questions. Every
item carries the **message permalink** (`permalink`, or
`https://slack.com/archives/{channel_id}/p{ts_without_dot}`) — never a channel
homepage.

**Calendar**: build one merged event list, then run the analysis below over it.

- **Calendar (xTiles), if selected in Stage 1.** Call `xtiles_list_calendar_events`
  for today — it aggregates whatever Google/Outlook calendars the user connected
  inside xTiles.
- **The Calendar connector, if also selected in Stage 1.** Call
  `list_events` for today and add its events to the same list.
- **Dedup across the two.** Drop an event from the Calendar connector when an
  xTiles-calendar event already has the same start time and the same title
  (case-insensitive) — the common case where xTiles is already synced to the
  same Google account the user also connected directly. Never show the same
  meeting twice.
- **If Calendar (xTiles) was selected and contributed zero events, don't treat
  the day as simply free.** The tool can't tell "nothing scheduled" apart from
  "no calendar linked." If the Calendar connector also contributed nothing (or
  wasn't selected), flag this once in the preview instead of silently showing
  an empty day: "No calendar events found today — this could mean nothing's
  scheduled, or that no calendar is linked inside xTiles yet." Skip the note if
  the Calendar connector did contribute at least one event — that already
  confirms the day has genuinely nothing from xTiles specifically.

Compute event count, hours occupied,
longest free focus window, overlaps, back-to-back runs, events after 20:00,
missing agendas, likely duplicates. For each event: time, title, participants,
meeting link. Find each meeting's agenda in this order — (1) a Granola/meeting
note with matching title or participants, (2) the most recent Gmail thread with
the organiser or attendees, (3) the event description. **If none yields
anything, omit the agenda line** — never paraphrase the title back.

**LinkedIn**: with its read-only tool, pull messages, mentions and notifications
from the last 24 h, plus notable engagement on the user's recent posts. Keep the
sender's name and a link to each message or notification; anything that needs a
reply becomes an action item.

**Other selected sources**: summarize only actual records their read-only tool
returns.

**Derive action items.** Every 🔴 email, every ⚡ mention, and every meeting that
genuinely needs preparation yields one verb-first action item. While deriving,
capture whether the source stated a **real deadline** and whether it carries
**genuine urgency**. A stated deadline overrides the default date and genuine
urgency sets `priority`; absent either, the task still takes the page's day as
its `dueDate` and gets no `priority`.

---

## Stage 5 — Preview

Show the real brief in chat, selected sections only, with real names, counts and
links. Render empty reads explicitly ("No unread emails", "No Slack updates
today") and failures explicitly ("Could not fetch Calendar — connector error").
Blank line between items. Newsletters stay separate from email.

Before the approval form, call `xtiles_get_current_user` and show the
destination email on the line above it — approval confirms that account.

Then **stop and wait**. Nothing is written yet.

---

## Stage 6 — Approval

```
genui{"ask_user_input":{"questions":[
  {"question":"Create this Daily in xTiles?","options":["Create it","Change something","Cancel"],"type":"single_select","free_text_placeholder":"Say what to change"}
]}}
```

`Change something` → ask what, update **only** that part, re-show the preview,
emit this form again. `Cancel` → acknowledge and stop.

---

## Stage 7 — Write + layout

Only after `Create it` (or immediately, on a scheduled run).

1. `xtiles_get_user_timezone` → today's local date as `yyyy-MM-dd`.
2. `xtiles_get_planner_content` for that date. Compare `###` headings and append
   **only** sections that do not exist yet. If all of them exist: on a manual
   run ask `Replace all` / `Append anyway` / `Cancel` as a form; on a scheduled
   run write nothing.
3. **One** call to `xtiles_create_tiles_from_markdown_in_my_planner` with
   `period: "day"`, today's `date`, and all sections in a single markdown
   string. Inspect the schema first: it must accept `date`, `period`, `markdown`
   without `projectId`/`viewId`. If it demands either, it is the wrong tool —
   report that the personal Daily write is unavailable and stop. Never fall back
   to project or view creation.
4. **Layout pass — mandatory, silent, never asked about.** Take `view_id` and
   the ordered `tile_ids` straight from the write response (do not re-derive
   them), call `xtiles_get_workflow` with id `tile-layout` and follow it,
   passing those tiles as the added tiles and the markdown just written as their
   content. Hints: default 2 tiles per row; give a heavy tile (📩 Emails,
   💬 Slack — Topics, 📅 Workload) its own full-width row.

### Tile format

Each `###` section carries its colour annotations immediately under the heading,
no blank line:

```
### 📩 Emails
@colorSize: LIGHTER
@color: SAIL
```

`@colorSize` is always `LIGHTER`. `@color` is picked from
`GHOST, CUMULUS, GOSSIP, COLDTURKEY, BLUE_CHALK, MILK_PUNCH, HAWKES_BLUE,
PATTENS_BLUE, SAIL, ATHENS_GRAY, BERMUDA, PERFUME, SELAGO, RICE_FLOWER,
WHITE_LINEN, POLAR` — never a semantic name (RED, BLUE, GREY…), and never the
same colour twice in a row.

- **No date or title-only tile.** Start with content.
- The heading emoji names the *subject* (📩 📧 📌 ⚡ 💬 📅). Status markers
  (🔴 🟡 ⚪ ⏳ ✅ ❓) belong on item lines inside a tile, never in a heading.
- **Links are inline hyperlinks inside a sentence** — `… → [Open email](url)`,
  `— [#channel](permalink)`, `· [Google Meet](url)`. A line containing only a
  link renders as a block card, which is not what the user wants. Never a bare
  URL, never a link on its own line, never a link inside a `<task>`.
- Blank line between every item.

**Action items are tasks, not checkboxes:**

```
**Action items**

<task dueDate="2026-08-11">Restore the Google ad account</task>

<task priority="high" dueDate="2026-08-10">Sign the Acme contract</task>
```

One `<task>` per line, blank line between, never nested in a list item.
`dueDate` is always set — defaulting to the page's day (today for a Daily page),
resolved against the user's timezone — and a **later real deadline** from the
source overrides it (use the `dueDate="YYYY-MM-DD HH:MM"` form when a time is
stated). `priority` only when the source itself signals it — at most a third of
a morning's tasks should be `high`. Never `completed="true"`.

### Tiles

- **`### 📩 Emails`** — three labelled blocks 🔴 **Needs action (N)** /
  🟡 **FYI (N)** / ⚪ **Noise**, then `**Action items**` with one `<task>` per 🔴
  item. Omit the action block when there are no 🔴 emails. When the user asked
  for topic splitting, emit one `### 📩 [Topic]` tile per topic with the same
  three-block structure, plus a catch-all `### 📩 Emails`.
- **`### 📧 Newsletters`** — all newsletters in one tile, one line each:
  `**[Publication](thread_url)** — one-line summary.` Omit the tile only when
  there are none.
- **Slack → exactly three tiles**, never more:
  - `### 📌 Slack — Action Points` — one poke-style line per ⚡ mention with its
    permalink, then a `**Tasks**` block. Omit if there are none.
  - `### ⚡ Slack — Mentions` — `- **@Name** in [#channel](permalink) — what they
    asked` (+ ` ⚡` if a reply is needed). Omit if there are none.
  - `### 💬 Slack — Topics` — `**Channels:** #a (N) · #b (N)`, then one line per
    topic, then a `**✅ Decisions**` block and a `**❓ Open**` block folded in
    (never their own tiles). **Always create this tile** — if there is nothing,
    write `No updates today.`; its absence reads as a connector failure.
- **`### 📅 Workload`** — never `### 📅 Calendar`. The name is the promise: an
  analysis of the day, not a reprint of the schedule.
  - Bold summary line first: `**N events · ~X h occupied · longest focus window
    HH:MM–HH:MM (X h)**`.
  - 🎯 **focus recommendation — one concrete sentence, never omitted**: which
    window to protect and for what, or which meeting decides the day.
  - Each event on its own bold line `**HH:MM–HH:MM · Title**` + participants and
    meeting link; 📋 agenda in the paragraph under it; one `<task>` prep item
    under that, only where preparation is genuinely implied.
  - Group events into 2–4 groups derived from the actual day (⭐ Important ·
    🤝 Client & external · 🔁 Recurring · 🧑‍🤝‍🧑 1:1s). **Fewer than 4 events →
    no groups**, list flat.
  - ⚠️ anomalies collected at the bottom, one per line.
  - Omit the tile if the merged event list is empty. Always one tile, even when
    both the xTiles calendar and the Calendar connector contributed events —
    never split them into two.
- **`### 💼 LinkedIn`** — messages, mentions and notifications that need
  attention, plus notable engagement on the user's recent posts. One line per
  item — `- **Name** — what they said/asked` with an inline link to the message
  or notification — then a `**Tasks**` block with one `<task>` per item that
  needs a reply. Omit the tile when the connector returned nothing.
- **Optional sources** get one tile each, only when their tool returned data.

---

## Stage 8 — CTA and schedule

1. Confirm in one line: `✅ Daily created.`
2. **CTA** — use the `parent_resource_url` (or equivalent user-facing URL) from
   the write response **byte-for-byte**. Never rebuild it from `view_id`, never
   swap the host, never substitute `/my-planner`, never present a raw
   `view_id`/`tile_id` as the result. If the write returned no user-facing URL,
   say exactly that and stop before the schedule offer.
   `✅ Daily created. [Open in xTiles →]({parent_resource_url})`
3. Schedule form:

```
genui{"ask_user_input":{"questions":[
  {"question":"Run this automatically every morning?","options":["Schedule it","No schedule"],"type":"single_select","free_text_placeholder":"Another cadence"}
]}}
```

On `Schedule it`, one follow-up form for cadence and time:

```
genui{"ask_user_input":{"questions":[
  {"question":"Which days?","options":["Weekdays","Every day"],"type":"single_select","free_text_placeholder":"Specific days"},
  {"question":"What time?","options":["08:00","09:00","10:00"],"type":"single_select","free_text_placeholder":"Another local time"}
]}}
```

Resolve the timezone with `xtiles_get_user_timezone`, then create the automation
with an exact schedule whose prompt calls this skill and embeds the **full
config** — role, sources, selected content, Slack consent, channels,
newsletters, language — plus the instruction to run silently, use the last 24
hours, append only absent sections, and run the layout pass. Do not schedule
until every selected connector has passed its read-only check. Confirm the
chosen time and timezone afterwards.

---

## Stage 9 — Related workflows

Offer the other four workflows, each with a one-line description of what it
does — never a bare list of names:

```
genui{"ask_user_input":{"questions":[
  {"question":"Want to set up anything else on xTiles?","options":["🌙 Evening Reflection — an end-of-day synthesis and a seed for tomorrow","📰 Today News — a daily topic-based news digest from the live web","📊 Weekly Review — what actually moved forward this week","🧭 Life Brief — personal priorities and open loops beyond work tools","Nothing else"],"type":"single_select","free_text_placeholder":"Something else"}
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
| Life Brief | `life-brief-with-gpt` |

Never print a handoff command, a `workflow_id`, or `Use $...` as user-facing
text, and never make the user repeat the choice. `Nothing else` → acknowledge
and stop.

---

## Closing rule

After a successful write, **every** terminal response repeats the same labelled
CTA link as its final line — after `No schedule`, after `Nothing else`, after a
later correction, after a connector clarification. A successful manual run never
ends without it. Scheduled runs end silently after the layout pass.
