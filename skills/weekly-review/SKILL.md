---
name: weekly-review
description: >
  Run a weekly review — analyse what was worked on this week and write a summary
  to the xTiles Weekly planner page. Reads Daily pages, connected tools, and any
  Goals or Milestones tile on the weekly page to assess progress.
  Setup triggers: "set up weekly review", "run every Friday", "weekly summary".
  Run triggers: "run my weekly review", "what did I do this week",
  "weekly recap", "show me my week".
allowed-tools: >
  mcp__xtiles__xtiles_get_planner_content,
  mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner,
  mcp__xtiles__xtiles_get_user_timezone,
  mcp__claude_ai_Slack__slack_search_channels,
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
3. **Match the user's language** throughout — match the language of the first message and adapt if they switch.

---

## Algorithm

**Period is always Weekly.**

**Run mode — detect before step 1:**
- **Scheduled run**: the incoming message contains `role:`, `tools:`, and `weekly_content:`. Skip the survey. Extract config, fetch data, write the review. Jump to **step 3**.
- **Fast-track**: user asks "run my weekly review" or similar — skip setup, use all available connectors, jump to **step 3**.
- **Manual setup**: general request ("set up weekly review") — run the full flow.

---

### 1. Fast-track

If the user's request is specific enough to infer intent — skip the setup widget and jump to step 3, pulling from all detected connectors.

If the request is general — show the setup widget.

---

### 2. Setup survey

Show the **Setup widget HTML** (see below) in Cowork. In Claude Code, ask the same questions inline.

After receiving answers — detect which MCP tools are actually available:

| Connector    | Identifying MCP tool                                      |
|--------------|-----------------------------------------------------------|
| Slack        | `mcp__claude_ai_Slack__slack_search_channels`             |
| Gmail        | `mcp__claude_ai_Gmail__search_threads`                    |
| Calendar     | `mcp__claude_ai_Google_Calendar__list_events`             |
| Granola      | `mcp__claude_ai_Granola__list_meetings`                   |
| Google Drive | `mcp__claude_ai_Google_Drive__list_recent_files`          |
| Linear       | `mcp__claude_ai_Linear__list_issues`                      |

**If xTiles is not connected** — stop and walk the user through connecting it first (see **How to connect connectors**).

**If a selected connector isn't connected** — walk through connecting it via `mcp__mcp-registry__suggest_connectors`. Wait for confirmation before continuing.

---

### 3. Silent data fetch

**Silently**, without messaging the user, collect data for the current week (Monday–today):

**xTiles planner — previous week (for comparison):**
Call `mcp__xtiles__xtiles_get_planner_content` with `period: "week"` for last week. Count accomplishments and open items — used for the week-over-week delta in the summary header.

**xTiles planner — Daily pages:**
Call `mcp__xtiles__xtiles_get_planner_content` with `period: "day"` for each day of the current week. Extract:
- Completed tasks (checked items, done/completed markers)
- Overdue or open tasks still present at end of day
- Notes, saved content, decisions recorded

**xTiles planner — Weekly page:**
Call `mcp__xtiles__xtiles_get_planner_content` with `period: "week"` for the current week. Scan all tiles for any titled with keywords: `Goal`, `Goals`, `Milestone`, `Milestones`, `OKR`, `Target`, `Focus`. If found — extract the goal statements verbatim. These are used in the Goal progress section.

**Connected sources** (only those selected / detected):
- **Slack**: `mcp__claude_ai_Slack__slack_read_channel` for each configured channel — messages this week, filter for decisions, shipped items, open threads
- **Gmail**: `mcp__claude_ai_Gmail__search_threads` — query `is:important newer_than:7d` — extract key threads that represent decisions or open actions
- **Calendar**: `mcp__claude_ai_Google_Calendar__list_events` — this week's events. Count meetings, identify recurring vs one-off, flag any unresolved follow-ups
- **Granola**: `mcp__claude_ai_Granola__list_meetings` — meeting notes from this week. Extract action items and decisions
- **Google Drive**: `mcp__claude_ai_Google_Drive__list_recent_files` — documents created or edited this week
- **Linear**: `mcp__claude_ai_Linear__list_issues` — issues closed, opened, and still in progress this week

