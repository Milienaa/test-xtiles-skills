---
name: db-survey
description: >
  INTERNAL shared step — not a user-facing workflow. The interactive rich-widget
  presentation layer for the daily-brief setup flow: renders the two-screen
  survey, the Slack channel-picker, and the newsletter-picker, then returns the
  user's selection. Invoked only by daily-brief via
  xtiles_get_workflow("db-survey"), at each interactive-pick branch. Never match
  this to a user request, never list or offer it to a user, never run it
  standalone.
allowed-tools: show_widget, AskUserQuestion
---

# db-survey — Interactive Widget Presentation (shared step)

The rich-widget presentation layer reused by `daily-brief`. The caller computes
the data (probe results, discovered channels, discovered newsletters) and calls
this sub-workflow to render the matching widget and read the user's choice. This
sub-workflow owns **only** rendering and response-reading — no connector probe,
no discovery, no fetch. It is **not** a standalone workflow — never run it on its
own and never offer it to a user.

## Modes

The caller invokes one of three modes and passes the inputs / receives the
outputs below:

| Mode | Input from caller | Output to caller |
|------|-------------------|------------------|
| `survey` | detected connectors (to pre-select tool cards) | `{role, tools, daily_content}` from the submit string |
| `channels` | discovered channel list + which to pre-mark (`sel`) | selected channels from the submit string |
| `newsletters` | discovered newsletter/sender list | selected newsletters from the submit string |

## Render pattern (all modes)

1. Take the mode's HTML block below **verbatim** (byte-stable — run-to-run render
   consistency depends on it; change nothing except the documented injection
   points).
2. Populate the injection points from the caller's data: inject one card/pill per
   item where the block's HTML comment says to, and mark pre-selected cards by
   adding `sel` to the card's `class` **and** setting its matching data attribute
   (`data-tool` for survey tool cards, `data-v` for channels) — both, or a
   deselect click silently does nothing and the item leaks back in. Change
   nothing else.
3. Render with the environment's visualize tool (`show_widget` in Claude Cowork;
   its analog in GPT Cowork).
4. Read the user's submission from the `sendPrompt` string the widget emits, and
   return the parsed selection to the caller.

**Fallback — never stop.** If the visualize tool is unavailable, degrade to the
environment's native question mechanism (`AskUserQuestion`) with the same
options; for a list of more than 4 items, present it as plain text and ask the
user to reply with their picks. Return the same output shape to the caller.

## Mode: `survey`

Two-screen form collecting role, tools, and Daily content in one submit. Before
rendering, pre-select each detected connector's tool card by adding `sel` to its
`class` **and** keeping its `data-tool` (both, per the render pattern). The submit
string has the form
`Daily planner setup — role: … · tools: … · daily_content: …` — parse `role`,
`tools`, and `daily_content` from it; the submitted `tools` list is authoritative
downstream. This block is self-contained (its own `<style>`) — no CSS injection.

