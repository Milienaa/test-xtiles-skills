---
name: weekly-review
description: >
  Run a weekly review — analyse what was worked on this week and write a summary
  to the xTiles Weekly planner page. Reads Daily pages, connected tools, and any
  Goals or Milestones tile on the weekly page to assess progress.
  Setup triggers: "set up weekly review", "run every Friday", "weekly summary".
  Run triggers: "run my weekly review", "what did I do this week",
  "weekly recap", "show me my week".

  Environment: this is the Claude / Cowork variant — it renders `show_widget`
  and `AskUserQuestion`. In ChatGPT Work, where every form is an inline
  `ask_user_input` / `genui` surface and `show_widget` does not exist, use
  `weekly-review-with-gpt` instead.

  Environment triggers: "Weekly Review in Claude", "the Claude version",
  "Claude Weekly Review".
allowed-tools: >
  mcp__xtiles__xtiles_get_planner_content,
  mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner,
  mcp__xtiles__xtiles_create_notification,
  mcp__xtiles__xtiles_get_page_layout,
  mcp__xtiles__xtiles_set_page_layout,
  mcp__xtiles__xtiles_get_workflow,
  mcp__xtiles__xtiles_get_user_timezone,
  mcp__xtiles__xtiles_list_calendar_events,
  mcp__claude_ai_Slack__slack_search_channels,
  mcp__claude_ai_Slack__slack_search_public_and_private,
  mcp__claude_ai_Slack__slack_read_channel,
  mcp__claude_ai_Slack__slack_send_message,
  mcp__claude_ai_Gmail__search_threads,
  mcp__claude_ai_Gmail__get_thread,
  mcp__claude_ai_Google_Calendar__list_events,
  mcp__claude_ai_Granola__list_meetings,
  mcp__claude_ai_Google_Drive__list_recent_files,
  mcp__claude_ai_Linear__list_issues,
  mcp__mcp-registry__suggest_connectors,
  AskUserQuestion,
  anthropic-skills:schedule,
  mcp__scheduled-tasks__create-scheduled-tasks
---

# xTiles Weekly Review

Analyse everything worked on during the week — Daily planner pages, connected
tools, and weekly goals — then write a focused summary to the Weekly page.

## Three principles

