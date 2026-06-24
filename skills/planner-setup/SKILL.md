---
name: planner-setup
description: >
  Personal setup of the cascading planner in xTiles.
  System: Month sets goals → Week breaks them into focus areas → Day = live brief
  from connectors + signals that need attention.

  Trigger when the user wants to:
  - "set up my planner", "create daily/weekly/monthly"
  - "personalize my workspace", "connect my planner to my tools"
  - onboard a new xTiles user into the Planner
allowed-tools: mcp__xtiles__xtiles_get_planner_content, mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner, AskUserQuestion, mcp__claude_ai_Slack__slack_search_channels
---

# Planner Setup — Smooth Setup Flow

## Three principles

1. **Survey first, write to xTiles last.** Nothing gets created until the user has seen the preview and said "yes".
2. **Daily ≠ Weekly ≠ Monthly.** Each period solves a different problem — content and questions differ accordingly.
3. **Real data, not placeholders.** Pull from connectors before preview so the user sees live content.

**Language:** match the language of the user's first message.

---

## Algorithm

### 1. Fast-track

If the user is specific ("give me daily for today", "I want to see Slack in the morning") — skip the full flow. Collect the minimum needed and jump to Silent data fetch (step 4).

If the request is general — run the full flow.

---

### 2. Survey — who are you and what's connected

**Show the survey widget** (HTML form) if in Cowork. Fallback — AskUserQuestion batches.

The form collects:
- Role (single select + Other)
- Tools (multi select: Slack, Gmail, Calendar, Analytics, Figma, Linear/Jira, Notion, Other)
- Which pages (Daily / Weekly / Monthly)

**After receiving answers — detect connectors:**

| Connector | Identifying MCP tools |
|-----------|----------------------|
| Slack | `slack_send_message`, `slack_read_channel` |
| Gmail | `search_threads`, `list_labels`, `get_thread` |
| Calendar | `list_events`, `create_event` |
| PostHog | `query_chart` + `get_from_url` + `get_events` |
| Amplitude | `query_chart` + `get_experiments` without `get_from_url` |
| xTiles | `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner` |

If only xTiles is connected — ask: set up statically or connect tools first?

---

### 3. Per-period clarification

**For each selected period — a separate question about content.**
Each period solves a different problem, so ask separately and don't impose structure.

Use AskUserQuestion with multiSelect. For each question —
4 suggestion options + Other where the user can write anything custom.

#### Daily

Question: "What do you want to see on your Daily each morning?"

Suggested options (pick the ones relevant to connected connectors):
- Important emails — unread messages that need a reply
- Chat messages — Slack or another work messenger
- Today's meetings — what's in Calendar today
- Metrics — key numbers if you check them daily
- Evening reflection — questions for yourself at end of day

Do NOT suggest tasks — they're already in xTiles by default.
If the user writes something custom via Other — fine, add it.

#### Weekly

Question: "What do you want to see on your Weekly?"

Suggested options:
- Weekly focus — main priorities
- Week's meetings — schedule from Calendar
- Team updates — key things to know from chat
- Week summary — Friday reflection

User can pick any combination or write their own.

#### Monthly

Question: "What do you want to see on your Monthly?"

Suggested options:
- Goals or intention for the month
- Meetings and events — full list from Calendar
- Project status — where things stand
- Retrospective — what worked, what didn't

User can pick any combination or write their own.

---

**General rule for all three:** don't box the user in.
If the user says "I want to see X" and X isn't in the list — fine, add X.
If something is unclear after the questions — ask directly, don't guess.

**Additional clarification if connectors are present:**

- **Slack connected** → call `mcp__claude_ai_Slack__slack_search_channels`, show top 4 channels, ask which ones they open first each morning
- **Amplitude/PostHog connected and user wants metrics** → ask for chart links or metric names
- **PostHog**: use `get_from_url` + `query_chart`
- **Amplitude**: `get_from_url` is unavailable — save URL as text, fetch data via `query_chart` / `get_experiments`

---

### 4. Silent data fetch

**Silently, without messaging the user**, pull fresh data from connectors:

- **Calendar**: events today, this week, this month (depending on selected pages)
- **Gmail**: unread important messages (`is:unread is:important in:inbox newer_than:3d`)
- **Slack**: messages from the user's chosen channels (top 20–30 most recent)

Analyze what you get — identify what needs the user's attention:
- 🔴 needs a decision / reply / deadline today
- 🟡 FYI / can wait / informational

---

### 5. Preview — show the content as text in chat

**This is the most important step.** Generate real content — not structure, not headings, but live text with real data.

Preview format in chat:

```
Here's what I've prepared for you:

---
📅 DAILY — May 7

### Chat signals
🔴 #team-sales · Iryna: agency mock-up is ready — can you help with the demo account? | [xTiles](...)
🔴 #team-sales · Iryna: your task — MCP Authorization, critical priority
🟡 #team-growth · Andrew: feedback needed on MCP landing mock | [link](...)

### Today's meetings
09:30 Growth-Sync · Andrey, Vadym, Iryna | [Meet](...)
16:30 Sync: Product plan · Volokovykh | [Meet](...)

### Gmail
No important unread emails — inbox clear

---
📆 WEEKLY — May 4–10

### Weekly focus
[what the user named as priorities]

### Week's meetings
Mon: Product marketing sync (1:00 PM)
...

---
🗓 MONTHLY — May

### Monthly goals
[what the user formulated]

### Meetings & events
May 1 Growth weekly meet ✓
May 8 Team Bi-weekly Sync
...
```

**Preview rules:**
- Show only the selected pages
- Content is live — from real connector data, no placeholders
- If there's no data for a section — either skip it or explain why it's empty
- After the preview ask: "Does this look right? Or anything to change?"

---

### 6. Approval

AskUserQuestion (single select):
- **"Yes, create it"** — proceed to write
- **"Change something"** — user says what, update only that part of the preview without restarting
- **"Cancel"** — stop

If the user wants a change — clarify exactly what, update only that section, show the preview again and ask again.

---

### 7. Write to xTiles

**Only after explicit "yes" from the user.**

Tool: `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`
- `period`: "day" / "week" / "month"
- `date`: current date in ISO 8601

**Order:** day → week → month.

**If the page already exists:**
1. Call `mcp__xtiles__xtiles_get_planner_content`
2. Compare existing H3 headers (`###`) with what you're about to add
3. Append only sections whose headers don't exist yet
4. If everything already exists — ask the user whether they want to replace it

**If an error occurs:** briefly say what went wrong, offer to retry or skip.

---

### 8. Schedule (optional)