```html
<style>
    :root{--color-background-primary:#fff;--color-background-secondary:#f5f5f5;--color-background-tertiary:#f8f8f8;--color-text-primary:#1a1a1a;--color-text-secondary:#888;--color-border-secondary:#aaa;--color-border-tertiary:#e0e0e0}
    @media(prefers-color-scheme:dark){:root{--color-background-primary:#2c2c2c;--color-background-secondary:#383838;--color-background-tertiary:#222;--color-text-primary:#f0f0f0;--color-text-secondary:#999;--color-border-secondary:#666;--color-border-tertiary:#3d3d3d}}
    *{box-sizing:border-box;margin:0;padding:0}
    body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:transparent;color:var(--color-text-primary)}
    .wrap{max-width:560px;margin:0 auto;background:var(--color-background-primary);border-radius:16px;padding:28px;box-shadow:0 2px 12px rgba(0,0,0,.08)}
    h2{font-size:18px;font-weight:700;margin-bottom:4px;color:var(--color-text-primary)}
    .step-label{font-size:12px;color:var(--color-text-secondary);margin-bottom:20px}
    .sec{margin-bottom:22px}
    .sec-title{font-size:13px;font-weight:600;color:var(--color-text-primary);margin-bottom:8px}
    .hint{font-size:12px;color:var(--color-text-secondary);margin-bottom:8px}
    .pills{display:flex;flex-wrap:wrap;gap:7px}
    .pill{padding:6px 14px;border-radius:20px;border:1.5px solid var(--color-border-tertiary);font-size:13px;cursor:pointer;background:var(--color-background-primary);color:var(--color-text-primary);user-select:none;transition:all .15s}
    .pill:hover{border-color:var(--color-border-secondary)}
    .pill.sel{background:var(--color-text-primary);color:var(--color-background-primary);border-color:var(--color-text-primary)}
    .cards{display:flex;flex-wrap:wrap;gap:8px}
    .card{display:flex;align-items:center;gap:7px;padding:8px 13px;border-radius:10px;border:1.5px solid var(--color-border-tertiary);font-size:13px;cursor:pointer;background:var(--color-background-primary);color:var(--color-text-primary);user-select:none;transition:all .15s}
    .card:hover{border-color:var(--color-border-secondary)}
    .card.sel{background:var(--color-text-primary);color:var(--color-background-primary);border-color:var(--color-text-primary)}
    .chk{width:15px;height:15px;border-radius:4px;border:1.5px solid var(--color-border-secondary);display:flex;align-items:center;justify-content:center;font-size:9px;flex-shrink:0}
    .card.sel .chk{background:var(--color-background-primary);border-color:var(--color-background-primary);color:var(--color-text-primary)}
    .custom-in{margin-top:9px}
    .custom-in input{width:100%;padding:7px 11px;border:1.5px solid var(--color-border-tertiary);border-radius:8px;font-size:13px;outline:none;background:var(--color-background-primary);color:var(--color-text-primary)}
    .custom-in input:focus{border-color:var(--color-border-secondary)}
    .checks{display:flex;flex-direction:column;gap:5px;margin-top:6px}
    .ci{display:flex;align-items:center;gap:9px;padding:7px 11px;border-radius:8px;border:1.5px solid var(--color-border-tertiary);font-size:13px;cursor:pointer;background:var(--color-background-primary);color:var(--color-text-primary);user-select:none;transition:all .15s}
    .ci:hover{border-color:var(--color-border-secondary)}
    .ci.sel{border-color:var(--color-text-primary);background:var(--color-background-secondary)}
    .ci .chk{flex-shrink:0}
    .ci.sel .chk{background:var(--color-text-primary);border-color:var(--color-text-primary);color:var(--color-background-primary)}
    .divider{height:1px;background:var(--color-border-tertiary);margin:18px 0}
    .btn-row{display:flex;gap:10px;margin-top:22px}
    .btn{padding:9px 18px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:all .15s}
    .btn-p{background:var(--color-text-primary);color:var(--color-background-primary);flex:1}
    .btn-p:hover:not(:disabled){opacity:0.9}
    .btn-p:disabled{opacity:0.5;cursor:not-allowed}
    .btn-s{background:var(--color-background-secondary);color:var(--color-text-primary)}
    .btn-s:hover{background:var(--color-border-secondary)}
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
        <div class="card" data-tool="Slack" onclick="togTool(this,'Slack')"><div class="chk">✓</div>Slack</div>
        <div class="card" data-tool="Gmail" onclick="togTool(this,'Gmail')"><div class="chk">✓</div>Gmail</div>
        <div class="card" data-tool="Calendar" onclick="togTool(this,'Calendar')"><div class="chk">✓</div>Calendar</div>
        <div class="card" data-tool="Granola" onclick="togTool(this,'Granola')"><div class="chk">✓</div>Granola</div>
        <div class="card" data-tool="Linear" onclick="togTool(this,'Linear')"><div class="chk">✓</div>Linear</div>
        <div class="card" data-tool="GoogleDrive" onclick="togTool(this,'GoogleDrive')"><div class="chk">✓</div>Google Drive</div>
      </div>
      <div class="custom-in" style="margin-top:8px">
        <input type="text" id="other-tool" placeholder="Other connector (e.g. Plaud, Notion)…">
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
  'Slack':       {daily:['Slack messages — work chat signals']},
  'Gmail':       {daily:['Important emails — unread inbox','Newsletters — curated summaries']},
  'Calendar':    {daily:['Workload — calendar analysis']},
  'Granola':     {daily:['Granola — meeting notes & summaries']},
  'Linear':      {daily:['Linear issues — new & updated']},
  'GoogleDrive': {daily:['Google Drive — shared files updated']}
};
var AM=[];

var ROLE_DEFAULTS={
  'Product Manager':   ['Slack messages — work chat signals','Important emails — unread inbox','Workload — calendar analysis','Linear issues — new & updated','Granola — meeting notes & summaries'],
  'Designer':          ['Slack messages — work chat signals','Important emails — unread inbox','Workload — calendar analysis'],
  'Engineer':          ['Slack messages — work chat signals','Important emails — unread inbox','Linear issues — new & updated','Workload — calendar analysis'],
  'Growth & Marketing':['Important emails — unread inbox','Newsletters — curated summaries','Slack messages — work chat signals','Workload — calendar analysis'],
  'Founder / CEO':     ['Slack messages — work chat signals','Important emails — unread inbox','Newsletters — curated summaries','Granola — meeting notes & summaries','Workload — calendar analysis'],
  'Support & Success': ['Important emails — unread inbox','Slack messages — work chat signals']
};

document.querySelectorAll('#tool-cards .card.sel').forEach(function(el){tools.add(el.dataset.tool)});
function pickRole(el,v){
  document.querySelectorAll('#role-pills .pill').forEach(function(p){p.classList.remove('sel')});
  el.classList.add('sel'); role=v;
  document.getElementById('role-other-wrap').style.display=v==='__other__'?'block':'none';
  chkNext();
}
function togTool(el,v){
  el.classList.toggle('sel');
  el.classList.contains('sel')?tools.add(v):tools.delete(v);
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
  // Always pre-select content for every tool the user explicitly picked
  tools.forEach(function(t){if(TM[t]&&TM[t].daily)TM[t].daily.forEach(function(v){content.add(v);});});
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
  document.getElementById('daily-content').innerHTML=html;
}
function togCI(el,v){el.classList.toggle('sel');el.classList.contains('sel')?content.add(v):content.delete(v);}
function submit(){
  document.querySelectorAll('.btn').forEach(function(b){b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';});
  var r=role==='__other__'?document.getElementById('role-other-in').value.trim():role;
  var oth=document.getElementById('other-tool').value.trim();
  if(oth)tools.add(oth);
  var tArr=Array.from(tools);
  var valid=[];
  tArr.forEach(function(t){if(TM[t]&&TM[t].daily)TM[t].daily.forEach(function(i){if(valid.indexOf(i)<0)valid.push(i)});});
  AM.forEach(function(i){if(valid.indexOf(i)<0)valid.push(i)});
  var items=Array.from(content).filter(function(i){return valid.indexOf(i)>=0;});
  var parts=['Daily planner setup — role: '+r+' · tools: '+(tArr.join(', ')||'none')+' · daily_content: '+(items.join(', ')||'none')];
  sendPrompt(parts.join(' · '));
}
</script>
```