1. **Real data only.** Read actual content from the planner and connectors. Never invent accomplishments, decisions, or goal assessments.
2. **Progress over activity.** The point is not a log of everything done, but whether the week moved things forward.
3. **Match the user's language** throughout — match the language of the first message and adapt if they switch. On a **scheduled run** there is no user message: use the language of the scheduled-task config prompt, and if that is ambiguous, the language of the fetched content. Default to English when still unclear. **Every template, label, and example in this file is written in English as a placeholder** — translate them into the detected language, and never emit a label in a language the user has not used.
4. **Every write to the planner is immediately followed by a layout pass — automatically, no exceptions.** The instant tiles are added in step 6, re-lay-out only those new tiles into a justified grid (step 6's layout pass, via the shared `tile-layout` workflow) *before* anything else — before the CTA button, before the schedule widget, before any message to the user. This never waits for the user to ask, is never skipped as "not needed this time," and is never left for a later run.

---

## Algorithm

**Period is always Weekly.**

**Run mode — detect before step 1:**
- **Scheduled run**: the incoming message starts with `Run weekly review —` AND contains `role:`, `tools:`, and `weekly_content:`. Skip the survey AND skip the preview — fetch data and write directly. Extract `notify:` from the config too if present (default `false` for older schedules that predate this setting). After writing, **skip the schedule widget and Slack sharing widget** — the task is already scheduled. Jump to **step 3**.
- **Fast-track**: user asks "run my weekly review" or similar conversational phrase — skip setup, use all available connectors, jump to **step 3**. Preview (step 5) is still required.
- **Manual setup**: general request ("set up weekly review") or setup form submission (message contains "Weekly review setup") — run the full flow. Preview (step 5) is still required.

---

### 1. Fast-track

If the user's request is specific enough to infer intent — skip the setup widget and jump to step 3, pulling from all detected connectors. For Slack — first call `mcp__claude_ai_Slack__slack_search_channels` for universal names (`general`, `all`, `team`, `announcements`, `product`) to collect base channels; then reason from any available context (user role, prior messages) to derive 1–2 phrases reflecting what this person discusses in Slack, and call `mcp__claude_ai_Slack__slack_search_public_and_private` with those phrases to discover additional active channels (public and private). Merge both results and use the top 8 as the channel list.

If the request is general — show the setup widget.

---

### 2. Setup survey

Show the **Setup widget HTML** (see below) in Cowork. In Claude Code, ask the same questions inline.

After receiving answers — detect which MCP tools are actually available:

| Connector    | Identifying MCP tool                                      |
|--------------|-----------------------------------------------------------|
| Slack        | `mcp__claude_ai_Slack__slack_search_channels`             |
| Gmail        | `mcp__claude_ai_Gmail__search_threads`                    |
| Calendar (xTiles) | `mcp__xtiles__xtiles_list_calendar_events`            |
| Calendar     | `mcp__claude_ai_Google_Calendar__list_events`             |
| Granola      | `mcp__claude_ai_Granola__list_meetings`                   |
| Google Drive | `mcp__claude_ai_Google_Drive__list_recent_files`          |
| Linear       | `mcp__claude_ai_Linear__list_issues`                      |

**Calendar (xTiles) is a distinct, optional card — not the same as Calendar.** It aggregates whatever Google/Outlook calendars the user connected inside xTiles itself, via `mcp__xtiles__xtiles_list_calendar_events`. `Calendar` stays the separate, optional card for a Google account connected *directly*, outside xTiles. When both are selected, merge their events for the week and dedup: drop a Calendar-connector event when an xTiles-calendar event already has the same start time and the same title (case-insensitive) — never show the same meeting twice.

**For an "Other" app the user typed** (the free-text field in the widget) — treat it exactly like the known connectors: attempt detection via any available MCP tool for it; if not detected, say its name explicitly ("I'll connect Plaud for you"), walk the user through connecting it via `mcp__mcp-registry__suggest_connectors`, then resume with the full tool list. Never silently drop a custom app the user named — carry it through the fetch (step 3) and analysis.

**If xTiles is not connected** — stop and walk the user through connecting it first (see **How to connect connectors**).

**If a selected connector isn't connected** — walk through connecting it via `mcp__mcp-registry__suggest_connectors`. Wait for confirmation before continuing. **Calendar (xTiles) can't be checked this way** — `xtiles_list_calendar_events` has no auth-error path for "no calendar linked" (an empty result looks identical to a genuinely quiet week), so its selection is taken at face value; the "maybe not linked" ambiguity is resolved later, in step 3, if the merged event list ends up empty.

---

### 3. Silent data fetch

**Silently**, without messaging the user, collect data for the current week (Monday–today):

**xTiles planner — previous week (for comparison):**
Call `mcp__xtiles__xtiles_get_planner_content` with `period: "week"` for last week. Count accomplishments and open items — used for the week-over-week delta in the summary header.

**xTiles planner — Daily pages:**
Call `mcp__xtiles__xtiles_get_planner_content` with `period: "day"` for each day from Monday through today only — do not call for future days. Extract:
- Completed tasks (checked items, done/completed markers)
- Overdue or open tasks still present at end of day
- Notes, saved content, decisions recorded

**xTiles planner — Weekly page:**
Call `mcp__xtiles__xtiles_get_planner_content` with `period: "week"` for the current week. Scan all tiles for any titled with keywords: `Goal`, `Goals`, `Milestone`, `Milestones`, `OKR`, `Target`, `Focus`. If found — extract the goal statements verbatim. These are used in the Goal progress section.

**Connected sources — MANDATORY. Call every detected connector. Daily pages alone are not sufficient.**

For each connector that was selected or detected — call it now, before analysis. Do not skip connectors because Daily pages were already read. Connector data supplements and cross-references Daily page content and is required for Activities analysis.

- **Slack**: `mcp__claude_ai_Slack__slack_read_channel` for each configured channel. After reading, filter to messages with timestamp ≥ Monday 00:00 of the current week — discard older messages. From filtered messages extract: decisions, shipped items, open threads
- **Gmail**: `mcp__claude_ai_Gmail__search_threads` — query `is:important in:inbox after:{Monday-YYYY/MM/DD}` using this week's Monday date (compute from current date) — extract key threads that represent decisions or open actions
- **Calendar**: build one merged event list for the week, then analyse it.
  - **Calendar (xTiles), if selected.** Call `mcp__xtiles__xtiles_list_calendar_events` for Monday through today.
  - **Calendar (the Google Calendar connector), if also selected.** Call `mcp__claude_ai_Google_Calendar__list_events` for the same range and add its events to the same list.
  - **Dedup across the two.** Drop a Calendar-connector event when an xTiles-calendar event already has the same start time and the same title (case-insensitive) — never show the same meeting twice.
  - **If Calendar (xTiles) was selected and contributed zero events for the whole week, don't assume the week was simply quiet.** The tool can't tell "nothing scheduled" from "no calendar linked." If the Calendar connector also contributed nothing (or wasn't selected), note this once in the summary instead of silently showing an empty week. Skip the note if the Calendar connector *did* contribute at least one event.
  - From the merged list: count meetings, identify recurring vs one-off, note people present
- **Granola**: `mcp__claude_ai_Granola__list_meetings` — meeting notes from this week. Extract action items, decisions, and attendees
- **Google Drive**: `mcp__claude_ai_Google_Drive__list_recent_files` — documents created or edited this week
- **Linear**: `mcp__claude_ai_Linear__list_issues` — issues closed, opened, and still in progress this week
- **Custom ("Other") app** the user named: call whatever MCP tool it exposes for this week's activity and extract what's relevant to accomplishments/decisions/open items. If it has no working tool or the call fails, note it — never silently omit an app the user selected.

If a connector call fails — note the failure, continue with remaining data. Do not fabricate data for failed connectors.

---

### 4. Analyse

**Be concise throughout.** Each item is one line. No multi-sentence explanations inside tiles. Dense but scannable.

Synthesise the collected data into **3 tiles**, each with bold `**[Title]**` subheadings inside (add a leading emoji where one fits, as in the reference below) — never `####`/`#####`.

---

**Tile 1 — ✅ Week recap** contains three subheadings:

`**✅ Accomplishments**` — concrete things finished, shipped, or resolved. Include results from Daily pages AND connectors (Slack threads closed, Linear issues shipped, Drive docs published, etc.). First line: WoW delta (`↑ More than last week (N vs M)` / `↓ Less` / `≈ Similar`). Each item: **numbered** (`1.`, `2.`…), one line, source attribution at the end (`— [#channel](url)`, `— [Linear #123](url)`, `— Granola`).

`**🎯 Goals**` — only if a Goals/Milestones tile was found on the weekly page. Each goal as a bullet: `- **[Goal name]** — ✅ Clear progress / 🔄 Some / ⬜ No movement / 🚫 Blocked — one-line assessment`. Omit entire subheading if no Goals tile found.

`**💡 Decisions**` — choices made this week (Granola, Slack, Daily pages). **Numbered** (`1.`, `2.`…), one decision per line + source in parentheses.

---

**Tile 2 — 🔍 Activities** — patterns derived from ALL collected data (Daily pages + every connector). Contains four subheadings:

`**🗂️ Dominant topics**` — group all topics semantically across the week. Identify top 5 by frequency (how many days they appeared) + attention volume. Format per topic as a bullet: `- **[Topic name]** — N days — [one sentence on what happened]`

`**⚡ Activity type**` — classify every action from the week into three types. Output as a **single percentage line, no bullets**: `Initiative 40% · Reactive 45% · Decisions 15%`

`**📊 Productivity pattern**` — most active day, quietest day, morning/afternoon/evening split, any anomalies. Format each observation as a bullet: `- [observation]`

`**👥 Key interactions**` — top 5 people by interaction count this week. Format each as a bullet: `- **[Name]** — N interactions — [topic] — [decision / discussion]`

---

**Tile 3 — → Next week** contains two subheadings:

`**🔄 Open**` — tasks, threads, or decisions not resolved this week. Format: `📌 [What needs to happen] — [source]`

`**Suggested priorities**` — suggested top 3 for next week. **Priority logic:**
1. If a Goals tile was found — derive priorities from goal blockers and next steps toward those goals first
2. Fill remaining slots (or all 3 if no goals) from "Open" by importance
Format: `- [ ] [priority]` — one line each, specific and actionable. No assignee, no due date.

---

If a subheading has no data — omit it entirely rather than writing a filler line.

---

### 5. Preview

**Mandatory for all non-scheduled runs.** Never skip from step 4 directly to step 6.

**Show it with the Preview widget HTML** (see below) via `show_widget` — never
as a plain-text chat dump. Build it fully dynamically: inject all 3 tiles'
real, already-assembled content into `#digest`, following the injection
pattern in the template's comment block — not placeholders. The widget's own
three buttons (**Looks good — save it** / **Change something** / **Cancel**)
*are* the approval step — do not also ask separately.

Do not call the xTiles write tool until the user clicks "Looks good — save it". If the user asks for a change — update only that section, then show the Preview widget again with the revised content — never fall back to a plain-text re-preview.

---

### 6. Write to xTiles

**Only after explicit approval.**

Tool: `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`
- `period`: `"week"`
- `date`: current week's Monday in ISO 8601

Write all sections in a **single call**.

**This skill has no way to update an existing tile's content, so every run creates a fresh set of three tiles.** If the Weekly page already has a `### ✅ Week recap` / `### 🔍 Activities` / `### → Next week` from an earlier run this week or from the user's own saved template, still write all three in the create call below — do not fetch the page to check for matching headings first, and do not skip any of them because a same-titled tile might already exist. A re-run is expected to produce a fresh, current review, not to silently do nothing.

**Tile structure** — every tile must follow this pattern exactly:

```
### [emoji] [Section title]
@colorSize: LIGHTER
@color: [COLOR]

[content]
```

- `@colorSize` is always `LIGHTER`
- `@color` — pick from this list **exactly as written**, different color per tile:
  `GHOST, CUMULUS, GOSSIP, COLDTURKEY, BLUE_CHALK, MILK_PUNCH, HAWKES_BLUE, PATTENS_BLUE, SAIL, ATHENS_GRAY, BERMUDA, PERFUME, SELAGO, RICE_FLOWER, WHITE_LINEN, POLAR`
  **CRITICAL: never use semantic color names (RED, BLUE, GREY, ORANGE, GREEN, etc.) — they will not render. Only the exact names from the list above.**
- Each tile gets a different color — do not repeat the same color twice in a row
- Do not create a date/header tile — start directly with content tiles

**Always write exactly 3 tiles** using bold `**[Title]**` subheadings inside each (leading emoji where it fits) — never `####`/`#####`.

- **Tile 1 — `### ✅ Week recap`**: accomplishments + goal progress + decisions
- **Tile 2 — `### 🔍 Activities`**: dominant topics + activity type % + productivity pattern + key interactions
- **Tile 3 — `### → Next week`**: open items + suggested priorities

**Content formatting — applies to all tiles:**

- **All links must be Markdown hyperlinks** — always `[text](url)`, never a bare URL
- **Separate every item with a blank line** — never a continuous block
- **One line per item** — no multi-sentence explanations inside tiles; be dense and scannable

**Tile 1 — ✅ Week recap:**

`**✅ Accomplishments**` — WoW delta as first line. Each accomplishment: **numbered** (`1.`, `2.`, `3.`), one line + source (`— [#channel](url)`, `— [Linear #123](url)`, `— Granola`). Include results from connectors, not only Daily pages.

`**🎯 Goals**` — only if Goals tile found. Each goal as a bullet: `- **[name]** — [status badge] — one-line assessment`. Omit subheading entirely if no Goals tile.

`**💡 Decisions**` — **numbered** (`1.`, `2.`, `3.`), one decision per line + source in parentheses.

**Tile 2 — 🔍 Activities:**

`**🗂️ Dominant topics**` — top 5 topics, each as a bullet: `- **[Topic]** — N days — [one-sentence summary]`

`**⚡ Activity type**` — single line, no bullets: `Initiative N% · Reactive N% · Decisions N%`

`**📊 Productivity pattern**` — each observation as a bullet: `- [observation about most active day, morning/afternoon/evening split, anomalies]`

`**👥 Key interactions**` — top 5 people, each as a bullet: `- **[Name]** — N interactions — [topic] — [decision / discussion]`

**Tile 3 — → Next week:**

`**🔄 Open**` — each item: `📌 [what needs to happen] — [source]`

`**Suggested priorities**` — top 3, format `- [ ] [priority]`, one line each. Derive from goal blockers first (if Goals tile found), then from "Open".

**Terminal sequence — after a successful write, these come in this exact order, each as its own separate widget/question, none skipped, none merged into another:**
**(a) CTA link → (b) Schedule offer → (c) Slack sharing offer → (d) Related workflows.** Never fold the Slack offer into the schedule step and never propose Slack while asking about scheduling — they are two distinct, consecutive widgets. Losing or reordering any of these four is a failed run. On a scheduled run, stop after the layout pass and the notification (step 5 below) — none of (a)–(d) are shown; nobody is watching chat during an unattended run.

**After a successful write — run these steps in order, no exceptions. Step 2 (the layout pass) is not optional and is never deferred, asked about, or judgment-called away — it runs automatically, immediately after every single write, before step 3's CTA button is even composed:**

1. Write `✅ Weekly Review saved.`
2. **Layout pass — mandatory, silent, automatic, every single run (scheduled runs included, fast-track included, any tile count included).** Using the `view_id` and `tile_ids` returned by the write call above (`tile_ids` is ordered to match the 3 `###` sections you just wrote), apply the shared justified-grid layout rules: call `mcp__xtiles__xtiles_get_workflow` with id `tile-layout` and follow it exactly — treat the tiles in `tile_ids` as its "added tiles" and the markdown you just composed as their content. **Layout hints for this workflow:** always exactly 3 tiles (Week recap, Activities, Next week) in that order · try all 3 in one row if they fit, else the heaviest tile (usually Activities) takes more width or its own row. Do not message the user about this pass, do not ask for confirmation, and never skip it — not even for a single tile or a scheduled run. (You may fetch `tile-layout` once per session and reuse it on later runs.)
3. **For non-scheduled runs only: link to the first tile (`### ✅ Week recap`), not the page**, so the user lands right on the review. (On a scheduled run, skip this widget entirely — step 5 below reaches the user instead.)
   - Use the `resource_url` of the **first** entry in the create call's response `tiles` array (a deep link that opens the Weekly page focused on that tile) as `{VIEW_URL}` directly.
   - **Only if that entry has no `resource_url` at all** — fall back to `https://xtiles.app/{view_id}`, also read from the create call's response.
   Call `show_widget` with the **CTA widget HTML** (see below) using that `{VIEW_URL}`. Translate the button label into the user's language. **Never leave `{VIEW_URL}` unresolved and never output a markdown link instead of the widget — the button must render on every non-scheduled run.**
4. **For non-scheduled runs only**: Immediately call `show_widget` with the **Schedule widget HTML** (see below). Do not skip, do not ask first, and never substitute `AskUserQuestion` here — this is a Cowork chat widget, not a question.
5. **Scheduled runs only, and only if the config's `notify:` flag is `true`.** The user opted into `mcp__xtiles__xtiles_create_notification` instead of a widget when they scheduled this (see step 7's notification toggle). If `notify:false` (or missing, for an older schedule) — skip this step entirely, silently, no notification:
   - `url`: the **page** URL, `https://xtiles.app/{view_id}` — a page link, never the tile-specific `{VIEW_URL}` from step 3 (which this step skips anyway).
   - `text`: **always the fixed string `"Your Weekly Review is ready — recap your week in 2 min."`, translated into the user's language — no customized/dynamic part.** Do not append a highlight, a count, or any detail pulled from this week's content — this text never changes run to run beyond translation.
   - `agent_source`: `"Claude"`.

**If an error occurs:** say what went wrong and offer to retry.

---

### 7. Schedule (optional)

The schedule widget was shown as the final step of the write flow above (the
Schedule widget, right after the CTA link) — this section handles the response.
**This is the schedule offer only; do not mention or offer Slack here** — Slack
sharing is the separate step 8 that comes next.

In Claude Code: after writing, ask inline: "Want me to run this automatically every Friday at 4 PM?"

- If **"Yes, schedule it"**: the widget response also carries `notify:true`/`notify:false` from its notification toggle. Invoke `anthropic-skills:schedule`, then `mcp__scheduled-tasks__create-scheduled-tasks`. Pass to both:
  - `prompt`: `Run weekly review — role: {role} · tools: {tools} · weekly_content: {content} · notify: {true/false} (if true, call xtiles_create_notification at the very end of this scheduled run — mandatory, do not skip) · schedule: weekly-friday-4pm`  
    Replace every placeholder with real values, `notify` straight from the widget.
  - `schedule`: cron from widget — default `0 16 * * 5` (Friday 4 PM). Widget sends a pre-built cron expression in the message (e.g. `cron: 00 16 * * 5`) — extract and use it directly as the `schedule` value.
  - `timezone`: from `mcp__xtiles__xtiles_get_user_timezone`

  **If `notify:true`** — the review that's already on the page right now (from step 6's write) is also worth notifying about; don't make the user wait for next Friday to see it work. Call `mcp__xtiles__xtiles_create_notification` immediately, right here: `url` is the page URL (`https://xtiles.app/{view_id}`), `text` is the fixed notification string per step 6.5 above, `agent_source` is `"Claude"`.

  After scheduling: confirm "Done — your Weekly Review will run every Friday at 4 PM." (append ", and I'll notify you in xTiles each time" if `notify:true`). **Do not show the CTA widget again here** — it was already shown once, right after the write in step 6; repeating it after the schedule confirmation is redundant. **This confirmation is not the end of the run — immediately continue to step 8 (Slack sharing) in the same turn.**

