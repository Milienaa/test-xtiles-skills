---
name: evening-reflection
description: >
  Use when setting up or running xTiles Evening Reflection — an end-of-day
  synthesis written to the Daily planner page as a "Day Characteristic" tile
  with a seed for tomorrow. Only Daily period is supported.

  Setup triggers: "set up evening reflection", "personalize my evening review",
  "connect reflection to tools", "onboard me into evening reflection".

  Run triggers: "reflect on my day", "characterize my day", "evening review",
  "wrap up my day", "what did I get done today". Also runs on schedule.

  Environment: this is the Claude / Cowork variant — it renders `show_widget`
  and `AskUserQuestion`. In ChatGPT Work, where every form is an inline
  `ask_user_input` / `genui` surface and `show_widget` does not exist, use
  `evening-reflection-with-gpt` instead.

  Environment triggers: "Evening Reflection in Claude", "the Claude version",
  "Claude Evening Reflection".
allowed-tools: >
  mcp__xtiles__xtiles_get_current_user,
  mcp__xtiles__xtiles_get_user_timezone,
  mcp__xtiles__xtiles_get_planner_content,
  mcp__xtiles__xtiles_list_tasks,
  mcp__xtiles__xtiles_create_tasks,
  mcp__xtiles__xtiles_update_task,
  mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner,
  mcp__xtiles__xtiles_create_notification,
  mcp__xtiles__xtiles_get_page_layout,
  mcp__xtiles__xtiles_set_page_layout,
  mcp__xtiles__xtiles_get_workflow,
  mcp__xtiles__xtiles_list_calendar_events,
  mcp__claude_ai_Slack__slack_search_channels,
  mcp__claude_ai_Slack__slack_search_public_and_private,
  mcp__claude_ai_Slack__slack_read_channel,
  mcp__claude_ai_Slack__slack_read_thread,
  mcp__claude_ai_Gmail__search_threads,
  mcp__claude_ai_Gmail__get_thread,
  recent_chats,
  conversation_search,
  mcp__mcp-registry__suggest_connectors,
  AskUserQuestion,
  anthropic-skills:schedule,
  mcp__scheduled-tasks__create-scheduled-tasks
---

# xTiles Evening Reflection — Setup & Daily Wrap-Up

The evening bookend to the morning Daily Brief. Where the morning brief points
forward at signals to act on, the evening reflection looks back: it synthesizes
what actually happened today, optionally logs it as completed tasks, and seeds
tomorrow.

## Three principles

1. **Survey first, write to xTiles last.** On setup and on the first manual run,
   the **reflection tile itself** is never written until the user has seen the
   step 6 preview and approved it. **Auto-log task creation (step 5) is the one
   exception**: it follows the `autolog` setting exactly as configured — an
   explicit "Yes, automatically" is itself the user's approval, honored from the
   very first run, never re-confirmed "just this once."
2. **Real data, not placeholders.** Always pull from the user's own past Claude
   chats via `recent_chats` (no connector needed) and from any connected tools
   before preview so the user sees live content. Never invent names, meetings, or
   messages — and never reconstruct the day from this session's context instead
   of reading it.
3. **Match the user's language** throughout the entire flow — match the language
   of the user's first message and adapt if they switch. On a **scheduled run**
   there is no user message: use the language of the scheduled-task config
   prompt, and if that is ambiguous, the language of the fetched content.
   Default to English when still unclear. **Every template, label, and example in
   this file is written in English as a placeholder** — translate them into the
   detected language, and never emit a label in a language the user has not used.
