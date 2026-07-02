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
allowed-tools: >
  mcp__xtiles__xtiles_get_current_user,
  mcp__xtiles__xtiles_get_user_timezone,
  mcp__xtiles__xtiles_get_planner_content,
  mcp__xtiles__xtiles_list_tasks,
  mcp__xtiles__xtiles_create_tasks,
  mcp__xtiles__xtiles_update_task,
  mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner,
  mcp__claude_ai_Slack__slack_search_channels,
  mcp__claude_ai_Slack__slack_search_public_and_private,
  mcp__claude_ai_Slack__slack_read_channel,
  mcp__claude_ai_Slack__slack_read_thread,
  mcp__claude_ai_Gmail__search_threads,
  mcp__claude_ai_Gmail__get_thread,
  mcp__session_info__list_sessions,
  mcp__session_info__read_transcript,
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
   nothing is created until the user has seen a preview and approved it. Only
   scheduled runs (with an approved config) act autonomously.
2. **Real data, not placeholders.** Always pull from Claude's own chat history
   (no connector needed) and from any connected tools before preview so the user
   sees live content. Never invent names, meetings, or messages.
3. **Match the user's language** throughout the entire flow — match the language
   of the user's first message and adapt if they switch.

---

## Algorithm

**Period is always Daily.** At the start of setup, tell the user: "I'll set up
your **Evening Reflection** — a short end-of-day synthesis written to your Daily
planner page." Never ask which period to set up. If the user asks for Weekly or
Monthly, say only the Daily reflection is currently supported.

**Run mode — detect before step 1:**

- **Scheduled run**: the incoming message contains `role:`, `tools:`,
  `evening_content:`, `tone:`, and `autolog:` (config injected by the `schedule`
  skill). Do not show the survey. Extract the config from the message. If a
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
| xTiles    | `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`, `mcp__xtiles__xtiles_list_tasks`               |
| Slack     | `mcp__claude_ai_Slack__slack_search_channels`, `mcp__claude_ai_Slack__slack_read_channel`                       |
| Gmail     | `mcp__claude_ai_Gmail__search_threads`, `mcp__claude_ai_Gmail__get_thread`                                       |
| Calendar  | the available calendar/events MCP tool, if any                                                                   |
| Linear    | `mcp__claude_ai_Linear__list_issues`                                                                             |

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

Ask three things (folded into the survey widget; inline in Claude Code):

**3.1 What to reflect on each evening.** Options — include only those relevant to
connected tools:
- Slack threads you were active in *(only if Slack connected)*
- Emails you sent / that needed action *(only if Gmail connected)*
- Meetings & calls (from calendar/notes) *(if calendar/Granola connected)*
- Tasks you completed today *(xTiles — on by default)*
- Other (describe in next message)

**3.2 Tone.** Single select:
- **Honest coach** — direct, names weak days plainly, anti-fluff (default)
- **Gentle** — supportive and encouraging
- **Neutral** — plain factual summary, no editorializing

**3.3 Auto-log completed tasks.** Single select:
- **Yes, with a preview first** — show me the tasks before creating them, then do
  it automatically on scheduled runs (recommended default)
- **Yes, automatically** — just create and complete them, no preview
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
  `per_page: 50`. Repeat with `completed: "false"` for open tasks due today. Also
  `mcp__xtiles__xtiles_get_planner_content` with `period: "day"` and today's date
  for full day context.
- **Slack**: `slack_search_public_and_private` with `on:[today] from:[user]` to
  find where the user was active, then `slack_read_thread` on the important
  threads. Apply the ignore filter from step 3.
- **Gmail**: `mcp__claude_ai_Gmail__search_threads` for mail sent today and
  important received mail that needed action (`newer_than:1d`), then `get_thread`
  for sender/subject/threadId.
- **Calendar / meeting notes** *(if connected)*: today's events; separate
  meetings-with-others (attendees > 1) from solo work blocks.