- If **"No, thanks"** — acknowledge briefly, then **immediately continue to step 8 (Slack sharing) in the same turn.**

---

### 8. Slack sharing (optional)

After the schedule widget response — call `show_widget` with the **Slack sharing widget HTML** (see below).

**If "Yes, share to Slack":**
1. If Slack channels were configured during setup — use those. Otherwise call `mcp__claude_ai_Slack__slack_search_channels` with query `general` to find main team channels, then call `show_widget` with a channel-picker HTML (same pattern as the channel picker in daily-brief).
2. Compose a concise Slack status message from the review — 3–5 bullet points max, plain text, no markdown tiles. Format:
   ```
   📋 Week of [dates] — status update

   ✅ [top 2–3 accomplishments, one line each]
   🔄 [1–2 open items]
   → Next week: [top priority]
   ```
3. Show the message preview in chat and call `show_widget` with a confirm/cancel widget (2 buttons: "Send" / "Cancel") before sending.
4. Call `mcp__claude_ai_Slack__slack_send_message` with the channel and message.
5. Confirm: "Sent to #[channel]." **This confirmation is not the end of the run — immediately continue to step 9 (Related workflows) in the same turn.**

**If "No, keep it personal"** — acknowledge briefly, then **immediately continue to step 9 (Related workflows) in the same turn.**