If a connector call fails — note the failure, continue with remaining data. Do not fabricate data for failed connectors.

---

### 4. Analyse

Synthesise the collected data into **2 tiles**, each with `####` subheadings inside.

**Tile 1 — ✅ Week recap** contains three subheadings:

`#### ✅ Accomplishments` — concrete things finished, shipped, or resolved this week. Prefer specifics over generics. First line: week-over-week delta (`↑ More than last week (N vs M)` / `↓ Less than last week` / `≈ Similar`). Omit delta if no previous week data.

`#### 🎯 Goals` — only include if a Goals / Milestones tile was found on the weekly page. For each goal: assess movement (✅ Clear progress / 🔄 Some progress / ⬜ No movement / 🚫 Blocked). Be honest — "no movement" is a valid and useful output. Omit the entire subheading if no Goals tile was found.

`#### 💡 Decisions` — choices made this week worth recording (from Granola notes, Slack threads, Daily page notes).

**Tile 2 — → Next week** contains two subheadings:

`#### 🔄 Still open` — tasks, threads, or decisions that started this week but aren't resolved.

`#### Priorities` — top 3 most important items from "Still open" as numbered priorities. Be specific: what exactly needs to happen, not just a topic name.

If a subheading has no data — omit it entirely rather than writing a filler line.

---

### 5. Preview

Show the assembled review in chat. Then ask (single select via `AskUserQuestion`):
- "Looks good — save it"
- "Change something"
- "Cancel"

If the user asks for a change — update only that section, re-show preview, ask again.

---

### 6. Write to xTiles

**Only after explicit approval.**

Tool: `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`
- `period`: `"week"`
- `date`: current week's Monday in ISO 8601

Write all sections in a **single call**.

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

**Always write exactly 2 tiles.** All 5 sections fit inside them using `####` subheadings — the same pattern as the Slack tile in daily-brief.

- **Tile 1 — `### ✅ Week recap`**: accomplishments + goal progress + key decisions
- **Tile 2 — `### → Next week`**: still open + numbered priorities

**Content formatting inside each tile:**

- **All links must be Markdown hyperlinks** — always `[text](url)`, never a bare URL
- **Separate every item with a blank line** — never write items as a continuous block

**Tile 1 — ✅ Week recap:**

`#### ✅ Accomplishments` subheading first:
- First line: week-over-week delta — `↑ More than last week (N vs M)` / `↓ Less than last week (N vs M)` / `≈ Similar to last week`. Omit if no previous week data.
- Then a blank line, then each accomplishment as its own paragraph (blank line between each)
- Attribute the source when available: `— [Linear #123](url)`, `— [#channel](url)`, `— Granola`, `— Google Drive`
- Prefer specifics: "Shipped X" beats "Worked on X"

`#### 🎯 Goals` subheading next (only if a Goals/Milestones tile was found on the weekly page — omit the subheading entirely if not):
- Each goal on its own block separated by a blank line
- Format: `**[Goal name]** — ✅ Clear progress` / `🔄 Some progress` / `⬜ No movement` / `🚫 Blocked`
- One-line honest assessment on the next line
- Never invent a goal — only goals found verbatim on the weekly page

`#### 💡 Decisions` subheading last:
- Each decision on its own paragraph (blank line between each)
- Decision statement + source in parentheses on the same line
- Source format: `(Granola — [Meeting name](url))` / `(Slack — [#channel](url))`

**Tile 2 — → Next week:**

`#### 🔄 Still open` subheading first:
- Each item on its own paragraph (blank line between each)
- Format: `📌 [What still needs to happen] — [source or owner if known]`
- Link to the source if available: `— [#channel](url)`, `— [Linear #456](url)`