If the user opted for an automatic schedule — after successful creation, run the `schedule` skill.
Only show relevant options:
- Daily at 9:00 AM — only if Daily was selected
- Weekly Mon 9:00 AM and Fri 5:00 PM — only if Weekly was selected
- Monthly summary — only if Monthly was selected

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
.period-box{padding:13px;background:#f9f9f9;border-radius:10px;margin-bottom:10px}
.period-title{font-size:13px;font-weight:600;color:#555;margin-bottom:8px}
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
        <div class="card" onclick="togTool(this,'Figma')"><div class="chk">✓</div>Figma</div>
        <div class="card" onclick="togTool(this,'Linear / Jira')"><div class="chk">✓</div>Linear / Jira</div>
        <div class="card" onclick="togTool(this,'Notion')"><div class="chk">✓</div>Notion</div>
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
    <h2>Your planner</h2>

    <div class="sec" style="margin-top:18px">
      <div class="sec-title">Which pages do you want to set up?</div>
      <div class="hint">You can select multiple</div>
      <div class="cards">
        <div class="card" onclick="togPage(this,'Daily')"><div class="chk">✓</div>📅 Daily</div>
        <div class="card" onclick="togPage(this,'Weekly')"><div class="chk">✓</div>📆 Weekly</div>
        <div class="card" onclick="togPage(this,'Monthly')"><div class="chk">✓</div>🗓 Monthly</div>
      </div>
    </div>

    <div id="page-content"></div>
    <div id="sched-wrap" style="display:none">
      <div class="divider"></div>
      <div class="sec">
        <div class="sec-title">Automatic schedule?</div>
        <div class="hint">I'll refresh your planner automatically</div>
        <div class="checks" id="sched-opts"></div>
      </div>
    </div>

    <div class="btn-row">
      <button class="btn btn-s" onclick="go1()">← Back</button>
      <button class="btn btn-p" onclick="submit()">Set up planner</button>
    </div>
  </div>
</div>

<script>
var role=null, tools=new Set(), pages=new Set(), content={}, sched=new Set();

var TM={
  'Slack':       {daily:['Slack messages — work chat signals'],weekly:['Team updates from Slack']},
  'Gmail':       {daily:['Important emails — unread inbox'],weekly:['Important letters of the week']},
  'Calendar':    {daily:["Today's meetings"],weekly:["Week's meetings — from Calendar"],monthly:['Meetings & events — full list']},
  'Analytics':   {daily:['Daily metrics'],weekly:['Weekly metrics'],monthly:['Monthly metrics']}
};
var AM={
  daily:['Evening reflection'],
  weekly:['Weekly focus — main priorities','Week summary — Friday reflection'],
  monthly:['Goals or intention for the month','Project status','Retrospective']
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
function go2(){document.getElementById('s1').style.display='none';document.getElementById('s2').style.display='block';}
function go1(){document.getElementById('s2').style.display='none';document.getElementById('s1').style.display='block';}

function togPage(el,p){
  el.classList.toggle('sel');
  el.classList.contains('sel')?pages.add(p):pages.delete(p);
  if(!content[p])content[p]=new Set();
  renderContent(); renderSched();
}
function getOpts(p){
  var key=p.toLowerCase(), items=new Set();
  tools.forEach(function(t){if(TM[t]&&TM[t][key])TM[t][key].forEach(function(i){items.add(i)})});
  (AM[key]||[]).forEach(function(i){items.add(i)});
  return Array.from(items);
}
function renderContent(){
  var order=['Daily','Weekly','Monthly'].filter(function(p){return pages.has(p)});
  if(!order.length){document.getElementById('page-content').innerHTML='';return;}
  var html='<div class="divider"></div>';
  order.forEach(function(p){
    if(!content[p])content[p]=new Set();
    var opts=getOpts(p), em=p==='Daily'?'📅':p==='Weekly'?'📆':'🗓';
    html+='<div class="period-box"><div class="period-title">'+em+' '+p+' — what do you want to see?</div><div class="checks" id="c-'+p.toLowerCase()+'">';
    opts.forEach(function(o){
      var s=content[p].has(o)?' sel':'';
      html+='<div class="ci'+s+'" onclick="togCI(this,\''+p+'\',\''+o.replace(/'/g,"\\'")+'\')" ><div class="chk">✓</div>'+o+'</div>';
    });
    html+='<div class="ci" onclick="togOther(this,\''+p+'\')" ><div class="chk">+</div>Something else…</div>';
    html+='<div class="custom-in" id="co-'+p.toLowerCase()+'" style="display:none"><input type="text" placeholder="What else…"></div>';
    html+='</div></div>';
  });
  document.getElementById('page-content').innerHTML=html;
}
function togCI(el,p,v){el.classList.toggle('sel');el.classList.contains('sel')?content[p].add(v):content[p].delete(v);}
function togOther(el,p){
  el.classList.toggle('sel');
  var w=document.getElementById('co-'+p.toLowerCase());
  if(w)w.style.display=el.classList.contains('sel')?'block':'none';
}
function renderSched(){
  var w=document.getElementById('sched-wrap'), o=document.getElementById('sched-opts');
  if(!pages.size){w.style.display='none';return;}
  var items=[];
  if(pages.has('Daily'))  items.push({v:'daily-9am',   l:'📅 Daily at 9:00 AM'});
  if(pages.has('Weekly')) items.push({v:'weekly-mon',  l:'📆 Weekly — Monday 9:00 AM'});
  if(pages.has('Weekly')) items.push({v:'weekly-fri',  l:'📆 Weekly — Friday 5:00 PM (recap)'});
  if(pages.has('Monthly'))items.push({v:'monthly',     l:'🗓 Monthly summary'});
  items.push({v:'none',l:'No schedule needed'});
  o.innerHTML=items.map(function(i){
    var s=sched.has(i.v)?' sel':'';
    return '<div class="ci'+s+'" onclick="togSched(this,\''+i.v+'\')" ><div class="chk">✓</div>'+i.l+'</div>';
  }).join('');
  w.style.display='block';
}
function togSched(el,v){
  if(v==='none'){
    sched.clear();
    document.querySelectorAll('#sched-opts .ci').forEach(function(e){e.classList.remove('sel')});
  } else {
    document.querySelectorAll('#sched-opts .ci').forEach(function(e){
      if(e.textContent.indexOf('No schedule')>-1)e.classList.remove('sel');
    });
    sched.delete('none');
    el.classList.toggle('sel');
    el.classList.contains('sel')?sched.add(v):sched.delete(v);
    return;
  }
  el.classList.add('sel'); sched.add(v);
}

function submit(){
  var r=role==='__other__'?document.getElementById('role-other-in').value.trim():role;
  var tArr=Array.from(tools);
  var tOther=document.getElementById('tool-other-in').value.trim();
  if(tOther)tArr.push(tOther);
  var parts=['Planner setup — role: '+r+' · tools: '+(tArr.join(', ')||'none')+' · pages: '+(Array.from(pages).join(', ')||'none')];
  ['Daily','Weekly','Monthly'].forEach(function(p){
    if(!pages.has(p))return;
    var items=Array.from(content[p]||[]);
    var inp=document.getElementById('co-'+p.toLowerCase());
    if(inp){var v=inp.querySelector('input');if(v&&v.value.trim())items.push(v.value.trim());}
    if(items.length)parts.push(p.toLowerCase()+'_content: '+items.join(', '));
  });
  var schArr=Array.from(sched).filter(function(v){return v!=='none'});
  if(schArr.length)parts.push('schedule: '+schArr.join(', '));
  sendPrompt(parts.join(' · '));
}
</script>
```

---

## How to behave

You're an assistant helping someone set up a planner that fits their real work rhythm.

- Don't create anything without a preview and approval
- If context is missing — ask, don't guess
- If the user gives new information along the way — pick it up, don't wait for the "right step"
- Real data from connectors always beats placeholders
- Daily, Weekly, Monthly — different tasks, different content, different questions
- Match the user's language (EN/UA), adapt if they switch