4. **Every write to the planner is immediately followed by a layout pass — automatically, no exceptions.** The instant the reflection tile is added in step 7, re-lay-out only that new tile into a justified grid (step 7's layout pass, via the shared `tile-layout` workflow) *before* anything else — before the CTA button, before the schedule widget, before any message to the user. This never waits for the user to ask, is never skipped as "not needed this time," and is never left for a later run.

---

## Algorithm

**Period is always Daily.** At the start of setup, tell the user: "I'll set up
your **Evening Reflection** — a short end-of-day synthesis written to your Daily
planner page." Never ask which period to set up. If the user asks for Weekly or
Monthly, say only the Daily reflection is currently supported.

**Run mode — detect before step 1:**

- **Scheduled run**: the incoming message contains `role:`, `tools:`,
  `evening_content:`, `tone:`, and `autolog:` (config injected by the `schedule`
  skill). Do not show the survey. Extract the config from the message,
  including `notify:` if present (default `false` for older scheduled configs
  that predate this setting). If a
  connector from the config is not detected — offer to walk the user through
  connecting it before continuing. Then jump to **step 4 (Silent data fetch)**.
  A scheduled run executes autonomously through to the written tile — but it
  still respects the `autolog` flag (only auto-creates tasks if it was approved
  at setup).
- **Fast-track or fresh manual run**: proceed to step 1.

---

### 1. Fast-track

If the user is specific ("reflect on my day using Slack and calendar") — skip the
full survey. Minimum needed: which connectors to pull from — infer from the
message, then check the detection table (step 2) to confirm availability. If a
required connector is not detected — offer to walk the user through connecting it
(see **How to connect connectors**); wait for confirmation. Pull only from
connectors that are both mentioned and confirmed available. Jump to **step 4**.

If the request is general — run the full flow.

---

### 2. Detect what's connected — silently, don't ask

**Never ask the user what they have connected.** Detect it yourself by checking
which MCP tools are present in this session, and treat that as the source of
truth. The survey (step 3) is only about *preferences*, never about connection
status.

| Connector | Identifying MCP tools                                                                                            |
|-----------|-----------------------------------------------------------------------------------------------------------------|
| **Claude (past chats)** | `recent_chats`, `conversation_search` — **not a connector**: present by default, nothing to connect, and never offered in a connect flow |
| xTiles    | `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`, `mcp__xtiles__xtiles_list_tasks`               |
| Slack     | `mcp__claude_ai_Slack__slack_search_channels`, `mcp__claude_ai_Slack__slack_read_channel`                       |
| Gmail     | `mcp__claude_ai_Gmail__search_threads`, `mcp__claude_ai_Gmail__get_thread`                                       |
| Calendar (xTiles) | `mcp__xtiles__xtiles_list_calendar_events`                                                                  |
| Calendar  | any other connected calendar/events MCP tool (e.g. Google Calendar), if any                                       |
| PM / issue tracker | **any connected PM tool** — Linear (`mcp__claude_ai_Linear__list_issues`), Jira, Asana, monday, ClickUp, GitHub Issues. Detect by the connector's issue/task read tools; treat all of them the same way (read-only). |

**Calendar (xTiles)'s tool presence is not proof of a linked calendar.** Unlike
Slack or Gmail, its tool comes from the required xTiles server itself, not from
a separate OAuth connection — so it's always "present" once xTiles is
connected, whether or not the user actually linked a Google/Outlook calendar
inside xTiles. Don't add it to the detected/pre-checked set on tool presence
alone; leave its survey card unchecked and resolve "maybe not linked" later, in
step 4, if today turns out empty. `Calendar` (the external one) doesn't have
this problem — a real connector, detected and pre-checked normally.

Build the **detected set** from this. Connecting is only ever raised in two cases:

- **xTiles is the only hard requirement.** If xTiles is not detected — stop and
  walk the user through connecting it (see **How to connect connectors**). Wait
  for confirmation before continuing.
- **The user explicitly wants a source that isn't detected** (e.g. picks "add
  another tool" in the survey, or names one). Then *offer* to connect it via
  `mcp__mcp-registry__suggest_connectors`. If they decline — just skip that
  source and continue. Never block the flow on an optional connector, and never
  prompt to connect something that is already detected.

These connectors are external and optional — they are not shipped with this
plugin.

---

### 3. Reflection preferences

Reconcile the survey's source picks against the **detected set** from step 2:
pre-check the tools that are already connected, pull only from detected sources,
and treat an "Other…" / not-detected pick as the *only* trigger to offer
connecting. Never re-prompt to connect something already detected.

**When rendering the Survey widget, only ever pre-check existing cards (add the
`sel` class, matching `tools`/`togTool` state) and relabel the PM-tool card to
whatever tool step 2 actually detected (Jira, Asana, monday… in place of
"Linear") — never rebuild the card list from scratch, and never drop the
"Other…" card or its free-text input.** It is the only way to add a source this
skill doesn't name explicitly, and it must render every time, connected sources
or not.

Ask three things (folded into the survey widget; inline in Claude Code):

**3.1 What to reflect on each evening.** Options — include only those relevant to
connected tools:
- Slack threads you were active in *(only if Slack connected)*
- Emails you sent / that needed action *(only if Gmail connected)*
- Meetings & calls (from calendar/notes) *(if Calendar (xTiles), Calendar, or Granola connected)*
- Tasks you completed today *(xTiles — on by default)*
- Issues you moved or were assigned in your PM tool — Linear, Jira, Asana, monday… *(only if a PM tool is connected)*
- Other (describe in next message)

**3.2 Tone.** Single select:
- **Honest coach** — direct, names weak days plainly, anti-fluff (default)
- **Gentle** — supportive and encouraging
- **Neutral** — plain factual summary, no editorializing

**3.3 Auto-log completed tasks.** Single select:
- **Yes, automatically** — just create and complete them, no preview (recommended default — this is what most users want)
- **Yes, with a preview first** — show me the tasks before creating them, then do
  it automatically on scheduled runs
- **No** — don't auto-create tasks, only write the reflection tile

**If Slack is selected and the user has not named channels:** call
`mcp__claude_ai_Slack__slack_search_channels` with query `general`, show up to 6 channel names. Ask via
`AskUserQuestion` (multi allowed): "Which channels reflect your real work? Pick
all that matter." Include found channels plus a fixed **"Other — I'll type the
names"** option. Add typed names as-is.

**Ignored Slack noise (configurable).** By default, ignore automated/bot channels
and alert streams (Sentry/error bots, build/deploy notifications, health/status
channels). Do **not** hardcode specific company channel names — derive the filter
from channel metadata (bot-posted, app-integration) and let the user add or
remove channels. If the user names channels to always ignore, store them in the
config.

**General rule:** if the user writes something custom — add it as-is. Don't
reshape it into a predefined option.

---

### 4. Silent data fetch

**Silently, without messaging the user**, pull today's data. First resolve
context:
- `mcp__xtiles__xtiles_get_user_timezone` — the user's IANA timezone and current
  local datetime. Use it to define "today" (00:00–23:59 local) and to resolve
  dates.
- `mcp__xtiles__xtiles_get_current_user` — the user's name, email, and xTiles user
  id. Do **not** rely on injected template variables for identity.

Then pull from selected connectors:

- **xTiles tasks (PRIMARY SOURCE)**: `mcp__xtiles__xtiles_list_tasks` with
  `completed: "true"`, `due_date_after`: today, `due_date_before`: tomorrow,
  `per_page: 50` — these completed tasks are their own signal for 🎯 Results,
  not just a reconciliation target for step 5. Repeat with `completed: "false"`
  for open tasks due today.
- **Daily Page notes**: from the same `mcp__xtiles__xtiles_get_planner_content`
  call (`period: "day"`, today's date), read every tile on the page beyond the
  task list — saved notes, decisions recorded, anything the user wrote or
  pasted in during the day. This is real signal for 🎯 Results and 🌟
  Opportunities, not just background context; don't fetch the page and then
  only look at its tasks.
- **Slack**: `slack_search_public_and_private` with `on:[today] from:[user]` to
  find where the user was active, then `slack_read_thread` on the important
  threads. Apply the ignore filter from step 3.
- **Gmail**: `mcp__claude_ai_Gmail__search_threads` for mail sent today and
  important received mail that needed action (`newer_than:1d`), then `get_thread`
  for sender/subject/threadId.
- **Calendar / meeting notes**: build one merged event list for today, then analyse it.
  - **Calendar (xTiles), if selected.** Call `mcp__xtiles__xtiles_list_calendar_events` for today.
  - **Calendar (the other connected calendar tool), if also selected.** Call its `list_events`-equivalent for today and add its events to the same list.
  - **Dedup across the two.** Drop an event from the Calendar connector when an xTiles-calendar event already has the same start time and the same title (case-insensitive) — never show the same meeting twice.
  - **If Calendar (xTiles) was selected and contributed zero events, don't assume the day was simply free.** The tool can't tell "nothing scheduled" from "no calendar linked." If Calendar also contributed nothing (or wasn't selected), note this once in the reflection instead of silently treating it as a free day. Skip the note if Calendar *did* contribute at least one event.
  - **Multiple connected xTiles calendars, free plan (mandatory disclosure).** `xtiles_list_calendar_events`'s response may itself carry a message that reading multiple calendars requires the Pro plan (only one of several connected accounts is readable, the rest withheld) — it states the real count of connected accounts and an upgrade URL. Watch for it every time this tool is called. If present, extract the account count (**N**) and the upgrade URL, and surface it — never absorb it silently: as the `⚠️` line in the reflection tile (see step 7) **and** as the first thing shown in the step 6 preview, before any tile content. Wording (translate, keep N and the URL real): "Only 1 of your N connected xTiles calendar accounts is readable on the free plan. Reading multiple calendars requires the Pro plan. [Upgrade]({url})."
  - From the merged list: separate meetings-with-others (attendees > 1) from solo work blocks.
- **PM / issue tracker** *(any connected — Linear, Jira, Asana, monday, ClickUp,
  GitHub Issues)*: pull, **read-only**, two sets — (a) issues **completed today**
  (status moved to Done/Closed today, assigned to or worked by the user) and (b)
  issues **newly created or newly assigned to the user today** that are still
  open. Keep each issue's title, status and permalink. This is a one-way pull —
  never write a status or a task back to the PM tool.
- **Goals check (opportunistic, optional).** Call
  `mcp__xtiles__xtiles_get_planner_content` twice — `period: "week"` and
  `period: "month"`, both anchored on today — purely to check **whether a
  Goals/Milestones tile exists**, nothing more. Scan the returned tiles for a
  heading containing `Goal`, `Goals`, `Milestone`, `Milestones`, `OKR`,
  `Target`, or `Focus`. **If found**, extract the goal statements verbatim —
  they feed the optional Goal progress section in step 6. **If neither page has
  one, drop this silently** — no note, no flag; most users won't have set one
  up, and that's not a connector failure worth mentioning (unlike the Calendar
  "maybe not linked" case above).
- **Claude chat history (today)** — **always run, no connector needed**: call
  `recent_chats` with `after` = today 00:00 and `before` = now, both ISO-8601 in
  the user's timezone from `xtiles_get_user_timezone` (never UTC — an evening run
  in a +2 zone would otherwise pull the wrong day), `sort_order: "desc"`, ~10
  chats. Page back with `before` set to the oldest chat you got while the results
  still fall inside today; stop at the first one that doesn't. Extract concrete
  *outcomes* — what was actually built, solved, decided, or shipped (e.g.
  "wrote the launch email", "fixed the auth bug", "researched competitors").
  Ignore abandoned or purely exploratory threads. These outcomes feed both the
  reflection and the auto-log.
  - Use `conversation_search` only to close a specific gap — a thread that
    clearly continues earlier work, or a task from the xTiles list you suspect
    was finished in a chat and want to confirm before auto-logging it. One query
    per gap, `max_results: 5`, and keep only hits from today. Never sweep the
    archive.
  - **Exclude this workflow's own chats** — the reflection run itself, the
    morning `daily-brief` run, and any setup/survey conversation are machinery,
    not outcomes. "Ran my evening reflection" is never an achievement.
  - Keep each outcome's conversation URL as returned by the tool — it is what
    lets the reflection link back to where the work happened. Never build a chat
    URL by hand.
  - **If `recent_chats` is unavailable or errors** — treat it exactly like a
    failed connector below: drop the source, say so explicitly, and build the
    reflection from xTiles tasks and the remaining connectors. Never fall back to
    this session's own context, and never invent a day.

Use only real data from connectors. If a connector call fails (error, timeout,
401) — record the failure and surface it explicitly later as "Could not fetch
     [connector] — connector error" (never silently write "No data").

---

### 5. Auto-log preview & write (respect the autolog setting)

Compare today's real activity — including outcomes pulled from Claude chat
history — against the existing xTiles task list, and decide per activity:

- **Close existing** — if the activity matches an **open** task due today (same
  work, even if worded differently), just mark that task complete with
  `mcp__xtiles__xtiles_update_task` (`completed: true`). Do **not** create a
  duplicate.
- **Create + close** — if the activity has no matching task, create it with
  `mcp__xtiles__xtiles_create_tasks` and immediately mark it complete.
- **Skip** — if a completed task for it already exists.

Match generously on meaning, not exact wording (e.g. a Claude chat "wrote the
launch email" closes an open task "Draft launch email").

**Sync from connected PM tools (one-way pull — never write back).** Fold the
PM/issue-tracker data into this same reconciliation:

- **Completed issue → completed task.** An issue that moved to Done/Closed today
  is a completed activity: close the matching open xTiles task, or create it and
  immediately close it if none exists. Match on meaning, never duplicate.
- **New issue → open task.** An issue newly created or newly assigned to the
  user today that is still open becomes a new **open** xTiles task — created but
  **not** marked complete — deduped against existing tasks. This is the one
  exception to "create + close": new work is logged open so it carries into
  tomorrow. In the autolog preview, show these as `🆕 [emoji] open task:
  [title]`, distinct from the completed ones.

**What counts as an activity** (derive categories from the data and the user's
role — do not force a fixed founder/PM template):
- 📞 Meetings & calls
- 💬 Interviews / CustDev / user or partner conversations
- 📝 Content created (posts, emails, docs, significant Slack messages)
- 🤝 Partnership or relationship moves — new contacts, agreements, follow-ups
- 🔧 Support / access granted
- 🔬 Research, analysis, deep dives
- 🛠 Materials prepared (decks, webinars)
- 🗺 Strategic decisions or priority shifts

**Behavior by `autolog` setting — always exactly as configured, including the
very first manual run right after setup. The setting itself is the user's
standing approval; never add an extra confirmation on top of it:**
- **Yes, with a preview first**: show the proposed list in chat, marking each as
  **new** or **closing an existing task** — `✅ [emoji] short specific title` for
  new, `☑️ closes: "[existing task]"` for matches — and ask via
  `AskUserQuestion`: "Log these for today?" → Apply all / Edit the list / Skip.
  Only after approval, apply them.
- **Yes, automatically**: create without preview — **on every run, including the
  first one.** Do not fall back to the preview flow "just this once"; the user
  already answered this question during setup.
- **No**: skip task creation entirely; go straight to the reflection tile.

To create: `mcp__xtiles__xtiles_create_tasks` with `assignee_email` (from
`get_current_user`), `due_date`: today (yyyy-MM-dd), `title`: short specific name
with an emoji category prefix. Then mark each completed:
`mcp__xtiles__xtiles_update_task` with `completed: true` — **except a new open
task synced from a PM tool, which stays open (never `completed: true`)**. Avoid
duplicates — never recreate a task that already exists for today.

---

### 6. Compose the reflection & preview

**Derive themes dynamically from ALL collected data — never from a template.**
Determine: the main themes of the day; what actually moved something forward vs
pure operations; opportunities (new contacts, ideas, competitive intel, insights);
promises & follow-ups (what was promised, who needs a message). Explicitly draw
on **all** of step 4's sources for this, not just chat/connector outcomes:
completed xTiles tasks in their own right (not only as a reconciliation
target), and any Daily Page notes — both are real signal for Results and
Opportunities, easy to under-weight next to the more narrative Slack/Gmail/chat
sources.

**If step 4's Goals check found a Goals/Milestones tile** (Weekly and/or
Monthly), add one optional **Goal progress** section: for each goal, one line —
`- **[Goal name]** — [✅ Clear progress / 🔄 Some / ⬜ No movement / 🚫
Blocked] — one-line note on how today connects to it, or honestly that it
didn't`. Skip a goal only if today genuinely has nothing to say about it — but
don't invent a connection to avoid an empty line either. **If no Goals tile was
found on either page, omit this section entirely** — most users won't have one
set up, and that's normal, not a gap to flag.

Apply the chosen **tone**. If the day was quiet, say so honestly — don't stretch
it. Optional sections appear only if there is something genuinely valuable.

**Show it with the Preview widget HTML** (see below) via `show_widget` — never
as a plain-text chat dump. Build it fully dynamically: inject the real
composed content into `#digest`, following the injection pattern in the
template's comment block. The widget's own three buttons (**Looks good —
write it** / **Change something** / **Cancel**) *are* the approval step — do
not also ask via `AskUserQuestion` afterward, that would be a second approval
for the same decision. If the user picks "Change something," update only that
section, then show the Preview widget again with the revised content — never
fall back to a plain-text re-preview. **If step 4's multi-calendar Pro-plan
disclosure applies**, it goes in first, above the reflection content — the
user must see it before approving, not just inside the tile itself. See the
widget's `.notice` element below. On scheduled runs, skip this step
entirely and write directly.

---

### 7. Write to xTiles

**Only after approval (or on a scheduled run).**

Tool: `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`
- `period`: "day"
- `date`: today in ISO 8601
- `markdown`: all sections in a **single call** — never split per section.

**This skill has no way to update an existing tile's content, so every run creates a fresh reflection tile.** The evening reflection writes to the same Daily page the morning brief uses. If today's page already has a `✨ Day Characteristic` tile — from a saved template or from an earlier run today — still create the new one with `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`: do not fetch the page to check for a matching heading first, and do not skip the write because one might already be there. A re-run is expected to produce a fresh, current reflection, not to silently do nothing.

**One single tile.** The whole reflection is **one** `###` tile titled
`✨ Day Characteristic — DD.MM.YYYY` — not separate tiles per section. The color
annotations sit on the **two lines directly below the title** (no blank line
between the title and the annotations):

```
### ✨ Day Characteristic — DD.MM.YYYY
@colorSize: LIGHTER
@color: SAIL

**[1–2 sharp sentences — the tone and essence of the day]**

---

**🎯 Results**

- [What was done]

---

**🌟 Opportunities**

- [Specific finds]

---

**🎯 Goal progress**

- **[Goal name]** — [✅ Clear progress / 🔄 Some / ⬜ No movement / 🚫 Blocked] — [one-line note on how today connects, or honestly that it didn't]

---

**→ Tomorrow**

<task dueDate="YYYY-MM-DD">[Specific action with names — "Message X about Y", not "continue X" — dueDate is tomorrow's date]</task>

<task dueDate="YYYY-MM-DD">[max 3 items total]</task>

⚠️ [unavailable connectors, only if some failed]
```

- `@colorSize` is always `LIGHTER`; `@color` is always `SAIL` for this tile. (Do
  **not** use plain names like "purple" — they will not render.)
- Sections inside the tile are **bold subheaders** (`**🎯 Results**`), separated by
  `---` dividers — never separate `###` tiles. Never `####`/`#####` headings —
  a tile has exactly one heading, its own `###` title.
- Drop any optional section (Opportunities, Goal progress) entirely if there's
  nothing genuinely valuable — don't leave an empty header. **Goal progress**
  specifically is dropped whenever step 4's Goals check found nothing — most
  days, most users.

**Content formatting inside the tile:**
- Separate each item with a blank line — never a continuous block.
- Slack/email entries that have a URL are Markdown hyperlinks with the priority
  emoji BEFORE the `[`, never inside the brackets. An outcome that came from a
  chat follows the same rule, linked to the conversation URL `recent_chats`
  returned — the chat title is the link text. If the tool returned no URL, write
  the title as plain text; never fabricate a `claude.ai` link.
- **Tomorrow's actions are real xTiles tasks, not checkboxes.** Write each as
  `<task dueDate="YYYY-MM-DD">Task name</task>` with `dueDate` set to
  **tomorrow's date** (this seeds tomorrow, it isn't due today), one empty line
  between each, never a numbered list. **Never `- [ ]`** — a plain markdown
  checkbox renders inside the tile's text but never becomes a real, trackable
  xTiles task; only `<task>` does (same rule `daily-brief` uses for its action
  items — see its "Action items are real tasks" section).
