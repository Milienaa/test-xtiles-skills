---
name: daily-brief
description: >
  Use when the user wants to set up OR run their xTiles Daily planner —
  a Daily page that serves as a live morning brief from connected tools
  (Slack, Gmail, Calendar, Analytics) plus signals that need attention.
  Only the Daily period is supported.

  Setup triggers: "set up my planner", "personalize my workspace",
  "connect my planner to my tools", "create daily",
  "onboard a new xTiles user into the Planner".

  Digest triggers: "show me my morning brief", "what do I need to know today",
  "run my digest". Also runs automatically via scheduled tasks.

  Config is read from the scheduled task prompt — no separate file needed.
  For manual runs: look for config in today's Planner; if there's none, start
  from the survey flow below.
allowed-tools: >
  mcp__xtiles__xtiles_get_planner_content,
  mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner,
  mcp__xtiles__xtiles_get_user_timezone,
  mcp__claude_ai_Slack__slack_search_channels,
  mcp__claude_ai_Slack__slack_read_channel,
  mcp__claude_ai_Gmail__search_threads,
  mcp__claude_ai_Gmail__list_labels,
  mcp__claude_ai_Gmail__get_thread,
  mcp__claude_ai_Google_Calendar__list_events,
  mcp__claude_ai_Amplitude__query_chart,
  mcp__claude_ai_Amplitude__get_experiments,
  mcp__mcp-registry__suggest_connectors,
  AskUserQuestion,
  anthropic-skills:schedule,
  mcp__scheduled-tasks__create-scheduled-tasks
---

# xTiles Daily Planner — Setup & Daily Digest

## Three principles

1. **Survey first, write to xTiles last.** Nothing gets created until the user has seen the preview and said "yes".
2. **Real data, not placeholders.** Pull from connectors before preview so the user sees live content.
3. **Match the user's language** throughout the entire flow — match the language of the user's first message and adapt if they switch.

---

## Algorithm

**Period is always Daily.** At the start of the flow, tell the user: "I'll set up your **Daily** planner page — a live morning brief from your connected tools." Never ask which period to set up.

**Run mode — detect before step 1:**
- **Scheduled run**: the incoming message contains `role:`, `tools:`, and `daily_content:` (config injected by the `schedule` skill). Do not show the survey. Extract the config from the message. If a connector from the config is not detected — offer to walk the user through connecting it before continuing. Then jump to **step 4 (Silent data fetch)**.
- **Fast-track or fresh manual run**: proceed to step 1.

### 1. Fast-track

If the user is specific ("give me daily for today", "I want to see Slack in the morning") — skip the full survey. Minimum needed: which connectors to pull from — infer from the message, then check the detection table (step 2) to confirm which are actually available. If a required connector is not detected — offer to walk the user through connecting it (see **How to connect connectors**); wait for confirmation before proceeding. Pull only from connectors that are both mentioned and confirmed available. Jump to **step 4**.

If the request is general — run the full flow.
 
---

### 2. Survey — who are you and what's connected

**Show the survey widget** (HTML form) in Cowork. In Claude Code (no Cowork environment), ask the same questions inline as plain text — role, tools, content preferences, schedule.

