---
name: weekly-review-with-gpt
description: >
  ChatGPT Work version of the xTiles Weekly Review — use this variant in
  ChatGPT; in Claude use `weekly-review` instead.

  Review the current week — Monday through now — from the personal Daily and
  Weekly planner pages plus selected work sources, assess progress against any
  Goals or Milestones on the Weekly page, and write three tiles (Week recap,
  Activities, Next week) to the personal Weekly planner. Can also share an
  approved summary to a Slack channel. Use when the user wants to see what
  moved forward this week or set up a recurring Friday review.

  Detect the surface, not the request: inline `ask_user_input` / `genui` forms
  mean ChatGPT Work — this variant. `show_widget` or `AskUserQuestion` mean
  Claude / Cowork — use `weekly-review` instead.

  Environment triggers: "Weekly Review for GPT", "Weekly Review in ChatGPT",
  "the GPT version".

  Triggers: "weekly review", "what did I do this week", "weekly recap",
  "review my goals", "run it every Friday", "run weekly-review-with-gpt".

  Writes to the Weekly period. For a single day use `daily-brief-with-gpt` or
  `evening-reflection-with-gpt`.
---

# xTiles Weekly Review — GPT

Turn Monday-through-today activity into **three** tiles on the user's personal
xTiles **Weekly** page: `### ✅ Week recap`, `### 🔍 Activities`,
`### → Next week`.

## Five principles

1. **Ask in forms, always.** Setup, Slack consent, channels, approval,
   schedule, sharing, send confirmation, related choice — every decision goes
   through an inline `ask_user_input` form. Never prose, never a numbered list.
2. **Progress over activity.** Prefer what actually moved forward to raw volume.
   Every item is one line, deduplicated across planner pages and connectors,
   keeping the strongest source link.
3. **Real data only.** An empty successful read and a connector error are never
   conflated. Never invent an outcome, a person, a decision, or a link.
4. **Personal Weekly planner only.** The single allowed write is
   `xtiles_create_tiles_from_markdown_in_my_planner` with `period: "week"`.
   Never a project, a view, or a standalone page.
5. **Write → layout → verify → link.** All three tiles in one call, then the
   layout pass, then re-read the Weekly page to confirm, then the link.

Match the language of the user's latest message — translate every question,
option and label. **Keep the three H3 tile titles stable in English.**

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

- Emit it yourself. Do not check or discuss chat modes, do not call
  `functions.request_user_input`, and do not mention form availability, plans,
  or implementation limitations.
- Never substitute another surface: no `show_widget`, `sendPrompt`,
  `visualize`, HTML fragments, widgets, scripts, `AskUserQuestion`, or Claude
  scheduling tools.
- Never replace a form with prose, a numbered list, or "reply with 1/2/3".
- Every question carries `question`, `options`, `type`
  (`single_select` | `multi_select`) and a short specific
  `free_text_placeholder`. **Max 3 questions per form. Nothing pre-selected.**
- **One short sentence before the form, then the form, then end the turn.**
  No text after the directive.
- The next user message is the answer — including the rendered `> Question` /
  answer format. Parse it and continue. Never re-ask a validly answered stage.
- Empty answer (`Не вибрано`, `Не выбрано`, `Not selected`, blank) → one
  sentence saying what is required, then re-emit **the same** form. Never infer
  a remembered preference in place of an answer.
- Free text is kept **verbatim**; the accumulated config carries across turns.

The chain: **Setup → Slack consent + channels (if needed) → fetch → preview →
Approval → write + layout + verify → CTA → Schedule → Slack sharing →
Related**.

---

## Run modes

- **Scheduled run** — the message carries a `weekly-review config:` block or
  states it is automated. **Silent by contract**: no forms, no preview, no
  approval, no CTA, no schedule, no sharing, no related offer. Parse the saved
  role, sources, Slack scope, channels, language and target; fetch,
  deduplicate, write only missing tiles, lay out, verify, stop. Report only a
  failure.
- **Specific manual run** — the user names sources. Use only those, ask the
  **smallest** missing form (Slack consent included when applicable). Preview
  and approval still mandatory.
- **General setup or run** — everything else. Start at Stage 1.

---

## Stage 1 — Setup