**Either way, step 9 still has to run before this turn ends — do not wait for the user to ask.**

---

### 9. Related workflows

**After every manual run, once step 8 is resolved** (shared or kept personal)
— offer related workflows. Skip this on scheduled runs, which end after step
6's notification — no widgets, nobody to ask.

Ask via `AskUserQuestion` (single select): "Want to set up anything else on
xTiles?"
- 🌅 Daily Brief — a live morning brief from your connected tools
- 🌙 Evening Reflection — an end-of-day synthesis seeded for tomorrow
- 📰 Today News — a daily news digest on topics you care about
- Nothing else, thanks

**Never list these as plain text requiring the user to retype a choice —
always use the interactive question.**

On selection, send the exact matching phrase to hand off to that skill (do
not attempt to run it yourself):
- Daily Brief → `Set workflow of Daily Brief (daily-brief) on xTiles MCP`
- Evening Reflection → `Set workflow of Evening Reflection (evening-reflection) on xTiles MCP`
- Today News → `Set workflow of Today News (today-news) on xTiles MCP`
- "Nothing else" — acknowledge briefly and stop.

---

## How to connect connectors

Call `mcp__mcp-registry__suggest_connectors` passing the names of missing connectors. Show the **Done widget HTML** immediately after. When the user clicks Done — confirm "Connected. Continuing…" and resume.

