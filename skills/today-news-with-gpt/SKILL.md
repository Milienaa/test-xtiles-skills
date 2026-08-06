---
name: today-news-with-gpt
description: >
  Build, run, or automate a live topic-based news digest in ChatGPT Work and
  save it to the user's personal xTiles Daily planner — one `Today's News` tile
  and, only when asked, one separate `Rumors & Leaks` tile. Every user decision
  is collected through inline `ask_user_input` interactive forms; every story
  comes from a live search with a working source link; nothing is written
  without approval on a manual run.

  Triggers: "today's news", "morning news", "latest on <topic>", "news digest",
  "monitor this company", "set up Today News", "run today-news-with-gpt",
  "Set workflow of Today News (today-news-with-gpt) on xTiles MCP".

  Only the Daily period is supported.
---

# xTiles Today News — GPT

Build a fresh, source-linked digest from the **live web** and add it to today's
personal Daily page. One `### 📰 Today's News` tile, plus one
`### 🕵️ Rumors & Leaks` tile only when the user explicitly enabled rumors.

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
option, summary and label. **Keep the two H3 tile titles stable in English.**

---

**Tool names** in this file are capability names, not host-specific ones
(`xtiles_…`, `slack_…`, `search_threads`, `list_events`). Call whatever the current
surface exposes for that capability. If a required xTiles capability is genuinely
missing, say so plainly and stop — never substitute a different write path.

---

## Form protocol

Emit the directive **directly in your assistant message**:

```
genui{"ask_user_input":{"questions":[ … ]}}
```

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
genui{"ask_user_input":{"questions":[
  {"question":"What do you want news about?","options":["<four to six inferred topics>"],"type":"multi_select","free_text_placeholder":"Add another topic"},
  {"question":"How should sources be verified?","options":["Cross-check every story","Standard source check"],"type":"single_select","free_text_placeholder":"Add another verification preference"},
  {"question":"Include rumors and leaks in a separate tile?","options":["No, verified news only","Yes, include clearly labelled rumors"],"type":"single_select","free_text_placeholder":"Add another preference"}
]}}
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

Read yesterday's Daily and collect the URLs already in `Today's News` and
`Rumors & Leaks`. Drop an identical URL from today's result, and drop the same
event under a different URL unless there is a substantive new development — when
it is an update, **state what changed**.

Record a web failure separately from an empty result. **Never turn a search
error into "No news."**

---

## Stage 4 — Build the digest

All verified reporting goes into **one** tile:

```markdown
### 📰 Today's News
@colorSize: LIGHTER
@color: CUMULUS

**[Topic]**

**[Headline — Source](URL)** ✅

[Two- or three-sentence factual summary]
```

- Group stories by topic; two or three high-signal stories per topic.
- Omit the verification badges under `Standard source check` — normal source
  quality still applies.
- A topic with no qualifying story after a **successful** search:
  `No meaningful news found in the last 24 hours.`
- A topic whose search **failed**:
  `Could not fetch current news for this topic — web search error.`

When rumors are enabled, add **exactly one** more tile:

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
WHITE_LINEN, POLAR` — never a semantic name. Labelled Markdown links only,
never a bare URL. Blank line between every item. **No date-only or title-only
tile.**

---

## Stage 5 — Preview and approval

Show the full digest with real content and real links. Directly above the
approval form state:

- the exact search window (and that it was widened to 48 h, if it was);
- the verification mode and whether rumors are included;
- the connected xTiles account name and email;
- the destination — today's personal Daily planner.

```
genui{"ask_user_input":{"questions":[
  {"question":"Save this digest to today's Daily?","options":["Save to Daily","Change something","Cancel"],"type":"single_select","free_text_placeholder":"Say what to change"}
]}}
```

`Change something` → collect the correction in a form, revise **only** that
part, re-show the full preview, ask again. `Cancel` → stop without writing.

**Existing digest.** Before writing, read today's Daily and compare the intended
H3 titles. If they are all already there:

```
genui{"ask_user_input":{"questions":[
  {"question":"Today already has a news digest. What should I do?","options":["Replace today's digest","Append another digest","Cancel"],"type":"single_select","free_text_placeholder":"Something else"}
]}}
```

Replace only through a safe exact content-update capability; otherwise offer
Append or Cancel. On a scheduled run, do nothing when all intended tiles already
exist.

---

## Stage 6 — Write, layout, verify

1. Call `xtiles_create_tiles_from_markdown_in_my_planner` with `period: "day"`,
   `date`: today's local ISO date, `markdown`: News plus optional Rumors in
   **one** payload. Inspect the schema first — it must accept `date`, `period`,
   `markdown` without a project or view ID. If it demands one, do not call it
   and do not fall back to project creation.
2. **Layout pass — immediate, silent, never asked about.** Take `view_id` and
   the ordered `tile_ids` from the write response (never re-derive them), call
   `xtiles_get_workflow` with id `tile-layout` and follow it, passing those
   tiles as the added tiles and the markdown just written as their content.
   Hints: a heavy `Today's News` gets its own full-width row; if both tiles are
   light, one equal-width row of two; otherwise `Rumors & Leaks` also takes a
   full-width row. Preserve content order.
3. **Verify** — re-read today's Daily and confirm every intended title. Retry
   the write **once** only if the write reported success but none of the titles
   appear.

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
genui{"ask_user_input":{"questions":[
  {"question":"Run this digest automatically?","options":["Schedule it","No schedule"],"type":"single_select","free_text_placeholder":"Another cadence"}
]}}
```

On `Schedule it`:

```
genui{"ask_user_input":{"questions":[
  {"question":"Which days?","options":["Weekdays","Every day"],"type":"single_select","free_text_placeholder":"Specific days"},
  {"question":"What time?","options":["07:00","08:00","09:00"],"type":"single_select","free_text_placeholder":"Another local time"}
]}}
```

Resolve the next occurrence in the user's timezone and create the automation
with `timing_mode: exact_schedule`, a VEVENT `DTSTART` and the matching RRULE.
Its prompt must invoke this workflow and embed a complete `today-news config:`
JSON block — topics, language, verification mode, rumors flag, personal-Daily
target, the silent-run instruction, the 24-hour window, cross-day deduplication,
and the layout requirement. **Never create a duplicate automation.**

---

## Stage 8 — Related

```
genui{"ask_user_input":{"questions":[
  {"question":"Want to set up anything else on xTiles?","options":["Daily Brief","Evening Reflection","Weekly Review","Nothing else"],"type":"single_select","free_text_placeholder":"Something else"}
]}}
```

Treat the selection as a direct invocation: **in the same turn**, call
`xtiles_get_workflow` with the matching id and continue from its first
applicable stage — `daily-brief-with-gpt`, `evening-reflection-with-gpt`,
`weekly-review-with-gpt`. Never print a handoff command, a `workflow_id`, or
`Use $...` as user-facing text, and never make the user repeat the choice.
`Nothing else` → acknowledge and stop.

---

## Closing rule

After a successful write, **every** terminal response repeats the same labelled
link as its final line — after `No schedule`, after `Nothing else`, after any
later correction. A successful manual run never ends without it. Scheduled runs
end silently after search, write, layout and verification.