```
genui{"ask_user_input":{"questions":[
  {"question":"What is your role?","options":["Product Manager","Designer","Engineer","Growth & Marketing","Founder / CEO","Support & Success"],"type":"single_select","free_text_placeholder":"Add another role"},
  {"question":"Which sources should feed your Weekly Review?","options":["Slack","Gmail","Calendar","Calendar (xTiles)","Granola","Linear","GitHub","Google Drive"],"type":"multi_select","free_text_placeholder":"Add another source"}
]}}
```

Require one role and at least one source. Selecting a source is permission to
read it for this review. **xTiles planner content is always the base source** —
it is not offered as an option.

**`Calendar (xTiles)` and `Calendar` are distinct.** The former aggregates
whatever Google/Outlook calendars the user connected inside xTiles itself, via
`xtiles_list_calendar_events`; the latter is a Google account connected
*directly*, outside xTiles. Either, both, or neither may be selected — when
both are, their events merge for the week (Stage 4).

---

## Stage 2 — Slack consent and channels

Only if Slack was selected. Consent first, as its own required question:

```
genui{"ask_user_input":{"questions":[
  {"question":"May I search your public and accessible private Slack conversations?","options":["Yes, search public and private","No, public channels only"],"type":"single_select","free_text_placeholder":"Add a condition"}
]}}
```

Then discover channels: search universal names (`general`, `all`, `team`,
`announcements`, `product`); with consent also search `from:me`, `to:me`, and
one or two role-relevant terms for the current week. Rank by recent personal
activity, then frequency, then work relevance. Strip bots, tests, random, fun
and off-topic noise. Show up to **eight** useful candidates, all unselected.

```
genui{"ask_user_input":{"questions":[
  {"question":"Which channels should I read for this week?","options":["<up to eight discovered channels, strongest first>"],"type":"multi_select","free_text_placeholder":"Add another channel"}
]}}
```

Typed channel names are kept exactly as entered. **Never search private
conversations or DMs without the affirmative consent answer.**

---

## Stage 3 — Connector checks

1. Resolve the active user and IANA timezone: `xtiles_get_current_user`,
   `xtiles_get_user_timezone`. xTiles is required.
2. Perform a harmless read-only call on every selected external source that can
   actually fail auth. Tool presence alone is not authorization.
3. If a source needs reconnecting, keep the full config, offer the native
   connect action, and resume from this check — never from Stage 1. The user may
   explicitly skip that source. **Skip this check for `Calendar (xTiles)`** — it
   has no auth-error path to test; its "maybe not linked" ambiguity is handled
   later, in Stage 4, if the merged event list comes back empty for the whole
   week.
4. **Never silently omit a selected source.**
5. Repeat these checks before creating any automation.

---

## Stage 4 — Fetch the current week

The week is **Monday 00:00 → the current local time** in the user's timezone.
**Never read future days.** Fetch silently, in parallel where possible.

- **Previous Weekly page** — count last week's accomplishments and open items
  for the week-over-week comparison.
- **This week's Daily pages** — Monday through today: completed work, open or
  overdue work, decisions, notes.
- **Current Weekly page** — tiles whose titles contain Goal, Goals, Milestone,
  Milestones, OKR, Target, or Focus. **Preserve each goal's wording.**
- **Slack** — configured channels for the current week within the approved
  scope: shipped work, decisions, important threads, open questions, people.
  Message permalinks, never channel homepages.
- **Gmail** — important mail from Monday onward; read only the threads needed to
  identify outcomes, decisions and unresolved actions. Keep thread links.