- Append the final `⚠️ [unavailable connectors]` line only if a connector failed. **If the multi-calendar Pro-plan disclosure applies (see step 4), it is always the first `⚠️` line, ahead of the unavailable-connectors line** — e.g. `⚠️ Only 1 of your 3 connected xTiles calendar accounts is readable on the free plan. Reading multiple calendars requires the Pro plan. [Upgrade](url)`.

**Terminal sequence — after a successful write, these come in this exact order, each as its own separate widget/question, none skipped, none merged into another:**
**(a) CTA link → (b) Schedule offer → (c) Related workflows.** The offer to open the reflection in xTiles (the CTA) and the schedule offer are distinct moments — never drop either, and never collapse the schedule offer into the CTA or the related step. Losing or reordering any of these is a failed run. On a scheduled run, stop after the layout pass and the notification (step 5 below) — none of (a)–(c) are shown; nobody is watching chat during an unattended run.

**After a successful write — run these steps in order, no exceptions. Step 2 (the layout pass) is not optional and is never deferred, asked about, or judgment-called away — it runs automatically, immediately after every single write, before step 3's CTA button is even composed:**

1. Write `✅ Evening reflection saved.`
2. **Layout pass — mandatory, silent, automatic, every single run (scheduled runs included, fast-track included, any tile count included).** Using the `view_id` and `tile_ids` returned by the write call above (`tile_ids` is ordered to match the `###` sections you just wrote — here a single reflection tile), apply the shared justified-grid layout rules: call `mcp__xtiles__xtiles_get_workflow` with id `tile-layout` and follow it exactly — treat the tiles in `tile_ids` as its "added tiles" and the markdown you just composed as their content. **Layout hints for this workflow:** always exactly 1 tile (the reflection) · give it a generous width — max_width, or the largest free band next to existing tiles. Do not message the user about this pass, do not ask for confirmation, and never skip it — not even for a single tile or a scheduled run. (You may fetch `tile-layout` once per session and reuse it on later runs.)
3. **For non-scheduled runs only: link to the reflection tile, not the page.** (On a scheduled run, skip this widget entirely — step 5 below reaches the user instead.)
   - Take the `resource_url` of the **first** entry in the create call's response `tiles` array (a deep link that opens the Daily page focused on that tile) and use it as `{VIEW_URL}` directly.
   - **Only if that entry has no `resource_url` at all** — fall back to `https://xtiles.app/{view_id}`, also read from the create call's response.
   Call `show_widget` with the **CTA widget HTML** (see below) using that `{VIEW_URL}`. Translate the button label into the user's language. **Never leave `{VIEW_URL}` unresolved and never output a markdown link instead of the widget — the button must render every non-scheduled run.**