**Connected tools** (multi select, show all regardless of what's actually detected):
- Slack
- Gmail
- Google Calendar
- PostHog
- Amplitude
- Other (describe in next message)

If "Other" is selected — ask a follow-up: "Which tool(s)? Just name them — I'll figure out what's available."

After receiving answers — detect which MCP tools are actually available:

| Connector | Identifying MCP tools                                                                                              |
|-----------|--------------------------------------------------------------------------------------------------------------------|
| Slack     | `mcp__claude_ai_Slack__slack_send_message`, `mcp__claude_ai_Slack__slack_read_channel`                             |
| Gmail     | `mcp__claude_ai_Gmail__search_threads`, `mcp__claude_ai_Gmail__list_labels`, `mcp__claude_ai_Gmail__get_thread`    |
| Calendar  | `mcp__claude_ai_Google_Calendar__list_events`, `mcp__claude_ai_Google_Calendar__create_event`                      |
| PostHog   | `query_chart` + `get_from_url` + `get_events`                                                                      |
| Amplitude | `mcp__claude_ai_Amplitude__query_chart` + `mcp__claude_ai_Amplitude__get_experiments`                              |
| xTiles    | `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`                                                     |

These connectors are external and optional — they are not shipped with this plugin. The user must connect them separately.

**If xTiles is not connected** — do not continue. Immediately walk the user through connecting xTiles (see **How to connect connectors** below). Wait for confirmation that xTiles is connected before proceeding.

**If a connector the user selected isn't connected** (Gmail, Slack, etc.) — immediately walk them through connecting it step by step. Do not move to the next step until they confirm it's connected or explicitly choose to skip that connector.

---

### 3. Daily content clarification

Question: "What do you want to see on your Daily each morning?"

Options — include only those relevant to connected tools:
- Unread emails that need a reply *(only if Gmail connected)*
- Slack messages from key channels *(only if Slack connected)*
- Today's meetings *(only if Calendar connected)*
- Key metrics *(only if PostHog or Amplitude connected)*
- Evening reflection prompts
- Other (describe in next message)

Do NOT suggest tasks — they're already in xTiles by default.

**If Slack is selected and the user has not already named their channels:**
Call `mcp__claude_ai_Slack__slack_search_channels`, show up to 6 channel names. Ask via `AskUserQuestion` (multi allowed):
"Which channels do you open first each morning? Pick all that matter."
Include the found channels as options plus one fixed option: **"Other — I'll type the names"**.
If the user selects "Other" — follow up: "Which channels? Type the names, comma-separated." Add them to the list as-is.

**If Analytics (PostHog or Amplitude) is selected and detected as connected:**
Ask for chart links or metric names to pull.
- PostHog: use `get_from_url` + `query_chart`
- Amplitude: `get_from_url` unavailable — save URL as text, fetch via `mcp__claude_ai_Amplitude__query_chart` / `mcp__claude_ai_Amplitude__get_experiments`

**General rule:** if the user writes something custom — add it as-is. Don't reshape it into a predefined option.

---

### 4. Silent data fetch

**Silently, without messaging the user**, pull fresh data from connectors based on selected sections and content choices:

- **Calendar**: today's events only (`mcp__claude_ai_Google_Calendar__list_events`)
- **Gmail**: unread important messages (`mcp__claude_ai_Gmail__search_threads` — query `is:unread is:important in:inbox newer_than:3d`)
- **Slack**: recent messages from the user's chosen channels (`mcp__claude_ai_Slack__slack_read_channel`, top 20–30)

Analyze what you get. Classify each signal:
- 🔴 needs a decision, reply, or action today
- 🟡 informational / can wait

Use only real data from connectors. Do not invent names, events, or messages.
All names, meeting titles, and message content must come directly from API responses — never from examples in this skill file.

If a connector call fails (error, timeout, 401) — record the failure. Do not write "No data" for a failed call — surface the error explicitly in step 5 so the user knows the connector did not respond.

---

### 5. Preview — show content in chat

Show real content with real data. Not structure, not headings with "(TBD)" — actual text.

Format (adapt to selected pages):

```
Here's what I've prepared:

---
📅 DAILY — [actual dateп]

### [Section based on user's choices]
🔴 [Real signal from connector] | [link if available]
🟡 [Real signal from connector]

### Today's meetings
[Real events from Calendar with actual titles and times]

### Gmail
[Real unread threads or "Inbox clear — no important unread email"]

---
```

**Rules:**
- Show only selected sections the user asked for
- If a connector returned no data — write exactly that ("No unread emails", "No meetings today") — don't skip silently
- If a connector call failed — write "Could not fetch [connector] data — connector error" (not "No data")
- No placeholder names, example events, or invented data — ever
- After the preview, ask: "Does this look right? Anything to change?"
---

### 6. Approval

Ask the user (single select):
- **"Looks good — create it"** → proceed to write
- **"Change something"** → ask what, update only that section, show preview again
- **"Cancel"** → stop

If the user asks for a change — clarify exactly what, update only that section, re-show preview, ask again.

---

### 7. Write to xTiles

**Only after explicit approval.**

Tool: `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`
- `period`: "day"
- `date`: current date in ISO 8601

**Write all sections in a single call.** Combine Gmail, Slack, Calendar, and any other selected connectors into one markdown and call the tool once — never split into separate calls per connector.

**If xTiles is not connected** — do not output the digest as plain text in chat. Walk the user through connecting xTiles (see **How to connect connectors**), wait for confirmation, then write.

**If the page already exists:**
1. Call `mcp__xtiles__xtiles_get_planner_content`
2. Compare existing H3 headers (`###`) with what you're about to add
3. Append only sections whose headers don't exist yet
4. If everything already exists — ask: replace all, append anyway, or cancel?
   **After each successful write:**
   Call `mcp__xtiles__xtiles_get_planner_content` with the same `date` and `period`.
   Extract the `view_id` from the response and include a link in the confirmation:

```
✅ [Page name] created.
🔗 [Open in xTiles](https://xtiles.app/{view_id})
```

Translate the link label ("Open in xTiles") into the user's language.

**If an error occurs:** briefly say what went wrong, offer to retry or skip that page.

---

### 8. Schedule (optional)

**After every successful write — always show the schedule widget** (see **Schedule widget HTML** below), regardless of what was selected in the setup survey. In Claude Code (no Cowork), ask inline: "Want me to run this every morning automatically? I can set it up so your Daily is ready in xTiles by 9:00 AM."

- If the user selects **"Yes, schedule it"** — first invoke `anthropic-skills:schedule`, then call `mcp__scheduled-tasks__create-scheduled-tasks`. Pass to both:
  - **`prompt`**: the full config string assembled from values collected during setup —
    ```
    Run daily digest — role: {role} · tools: {tools} · daily_content: {content} · schedule: daily-9am
    ```
    Replace `{role}`, `{tools}`, `{content}` with the actual values. Do not leave placeholders.
  - **`schedule`**: `0 9 * * *` (every day at 09:00)
  - **`timezone`**: the user's local timezone — call `mcp__xtiles__xtiles_get_user_timezone` to get it before scheduling if it hasn't been fetched yet.

  This prompt fires each morning and triggers `daily-brief` in scheduled-run mode — the full config must be embedded so the survey is skipped automatically.

  After scheduling succeeds, confirm: "Done — your Daily will be ready in xTiles every morning at 9:00 AM."
- If the user selects **"No, thanks"** — acknowledge briefly and stop.
---

## How to connect connectors

Do not send the user to settings manually and do not give a URL to follow.
Call `mcp__mcp-registry__suggest_connectors` — it renders interactive connect buttons directly in the Cowork UI.

**Flow:**
1. Call `mcp__mcp-registry__suggest_connectors` passing the names of the missing connectors.
2. Show the **Done widget** (see **Done widget HTML** below) directly under the connector form.
3. The user clicks the connect buttons in the UI — the auth flow runs natively. When finished, they click **"Done"**.
4. Confirm: "Connected. Continuing…" and resume the flow from where it was interrupted.
---

## Done widget HTML

Show this widget via `show_widget` immediately after calling `mcp__mcp-registry__suggest_connectors`.
The user clicks **Done** when they have finished connecting — this sends a message to chat and resumes the flow.

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

## Survey widget HTML

Show this form via `show_widget` at the start of setup in Cowork.
After Submit, the user sends a string of answers to chat — process it and continue the flow.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:#f8f8f8;color:#1a1a1a}
.wrap{max-width:560px;margin:0 auto;background:#fff;border-radius:16px;padding:28px;box-shadow:0 2px 12px rgba(0,0,0,.08)}
h2{font-size:18px;font-weight:700;margin-bottom:4px}
.step-label{font-size:12px;color:#aaa;margin-bottom:20px}
.sec{margin-bottom:22px}
.sec-title{font-size:13px;font-weight:600;color:#444;margin-bottom:8px}
.hint{font-size:12px;color:#aaa;margin-bottom:8px}
.pills{display:flex;flex-wrap:wrap;gap:7px}
.pill{padding:6px 14px;border-radius:20px;border:1.5px solid #e0e0e0;font-size:13px;cursor:pointer;background:#fff;color:#333;user-select:none;transition:all .15s}
.pill:hover{border-color:#aaa}
.pill.sel{background:#1a1a1a;color:#fff;border-color:#1a1a1a}
.cards{display:flex;flex-wrap:wrap;gap:8px}
.card{display:flex;align-items:center;gap:7px;padding:8px 13px;border-radius:10px;border:1.5px solid #e0e0e0;font-size:13px;cursor:pointer;background:#fff;color:#333;user-select:none;transition:all .15s}
.card:hover{border-color:#aaa}
.card.sel{background:#1a1a1a;color:#fff;border-color:#1a1a1a}
.chk{width:15px;height:15px;border-radius:4px;border:1.5px solid #ccc;display:flex;align-items:center;justify-content:center;font-size:9px;flex-shrink:0}
.card.sel .chk{background:#fff;border-color:#fff;color:#1a1a1a}
.custom-in{margin-top:9px}
.custom-in input{width:100%;padding:7px 11px;border:1.5px solid #e0e0e0;border-radius:8px;font-size:13px;outline:none}
.custom-in input:focus{border-color:#999}
.checks{display:flex;flex-direction:column;gap:5px;margin-top:6px}
.ci{display:flex;align-items:center;gap:9px;padding:7px 11px;border-radius:8px;border:1.5px solid #e0e0e0;font-size:13px;cursor:pointer;background:#fff;user-select:none;transition:all .15s}
.ci:hover{border-color:#aaa}
.ci.sel{border-color:#1a1a1a;background:#f5f5f5}
.ci .chk{flex-shrink:0}
.ci.sel .chk{background:#1a1a1a;border-color:#1a1a1a;color:#fff}
.divider{height:1px;background:#f0f0f0;margin:18px 0}
.btn-row{display:flex;gap:10px;margin-top:22px}
.btn{padding:9px 18px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:all .15s}
.btn-p{background:#1a1a1a;color:#fff;flex:1}
.btn-p:hover:not(:disabled){background:#333}
.btn-p:disabled{background:#ccc;cursor:not-allowed}
.btn-s{background:#f0f0f0;color:#333}
.btn-s:hover{background:#e0e0e0}
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
      <div class="sec-title">Which tools do you use?</div>
      <div class="hint">Select all that apply — I'll pull live data from them</div>
      <div class="cards" id="tool-cards">
        <div class="card" onclick="togTool(this,'Slack')"><div class="chk">✓</div>Slack</div>
        <div class="card" onclick="togTool(this,'Gmail')"><div class="chk">✓</div>Gmail</div>
        <div class="card" onclick="togTool(this,'Calendar')"><div class="chk">✓</div>Calendar</div>
        <div class="card" onclick="togTool(this,'Analytics')"><div class="chk">✓</div>Analytics</div>
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
    <h2>Your Daily</h2>

    <div class="sec" style="margin-top:18px">
      <div class="sec-title">What do you want to see each morning?</div>
      <div class="checks" id="daily-content"></div>
    </div>

    <div class="btn-row">
      <button class="btn btn-s" onclick="go1()">← Back</button>
      <button class="btn btn-p" onclick="submit()">Set up Daily</button>
    </div>
  </div>
</div>

<script>
var role=null, tools=new Set(), content=new Set();

var TM={
  'Slack':    {daily:['Slack messages — work chat signals']},
  'Gmail':    {daily:['Important emails — unread inbox']},
  'Calendar': {daily:["Today's meetings"]},
  'Analytics':{daily:['Daily metrics']}
};
var AM=['Evening reflection'];

var ROLE_DEFAULTS={
  'Product Manager':   ['Slack messages — work chat signals',"Today's meetings",'Important emails — unread inbox'],
  'Designer':          ['Slack messages — work chat signals',"Today's meetings"],
  'Engineer':          ['Slack messages — work chat signals','Important emails — unread inbox'],
  'Growth & Marketing':['Daily metrics','Important emails — unread inbox',"Today's meetings"],
  'Founder / CEO':     ['Slack messages — work chat signals','Important emails — unread inbox',"Today's meetings",'Daily metrics'],
  'Support & Success': ['Important emails — unread inbox','Slack messages — work chat signals']
};

function pickRole(el,v){
  document.querySelectorAll('#role-pills .pill').forEach(function(p){p.classList.remove('sel')});
  el.classList.add('sel'); role=v;
  document.getElementById('role-other-wrap').style.display=v==='__other__'?'block':'none';
  chkNext();
}
function togTool(el,v){
  el.classList.toggle('sel');
  if(v==='__other__'){
    document.getElementById('tool-other-wrap').style.display=el.classList.contains('sel')?'block':'none';
  } else {
    el.classList.contains('sel')?tools.add(v):tools.delete(v);
  }
  chkNext();
}
function chkNext(){
  var ok=role&&(role!=='__other__'||document.getElementById('role-other-in').value.trim());
  document.getElementById('next-btn').disabled=!ok;
}
function go2(){
  if(!content.size){
    var r=role==='__other__'?null:role;
    (ROLE_DEFAULTS[r]||[]).forEach(function(v){content.add(v);});
  }
  renderContent();
  document.getElementById('s1').style.display='none';
  document.getElementById('s2').style.display='block';
}
function go1(){document.getElementById('s2').style.display='none';document.getElementById('s1').style.display='block';}

function renderContent(){
  var items=[];
  tools.forEach(function(t){if(TM[t]&&TM[t].daily)TM[t].daily.forEach(function(i){if(items.indexOf(i)<0)items.push(i)})});
  AM.forEach(function(i){if(items.indexOf(i)<0)items.push(i)});
  var html='';
  items.forEach(function(o){
    var s=content.has(o)?' sel':'';
    html+='<div class="ci'+s+'" onclick="togCI(this,\''+o.replace(/'/g,"\\'")+'\')" ><div class="chk">✓</div>'+o+'</div>';
  });
  html+='<div class="ci" onclick="togOther(this)"><div class="chk">+</div>Something else…</div>';
  html+='<div class="custom-in" id="co-daily" style="display:none"><input type="text" placeholder="What else…"></div>';
  document.getElementById('daily-content').innerHTML=html;
}
function togCI(el,v){el.classList.toggle('sel');el.classList.contains('sel')?content.add(v):content.delete(v);}
function togOther(el){
  el.classList.toggle('sel');
  var w=document.getElementById('co-daily');
  if(w)w.style.display=el.classList.contains('sel')?'block':'none';
}
function submit(){
  var r=role==='__other__'?document.getElementById('role-other-in').value.trim():role;
  var tArr=Array.from(tools);
  var tOther=document.getElementById('tool-other-in').value.trim();
  if(tOther)tArr.push(tOther);
  var items=Array.from(content);
  var inp=document.getElementById('co-daily');
  if(inp){var v=inp.querySelector('input');if(v&&v.value.trim())items.push(v.value.trim());}
  var parts=['Daily planner setup — role: '+r+' · tools: '+(tArr.join(', ')||'none')+' · daily_content: '+(items.join(', ')||'none')];
  sendPrompt(parts.join(' · '));
}
</script>
```

---

## Schedule widget HTML

Show this widget via `show_widget` after a successful write in Cowork.
After the user clicks a button, the widget calls `sendPrompt()` and the response lands in chat.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:#f8f8f8;color:#1a1a1a}
.wrap{max-width:480px;margin:0 auto;background:#fff;border-radius:16px;padding:28px;box-shadow:0 2px 12px rgba(0,0,0,.08);text-align:center}
.icon{font-size:36px;margin-bottom:12px}
h2{font-size:17px;font-weight:700;margin-bottom:6px}
.sub{font-size:13px;color:#888;margin-bottom:24px;line-height:1.5}
.time{display:inline-flex;align-items:center;gap:6px;background:#f3f3f3;border-radius:8px;padding:6px 14px;font-size:13px;font-weight:600;color:#444;margin-bottom:24px}
.btns{display:flex;flex-direction:column;gap:10px}
.btn{padding:11px 20px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:all .15s}
.btn-yes{background:#1a1a1a;color:#fff}
.btn-yes:hover{background:#333}
.btn-no{background:#f0f0f0;color:#555}
.btn-no:hover{background:#e0e0e0}
</style>

<div class="wrap">
  <div class="icon">⏰</div>
  <h2>Run this every morning?</h2>
  <p class="sub">I'll fetch your signals and write your Daily to xTiles automatically — no need to ask each time.</p>
  <div class="time">📅 Every day at 9:00 AM</div>
  <div class="btns">
    <button class="btn btn-yes" onclick="sendPrompt('Yes, schedule my daily digest at 9:00 AM every day')">Yes, schedule it</button>
    <button class="btn btn-no" onclick="sendPrompt('No schedule needed')">No, thanks</button>
  </div>
</div>
```

---

## How to behave

- Use the survey widget for setup; ask inline for approval and any follow-up clarifications
- **Never output the digest as plain text in chat** and ask the user to copy it manually — always write to xTiles directly, or walk through connecting xTiles first
- **Never skip a connector** the user selected — if it's not connected, walk through the connection before continuing, don't silently drop it
- Never create anything without preview and explicit approval
- Never put example names, example events, or example messages into the preview — only real data from connectors
- **All clarifying questions after the main survey form** (channel selection, metric links, approval, change requests) — use `AskUserQuestion`, not plain text
- If context is missing — ask, don't guess
- If the user gives new information along the way — pick it up, don't wait for the "right step"
- Real data from connectors always beats placeholders
- Daily is the only period. If the user asks for Weekly or Monthly, tell them only the Daily planner is currently supported and offer to create a Daily page instead — never silently downscope.
- Match the user's language, adapt if they switch
- Show the survey widget in Cowork only — in Claude Code, ask the same questions inline
 