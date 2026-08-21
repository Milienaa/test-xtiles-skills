---
name: today-news-with-gpt
description: >
  ChatGPT Work version of xTiles Today News — use this variant in ChatGPT; in
  Claude use `today-news` instead.

  Generate a fresh topic-based news digest from the live web and save it to
  today's personal xTiles Daily page — one tile for a small digest, a tile per
  topic once there is enough content — plus an optional separate tile for
  clearly-labelled rumors and leaks. Use when the user asks for news, wants
  updates on a subject, or wants to track a company, market or technology every
  morning.

  Detect the surface, not the request: inline `ask_user_input` / `genui` forms
  mean ChatGPT Work — this variant. `show_widget` or `AskUserQuestion` mean
  Claude / Cowork — use `today-news` instead.

  Environment triggers: "Today News for GPT", "Today News in ChatGPT",
  "the GPT version".

  Triggers: "morning news", "today's news", "what's happening in AI today",
  "latest on <topic>", "set up daily news", "run today-news-with-gpt".

  Stories always come from a live search, never from memory. For a digest of the
  user's own work signals use `daily-brief-with-gpt`.
---

# xTiles Today News — GPT

Build a fresh, source-linked digest from the **live web** and add it to today's
personal Daily page. A small digest is one `### 📰 Today's News` tile; once
there is enough content it becomes **one tile per topic** (see the split rule in
Stage 4). Rumors, when explicitly enabled, are always exactly one
`### 🕵️ Rumors & Leaks` tile — never split, never mixed into a news tile.

## Five principles

1. **Ask in forms, always.** Topics, verification, rumors, approval, schedule —
   every decision goes through an inline `ask_user_input` form. The setup form
   is the **required entry point** even for a bare "today's news": never search
   or write before it.
2. **Live sources only.** Never answer a current-news request from memory, and
   never invent a headline, date, quote, metric, name, or link. Every item
   carries a working URL.
3. **Nothing is written without approval** on a manual run — the complete real
   digest and the destination account are shown first.
4. **Personal planner only.** The single allowed write is
   `xtiles_create_tiles_from_markdown_in_my_planner` with `period: "day"`.
   Never a project, a view, or a standalone page.
5. **Write → layout → verify → link.** After the write, run the layout pass,
   re-read the Daily to confirm the tiles exist, and end with an openable link.

Match the language of the user's latest message — translate every question,
option, summary and label. **Keep the fixed H3 tile titles stable in English** —
`📰 Today's News` and `🕵️ Rumors & Leaks`. A per-topic tile title carries the
user's own topic wording **verbatim**, in whatever language they typed it.

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
  sentence saying what is required, then re-emit **the same** form.
- Typed topics are preserved **verbatim**; at least one topic is required. The
  accumulated config carries across turns.

The chain: **Setup → search → preview → Approval → write + layout + verify →
CTA → Schedule → Related**.

---

## Run modes

- **Scheduled run** — the message carries a `today-news config:` block or states
  it is automated. **Silent by contract**: no forms, no preview, no approval, no
  CTA, no schedule or related offer. Parse topics, language, verification mode
  and rumors flag; search, deduplicate, write only missing tiles, lay out,
  verify, stop. Report only a failure.
- **Specific manual run** — the user names topics. Use **those exact topics**,
  default to verified news only with standard source checks, and skip the rest
  of setup unless they also ask to personalize. Preview and approval still
  mandatory.
- **Full setup / quick manual run** — everything else, including a bare "Today
  News". Start at Stage 1.

---

## Stage 1 — Setup

Infer **four to six plausible topic labels** from visible conversation context
only, and use them as the unselected options. With no useful context, fall back
to `AI & product`, `Tech industry`, `Startups`, `Business`. Inference populates
the choices — it never decides them.

```
genui{"ask_user_input":{"questions":[
  {"question":"What do you want news about?","options":["<four to six inferred topics>"],"type":"multi_select","free_text_placeholder":"Add another topic"},
  {"question":"How should sources be verified?","options":["Cross-check every story","Standard source check"],"type":"single_select","free_text_placeholder":"Add another verification preference"},
  {"question":"Include rumors and leaks in a separate tile?","options":["No, verified news only","Yes, include clearly labelled rumors"],"type":"single_select","free_text_placeholder":"Add another preference"}
]}}
```

Translate labels and placeholders; replace the sample topics with the real
inferred ones before rendering. Require at least one topic.

---

## Stage 2 — Resolve xTiles and time

Before searching: `xtiles_get_current_user` and `xtiles_get_user_timezone` —
harmless read-only calls that give the destination account and the local date
defining *today* and *yesterday*. If xTiles is unavailable, keep the topic
configuration, offer the native connector action, and resume after connection.

Repeat a harmless xTiles read before creating any automation. Never create an
automation whose connector is not authorized.

---

## Stage 3 — Search the live web

Default window: meaningful developments published or materially updated in the
**last 24 hours**. Expand to 48 hours only when a topic is quiet, and **label
that wider window in the preview**.

Per topic:

1. Search broadly enough to identify the main developments, then open the top
   relevant sources.