4. **For non-scheduled runs only:** immediately continue to **step 8 (Schedule)** — do not skip, do not ask
   first, and never substitute `AskUserQuestion` for the schedule widget.
5. **Scheduled runs only, and only if the config's `notify:` flag is `true`.** The user opted into `mcp__xtiles__xtiles_create_notification` instead of a widget when they scheduled this (see step 8's notification toggle). If `notify:false` (or missing, for an older schedule) — skip this step entirely, silently, no notification:
   - `url`: the **tile-focused deep link**, so clicking the notification scrolls straight to the reflection instead of dropping the user at the top of the page. Read the `resource_url` of the **first** entry in the create call's response `tiles` array directly (step 3, which normally computes this as `{VIEW_URL}`, is skipped on a scheduled run — so read it fresh here from the same response). **Only if that entry has no `resource_url` at all** — fall back to the page URL, `https://xtiles.app/{view_id}`.
   - `text`: **always the fixed string `"Your Evening Reflection is ready — take 2 min to look back."`, translated into the user's language — no customized/dynamic part.** Do not append a highlight, a count, or any detail pulled from tonight's content — this text never changes run to run beyond translation.
   - `agent_source`: `"Claude"`.

On error, say briefly what went wrong and offer to retry.

---

### 8. Schedule (optional)