- **Calendar** — build one merged event list for the week, then analyse it:
  - `Calendar (xTiles)`, if selected — `xtiles_list_calendar_events` for Monday
    through today.
  - `Calendar`, if also selected — `list_events` for the same range, added to
    the same list.
  - Dedup: drop a `Calendar` event when an xTiles-calendar event already has
    the same start time and the same title (case-insensitive) — never show the
    same meeting twice.
  - If `Calendar (xTiles)` was selected and contributed zero events for the
    whole week, and `Calendar` contributed nothing too (or wasn't selected),
    don't assume the week was simply quiet — note the ambiguity once instead of
    silently showing an empty week.
  - From the merged list: separate meetings with others from solo blocks, and
    recurring from one-off.
- **Granola / meeting notes** — grounded decisions, actions, attendees.
- **Linear / GitHub** — work created, updated, merged/closed, and still open
  during the week.
- **Google Drive** — files created, edited, or shared during the week.

Record a failure as `Could not fetch [source] — connector error`. **Never
translate a failure into "No activity."**

---

## Stage 5 — Analyse into three tiles

Deduplicate the same event across planner pages and connectors, keeping the
strongest source link. One line per item.

### `### ✅ Week recap`

- `##### ✅ Accomplishments` — first line is the week-over-week comparison:
  `↑ More than last week (N vs M)`, `↓ Less than last week (N vs M)`, or
  `≈ Similar to last week (N vs M)`. Then numbered concrete outcomes with source
  attribution.
- `##### 🎯 Goals` — only when weekly goals were found. Preserve each goal name
  and assess it as `✅ Clear progress`, `🔄 Some progress`, `⬜ No movement`, or
  `🚫 Blocked`, each with one grounded explanation.
- `##### 💡 Decisions` — numbered decisions with sources. Omit when empty.

### `### 🔍 Activities`

- `##### 🗂️ Dominant topics` — top five semantic topics, each with the number of
  active days and a one-line summary.
- `##### ⚡ Activity type` — one percentage line summing to 100:
  `Initiative N% · Reactive N% · Decisions N%`.
- `##### 📊 Productivity pattern` — grounded observations about active and quiet
  days and time-of-day distribution. **Never infer precise times when the
  sources carry no timestamps.**
- `##### 👥 Key interactions` — up to five people by meaningful interaction
  count, the topic, and whether a decision occurred. Omit when identity cannot
  be grounded.

### `### → Next week`

- `##### 🔄 Open` — unresolved tasks, threads, promises and decisions, each with
  a source.
- `##### Suggested priorities` — **exactly three** checkbox actions when there
  is enough data. Goal blockers first, then important open work. Specific, with
  no invented assignee or date.

Omit an empty H5 subsection, but **always create all three H3 tiles**. Real
empty-state text goes inside a required tile only when the whole category has no
data.

---

## Stage 6 — Preview and approval

Show the complete review exactly as it will appear — real links, real empty
reads, real connector failures. Directly above the approval form state:

- the current local week range;
- the connected xTiles account name and email;
- the destination — the personal Weekly planner.

```
genui{"ask_user_input":{"questions":[
  {"question":"Save this review to your Weekly page?","options":["Save to Weekly","Change something","Cancel"],"type":"single_select","free_text_placeholder":"Say what to change"}
]}}
```

`Change something` → collect the change in a form, revise **only** that part,
re-show the full preview, ask again. `Cancel` → stop without writing.

**Existing review.** Before writing, read the current Weekly page and compare
the three H3 titles. If all three exist:

```
genui{"ask_user_input":{"questions":[
  {"question":"This week already has a review. What should I do?","options":["Replace the existing review","Append another review","Cancel"],"type":"single_select","free_text_placeholder":"Something else"}
]}}
```

Replace only through a content-update capability that can safely replace those
three sections; otherwise say replacement is unavailable and offer Append or
Cancel. On a scheduled run, do nothing when all three already exist.

---

## Stage 7 — Write, layout, verify

1. Call `xtiles_create_tiles_from_markdown_in_my_planner` with `period: "week"`,
   `date`: **the current week's Monday** in ISO 8601, `markdown`: all three
   approved H3 sections in **one** payload. The schema must accept `date`,
   `period`, `markdown` without a project or view ID — if it demands one, it is
   the wrong tool; do not fall back to project creation.

   Each tile begins exactly with its stable title plus colour annotations, no
   blank line between:

   ```markdown
   ### ✅ Week recap
   @colorSize: LIGHTER
   @color: SAIL
   ```

   Three **different** adjacent colours from `GHOST, CUMULUS, GOSSIP,
   COLDTURKEY, BLUE_CHALK, MILK_PUNCH, HAWKES_BLUE, PATTENS_BLUE, SAIL,
   ATHENS_GRAY, BERMUDA, PERFUME, SELAGO, RICE_FLOWER, WHITE_LINEN, POLAR` —
   never a semantic name. Labelled Markdown links only, never a bare URL. Blank
   line between items.

2. **Layout pass — immediate, silent, never asked about.** Take `view_id` and
   the ordered `tile_ids` from the write response (never re-derive them), call
   `xtiles_get_workflow` with id `tile-layout` and follow it with those tiles as
   the added tiles, the markdown just written as their content, and these hints:
   **three tiles; `Activities` is the heavy one and takes its own full-width
   row; pair `Week recap` with `Next week` in an equal-width row.**

   If `tile-layout` is unavailable, do not stop — lay out directly: treat every
   pre-existing tile as a fixed obstacle, estimate wrapping with
   `chars_per_line(w) = (w * 25 - 48) / 7.5`, derive height from headers, short
   lines, paragraphs, gaps and tile chrome, clamp to server minimums, fill the
   row width exactly, equalize heights within a row, and place new rows in the
   first free bands without moving any obstacle. Validate bounds, minimum sizes,
   row widths and overlaps before writing. If the layout write is rejected,
   retry once with the same paired-row plus full-width-heavy arrangement below
   all obstacles; if that fails, continue silently.

3. **Verify** — re-read the same Weekly page and confirm the three titles. Retry
   the write **once** only if the write reported success but none of the tiles
   appear.

---

## Stage 8 — CTA and schedule

1. Use the exact user-facing `parent_resource_url` (or equivalent) returned by
   xTiles, **byte-for-byte** — same host and path. Never build a URL from
   `view_id`, never substitute `/my-planner`, never show a raw project, view, or
   tile ID.
2. If no valid user-facing URL comes back, say the review was saved but xTiles
   returned no openable link, and stop before the optional stages.
3. Show it as a labelled link — `[Open Weekly Review](url)` — and retain it.
4. Schedule form:

```
genui{"ask_user_input":{"questions":[
  {"question":"Run this review automatically every week?","options":["Schedule it","No schedule"],"type":"single_select","free_text_placeholder":"Another cadence"}
]}}
```

On `Schedule it`:

```
genui{"ask_user_input":{"questions":[
  {"question":"Which day?","options":["Friday","Thursday","Sunday"],"type":"single_select","free_text_placeholder":"Another day"},
  {"question":"What time?","options":["16:00","17:00","18:00"],"type":"single_select","free_text_placeholder":"Another local time"}
]}}
```

Resolve the timezone and the next occurrence, then create the automation with
`timing_mode: exact_schedule`, a VEVENT `DTSTART`, and
`RRULE:FREQ=WEEKLY;BYDAY=<chosen day>`. Its prompt must invoke this workflow and
embed a complete `weekly-review config:` JSON block — role, sources, Slack scope
and channels, language, personal-Weekly destination, the silent-run instruction,
deduplication, and the layout requirement. **Never create a duplicate
automation.**

---

## Stage 9 — Slack sharing

```
genui{"ask_user_input":{"questions":[
  {"question":"Share a summary of this review to Slack?","options":["Share to Slack","Keep it personal"],"type":"single_select","free_text_placeholder":"Somewhere else"}
]}}
```

On `Share to Slack`: resolve a **real** channel through a channel form — never a
guessed channel and never a direct message to a person. Then show the actual
message (week range, top outcomes, open items, next priority — 3–5 bullets) and
require explicit confirmation:

```
genui{"ask_user_input":{"questions":[
  {"question":"Send this to the channel?","options":["Send","Change","Cancel"],"type":"single_select","free_text_placeholder":"Say what to change"}
]}}
```

Call the Slack send tool only after `Send`.

---

## Stage 10 — Related

Offer the other four workflows, each with a one-line description of what it
does — never a bare list of names:

```
genui{"ask_user_input":{"questions":[
  {"question":"Want to set up anything else on xTiles?","options":["☀️ Daily Brief — tomorrow morning's digest from Slack, Gmail and Calendar","🌙 Evening Reflection — an end-of-day synthesis and a seed for tomorrow","📰 Today News — a daily topic-based news digest from the live web","🧭 Life Brief — personal priorities and open loops beyond work tools","Nothing else"],"type":"single_select","free_text_placeholder":"Something else"}
]}}
```

Treat the selection as a direct invocation: **in the same turn**, call
`xtiles_get_workflow` with the matching id and continue from its first
applicable stage:

| Option | Workflow id |
| --- | --- |
| Daily Brief | `daily-brief-with-gpt` |
| Evening Reflection | `evening-reflection-with-gpt` |
| Today News | `today-news-with-gpt` |
| Life Brief | `life-brief-with-gpt` |

Never print a handoff command, a `workflow_id`, or `Use $...` as user-facing
text, and never make the user repeat the choice. `Nothing else` → acknowledge
and stop.

---

## Closing rule

After a successful write, **every** terminal response repeats the same labelled
link as its final line — after `No schedule`, after `Keep it personal`, after
`Nothing else`, after any later correction. A successful manual run never ends
without it. Scheduled runs end silently after write, layout and verification.