---

## Review tile format (reference)

Use the example below as the ground-truth formatting reference — match it character for character (blank lines, bold subheadings, `@colorSize`, `@color`, emoji placement, source attribution).

```markdown
### ✅ Week recap
@colorSize: LIGHTER
@color: GOSSIP

**✅ Accomplishments**

↑ More than last week (5 vs 3)

1. Shipped affiliate integration with Impact — [Linear #312](https://linear.app/issues/312)

2. Closed 3 auth sprint issues, E2E tests passing — [#eng-core](https://slack.com/...)

3. Aligned with Andrew on Q3 OKRs — Granola

4. Submitted MCP plugin draft to marketplace — Google Drive

**🎯 Goals**

- **Launch MCP plugin to marketplace** — ✅ Clear progress — draft submitted Monday, review pending since Wednesday

- **Grow to 10k DAU** — ⬜ No movement — no experiments ran; deprioritised for plugin launch

- **Reduce support tickets by 20%** — 🔄 Some progress — new FAQ live; tickets down ~8%, trend positive

**💡 Decisions**

1. PartnerStack postponed — Impact affiliate only for July (Slack — [#growth](https://slack.com/...))

2. Daily Brief prioritised over new connectors this sprint (Granola — [Plugin Sync Jun 24](https://granola.ai/...))

3. Influencer strategy: macro → micro, budget reallocated (Slack — [#marketing](https://slack.com/...))

### 🔍 Activities
@colorSize: LIGHTER
@color: BLUE_CHALK

**🗂️ Dominant topics**

- **Affiliate & partnerships** — 4 days — Impact negotiations, EchoMe integration closing

- **Plugin launch** — 3 days — marketplace prep, Daily Brief fixes

- **Q3 planning** — 3 days — OKR syncs, budget, resources

- **Influencer strategy** — 2 days — macro → micro shift, Sara Aratake brief

- **Auth sprint** — 2 days — 3 issues closed, E2E passing

**⚡ Activity type**

Initiative 38% · Reactive 47% · Decisions 15%

**📊 Productivity pattern**

- Most active: Tuesday. Quietest: Friday (2 meetings, little async).

- Peak hours 10:00–13:00. Anomaly: Monday — 3 back-to-back calls after 18:00.

**👥 Key interactions**

- **Andrew** — 6 interactions — Q3 OKRs — decision (OKRs approved)

- **Todd Savard** — 4 interactions — EchoMe integration — discussion (no decision yet)

- **Sara Aratake** — 3 interactions — influencer brief — discussion

- **Alex** — 2 interactions — Q3 budget — awaiting confirmation

- **Maria (Design)** — 2 interactions — plugin UI — decision (mockup approved)

### → Next week
@colorSize: LIGHTER
@color: HAWKES_BLUE

**🔄 Open**

📌 Todd Savard — EchoMe follow-up, waiting on their tech team — [#partnerships](https://slack.com/...)

📌 Sara Aratake — influencer brief due Monday — [Linear #341](https://linear.app/issues/341)

📌 Q3 budget — Alex to confirm by EOW — Granola

**Suggested priorities**

- [ ] Merge review-fixes branch and send for review — blocks marketplace submission (goal)

- [ ] Send EchoMe follow-up to Todd Savard — move or close by Wednesday

- [ ] Finalise Sara Aratake influencer brief before Monday standup
```

