---
name: evening-reflection-with-gpt
description: >
  ChatGPT Work version of the xTiles Evening Reflection — use this variant in
  ChatGPT; in Claude use `evening-reflection` instead.

  Close out the day: synthesize what actually happened from today's planner,
  tasks and connected sources, optionally log completed work as xTiles tasks,
  and write one "Day Characteristic" tile to today's personal Daily planner.
  Use when the user wants to reflect on the day, record what got done, or set up
  a recurring evening review.

  Detect the surface, not the request: inline `ask_user_input` / `genui` forms
  mean ChatGPT Work — this variant. `show_widget` or `AskUserQuestion` mean
  Claude / Cowork — use `evening-reflection` instead.

  Environment triggers: "Evening Reflection for GPT", "Evening Reflection in ChatGPT",
  "the GPT version".

  Triggers: "reflect on my day", "wrap up today", "what did I get done today",
  "log what I did", "set up Evening Reflection",
  "run evening-reflection-with-gpt".

  Only the Daily period is supported. For the morning version use
  `daily-brief-with-gpt`; for a whole-week summary use
  `weekly-review-with-gpt`.
---

# xTiles Evening Reflection — GPT

Synthesize what actually happened today, reconcile completed work with xTiles
tasks when the user allows it, and add **one** `Day Characteristic` tile to
today's personal Daily planner. **Period is always Daily** — if asked for a
weekly or monthly reflection, say plainly that only Daily is supported rather
than silently downscaling.

## Five principles

1. **Ask in forms, always.** Every decision goes through an inline
   `ask_user_input` form — see the protocol below. The setup form is the
   **required entry point** even for a bare "reflect on my day": never jump
   straight to the reflection.
2. **Real data only.** Never invent completed work, promises, people, meetings,
   tasks, or links. A quiet day is reported as a quiet day.
3. **Nothing mutates without approval.** On a manual run, both the task changes
   and the reflection are previewed and approved separately. Automatic task
   changes exist only inside a scheduled run whose config explicitly approves
   them.
4. **Personal planner only.** The single allowed write is
   `xtiles_create_tiles_from_markdown_in_my_planner` with `period: "day"`.
   Never a project, a view, or a standalone page.
5. **Write → layout → verify → link.** After the write, run the layout pass,
   re-read the Daily to confirm the tile exists, and end with an openable link.

Match the language of the user's latest message — translate every question,
option and label, but keep factual names and source labels intact. Labels in
this file (`Results`, `Opportunities`, `Tomorrow`…) are English placeholders.

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
  `visualize`, HTML fragments, `AskUserQuestion`, or Claude scheduling tools.
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
- Free text is kept **verbatim**; the accumulated config carries across turns.

The chain: **Setup → Preferences → Slack consent + channels (if needed) →
fetch → task preview → Approval → write + layout + verify → CTA → Schedule →
Related**.

---

## Run modes

- **Scheduled run** — the message carries an `evening-reflection config:` block
  or states it is automated. **Silent by contract**: no forms, no previews, no
  CTA, no schedule or related offer. Use the saved config, respect its
  `autolog`, and finish after task reconciliation (when allowed), write, layout
  and verification. Report only a failure.
- **Specific manual run** — the user names sources or a tone. Infer only what is
  explicit and ask the **smallest** missing form. Slack consent is still
  required; both previews are still mandatory.
- **Full setup / quick manual run** — everything else, including a bare
  "Evening Reflection". Start at Stage 1.

---

## Stage 1 — Setup

One sentence, then:

```
genui{"ask_user_input":{"questions":[
  {"question":"What is your role?","options":["Product Manager","Designer","Engineer","Growth & Marketing","Founder / CEO","Support & Success"],"type":"single_select","free_text_placeholder":"Add another role"},
  {"question":"Which sources should feed your Evening Reflection?","options":["xTiles Planner","Slack","Gmail","Calendar","Granola","Linear"],"type":"multi_select","free_text_placeholder":"Add another source"}
]}}
```

Require one role and at least one source. Selecting a source is permission to
read it for this reflection. xTiles access is always required — for the
destination and for deduplication — regardless of whether `xTiles Planner` is
selected as an evidence source.

---

## Stage 2 — Preferences

Build the first question's options **only** from the selected sources, plus the
four universal ones.

```
genui{"ask_user_input":{"questions":[
  {"question":"What should the reflection cover?","options":["Tasks completed today","Decisions and progress","Promises and follow-ups","Tomorrow's priorities","<one option per selected source>"],"type":"multi_select","free_text_placeholder":"Add something else"},
  {"question":"What tone should it use?","options":["Honest coach","Gentle","Neutral"],"type":"single_select","free_text_placeholder":"Describe another tone"},
  {"question":"Auto-log completed activities as xTiles tasks?","options":["Yes — preview first","Yes — automatically on scheduled runs","No — reflection only"],"type":"single_select","free_text_placeholder":"Another rule"}
]}}
```