`#### Priorities` subheading next:
- Three numbered priorities extracted directly from "Still open"
- Each on its own paragraph (blank line between): `1.`, `2.`, `3.`
- Be specific — what exactly needs to happen, not just a topic name

**After a successful write — run these steps in order:**

1. Write `✅ Weekly Review saved.`
2. Call `mcp__xtiles__xtiles_get_planner_content` with `period: "week"` and this week's date. Extract `view_id`. Call `show_widget` with the **CTA widget HTML** (see below), replacing `{VIEW_URL}` with `https://xtiles.app/{view_id}`. Translate the button label into the user's language. Never output a markdown link instead of the widget.
3. Immediately call `show_widget` with the **Schedule widget HTML** (see below). Do not skip.

**If an error occurs:** say what went wrong and offer to retry.

---

### 7. Schedule (optional)

The schedule widget is shown in step 6 above. This step handles the response.

In Claude Code: after writing, ask inline: "Want me to run this automatically every Friday at 4 PM?"

- If **"Yes, schedule it"**: invoke `anthropic-skills:schedule`, then `mcp__scheduled-tasks__create-scheduled-tasks`. Pass to both:
  - `prompt`: `Run weekly review — role: {role} · tools: {tools} · weekly_content: {content} · schedule: weekly-friday-4pm`  
    Replace every placeholder with real values.
  - `schedule`: cron from widget — default `0 16 * * 5` (Friday 4 PM). Widget sends `cron: HH:MM weekday` — parse and build accordingly.
  - `timezone`: from `mcp__xtiles__xtiles_get_user_timezone`

  After scheduling: confirm "Done — your Weekly Review will run every Friday at 4 PM." Then call `show_widget` with the **CTA widget HTML** again using the same `view_id`.

- If **"No, thanks"** — acknowledge briefly and stop.

---

### 8. Slack sharing (optional)

After the schedule widget response — ask via `AskUserQuestion` (single select):

**"Want to share this as a status update to your team?"**
- "Yes, share to Slack"
- "No, keep it personal"

**If "Yes, share to Slack":**
1. If Slack channels were configured during setup — use those. Otherwise call `mcp__claude_ai_Slack__slack_search_channels` and ask the user to pick a channel via `AskUserQuestion`.
2. Compose a concise Slack status message from the review — 3–5 bullet points max, plain text, no markdown tiles. Format:
   ```
   📋 Week of [dates] — status update

   ✅ [top 2–3 accomplishments, one line each]
   🔄 [1–2 open items]
   → Next week: [top priority]
   ```
3. Show the message preview and ask for confirmation before sending.
4. Call `mcp__claude_ai_Slack__slack_send_message` with the channel and message.
5. Confirm: "Sent to #[channel]."

**If "No, keep it personal"** — acknowledge and stop.

---

## How to connect connectors

Call `mcp__mcp-registry__suggest_connectors` passing the names of missing connectors. Show the **Done widget HTML** immediately after. When the user clicks Done — confirm "Connected. Continuing…" and resume.

---

## Review tile format (reference)