**After every successful manual write — call `show_widget` with the schedule widget** (see below),
regardless of survey answers. Do not ask first and never substitute `AskUserQuestion` for it — this is a Cowork chat widget, not a question. In Claude Code, ask inline: "Want me to run this
every evening automatically? What time? (default: 9:00 PM)".

- If the user schedules it — the widget response also carries `notify:true`/`notify:false` from its notification toggle. Invoke `anthropic-skills:schedule`, then
  `mcp__scheduled-tasks__create-scheduled-tasks`. Pass to both:
    - **`prompt`**: full config assembled from setup —
      ```
      Run evening reflection — role: {role} · tools: {tools} · evening_content: {content} · tone: {tone} · autolog: {on/preview/off} · notify: {true/false} (if true, call xtiles_create_notification at the very end of this scheduled run — mandatory, do not skip) · schedule: daily-9pm
      ```
      Replace all placeholders with real values, `notify` straight from the widget.
    - **`schedule`**: cron derived from the widget. The widget sends `cron: HH:MM days:1-5` (weekdays) or `cron: HH:MM days:*` (every day) — parse both values and build: `M H * * 1-5` for weekdays, `M H * * *` for every day. Default `0 21 * * 1-5` (9:00 PM on weekdays) if not found.
    - **`timezone`**: from `mcp__xtiles__xtiles_get_user_timezone`.
      This prompt fires each evening and triggers `evening-reflection` in
      scheduled-run mode — the full config must be embedded so the survey is skipped.

      **If `notify:true`** — the reflection that's already on the page right now (from step 7's write) is also worth notifying about; don't make the user wait for tomorrow to see it work. Call `mcp__xtiles__xtiles_create_notification` immediately, right here: `url` is `{VIEW_URL}` (the same tile-focused deep link already resolved back in step 7.3, so this notification scrolls straight to the tile too), `text` is the fixed notification string per step 7.5 above, `agent_source` is `"Claude"`.

      Confirm: "Done — your reflection will write to xTiles every [weekday evening / evening] at [time]." (say "weekday evening" if `days:1-5`, "every evening" if `days:*`; append ", and I'll notify you in xTiles each time" if `notify:true`) **This confirmation is not the end of the run — immediately continue to step 9 (Related workflows) in the same turn.**
- If the user declines — acknowledge briefly, then **immediately continue to step 9 (Related workflows) in the same turn.**

**Either way, step 9 still has to run before this turn ends — do not wait for the user to ask.**

---

### 9. Related workflows

**After every manual run, once step 8 is resolved** (scheduled or declined) —
offer related workflows. Skip this on scheduled runs, which end after step
7's notification — no widgets, nobody to ask.

Ask via `AskUserQuestion` (single select): "Want to set up anything else on
xTiles?"
- 🌅 Daily Brief — a live morning brief from your connected tools
- 📰 Today News — a daily news digest on topics you care about
- 📊 Weekly Review — a weekly summary of what moved forward this week
- Nothing else, thanks

**Never list these as plain text requiring the user to retype a choice —
always use the interactive question.**

On selection, send the exact matching phrase to hand off to that skill (do
not attempt to run it yourself):
- Daily Brief → `Set workflow of Daily Brief (daily-brief) on xTiles MCP`
- Today News → `Set workflow of Today News (today-news) on xTiles MCP`
- Weekly Review → `Set workflow of Weekly Review (weekly-review) on xTiles MCP`
- "Nothing else" — acknowledge briefly and stop.

---

## How to connect connectors

Do not send the user to settings manually and do not give a URL to follow.
Call `mcp__mcp-registry__suggest_connectors` — it renders interactive connect
buttons directly in the Cowork UI.

**Flow:**
1. Call `mcp__mcp-registry__suggest_connectors` with the names of missing
   connectors.
2. Show the **Done widget** (below) directly under the connector form.
3. The user clicks the connect buttons; auth runs natively. When finished, they
   click **"Done"**.
4. Confirm: "Connected. Continuing…" and resume from where the flow paused.

---

## Done widget HTML