---

## Setup widget HTML

```html
<style>
:root{--bg:#f8f8f8;--card:#fff;--text:#1a1a1a;--muted:#888;--line:#e0e0e0;--soft:#f1f1f1}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:var(--bg);color:var(--text)}
.wrap{max-width:540px;margin:0 auto;background:var(--card);border-radius:16px;padding:28px;box-shadow:0 2px 12px rgba(0,0,0,.08)}
h2{font-size:18px;font-weight:700;margin-bottom:4px}
.sub{font-size:13px;color:var(--muted);margin-bottom:22px;line-height:1.5}
.sec{margin-bottom:20px}
.sec-title{font-size:13px;font-weight:600;margin-bottom:10px}
.cards{display:flex;flex-wrap:wrap;gap:8px}
.card{display:flex;align-items:center;gap:7px;padding:8px 13px;border-radius:10px;border:1.5px solid var(--line);font-size:13px;cursor:pointer;background:var(--card);user-select:none;transition:all .15s}
.card:hover{border-color:#aaa}
.card.sel{background:var(--text);color:var(--card);border-color:var(--text)}
.chk{width:15px;height:15px;border-radius:4px;border:1.5px solid #aaa;display:flex;align-items:center;justify-content:center;font-size:9px;flex-shrink:0}
.card.sel .chk{background:var(--card);border-color:var(--card);color:var(--text)}
.pills{display:flex;flex-wrap:wrap;gap:8px}
.pill{padding:7px 14px;border-radius:20px;border:1.5px solid var(--line);font-size:13px;cursor:pointer;background:var(--card);user-select:none;transition:all .15s}
.pill:hover{border-color:#aaa}
.pill.sel{background:var(--text);color:var(--card);border-color:var(--text)}
.btn-row{display:flex;gap:10px;margin-top:24px}
.btn{padding:10px 18px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:all .15s}
.btn-p{background:var(--text);color:var(--card);flex:1}
.btn-p:hover{opacity:.9}
.btn-s{background:var(--soft);color:#555}
.btn-s:hover{background:#e0e0e0}
</style>
<div class="wrap">
  <h2>Weekly Review</h2>
  <p class="sub">I'll read your week from the planner and connected tools, then write a summary to your Weekly page.</p>

  <div class="sec">
    <div class="sec-title">Which tools should I pull from?</div>
    <div class="cards" id="tool-cards">
      <div class="card" onclick="tog(this,'Slack')"><div class="chk">✓</div>Slack</div>
      <div class="card" onclick="tog(this,'Gmail')"><div class="chk">✓</div>Gmail</div>
      <div class="card" onclick="tog(this,'CalendarXTiles')"><div class="chk">✓</div>Calendar (xTiles)</div>
      <div class="card" onclick="tog(this,'Calendar')"><div class="chk">✓</div>Calendar</div>
      <div class="card" onclick="tog(this,'Granola')"><div class="chk">✓</div>Granola</div>
      <div class="card" onclick="tog(this,'Linear')"><div class="chk">✓</div>Linear</div>
      <div class="card" onclick="tog(this,'GoogleDrive')"><div class="chk">✓</div>Google Drive</div>
    </div>
    <input type="text" id="other-tool" placeholder="Other app (e.g. Plaud, Notion)…" style="width:100%;margin-top:9px;padding:8px 12px;border:1.5px solid var(--line);border-radius:8px;font-size:13px;outline:none;background:var(--card);color:var(--text)">
  </div>

  <div class="btn-row">
    <button class="btn btn-s" id="btn-cancel" onclick="cancelIt()">Cancel</button>
    <button class="btn btn-p" onclick="submit()">Set up Weekly Review</button>
  </div>
</div>
<script>
var tools=new Set();
function lock(){document.querySelectorAll('.btn').forEach(function(b){b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';});}
function tog(el,v){el.classList.toggle('sel');el.classList.contains('sel')?tools.add(v):tools.delete(v);}
function submit(){
  lock();
  var oth=document.getElementById('other-tool').value.trim();
  if(oth)tools.add(oth);
  var t=Array.from(tools).join(', ')||'none';
  sendPrompt('Weekly review setup — tools: '+t+' · weekly_content: accomplishments, goal-progress, open-items, decisions');
}
function cancelIt(){lock();document.getElementById('btn-cancel').textContent='✓ Cancelled';sendPrompt('Cancel weekly review setup');}
</script>
```