```markdown
### ✅ Week recap
@colorSize: LIGHTER
@color: GOSSIP

#### ✅ Accomplishments

↑ More than last week (5 vs 3)

Shipped the affiliate integration with Impact — [Linear #312](https://linear.app/issues/312)

Closed 3 auth sprint issues, auth flow now passing all E2E tests — [#eng-core](https://slack.com/...)

Aligned with Andrew on Q3 growth priorities; OKRs locked — Granola

Submitted MCP plugin draft to marketplace review — Google Drive

#### 🎯 Goals

**Launch MCP plugin to marketplace** — ✅ Clear progress
Draft submitted Monday, marketplace review pending since Wednesday

**Grow to 10k DAU** — ⬜ No movement
No growth experiments ran this week; deprioritised in favour of plugin launch

**Reduce support ticket volume by 20%** — 🔄 Some progress
New FAQ page live; ticket volume down ~8% vs last week, trend positive

#### 💡 Decisions

PartnerStack postponed — moving forward with Impact only for July launch (Slack — [#growth](https://slack.com/...))

Daily Brief plugin prioritised over new connectors this sprint (Granola — [Plugin Sync Jun 24](https://granola.ai/...))

Influencer strategy shifted from macro to micro — budget reallocated (Slack — [#marketing](https://slack.com/...))

### → Next week
@colorSize: LIGHTER
@color: HAWKES_BLUE

#### 🔄 Still open

📌 Todd Savard follow-up on EchoMe integration — waiting on their tech team — [#partnerships](https://slack.com/...)

📌 Sara Aratake influencer brief — needs to go out before Monday — [Linear #341](https://linear.app/issues/341)

📌 Q3 budget approval from finance — Alex to confirm by EOW — Granola

#### Priorities

1. Send EchoMe integration follow-up to Todd Savard — move or close by Wednesday

2. Finalise and send Sara Aratake influencer brief before Monday standup

3. Chase Alex for Q3 budget sign-off — block calendar for finance sync
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
      <div class="card" onclick="tog(this,'Calendar')"><div class="chk">✓</div>Calendar</div>
      <div class="card" onclick="tog(this,'Granola')"><div class="chk">✓</div>Granola</div>
      <div class="card" onclick="tog(this,'Linear')"><div class="chk">✓</div>Linear</div>
      <div class="card" onclick="tog(this,'GoogleDrive')"><div class="chk">✓</div>Google Drive</div>
    </div>
  </div>

  <div class="sec">
    <div class="sec-title">Format</div>
    <div class="pills" id="fmt">
      <div class="pill sel" onclick="pickFmt(this,'status')">Status update</div>
      <div class="pill" onclick="pickFmt(this,'journal')">Personal journal</div>
    </div>
  </div>

  <div class="btn-row">
    <button class="btn btn-s" onclick="sendPrompt('Cancel weekly review setup')">Cancel</button>
    <button class="btn btn-p" onclick="submit()">Set up Weekly Review</button>
  </div>
</div>
<script>
var tools=new Set(), fmt='status';
function tog(el,v){el.classList.toggle('sel');el.classList.contains('sel')?tools.add(v):tools.delete(v);}
function pickFmt(el,v){document.querySelectorAll('#fmt .pill').forEach(function(p){p.classList.remove('sel')});el.classList.add('sel');fmt=v;}
function submit(){
  var t=Array.from(tools).join(', ')||'none';
  sendPrompt('Weekly review setup — tools: '+t+' · format: '+fmt+' · weekly_content: accomplishments, goal-progress, open-items, decisions');
}
</script>
```

---

## CTA widget HTML

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
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
.time-row{display:inline-flex;align-items:center;gap:8px;background:#f3f3f3;border-radius:10px;padding:8px 16px;font-size:13px;font-weight:600;color:#444;margin-bottom:24px;flex-wrap:wrap;justify-content:center}
.time-row select,.time-row input[type=time]{border:none;background:transparent;font-size:15px;font-weight:700;color:#1a1a1a;outline:none;cursor:pointer}
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
  <div class="btns">
    <button class="btn btn-yes" onclick="scheduleIt()">Yes, schedule it</button>
    <button class="btn btn-no" onclick="sendPrompt('No schedule needed')">No, thanks</button>
  </div>
</div>
<script>
var DAYS={1:'Monday',4:'Thursday',5:'Friday'};
function scheduleIt(){
  var d=document.getElementById('sched-day').value;
  var t=document.getElementById('sched-time').value||'16:00';
  var parts=t.split(':'),h=parseInt(parts[0],10),m=parts[1];
  var label=(h%12||12)+':'+m+' '+(h>=12?'PM':'AM');
  sendPrompt('Yes, schedule my weekly review every '+DAYS[d]+' at '+label+' (cron: '+m+' '+h+' * * '+d+')');
}
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
<button class="btn" onclick="sendPrompt('Done — connectors connected, continue the flow')">✓ Done</button>
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