- **Claude chat history (today)** — **always run, no connector needed**: call
  `mcp__session_info__list_sessions` to get today's sessions, then
  `mcp__session_info__read_transcript` on the relevant ones. Extract concrete
  *outcomes* — what was actually built, solved, decided, or shipped (e.g.
  "wrote the launch email", "fixed the auth bug", "researched competitors").
  Ignore abandoned or purely exploratory threads. These outcomes feed both the
  reflection and the auto-log.

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

Match generously on meaning, not exact wording (e.g. a Claude session "wrote the
launch email" closes an open task "Draft launch email").

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

**Behavior by `autolog` setting:**
- **Yes, with a preview first** (and on the first manual run regardless): show the
  proposed list in chat, marking each as **new** or **closing an existing task** —
  `✅ [emoji] short specific title` for new, `☑️ closes: "[existing task]"` for
  matches — and ask via `AskUserQuestion`: "Log these for today?" → Apply all /
  Edit the list / Skip. Only after approval, apply them.
- **Yes, automatically**: create without preview.
- **No**: skip task creation entirely; go straight to the reflection tile.

To create: `mcp__xtiles__xtiles_create_tasks` with `assignee_email` (from
`get_current_user`), `due_date`: today (yyyy-MM-dd), `title`: short specific name
with an emoji category prefix. Then mark each completed:
`mcp__xtiles__xtiles_update_task` with `completed: true`. Avoid duplicates — never
recreate a task that already exists for today.

---

### 6. Compose the reflection & preview in chat

**Derive themes dynamically from ALL collected data — never from a template.**
Determine: the main themes of the day; what actually moved something forward vs
pure operations; opportunities (new contacts, ideas, competitive intel, insights);
promises & follow-ups (what was promised, who needs a message).

Apply the chosen **tone**. If the day was quiet, say so honestly — don't stretch
it. Optional sections appear only if there is something genuinely valuable.

Show the preview in chat with real content (not headings with "TBD"):

```
Here's your reflection for [actual date]:

---
✨ DAY CHARACTERISTIC — DD.MM.YYYY
[1–2 sharp sentences — the tone and essence of the day, not a task list]

🎯 RESULTS
- [What was done]
- ...
[If no strategic movement: "Operational day — little strategic progress"]

🌟 OPPORTUNITIES   (only if non-trivial)
- [Specific finds: partnerships, leads, competitor intel, ideas]

→ TOMORROW
1. [Specific action with names — "Message X about Y", not "continue X"]
2. [max 3 items]
3. ...
---
```

After the preview, ask via `AskUserQuestion`: **Looks good — write it** /
**Change something** / **Cancel**. On scheduled runs, skip approval and write
directly.

---

### 7. Write to xTiles

**Only after approval (or on a scheduled run).**

Tool: `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`
- `period`: "day"
- `date`: today in ISO 8601
- `markdown`: all sections in a **single call** — never split per section.

**Append, don't clobber.** The evening reflection writes to the same Daily page
the morning brief uses. First call `mcp__xtiles__xtiles_get_planner_content` for
today; compare existing `###` headers; append only sections whose headers don't
exist yet. If everything already exists, ask: replace, append anyway, or cancel.

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

**→ Tomorrow**

1. [Specific action with names — "Message X about Y", not "continue X"]
2. [max 3 items total]

⚠️ [unavailable connectors, only if some failed]
```

- `@colorSize` is always `LIGHTER`; `@color` is always `SAIL` for this tile. (Do
  **not** use plain names like "purple" — they will not render.)
- Sections inside the tile are **bold subheaders** (`**🎯 Results**`), separated by
  `---` dividers — never separate `###` tiles.
- Drop any optional section (Opportunities) entirely if there's nothing genuinely
  valuable — don't leave an empty header.

**Content formatting inside the tile:**
- Separate each item with a blank line — never a continuous block.
- Slack/email entries that have a URL are Markdown hyperlinks with the priority
  emoji BEFORE the `[`, never inside the brackets.
- Tomorrow's actions as a numbered list.
- Append the final `⚠️ [unavailable connectors]` line only if a connector failed.

**After a successful write:** call `mcp__xtiles__xtiles_get_planner_content` for
the same date/period, extract `view_id`, and confirm:

```
✅ Evening reflection saved.
```

Then call `show_widget` with the **CTA widget HTML** (see below), replacing `{VIEW_URL}` with `https://xtiles.app/{view_id}`. This renders a tappable button — never rely on a markdown link alone.

Translate the button label into the user's language. On error, say briefly what went wrong and offer to retry.

---

### 8. Schedule (optional)

**After every successful manual write — show the schedule widget** (see below),
regardless of survey answers. In Claude Code, ask inline: "Want me to run this
every evening automatically? What time? (default: 9:00 PM)".

- If the user schedules it — invoke `anthropic-skills:schedule`, then
  `mcp__scheduled-tasks__create-scheduled-tasks`. Pass to both:
    - **`prompt`**: full config assembled from setup —
      ```
      Run evening reflection — role: {role} · tools: {tools} · evening_content: {content} · tone: {tone} · autolog: {on/preview/off} · schedule: daily-9pm
      ```
      Replace all placeholders with real values.
    - **`schedule`**: cron derived from the widget. The widget sends `cron: HH:MM days:1-5` (weekdays) or `cron: HH:MM days:*` (every day) — parse both values and build: `M H * * 1-5` for weekdays, `M H * * *` for every day. Default `0 21 * * 1-5` (9:00 PM on weekdays) if not found.
    - **`timezone`**: from `mcp__xtiles__xtiles_get_user_timezone`.
      This prompt fires each evening and triggers `evening-reflection` in
      scheduled-run mode — the full config must be embedded so the survey is skipped.
      Confirm: "Done — your reflection will write to xTiles every [weekday evening / evening] at [time]." (say "weekday evening" if `days:1-5`, "every evening" if `days:*`)
- If the user declines — acknowledge briefly and stop.

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
sends a string of answers to chat — process it and continue.

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
        <div class="pill sel" onclick="pickLog(this,'preview')">Yes — preview first</div>
        <div class="pill" onclick="pickLog(this,'auto')">Yes — automatically</div>
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
var role=null, tools=new Set(), content=new Set(), tone='Honest coach', autolog='preview';
var TM={
  'Slack':   ['Slack threads I was active in'],
  'Gmail':   ['Emails I sent / that needed action'],
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
.time-row{display:inline-flex;align-items:center;gap:8px;background:#f3f3f3;border-radius:10px;padding:8px 16px;font-size:13px;font-weight:600;color:#444;margin-bottom:24px}
.time-row select,.time-row input[type=time]{border:none;background:transparent;font-size:15px;font-weight:700;color:#1a1a1a;outline:none;cursor:pointer}
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
  <div class="btns">
    <button class="btn btn-yes" id="btn-yes" onclick="scheduleIt()">Yes, schedule it</button>
    <button class="btn btn-no" id="btn-no" onclick="noThanks()">No, thanks</button>
  </div>
</div>
<script>
function collapse(msg){document.querySelector('.btns').innerHTML='<p style="font-size:13px;color:#aaa;text-align:center;padding:4px 0">'+msg+'</p>';}
function scheduleIt(){
  var days=document.getElementById('sched-days').value;
  var t=document.getElementById('sched-time').value||'21:00';
  var parts=t.split(':'),h=parseInt(parts[0],10),m=parts[1];
  var label=(h%12||12)+':'+m+' '+(h>=12?'PM':'AM');
  var dLabel=days==='1-5'?'weekdays':'every day';
  collapse('⏳ Scheduling…');
  sendPrompt('Yes, schedule my evening reflection at '+label+' '+dLabel+' (cron: '+t+' days:'+days+')');
}
function noThanks(){collapse('✓ Got it');sendPrompt('No schedule needed');}
</script>
```

---

## CTA widget HTML

Show immediately after a successful write and after scheduling confirmation. Replace `{VIEW_URL}` with the real xTiles page URL before calling `show_widget`.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
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