Show via `show_widget` immediately after calling
`mcp__mcp-registry__suggest_connectors`.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:16px;background:transparent}
.btn{width:100%;padding:11px 20px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;background:#1a1a1a;color:#fff;transition:background .15s}
.btn:hover{background:#333}
</style>
<button class="btn" id="btn-done" onclick="doneIt()">✓ Done</button>
<script>
function doneIt(){var b=document.getElementById('btn-done');b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';b.textContent='⏳…';sendPrompt('Done — connectors connected, continue the flow');}
</script>
```

---

## Survey widget HTML

Show via `show_widget` at the start of setup in Cowork. After Submit, the user
sends a string of answers to chat — process it and continue. Adapt only by
pre-checking detected cards and relabeling the PM-tool card (see step 3) — the
`__other__` card and its free-text input (`togTool(this,'__other__')` /
`tool-other-in`) are part of the template exactly as shown below and must never
be removed.

```html
<style>
    :root{--color-background-primary:#fff;--color-background-secondary:#f5f5f5;--color-background-tertiary:#f8f8f8;--color-text-primary:#1a1a1a;--color-text-secondary:#888;--color-border-secondary:#aaa;--color-border-tertiary:#e0e0e0}
    *{box-sizing:border-box;margin:0;padding:0}
    body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:var(--color-background-tertiary);color:var(--color-text-primary)}
    .wrap{max-width:560px;margin:0 auto;background:var(--color-background-primary);border-radius:16px;padding:28px;box-shadow:0 2px 12px rgba(0,0,0,.08)}
    h2{font-size:18px;font-weight:700;margin-bottom:4px}
    .step-label{font-size:12px;color:var(--color-text-secondary);margin-bottom:20px}
    .sec{margin-bottom:22px}
    .sec-title{font-size:13px;font-weight:600;margin-bottom:8px}
    .hint{font-size:12px;color:var(--color-text-secondary);margin-bottom:8px}
    .pills{display:flex;flex-wrap:wrap;gap:7px}
    .pill{padding:6px 14px;border-radius:20px;border:1.5px solid var(--color-border-tertiary);font-size:13px;cursor:pointer;background:var(--color-background-primary);user-select:none;transition:all .15s}
    .pill:hover{border-color:var(--color-border-secondary)}
    .pill.sel{background:var(--color-text-primary);color:var(--color-background-primary);border-color:var(--color-text-primary)}
    .cards{display:flex;flex-wrap:wrap;gap:8px}
    .card{display:flex;align-items:center;gap:7px;padding:8px 13px;border-radius:10px;border:1.5px solid var(--color-border-tertiary);font-size:13px;cursor:pointer;background:var(--color-background-primary);user-select:none;transition:all .15s}
    .card:hover{border-color:var(--color-border-secondary)}
    .card.sel{background:var(--color-text-primary);color:var(--color-background-primary);border-color:var(--color-text-primary)}
    .chk{width:15px;height:15px;border-radius:4px;border:1.5px solid var(--color-border-secondary);display:flex;align-items:center;justify-content:center;font-size:9px;flex-shrink:0}
    .card.sel .chk{background:var(--color-background-primary);border-color:var(--color-background-primary);color:var(--color-text-primary)}
    .custom-in{margin-top:9px}
    .custom-in input{width:100%;padding:7px 11px;border:1.5px solid var(--color-border-tertiary);border-radius:8px;font-size:13px;outline:none}
    .checks{display:flex;flex-direction:column;gap:5px;margin-top:6px}
    .ci{display:flex;align-items:center;gap:9px;padding:7px 11px;border-radius:8px;border:1.5px solid var(--color-border-tertiary);font-size:13px;cursor:pointer;user-select:none;transition:all .15s}
    .ci:hover{border-color:var(--color-border-secondary)}
    .ci.sel{border-color:var(--color-text-primary);background:var(--color-background-secondary)}
    .ci.sel .chk{background:var(--color-text-primary);border-color:var(--color-text-primary);color:var(--color-background-primary)}
    .btn-row{display:flex;gap:10px;margin-top:22px}
    .btn{padding:9px 18px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:all .15s}
    .btn-p{background:var(--color-text-primary);color:var(--color-background-primary);flex:1}
    .btn-p:hover:not(:disabled){opacity:.9}
    .btn-p:disabled{opacity:.5;cursor:not-allowed}
    .btn-s{background:var(--color-background-secondary)}
</style>

<div class="wrap" id="app">
  <!-- STEP 1 -->
  <div id="s1">
    <div class="step-label">Step 1 of 2</div>
    <h2>About you</h2>
    <div class="sec" style="margin-top:18px">
      <div class="sec-title">What's your role?</div>
      <div class="pills" id="role-pills">
        <div class="pill" onclick="pickRole(this,'Product Manager')">Product Manager</div>
        <div class="pill" onclick="pickRole(this,'Designer')">Designer</div>
        <div class="pill" onclick="pickRole(this,'Engineer')">Engineer</div>
        <div class="pill" onclick="pickRole(this,'Growth & Marketing')">Growth & Marketing</div>
        <div class="pill" onclick="pickRole(this,'Founder / CEO')">Founder / CEO</div>
        <div class="pill" onclick="pickRole(this,'Support & Success')">Support & Success</div>
        <div class="pill" onclick="pickRole(this,'__other__')">Other role…</div>
      </div>
      <div class="custom-in" id="role-other-wrap" style="display:none">
        <input id="role-other-in" type="text" placeholder="Your role…" oninput="chkNext()">
      </div>
    </div>
    <div class="sec">
      <div class="sec-title">Which sources should feed your reflection?</div>
      <div class="hint">Pre-checked = already connected. Pick "Other…" only to add a source you haven't connected yet.</div>
      <div class="cards" id="tool-cards">
        <div class="card" onclick="togTool(this,'Slack')"><div class="chk">✓</div>Slack</div>
        <div class="card" onclick="togTool(this,'Gmail')"><div class="chk">✓</div>Gmail</div>
        <div class="card" onclick="togTool(this,'CalendarXTiles')"><div class="chk">✓</div>Calendar (xTiles)</div>
        <div class="card" onclick="togTool(this,'Calendar')"><div class="chk">✓</div>Calendar</div>
        <div class="card" onclick="togTool(this,'Granola')"><div class="chk">✓</div>Granola</div>
        <div class="card" onclick="togTool(this,'Linear')"><div class="chk">✓</div>Linear</div>
        <div class="card" onclick="togTool(this,'__other__')"><div class="chk">✓</div>Other…</div>
      </div>
      <div class="custom-in" id="tool-other-wrap" style="display:none">
        <input id="tool-other-in" type="text" placeholder="Other tools (comma-separated)…">
      </div>
    </div>
    <div class="btn-row">
      <button class="btn btn-p" id="next-btn" onclick="go2()" disabled>Next →</button>
    </div>
  </div>

  <!-- STEP 2 -->
  <div id="s2" style="display:none">
    <div class="step-label">Step 2 of 2</div>
    <h2>Your evening reflection</h2>
    <div class="sec" style="margin-top:18px">
      <div class="sec-title">What should the reflection cover?</div>
      <div class="checks" id="evening-content"></div>
    </div>
    <div class="sec">
      <div class="sec-title">Tone</div>
      <div class="pills" id="tone-pills">
        <div class="pill sel" onclick="pickTone(this,'Honest coach')">Honest coach</div>
        <div class="pill" onclick="pickTone(this,'Gentle')">Gentle</div>
        <div class="pill" onclick="pickTone(this,'Neutral')">Neutral</div>
      </div>
    </div>
    <div class="sec">
      <div class="sec-title">Auto-log today's activities as completed tasks?</div>
      <div class="pills" id="autolog-pills">
        <div class="pill sel" onclick="pickLog(this,'auto')">Yes — automatically</div>
        <div class="pill" onclick="pickLog(this,'preview')">Yes — preview first</div>
        <div class="pill" onclick="pickLog(this,'off')">No</div>
      </div>
    </div>
    <div class="btn-row">
      <button class="btn btn-s" onclick="go1()">← Back</button>
      <button class="btn btn-p" onclick="submit()">Set up reflection</button>
    </div>
  </div>
</div>

<script>
var role=null, tools=new Set(), content=new Set(), tone='Honest coach', autolog='auto';
var TM={
  'Slack':   ['Slack threads I was active in'],
  'Gmail':   ['Emails I sent / that needed action'],
  'CalendarXTiles':['Meetings & calls'],
  'Calendar':['Meetings & calls'],
  'Granola': ['Meeting notes & summaries'],
  'Linear':  ['Linear issues I moved']
};
var ROLE_DEFAULTS={
  'Product Manager':   ['Meetings & calls','Slack threads I was active in','Tasks I completed today'],
  'Designer':          ['Slack threads I was active in','Tasks I completed today'],
  'Engineer':          ['Linear issues I moved','Tasks I completed today','Slack threads I was active in'],
  'Growth & Marketing':['Emails I sent / that needed action','Tasks I completed today','Meetings & calls'],
  'Founder / CEO':     ['Meetings & calls','Slack threads I was active in','Emails I sent / that needed action','Tasks I completed today'],
  'Support & Success': ['Emails I sent / that needed action','Slack threads I was active in','Tasks I completed today']
};
function pickRole(el,v){document.querySelectorAll('#role-pills .pill').forEach(function(p){p.classList.remove('sel')});el.classList.add('sel');role=v;document.getElementById('role-other-wrap').style.display=v==='__other__'?'block':'none';chkNext();}
function togTool(el,v){el.classList.toggle('sel');if(v==='__other__'){document.getElementById('tool-other-wrap').style.display=el.classList.contains('sel')?'block':'none';}else{el.classList.contains('sel')?tools.add(v):tools.delete(v);}chkNext();}
function pickTone(el,v){document.querySelectorAll('#tone-pills .pill').forEach(function(p){p.classList.remove('sel')});el.classList.add('sel');tone=v;}
function pickLog(el,v){document.querySelectorAll('#autolog-pills .pill').forEach(function(p){p.classList.remove('sel')});el.classList.add('sel');autolog=v;}
function chkNext(){var ok=role&&(role!=='__other__'||document.getElementById('role-other-in').value.trim());document.getElementById('next-btn').disabled=!ok;}
function go2(){if(!content.size){var r=role==='__other__'?null:role;(ROLE_DEFAULTS[r]||['Tasks I completed today']).forEach(function(v){content.add(v);});}renderContent();document.getElementById('s1').style.display='none';document.getElementById('s2').style.display='block';}
function go1(){document.getElementById('s2').style.display='none';document.getElementById('s1').style.display='block';}
function renderContent(){
  var items=['Tasks I completed today'];
  tools.forEach(function(t){if(TM[t])TM[t].forEach(function(i){if(items.indexOf(i)<0)items.push(i)})});
  var html='';
  items.forEach(function(o){var s=content.has(o)?' sel':'';html+='<div class="ci'+s+'" onclick="togCI(this,\''+o.replace(/'/g,"\\'")+'\')"><div class="chk">✓</div>'+o+'</div>';});
  html+='<div class="ci" onclick="togOther(this)"><div class="chk">+</div>Something else…</div>';
  html+='<div class="custom-in" id="co-ev" style="display:none"><input type="text" placeholder="What else…"></div>';
  document.getElementById('evening-content').innerHTML=html;
}
function togCI(el,v){el.classList.toggle('sel');el.classList.contains('sel')?content.add(v):content.delete(v);}
function togOther(el){el.classList.toggle('sel');var w=document.getElementById('co-ev');if(w)w.style.display=el.classList.contains('sel')?'block':'none';}
function submit(){
  document.querySelectorAll('.btn').forEach(function(b){b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';});
  var r=role==='__other__'?document.getElementById('role-other-in').value.trim():role;
  var tArr=Array.from(tools);var tOther=document.getElementById('tool-other-in').value.trim();if(tOther)tArr.push(tOther);
  var items=Array.from(content);var inp=document.getElementById('co-ev');if(inp){var v=inp.querySelector('input');if(v&&v.value.trim())items.push(v.value.trim());}
  sendPrompt('Evening reflection setup — role: '+r+' · tools: '+(tArr.join(', ')||'none')+' · evening_content: '+(items.join(', ')||'none')+' · tone: '+tone+' · autolog: '+autolog);
}
</script>
```

---

## Preview widget HTML

Show via `show_widget` in step 6, for every manual run. Build fully
dynamically: inject the real, already-composed reflection into `#digest`
following the injection pattern in the comment block below. Omit any section
whose content doesn't exist for today (Opportunities, Goal progress) — don't
inject an empty card.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:16px;background:#f8f8f8;color:#1a1a1a}
.wrap{max-width:540px;margin:0 auto;background:#fff;border-radius:16px;padding:24px;box-shadow:0 2px 12px rgba(0,0,0,.08)}
.header{display:flex;align-items:center;justify-content:space-between;margin-bottom:16px}
h2{font-size:16px;font-weight:700}
.cnt{font-size:12px;color:#888;background:#f5f5f5;padding:2px 9px;border-radius:20px}
.scroll{max-height:380px;overflow-y:auto;margin-bottom:16px;display:flex;flex-direction:column;gap:12px}
.section{border-radius:12px;padding:14px;background:#f7f7f7}
.section-head{font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.5px;color:#888;margin-bottom:10px}
.char-line{font-size:13px;color:#1a1a1a;line-height:1.5;font-weight:600}
.item{font-size:12px;color:#333;line-height:1.5;margin-bottom:7px;padding-bottom:7px;border-bottom:1px solid rgba(0,0,0,.05)}
.item:last-child{border-bottom:none;margin-bottom:0;padding-bottom:0}
.item-title{font-weight:600;color:#1a1a1a}
.item-badge{display:inline-block;margin-right:4px}
.notice{border-radius:12px;padding:12px 14px;background:#fff6e5;border:1px solid #f3d9a4;color:#7a5a00;font-size:12px;line-height:1.5}
.notice a{color:#7a5a00;font-weight:600}
.feedback{display:none;margin-bottom:12px}
textarea{width:100%;padding:10px;border:1.5px solid #e0e0e0;border-radius:10px;font-size:13px;outline:none;resize:none;height:68px;font-family:inherit}
.btns{display:flex;flex-direction:column;gap:8px}
.btn{width:100%;padding:11px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:all .15s}
.btn-save{background:#eef6ff;color:#1a5fb4;border:1.5px solid #c3d9f7}
.btn-save:hover{background:#deeeff;border-color:#a8c8f0}
.btn-edit{background:#f0f0f0;color:#444}
.btn-edit:hover{background:#e0e0e0}
.btn-cancel{background:transparent;color:#bbb;font-size:13px;font-weight:400}
</style>
<div class="wrap">
  <div class="header">
    <h2>✨ Day Characteristic</h2>
    <span class="cnt" id="cnt">[actual date]</span>
  </div>
  <div class="scroll" id="digest">

    <!--
    INJECTION PATTERN — Claude fills in #digest with the real, already-composed reflection:

    If the multi-calendar Pro-plan disclosure applies (see step 4), inject this FIRST, above every .section — the user must see it before approving:
    <div class="notice">⚠️ Only 1 of your 3 connected xTiles calendar accounts is readable on the free plan. Reading multiple calendars requires the Pro plan. <a href="[upgrade url]">Upgrade</a></div>

    <div class="section">
      <div class="char-line">1–2 sharp sentences — the tone and essence of the day.</div>
    </div>

    <div class="section">
      <div class="section-head">🎯 Results</div>
      <div class="item">What was done.</div>
      <div class="item">Another result.</div>
    </div>

    Opportunities — only if non-trivial, omit the whole section otherwise:
    <div class="section">
      <div class="section-head">🌟 Opportunities</div>
      <div class="item">Specific find: partnership, lead, competitor intel, idea.</div>
    </div>

    Goal progress — only if step 4's Goals check found a tile, omit otherwise:
    <div class="section">
      <div class="section-head">🎯 Goal progress</div>
      <div class="item"><span class="item-badge">✅</span><span class="item-title">Goal name</span> — one-line note on how today connects.</div>
    </div>

    <div class="section">
      <div class="section-head">→ Tomorrow</div>
      <div class="item">Specific action with names — "Message X about Y", not "continue X".</div>
    </div>
    -->

  </div>
  <div class="feedback" id="fb">
    <textarea id="fbtext" placeholder="What should I change?"></textarea>
  </div>
  <div class="btns">
    <button class="btn btn-save" id="btn-save" onclick="doSave()">Looks good — write it</button>
    <button class="btn btn-edit" id="btn-edit" onclick="toggleEdit()">Change something</button>
    <button class="btn btn-cancel" id="btn-cancel" onclick="cancelIt()">Cancel</button>
  </div>
</div>
<script>
var editing=false;
function collapse(msg){document.querySelector('.btns').innerHTML='<p style="font-size:13px;color:#aaa;text-align:center;padding:4px 0">'+msg+'</p>';}
function doSave(){
  var fb=document.getElementById('fbtext').value.trim();
  collapse(fb?'⏳ Applying & saving…':'⏳ Saving…');
  sendPrompt(fb?'Apply this change then save: '+fb:'Looks good — save the reflection');
}
function cancelIt(){collapse('✓ Cancelled');sendPrompt('Cancel reflection');}
function toggleEdit(){
  editing=!editing;
  document.getElementById('fb').style.display=editing?'block':'none';
  document.getElementById('btn-save').textContent=editing?'Apply & Save':'Looks good — write it';
  document.getElementById('btn-edit').textContent=editing?'Never mind':'Change something';
}
</script>
```

---

## Schedule widget HTML

Show via `show_widget` after a successful write in Cowork. Default time is evening.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:#f8f8f8;color:#1a1a1a}
.wrap{max-width:480px;margin:0 auto;background:#fff;border-radius:16px;padding:28px;box-shadow:0 2px 12px rgba(0,0,0,.08);text-align:center}
.icon{font-size:36px;margin-bottom:12px}
h2{font-size:17px;font-weight:700;margin-bottom:6px}
.sub{font-size:13px;color:#888;margin-bottom:20px;line-height:1.5}
.time-row{display:inline-flex;align-items:center;gap:8px;background:#f3f3f3;border-radius:10px;padding:8px 16px;font-size:13px;font-weight:600;color:#444;margin-bottom:16px}
.time-row select,.time-row input[type=time]{border:none;background:transparent;font-size:15px;font-weight:700;color:#1a1a1a;outline:none;cursor:pointer}
.notify-row{display:flex;align-items:center;justify-content:space-between;text-align:left;padding:12px 14px;border-radius:10px;background:#f8f8f8;border:1.5px solid #eee;cursor:pointer;user-select:none;margin-bottom:24px}
.notify-info{flex:1;margin-right:12px}
.notify-label{font-size:13px;font-weight:600}
.notify-desc{font-size:11px;color:#888;margin-top:2px;line-height:1.4}
.sw{width:40px;height:22px;background:#e0e0e0;border-radius:11px;position:relative;transition:background .2s;flex-shrink:0}
.sw.on{background:#1a1a1a}
.knob{width:18px;height:18px;background:#fff;border-radius:50%;position:absolute;top:2px;left:2px;transition:left .2s;box-shadow:0 1px 3px rgba(0,0,0,.2)}
.sw.on .knob{left:20px}
.btns{display:flex;flex-direction:column;gap:10px}
.btn{padding:11px 20px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:all .15s}
.btn-yes{background:#1a1a1a;color:#fff}
.btn-yes:hover{background:#333}
.btn-no{background:#f0f0f0;color:#555}
.btn-no:hover{background:#e0e0e0}
</style>
<div class="wrap">
  <div class="icon">🌙</div>
  <h2>Reflect every evening?</h2>
  <p class="sub">I'll synthesize your day and write it to xTiles automatically — no need to ask each time.</p>
  <div class="time-row">
    📅 Every
    <select id="sched-days">
      <option value="1-5" selected>Weekdays</option>
      <option value="*">Day</option>
    </select>
    at <input type="time" id="sched-time" value="21:00">
  </div>
  <div class="notify-row" onclick="togNotify()">
    <div class="notify-info">
      <div class="notify-label">🔔 Notify me in xTiles</div>
      <div class="notify-desc">Get pinged the moment each reflection is ready</div>
    </div>
    <div class="sw on" id="notify-sw"><div class="knob"></div></div>
  </div>
  <div class="btns">
    <button class="btn btn-yes" id="btn-yes" onclick="scheduleIt()">Yes, schedule it</button>
    <button class="btn btn-no" id="btn-no" onclick="noThanks()">No, thanks</button>
  </div>
</div>
<script>
var notify=true;
function togNotify(){notify=!notify;document.getElementById('notify-sw').classList.toggle('on',notify);}
function collapse(msg){document.querySelector('.btns').innerHTML='<p style="font-size:13px;color:#aaa;text-align:center;padding:4px 0">'+msg+'</p>';}
function scheduleIt(){
  var days=document.getElementById('sched-days').value;
  var t=document.getElementById('sched-time').value||'21:00';
  var parts=t.split(':'),h=parseInt(parts[0],10),m=parts[1];
  var label=(h%12||12)+':'+m+' '+(h>=12?'PM':'AM');
  var dLabel=days==='1-5'?'weekdays':'every day';
  collapse('⏳ Scheduling…');
  sendPrompt('Yes, schedule my evening reflection at '+label+' '+dLabel+' (cron: '+t+' days:'+days+') · notify:'+notify);
}
function noThanks(){collapse('✓ Got it');sendPrompt('No schedule needed');}
</script>
```

---

## CTA widget HTML

Show immediately after a successful write — once only, never again after the schedule confirmation. Replace `{VIEW_URL}` with the tile-level link resolved in step 7.3 above (falling back to the page URL only if no tile-level link was available) before calling `show_widget`.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
html,body{overflow:hidden;height:auto}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:12px;background:transparent}
.btn{display:block;width:100%;padding:12px 20px;border-radius:10px;font-size:15px;font-weight:700;color:#fff;background:#1a1a1a;text-align:center;text-decoration:none;transition:background .15s}
.btn:hover{background:#333}
</style>
<a class="btn" href="{VIEW_URL}" target="_blank">Open in xTiles →</a>
```

---

## How to behave

- Use the survey widget for setup; use `AskUserQuestion` for every follow-up
  (channels, auto-log approval, preview approval, change requests). In Claude
  Code, ask the same questions inline.
- **Never write the reflection as plain chat text and ask the user to copy it** —
  always write to xTiles, or walk through connecting xTiles first.
- **Don't ask what's connected — detect it.** Only xTiles is mandatory to
  connect. For any other source the user explicitly wants but isn't connected,
  *offer* to connect; if they decline, skip it and continue. Never prompt to
  connect something already detected.
- **Never create tasks or tiles without preview and approval** on setup / first
  manual run. Only an approved scheduled run acts autonomously, and it still
  respects the `autolog` setting.
- **Respect the tone setting.** If the day was quiet, say so honestly — don't
  stretch it. Optional sections appear only when there's real value.
- **Derive themes and activity categories from the data**, never from a fixed
  template, and never hardcode company-specific channel names.
- Real data from connectors always beats placeholders — never invent names,
  meetings, or messages.
- Identity (name, email, user id) comes from `xtiles_get_current_user`; dates from
  `xtiles_get_user_timezone`.
- Daily is the only period. If asked for Weekly/Monthly, offer the Daily
  reflection instead — never silently downscope.
- Match the user's language; adapt if they switch.
- Show widgets in Cowork only — in Claude Code, ask the same questions inline.
- **The CTA button (step 7) is not optional.** Always call `show_widget` with
  the CTA widget after a successful write — never end the write step with just
  the confirmation text.
- **In Cowork, the CTA and Schedule offer are `show_widget` HTML widgets, never
  `AskUserQuestion` and never plain text** — `AskUserQuestion` is for step 9
  (Related workflows) only.
- **Never end the flow with a plain-text list of next steps.** After
  scheduling is resolved (step 8), always offer related workflows through the
  `AskUserQuestion` in step 9 — the user picks, they never retype a choice.
- **`autolog: auto` means auto from the first run, not the second.** Never add a
  one-time confirmation "just to be safe" on top of an explicit setting — that
  makes the setting meaningless.
- **The Survey widget's "Other…" card is not optional decoration.** It's the
  only way to add a connector this skill doesn't name explicitly — never drop it
  when adapting the widget to detected connectors.