## Mode: `channels`

Slack channel-picker. The caller passes the discovered channels grouped under the
three discovery labels and marks the up-to-5 pre-selected. Inject cards per the
HTML comment; add `sel` to the pre-selected ones. The submit string is
`Selected channels: …`. The `<style>` below is the shared picker CSS inlined.

```html
<style>
:root{--c-surface:#fff;--c-text:#1a1a1a;--c-border:#e0e0e0;--c-border2:#aaa;--c-btn-p:#1a1a1a;--c-btn-p-text:#fff}
@media(prefers-color-scheme:dark){:root{--c-surface:#2c2c2c;--c-text:#f0f0f0;--c-border:#3d3d3d;--c-border2:#666;--c-btn-p:#f0f0f0;--c-btn-p-text:#1a1a1a}}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:transparent}
.wrap{max-width:480px;margin:0 auto;background:var(--c-surface);border-radius:16px;padding:24px;box-shadow:0 2px 12px rgba(0,0,0,.12)}
h2{font-size:15px;font-weight:700;margin-bottom:14px;color:var(--c-text)}
.cards{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:12px}
.card{padding:6px 14px;border-radius:20px;border:1.5px solid var(--c-border);font-size:13px;cursor:pointer;background:var(--c-surface);color:var(--c-text);user-select:none;transition:all .15s}
.card:hover{border-color:var(--c-border2)}
.card.sel{background:var(--c-btn-p);color:var(--c-btn-p-text);border-color:var(--c-btn-p)}
.grp{flex-basis:100%;font-size:11px;font-weight:600;text-transform:uppercase;letter-spacing:.04em;color:var(--c-border2);margin:6px 0 -2px}
input{width:100%;padding:8px 12px;border:1.5px solid var(--c-border);border-radius:8px;font-size:13px;margin-bottom:14px;outline:none;background:var(--c-surface);color:var(--c-text)}
input:focus{border-color:var(--c-border2)}
.btn{width:100%;padding:11px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;background:var(--c-btn-p);color:var(--c-btn-p-text)}
</style>
<div class="wrap">
  <h2>Which channels do you open first each morning?</h2>
  <p style="font-size:12px;color:#888;margin:-8px 0 12px">Pre-selected: your most active and relevant channels. Add any others you want to see.</p>
  <div class="cards" id="ch">
    <!-- Group the discovered channels under the three Step F.4 labels, each preceded by a group heading. Omit any group that has no channels.
         <div class="grp">Core — relevant &amp; active</div>
         <div class="grp">Active — where you post most</div>
         <div class="grp">Relevant to your role</div>
         Inject one card per channel under its group: <div class="card[ sel]" data-v="#channelname" onclick="tog(this,'#channelname')">#channelname</div>
         Add " sel" to the class for the up-to-5 pre-selected channels from Step F.6 (all core first). -->
  </div>
  <input type="text" id="other-ch" placeholder="Other channel…">
  <button class="btn" id="sub-ch" onclick="submit()">Confirm</button>
</div>
<script>
var sel=new Set();
document.querySelectorAll('#ch .card.sel').forEach(function(el){sel.add(el.dataset.v)});
function tog(el,v){el.classList.toggle('sel');el.classList.contains('sel')?sel.add(v):sel.delete(v)}
function submit(){var b=document.getElementById('sub-ch');b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';b.textContent='⏳…';var o=document.getElementById('other-ch').value.trim();if(o)sel.add(o);sendPrompt('Selected channels: '+Array.from(sel).join(', '))}
</script>
```