---

## CTA widget HTML

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
html,body{overflow:hidden;height:auto}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:12px;background:transparent}
.btn{display:block;width:100%;padding:12px 20px;border-radius:10px;font-size:15px;font-weight:700;color:#fff;background:#1a1a1a;text-align:center;text-decoration:none;transition:background .15s}
.btn:hover{background:#333}
</style>
<a class="btn" href="{VIEW_URL}" target="_blank">Open Weekly Review →</a>
```

---

## Schedule widget HTML

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:#f8f8f8;color:#1a1a1a}
.wrap{max-width:480px;margin:0 auto;background:#fff;border-radius:16px;padding:28px;box-shadow:0 2px 12px rgba(0,0,0,.08);text-align:center}
.icon{font-size:32px;margin-bottom:12px}
h2{font-size:17px;font-weight:700;margin-bottom:6px}
.sub{font-size:13px;color:#888;margin-bottom:20px;line-height:1.5}
.time-row{display:inline-flex;align-items:center;gap:8px;background:#f3f3f3;border-radius:10px;padding:8px 16px;font-size:13px;font-weight:600;color:#444;margin-bottom:16px;flex-wrap:wrap;justify-content:center}
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
  <div class="icon">📋</div>
  <h2>Run this every week?</h2>
  <p class="sub">I'll read your week and write the review to xTiles automatically.</p>
  <div class="time-row">
    Every
    <select id="sched-day">
      <option value="5" selected>Friday</option>
      <option value="4">Thursday</option>
      <option value="1">Monday</option>
    </select>
    at <input type="time" id="sched-time" value="16:00">
  </div>
  <div class="notify-row" onclick="togNotify()">
    <div class="notify-info">
      <div class="notify-label">🔔 Notify me in xTiles</div>
      <div class="notify-desc">Get pinged the moment each week's review is ready</div>
    </div>
    <div class="sw on" id="notify-sw"><div class="knob"></div></div>
  </div>
  <div class="btns">
    <button class="btn btn-yes" id="btn-yes" onclick="scheduleIt()">Yes, schedule it</button>
    <button class="btn btn-no" id="btn-no" onclick="noThanks()">No, thanks</button>
  </div>
</div>
<script>
var DAYS={1:'Monday',4:'Thursday',5:'Friday'};
var notify=true;
function togNotify(){notify=!notify;document.getElementById('notify-sw').classList.toggle('on',notify);}
function collapse(msg){document.querySelector('.btns').innerHTML='<p style="font-size:13px;color:#aaa;text-align:center;padding:4px 0">'+msg+'</p>';}
function scheduleIt(){
  var d=document.getElementById('sched-day').value;
  var t=document.getElementById('sched-time').value||'16:00';
  var parts=t.split(':'),h=parseInt(parts[0],10),m=parts[1];
  var label=(h%12||12)+':'+m+' '+(h>=12?'PM':'AM');
  collapse('⏳ Scheduling…');
  sendPrompt('Yes, schedule my weekly review every '+DAYS[d]+' at '+label+' (cron: '+m+' '+h+' * * '+d+') · notify:'+notify);
}
function noThanks(){collapse('✓ Got it');sendPrompt('No schedule needed');}
</script>
```

---

## Preview widget HTML

Show via `show_widget` in step 5, for every non-scheduled run. Build fully
dynamically: inject all 3 tiles' real, already-assembled content into
`#digest`, following the injection pattern in the comment block below. Omit
any subsection whose content doesn't exist this week (Goals) — don't inject
an empty block.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:16px;background:#f8f8f8;color:#1a1a1a}
.wrap{max-width:560px;margin:0 auto;background:#fff;border-radius:16px;padding:24px;box-shadow:0 2px 12px rgba(0,0,0,.08)}
.header{display:flex;align-items:center;justify-content:space-between;margin-bottom:16px}
h2{font-size:16px;font-weight:700}
.cnt{font-size:12px;color:#888;background:#f5f5f5;padding:2px 9px;border-radius:20px}
.scroll{max-height:420px;overflow-y:auto;margin-bottom:16px;display:flex;flex-direction:column;gap:12px}
.section{border-radius:12px;padding:14px;background:#f7f7f7}
.section-head{font-size:12px;font-weight:700;color:#1a1a1a;margin-bottom:10px;padding-bottom:8px;border-bottom:1px solid rgba(0,0,0,.07)}
.sub-head{font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.4px;color:#888;margin:10px 0 6px}
.sub-head:first-of-type{margin-top:0}
.item{font-size:12px;color:#333;line-height:1.5;margin-bottom:6px}
.item:last-child{margin-bottom:0}
.item-title{font-weight:600;color:#1a1a1a}
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
    <h2>📊 Weekly Review</h2>
    <span class="cnt" id="cnt">[week of ...]</span>
  </div>
  <div class="scroll" id="digest">

    <!--
    INJECTION PATTERN — Claude fills in #digest with the real, already-assembled review:

    <div class="section">
      <div class="section-head">✅ Week recap</div>
      <div class="sub-head">Accomplishments</div>
      <div class="item">↑ More than last week (5 vs 3)</div>
      <div class="item"><span class="item-title">1.</span> Shipped affiliate integration — Linear #312</div>
      Goals — only if a Goals/Milestones tile was found, omit otherwise:
      <div class="sub-head">Goals</div>
      <div class="item"><span class="item-title">Launch MCP plugin</span> — ✅ Clear progress — draft submitted, review pending</div>
      <div class="sub-head">Decisions</div>
      <div class="item">1. PartnerStack postponed — Slack #growth</div>
    </div>

    <div class="section">
      <div class="section-head">🔍 Activities</div>
      <div class="sub-head">Dominant topics</div>
      <div class="item"><span class="item-title">Affiliate & partnerships</span> — 4 days — Impact negotiations</div>
      <div class="sub-head">Activity type</div>
      <div class="item">Initiative 38% · Reactive 47% · Decisions 15%</div>
      <div class="sub-head">Productivity pattern</div>
      <div class="item">Most active: Tuesday. Quietest: Friday.</div>
      <div class="sub-head">Key interactions</div>
      <div class="item"><span class="item-title">Andrew</span> — 6 interactions — Q3 OKRs — decision</div>
    </div>

    <div class="section">
      <div class="section-head">→ Next week</div>
      <div class="sub-head">Open</div>
      <div class="item">📌 Todd Savard — EchoMe follow-up — #partnerships</div>
      <div class="sub-head">Suggested priorities</div>
      <div class="item">☐ Merge review-fixes branch — blocks marketplace submission</div>
    </div>
    -->

  </div>
  <div class="feedback" id="fb">
    <textarea id="fbtext" placeholder="What should I change?"></textarea>
  </div>
  <div class="btns">
    <button class="btn btn-save" id="btn-save" onclick="doSave()">Looks good — save it</button>
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
  sendPrompt(fb?'Apply this change then save: '+fb:'Looks good — save it');
}
function cancelIt(){collapse('✓ Cancelled');sendPrompt('Cancel');}
function toggleEdit(){
  editing=!editing;
  document.getElementById('fb').style.display=editing?'block':'none';
  document.getElementById('btn-save').textContent=editing?'Apply & Save':'Looks good — save it';
  document.getElementById('btn-edit').textContent=editing?'Never mind':'Change something';
}
</script>
```

---

## Slack sharing widget HTML

Show via `show_widget` after the schedule widget response in step 8.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:16px;background:transparent}
.wrap{max-width:400px;margin:0 auto}
p{font-size:14px;color:#555;margin-bottom:12px;line-height:1.4}
.btns{display:flex;flex-direction:column;gap:8px}
.btn{width:100%;padding:11px 20px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:background .15s}
.btn-yes{background:#1a1a1a;color:#fff}
.btn-yes:hover{background:#333}
.btn-no{background:#f0f0f0;color:#555}
.btn-no:hover{background:#e0e0e0}
</style>
<div class="wrap">
  <p>Share this week's summary as a status update to your team?</p>
  <div class="btns">
    <button class="btn btn-yes" id="btn-yes" onclick="share()">Share to Slack</button>
    <button class="btn btn-no" id="btn-no" onclick="noThanks()">No, keep it personal</button>
  </div>
</div>
<script>
function collapse(msg){document.querySelector('.btns').innerHTML='<p style="font-size:13px;color:#aaa;text-align:center;padding:4px 0">'+msg+'</p>';}
function share(){collapse('⏳ Sharing…');sendPrompt('Yes, share to Slack');}
function noThanks(){collapse('✓ Got it');sendPrompt('No, keep it personal');}
</script>
```

---

## Done widget HTML

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

## How to behave

- Never create anything without preview and explicit approval, except on scheduled runs
- Never invent accomplishments, decisions, or goal assessments — only real data from the planner and connectors
- If a connector call fails — skip it, note it in the preview, do not fabricate data
- If the weekly page has no Goals tile — omit the Goal progress section entirely, do not invent goals
- All links must be Markdown hyperlinks — `[text](url)`, never a bare URL
- Match the user's language; adapt if they switch
- Show the setup widget in Cowork only — in Claude Code, ask the same questions inline
- **In Cowork, every terminal-sequence step (CTA, Schedule offer, Slack sharing offer) is a `show_widget` HTML widget, never `AskUserQuestion` and never plain text** — `AskUserQuestion` is for step 9 (Related workflows) only