Require at least one content option. **Whatever the autolog answer is, a manual
run always previews task mutations before applying them** — the "automatically"
option only authorizes silent mutation inside a scheduled run.

---

## Stage 3 — Slack consent and channels

Only if Slack was selected. Consent first, as its own required question:

```
genui{"ask_user_input":{"questions":[
  {"question":"May I search your public and accessible private Slack conversations?","options":["Yes, search public and private","No, public channels only"],"type":"single_select","free_text_placeholder":"Add a condition"}
]}}
```

Then discover channels within the approved scope — rank by where the user
actually posted today, then by role relevance. Drop bot, alert, deploy, health,
test, random and off-topic streams by default; never hardcode company channel
names.

```
genui{"ask_user_input":{"questions":[
  {"question":"Which channels should I read for today's work?","options":["<discovered channels, most active first>"],"type":"multi_select","free_text_placeholder":"Add another channel"}
]}}
```

Typed channel names are kept exactly as entered.

---

## Stage 4 — Connector checks

1. Resolve the active user and IANA timezone: `xtiles_get_current_user`,
   `xtiles_get_user_timezone`. xTiles is required — if it is unavailable, stop
   and connect it first.
2. Perform a harmless read-only call on every selected external source. Tool
   presence alone is not authorization.
3. If a source needs reconnecting, keep the full config, offer the native
   connect action, and resume from this check — never from Stage 1. The user may
   explicitly skip that source.
4. **Never silently omit a selected source.** A source that failed is named in
   the preview.

---

## Stage 5 — Fetch today's real activity

Today is **00:00 → now** in the user's local timezone. Fetch silently, in
parallel where possible.

- **xTiles Planner** (when selected): `xtiles_list_tasks` for tasks completed
  and open today, then `xtiles_get_planner_content` for today's Daily — notes,
  decisions, and any existing reflection. **Even when not selected**, read just
  enough planner content to detect an existing reflection tile and to
  deduplicate task changes.
- **Slack**: within the approved scope, `from:me` to locate the user's own work,
  and `to:me` for promises and follow-ups when that content was selected. Read
  important threads in the chosen channels. Keep message permalinks.
- **Gmail**: mail sent today, plus important received mail that needs action.
  Read only what is necessary; keep thread links.
- **Calendar**: today's events, separating meetings with others from solo focus
  blocks.
- **Granola / meeting notes**: decisions, action items and attendees from
  today's meetings.
- **Linear / issue tracker**: issues that moved meaningfully today.

Record a connector failure as `Could not fetch [source] — connector error`; an
empty successful read as `No activity found in [source] today.` The two are
never conflated.

---

## Stage 6 — Reconcile activities with tasks

Derive candidate activities from real outcomes only. Categories emerge from the
day (meetings, content, partnerships, support, research, implementation,
strategic decisions…) — never force a fixed role template.

Compare each activity **semantically** with today's personal tasks:

- **Close existing** — an open task represents the same work → propose marking
  it complete.
- **Create + close** — no equivalent task exists → propose a short, specific
  completed task.
- **Skip** — an equivalent completed task already exists, evidence is weak, or
  the activity is trivial.

Match meaning, not wording. Never create a duplicate. Use the current xTiles
user's identity and today's local ISO date.

**Manual run** — show the real preview first:

```
☑️ closes existing: "[task title]"
✅ new completed task: [emoji] [specific title]
```

```
genui{"ask_user_input":{"questions":[
  {"question":"Log these as tasks?","options":["Apply all","Change the list","Skip task logging"],"type":"single_select","free_text_placeholder":"Say what to change"}
]}}
```

Mutate tasks only after `Apply all`. `Change the list` → collect the change in a
form, re-preview, ask again. `Skip task logging` skips only the tasks — the
reflection continues.

**Scheduled run** — `autolog: off`, ambiguous, or missing → **no task
mutation**. Only an explicitly approved automatic mode closes matching tasks and
creates completed ones without a preview.

---

## Stage 7 — Compose and approve

Derive themes from everything successfully collected. Distinguish real progress
from pure operations. Surface promises and follow-ups. Apply the chosen tone
honestly — **do not inflate a quiet day**. Keep optional sections only when they
carry real value.

Preview this exact structure, filled with real content:

```markdown
### ✨ Day Characteristic — DD.MM.YYYY
@colorSize: LIGHTER
@color: SAIL

**[One or two sharp sentences describing the day]**

---

**🎯 Results**

- [Concrete outcome with a labelled source link when available]

---

**🌟 Opportunities**

- [Only genuinely useful opportunities]

---

**→ Tomorrow**

- [ ] [Specific action with a person or object; maximum three]

⚠️ [Unavailable sources — only when a source actually failed]
```

Rules for this markdown:

- **One H3 tile only.** `Results`, `Opportunities` and `Tomorrow` are bold
  subheaders inside it, never separate tiles.