## Mode: `newsletters`

Newsletter-picker. The caller passes the discovered publications/senders. Inject
one card per publication per the HTML comment (none pre-selected). The submit
string is `Selected newsletters: …`. If the caller found no publications, use the
fallback (ask for names as free text). The `<style>` below is the shared picker
CSS inlined.

```html
<style>
:root{--c-surface:#fff;--c-text:#1a1a1a;--c-border:#e0e0e0;--c-border2:#aaa;--c-btn-p:#1a1a1a;--c-btn-p-text:#fff}
@media(prefers-color-scheme:dark){:root{--c-surface:#2c2c2c;--c-text:#f0f0f0;--c-border:#3d3d3d;--c-border2:#666;--c-btn-p:#f0f0f0;--c-btn-p-text:#1a1a1a}}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:transparent}
.wrap{max-width:480px;margin:0 auto;background:var(--c-surface);border-radius:16px;padding:24px;box-shadow:0 2px 12px rgba(0,0,0,.12)}
h2{font-size:15px;font-weight:700;margin-bottom:14px;color:var(--c-text)}
.cards{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:12px}
.card{padding:6px 14px;border-radius:20px;border:1.5px solid var(--c-border);font-size:13px;cursor:pointer;background:var(--c-surface);color:var(--c-text);user-select:none;transition:all .15s}
.card:hover{border-color:var(--c-border2)}
.card.sel{background:var(--c-btn-p);color:var(--c-btn-p-text);border-color:var(--c-btn-p)}
.grp{flex-basis:100%;font-size:11px;font-weight:600;text-transform:uppercase;letter-spacing:.04em;color:var(--c-border2);margin:6px 0 -2px}
input{width:100%;padding:8px 12px;border:1.5px solid var(--c-border);border-radius:8px;font-size:13px;margin-bottom:14px;outline:none;background:var(--c-surface);color:var(--c-text)}
input:focus{border-color:var(--c-border2)}
.btn{width:100%;padding:11px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;background:var(--c-btn-p);color:var(--c-btn-p-text)}
</style>
<div class="wrap">
  <h2>Which newsletters do you want in your Daily?</h2>
  <div class="cards" id="nl">
    <!-- inject: <div class="card" onclick="tog(this,'Publication Name')">Publication Name</div> -->
  </div>
  <input type="text" id="other-nl" placeholder="Other newsletter…">
  <button class="btn" id="sub-nl" onclick="submit()">Confirm</button>
</div>
<script>
var sel=new Set();
function tog(el,v){el.classList.toggle('sel');el.classList.contains('sel')?sel.add(v):sel.delete(v)}
function submit(){var b=document.getElementById('sub-nl');b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';b.textContent='⏳…';var o=document.getElementById('other-nl').value.trim();if(o)sel.add(o);sendPrompt('Selected newsletters: '+Array.from(sel).join(', '))}
</script>
```

## Return to the caller

Once the user submits (or the fallback collects the picks), parse the selection
and return it to `daily-brief` in the mode's output shape. Do not perform any
fetch, discovery, or write here — that is the caller's job.