2. Prefer **primary and authoritative** sources: official announcements,
   filings, regulators, company or project posts, research papers, first-party
   documentation. For breaking general news, reputable original reporting. For
   technical claims, only official documentation or primary research.
3. Extract headline, publisher, event date, publication date, URL, and a
   factual two- or three-sentence summary.
4. Drop pure opinion with no new event, press-release rewrites with no added
   evidence, SEO aggregators, duplicate URLs, and stale stories.
5. Deduplicate the same event across outlets — keep the strongest primary or
   original-reporting link; corroborating sources are for verification only.
6. On `Cross-check every story`, corroborate each material claim with a second
   independent source where one exists. Mark `✅ verified` when corroborated and
   `⚠️ could not corroborate` when it cannot be confirmed. **Aggregators
   repeating each other is not corroboration.**

**Rumors** — only when explicitly enabled. Run a separate search for credible
leaks, insider reporting, relevant Reddit posts, or first-party social posts.
At most one or two claims per topic. Every item stays explicitly unverified even
when plausible. **Never mix rumors into the News tile.**

### Cross-day deduplication

Read yesterday's Daily and collect the URLs from every news tile there — the
single `Today's News` tile or any per-topic `📰` tiles — plus
`Rumors & Leaks`. Drop an identical URL from today's result, and drop the same
event under a different URL unless there is a substantive new development — when
it is an update, **state what changed**.

Record a web failure separately from an empty result. **Never turn a search
error into "No news."**

---

## Stage 4 — Build the digest

### One tile or one per topic — decide before previewing

Count the topics that actually produced at least one qualifying story, and the
total number of stories.

- **Single tile.** Two or fewer topics with stories, **or** four or fewer
  stories in total → everything goes into one `### 📰 Today's News` tile, topics
  as bold subheaders inside it.
- **Split by topic.** Three or more topics with stories **and** five or more
  stories in total → **one tile per topic**, titled `### 📰 [Topic]` with the
  user's own topic wording. Never split a single topic across two tiles, and
  never give a topic with one story its own tile — fold a lone story into the
  nearest related topic tile, or keep the digest in single-tile mode.
- Topics with **no story** or a **failed search** never get their own tile.
  They are reported as one short line at the end of the last news tile, so the
  user still sees what was covered.
- Never split to look fuller. A quiet news day stays one tile.

**Single-tile shape:**

```markdown
### 📰 Today's News
@colorSize: LIGHTER
@color: CUMULUS

**[Topic]**

**[Headline — Source](URL)** ✅

[Two- or three-sentence factual summary]
```

**Split shape — one tile per topic, in the order the topics were configured:**

```markdown
### 📰 [Topic]
@colorSize: LIGHTER
@color: CUMULUS

**[Headline — Source](URL)** ✅

[Two- or three-sentence factual summary]

### 📰 [Next topic]
@colorSize: LIGHTER
@color: SELAGO

**[Headline — Source](URL)** ✅

[Two- or three-sentence factual summary]

⚠️ No meaningful news found for [quiet topic]; could not fetch [failed topic] — web search error.
```

- Two or three high-signal stories per topic, in both shapes.
- Omit the verification badges under `Standard source check` — normal source
  quality still applies.
- A topic with no qualifying story after a **successful** search:
  `No meaningful news found in the last 24 hours.`
- A topic whose search **failed**:
  `Could not fetch current news for this topic — web search error.`
  The two are never conflated, in either shape.

When rumors are enabled, add **exactly one** more tile — regardless of how many
news tiles there are:

```markdown
### 🕵️ Rumors & Leaks
@colorSize: LIGHTER
@color: MILK_PUNCH

⚠️ *Unverified — based on leaks, social posts, or insider reports*

**[Topic]**

**[Claim — Source](URL)**

[One- or two-sentence neutral summary]
```

Formatting rules: colour annotations sit immediately under the heading with no
blank line; `@colorSize` is always `LIGHTER` and `@color` comes from `GHOST,
CUMULUS, GOSSIP, COLDTURKEY, BLUE_CHALK, MILK_PUNCH, HAWKES_BLUE,
PATTENS_BLUE, SAIL, ATHENS_GRAY, BERMUDA, PERFUME, SELAGO, RICE_FLOWER,
WHITE_LINEN, POLAR` — never a semantic name, and never the same colour on two
adjacent tiles. Labelled Markdown links only, never a bare URL. Blank line
between every item. **No date-only or title-only tile, and never an empty
tile** — a topic tile exists only because it has stories in it.

---

## Stage 5 — Preview and approval

Show the full digest with real content and real links. Directly above the
approval form state:

- the exact search window (and that it was widened to 48 h, if it was);
- the verification mode and whether rumors are included;
- how many tiles will be written and what they are titled;
- the connected xTiles account name and email;
- the destination — today's personal Daily planner.

```
genui{"ask_user_input":{"questions":[
  {"question":"Save this digest to today's Daily?","options":["Save to Daily","Change something","Cancel"],"type":"single_select","free_text_placeholder":"Say what to change"}
]}}
```