- Colour annotations sit immediately under the heading, no blank line.
  `@colorSize` is always `LIGHTER`; `@color` comes from `GHOST, CUMULUS, GOSSIP,
  COLDTURKEY, BLUE_CHALK, MILK_PUNCH, HAWKES_BLUE, PATTENS_BLUE, SAIL,
  ATHENS_GRAY, BERMUDA, PERFUME, SELAGO, RICE_FLOWER, WHITE_LINEN, POLAR` —
  never a semantic name (RED, BLUE, GREY…).
- Labelled inline hyperlinks only — never a bare URL and never a line containing
  only a link.
- Blank line between every item.
- Omit `Opportunities` entirely when there is nothing real to put in it.

Directly above the approval form show: today's local date, the connected xTiles
account name and email, and the destination — today's personal Daily planner.

```
genui{"ask_user_input":{"questions":[
  {"question":"Write this reflection to today's Daily?","options":["Write to Daily","Change something","Cancel"],"type":"single_select","free_text_placeholder":"Say what to change"}
]}}
```

`Change something` → collect the change in a form, revise **only** that part,
re-show the full preview, ask again. `Cancel` → stop without writing the tile;
task mutations already approved stay applied, since they were approved
explicitly.

**Existing reflection.** Before writing, compare the exact dated H3 title with
today's planner content. If it is already there:

```
genui{"ask_user_input":{"questions":[
  {"question":"Today already has a reflection. What should I do?","options":["Replace today's reflection","Append another reflection","Cancel"],"type":"single_select","free_text_placeholder":"Something else"}
]}}
```

Replace only through a safe exact content-update capability; if none exists,
offer Append or Cancel. On a scheduled run, an existing exact tile means do
nothing.

---

## Stage 8 — Write, layout, verify

After `Write to Daily` (or immediately on a scheduled run):

1. Call `xtiles_create_tiles_from_markdown_in_my_planner` with `period: "day"`,
   `date`: today's local ISO date, `markdown`: the single reflection H3 section.
   The schema must accept `date`, `period`, `markdown` without a project or view
   ID — if it demands one, do not call it and do not fall back to project
   creation; report that the personal Daily write is unavailable.
2. **Layout pass — immediate, silent, never asked about.** Take `view_id` and
   the single `tile_id` from the write response (never re-derive them), call
   `xtiles_get_workflow` with id `tile-layout` and follow it, passing that one
   tile as the added tile and the markdown just written as its content. Hint:
   **the reflection gets its own full-width row**, placed in the first free band
   without moving or overlapping any existing tile.
3. **Verify.** Re-read today's Daily and confirm the exact dated title is
   present. Retry the write **once** only if the write reported success but the
   title is absent.

---

## Stage 9 — CTA and schedule

1. Use the exact user-facing `parent_resource_url` (or equivalent) returned by
   xTiles, **byte-for-byte** — same host, environment and path. Never build a
   URL from `view_id`, never substitute a generic planner route, never show a
   raw ID.
2. If no valid user-facing URL comes back, say the reflection was saved but
   xTiles returned no openable link, and stop before the optional stages.
3. Show it as a labelled link — `[Open Evening Reflection in My Planner](url)` —
   and retain it.
4. Schedule form — shown on **every** successful manual run, immediately after
   the link:

```
genui{"ask_user_input":{"questions":[
  {"question":"Run the Evening Reflection automatically?","options":["Schedule it","No schedule"],"type":"single_select","free_text_placeholder":"Another cadence"}
]}}
```

On `Schedule it`:

```
genui{"ask_user_input":{"questions":[
  {"question":"Which days?","options":["Weekdays","Every day"],"type":"single_select","free_text_placeholder":"Specific days"},
  {"question":"What time?","options":["18:00","19:00","21:00"],"type":"single_select","free_text_placeholder":"Another local time"}
]}}
```

Resolve the next occurrence in the user's timezone and create the automation
with an exact schedule. Its prompt must invoke this workflow and embed a
complete `evening-reflection config:` JSON block — role, sources, content, tone,
autolog, Slack scope and channels, language, personal-Daily target, the
silent-run instruction, deduplication, and the layout requirement. **Never
create a duplicate automation.**

---

## Stage 10 — Related

Offer the other four workflows, each with a one-line description of what it
does — never a bare list of names:

```
genui{"ask_user_input":{"questions":[
  {"question":"Want to set up anything else on xTiles?","options":["☀️ Daily Brief — tomorrow morning's digest from Slack, Gmail and Calendar","📰 Today News — a daily topic-based news digest from the live web","📊 Weekly Review — what actually moved forward this week","🧭 Life Brief — personal priorities and open loops beyond work tools","Nothing else"],"type":"single_select","free_text_placeholder":"Something else"}
]}}
```

Treat the selection as a direct invocation: **in the same turn**, call
`xtiles_get_workflow` with the matching id and continue from its first
applicable stage:

| Option | Workflow id |
| --- | --- |
| Daily Brief | `daily-brief-with-gpt` |
| Today News | `today-news-with-gpt` |
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
end silently after write, layout and verification.