`Change something` → collect the correction in a form, revise **only** that
part, re-show the full preview, ask again. `Cancel` → stop without writing.

**Existing digest.** Before writing, read today's Daily and look for any `📰`
news tile — `Today's News` or a per-topic one — and the `🕵️ Rumors & Leaks`
tile. Match on the tiles that are there, not on the shape you happen to be
writing today: a digest written as one tile yesterday and split today is still
the same digest. If today's news is already on the page:

```
genui{"ask_user_input":{"questions":[
  {"question":"Today already has a news digest. What should I do?","options":["Replace today's digest","Append another digest","Cancel"],"type":"single_select","free_text_placeholder":"Something else"}
]}}
```

Replace only through a safe exact content-update capability, and replace the
**whole digest** — every existing news tile plus the rumors tile — so no tile
from the previous shape is left orphaned on the page. Otherwise offer Append or
Cancel. On a scheduled run, do nothing when the digest is already there.

---

## Stage 6 — Write, layout, verify

1. Call `xtiles_create_tiles_from_markdown_in_my_planner` with `period: "day"`,
   `date`: today's local ISO date, `markdown`: every approved tile — the single
   news tile or all per-topic tiles, plus optional Rumors — in **one** payload,
   in the previewed order. One call regardless of tile count; never one call per
   tile. Inspect the schema first — it must accept `date`, `period`, `markdown`
   without a project or view ID. If it demands one, do not call it and do not
   fall back to project creation.
2. **Layout pass — immediate, silent, never asked about.** Take `view_id` and
   the ordered `tile_ids` from the write response (never re-derive them), call
   `xtiles_get_workflow` with id `tile-layout` and follow it, passing those
   tiles as the added tiles and the markdown just written as their content.
   Hints: a heavy tile gets its own full-width row; lighter per-topic tiles go
   two per row; a single `Today's News` tile is normally full-width, and if it
   and `Rumors & Leaks` are both light they can share one equal-width row.
   Preserve content order.
3. **Verify** — re-read today's Daily and confirm every intended title, all of
   them, not just the first. Retry the write **once** only if the write reported
   success but none of the titles appear.

---

## Stage 7 — CTA and schedule

1. Use the exact user-facing `parent_resource_url` (or equivalent) returned by
   xTiles, **byte-for-byte** — same environment, host and path. Never build a
   URL from `view_id`, never substitute a generic planner route, never show a
   raw ID.
2. If no valid user-facing URL comes back, say the digest was saved but xTiles
   returned no openable link, and stop before the optional stages.
3. Show it as a labelled link — `[Open Today's News in My Planner](url)` — and
   retain it.
4. Schedule form — shown on **every** successful manual run, immediately after
   the link:

```
genui{"ask_user_input":{"questions":[
  {"question":"Run this digest automatically?","options":["Schedule it","No schedule"],"type":"single_select","free_text_placeholder":"Another cadence"}
]}}
```

On `Schedule it`:

```
genui{"ask_user_input":{"questions":[
  {"question":"Which days?","options":["Weekdays","Every day"],"type":"single_select","free_text_placeholder":"Specific days"},
  {"question":"What time?","options":["07:00","08:00","09:00"],"type":"single_select","free_text_placeholder":"Another local time"}
]}}
```

Resolve the next occurrence in the user's timezone and create the automation
with `timing_mode: exact_schedule`, a VEVENT `DTSTART` and the matching RRULE.
Its prompt must invoke this workflow and embed a complete `today-news config:`
JSON block — topics, language, verification mode, rumors flag, personal-Daily
target, the silent-run instruction, the 24-hour window, cross-day deduplication,
the single-tile / per-topic split rule, and the layout requirement. **Never
create a duplicate automation.**

---

## Stage 8 — Related

Offer the other four workflows, each with a one-line description of what it
does — never a bare list of names:

```
genui{"ask_user_input":{"questions":[
  {"question":"Want to set up anything else on xTiles?","options":["☀️ Daily Brief — tomorrow morning's digest from Slack, Gmail and Calendar","🌙 Evening Reflection — an end-of-day synthesis and a seed for tomorrow","📊 Weekly Review — what actually moved forward this week","🧭 Life Brief — personal priorities and open loops beyond work tools","Nothing else"],"type":"single_select","free_text_placeholder":"Something else"}
]}}
```

Treat the selection as a direct invocation: **in the same turn**, call
`xtiles_get_workflow` with the matching id and continue from its first
applicable stage:

| Option | Workflow id |
| --- | --- |
| Daily Brief | `daily-brief-with-gpt` |
| Evening Reflection | `evening-reflection-with-gpt` |
| Weekly Review | `weekly-review-with-gpt` |
| Life Brief | `life-brief-with-gpt` |

Never print a handoff command, a `workflow_id`, or `Use $...` as user-facing
text, and never make the user repeat the choice. `Nothing else` → acknowledge
and stop.

---

## Closing rule

After a successful write, **every** terminal response repeats the same labelled
link as its final line — after `No schedule`, after `Nothing else`, after any
later correction. A successful manual run never ends without it. Scheduled runs
end silently after search, write, layout and verification.
