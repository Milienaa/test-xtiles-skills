---
name: today-news
description: >
  Generate a fresh morning news digest from the web based on user interests.
  Use when the user asks for daily news, morning news, latest updates, a news
  briefing, or wants to track a topic, company, market, or technology.
  Trigger phrases: "give me morning news", "what's happening in AI today",
  "set up daily news", "today's news about fintech".
allowed-tools: WebSearch, WebFetch, show_widget, mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner, mcp__xtiles__xtiles_get_planner_content, mcp__xtiles__xtiles_get_user_timezone, AskUserQuestion, anthropic-skills:schedule, mcp__scheduled-tasks__create-scheduled-tasks
---

# Today News

Generate a fresh morning digest from live web sources based on the user's interests.

## Core behavior

- **Primary source: WebSearch + WebFetch.** Never use memory for current events.
- Match the language of the user's request throughout.
- Default scope: global, last 24 hours.
- **News** — verified against reputable primary sources (Reuters, AP, Bloomberg, FT, TechCrunch). Never aggregators or PR.
- **Rumors & Leaks** (optional) — unverified tips from X/Twitter, Reddit, insider blogs. Always labeled as unverified.
- Never invent events, dates, quotes, metrics, names, or URLs.

---

## Algorithm

**Run mode — detect before step 1:**
- **Scheduled run**: incoming message contains `topics:` and `schedule:`. Skip setup. Extract topics and flags (`verify`, `rumors`), search the web, write the digest to today's xTiles Daily.
- **Fast-track**: user names a specific topic ("news about AI"). Skip setup — infer topic and jump to step 3.
- **Setup**: general request ("set up daily news", "I want morning news"). Run the full flow.

---

### 1. Analyze interests

Infer 4–6 likely topics from conversation context — the user's role, companies or products they mentioned, tools they use. If no context is available, default to: AI & product, tech industry, startups, business.

### 2. Suggest topics

Build the **Topic widget HTML** (see below) with inferred topic labels. Call `show_widget`.

The widget lets the user:
- Select topic chips and/or type a custom topic
- Toggle **Verify sources** — enables cross-checking of stories from unreliable or outdated outlets
- Toggle **Rumors & Leaks** — adds a separate section with unverified tips from X, Reddit, and insider blogs

`submit()` sends all selections and toggle states via `sendPrompt()`.

### 3. Search

**Verified news (always):** For each topic run `WebSearch`:
- Tech/AI → `site:techcrunch.com OR site:theverge.com OR site:wired.com`
- Finance → `site:bloomberg.com OR site:ft.com OR site:reuters.com`
- General → `site:reuters.com OR site:apnews.com OR site:bbc.com`

Query: `[topic] news [today OR "last 24 hours"] [domain filters]`

Fetch top 2–3 results per topic with `WebFetch`. Extract: headline + publication + date, 2–3 sentence summary, URL. Deduplicate by URL. Drop: opinion without a news hook, PR content, anything older than 48 h.

**If "Verify sources" is on:**
For each story, assess source reputation and publication date. If the outlet is questionable or the date is ambiguous, run a corroborating `WebSearch` to confirm the event. Mark each story: ✅ verified or ⚠️ could not corroborate.

**Rumors & Leaks (if opted in):** Run a parallel search per topic:
- Sources: `site:x.com OR site:reddit.com OR site:9to5mac.com OR site:macrumors.com OR site:androidauthority.com`
- Query: `[topic] leak OR rumor OR exclusive OR insider [today OR this week]`
- Fetch top 1–2 results. Extract claim, source, corroboration clues. Always label as **unverified**.

### 4. Preview

Show the assembled digest using the **Preview widget HTML** (see below).

Claude builds the full widget HTML dynamically — inject the actual news content into `#digest` following the injection pattern in the template comment. The widget has three actions:
- **Save to xTiles** — triggers step 5
- **Change something** — reveals a text field; user submits correction text which re-enters the flow
- **Cancel** — ends the flow

**Never show the digest as plain text — always use the Preview widget.**

### 5. Write to xTiles

Tool: `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`
- `period`: `"day"`
- `date`: today in ISO 8601
- `markdown`: approved digest (one News tile + optional Rumors tile)

After write: call `mcp__xtiles__xtiles_get_planner_content`, get `view_id`, show the **CTA widget HTML** with `{VIEW_URL}` = `https://xtiles.app/{view_id}`.

### 6. Schedule (optional)

After every successful write — show the **Schedule widget HTML**.

If the user schedules: invoke `anthropic-skills:schedule`, then `mcp__scheduled-tasks__create-scheduled-tasks`:
- `prompt`: `Run today-news — topics: {topics} · flags: verify={true/false} rumors={true/false} · schedule: daily-{time}`
- `schedule`: cron from widget (e.g. `30 8 * * *`)
- `timezone`: from `mcp__xtiles__xtiles_get_user_timezone`

---

## Digest format

All content goes into **one News tile** (max two tiles total — News + optional Rumors). Do not create a separate tile per topic.

**News tile** — all topics in one tile:

```markdown
### 📰 Today's News
@colorSize: LIGHTER
@color: CUMULUS

**AI & Product**

**[Headline — Source](URL)** ✅
[2–3 sentence summary]

**[Headline — Source](URL)** ⚠️
[2–3 sentence summary]

**Tech Industry**

**[Headline — Source](URL)**
[2–3 sentence summary]
```

Omit ✅/⚠️ status markers if "Verify sources" was off.

If no news found for a topic, write:

```
**[Topic]**
No news found in the last 24 hours.
```

**Rumors & Leaks tile** — all topics in one tile (only if opted in):

```markdown
### 🕵️ Rumors & Leaks
@colorSize: LIGHTER
@color: MILK_PUNCH

⚠️ *Unverified — based on X posts, Reddit, or insider blogs*

**AI & Product**

**[Claim — Source](URL)**
[1–2 sentence summary]

**Tech Industry**

**[Claim — Source](URL)**
[1–2 sentence summary]
```

- Write all tiles in a **single** xTiles call
- `@colorSize` is always `LIGHTER`
- News tile always uses `CUMULUS`; Rumors tile always uses `MILK_PUNCH`

---

## Topic widget HTML

Build dynamically: replace `[Topic N]` with inferred labels (4–6 chips). Include only as many `.chip` divs as there are inferred topics.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:#f8f8f8;color:#1a1a1a}
.wrap{max-width:520px;margin:0 auto;background:#fff;border-radius:16px;padding:26px;box-shadow:0 2px 12px rgba(0,0,0,.08)}
h2{font-size:17px;font-weight:700;margin-bottom:16px}
.chips{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:16px}
.chip{padding:8px 14px;border:1.5px solid #e0e0e0;border-radius:20px;font-size:13px;cursor:pointer;user-select:none;transition:all .15s;background:#fff}
.chip:hover{border-color:#aaa}
.chip.sel{background:#1a1a1a;color:#fff;border-color:#1a1a1a}
input[type=text]{width:100%;padding:10px 12px;border:1.5px solid #e0e0e0;border-radius:10px;font-size:13px;outline:none;margin-bottom:16px}
input[type=text]:focus{border-color:#999}
.divider{height:1px;background:#f0f0f0;margin-bottom:14px}
.toggles{margin-bottom:20px}
.toggle-row{display:flex;align-items:center;justify-content:space-between;padding:10px 0;cursor:pointer;user-select:none}
.toggle-row+.toggle-row{border-top:1px solid #f5f5f5}
.toggle-info{flex:1;margin-right:12px}
.toggle-label{font-size:13px;font-weight:600}
.toggle-desc{font-size:11px;color:#888;margin-top:2px;line-height:1.4}
.sw{width:40px;height:22px;background:#e0e0e0;border-radius:11px;position:relative;transition:background .2s;flex-shrink:0}
.sw.on{background:#1a1a1a}
.knob{width:18px;height:18px;background:#fff;border-radius:50%;position:absolute;top:2px;left:2px;transition:left .2s;box-shadow:0 1px 3px rgba(0,0,0,.2)}
.sw.on .knob{left:20px}
.btn{width:100%;padding:11px;border-radius:10px;border:none;font-size:14px;font-weight:600;background:#1a1a1a;color:#fff;cursor:pointer}
.btn:hover{background:#333}
.btn:disabled{opacity:.4;cursor:not-allowed}
</style>
<div class="wrap">
  <h2>What do you want news about?</h2>
  <div class="chips" id="chips">
    <div class="chip" onclick="tog(this)">[Topic 1]</div>
    <div class="chip" onclick="tog(this)">[Topic 2]</div>
    <div class="chip" onclick="tog(this)">[Topic 3]</div>
    <div class="chip" onclick="tog(this)">[Topic 4]</div>
    <div class="chip" onclick="tog(this)">[Topic 5]</div>
    <div class="chip" onclick="tog(this)">[Topic 6]</div>
  </div>
  <input type="text" id="custom" placeholder="Add a topic…" oninput="chk()">
  <div class="divider"></div>
  <div class="toggles">
    <div class="toggle-row" onclick="togSw('verify')">
      <div class="toggle-info">
        <div class="toggle-label">Verify sources</div>
        <div class="toggle-desc">Cross-check unreliable or outdated stories with a second source</div>
      </div>
      <div class="sw" id="verify"><div class="knob"></div></div>
    </div>
    <div class="toggle-row" onclick="togSw('rumors')">
      <div class="toggle-info">
        <div class="toggle-label">Rumors &amp; Leaks</div>
        <div class="toggle-desc">Unverified tips from X, Reddit, and insider blogs — collected separately</div>
      </div>
      <div class="sw" id="rumors"><div class="knob"></div></div>
    </div>
  </div>
  <button class="btn" id="sub" onclick="submit()" disabled>Get news →</button>
</div>
<script>
var sw={verify:false,rumors:false};
function tog(el){el.classList.toggle('sel');chk();}
function togSw(id){sw[id]=!sw[id];document.getElementById(id).classList.toggle('on',sw[id]);}
function chk(){
  var hasSel=document.querySelectorAll('.chip.sel').length>0;
  var hasText=document.getElementById('custom').value.trim().length>0;
  document.getElementById('sub').disabled=!(hasSel||hasText);
}
function submit(){
  var sel=Array.from(document.querySelectorAll('.chip.sel')).map(function(e){return e.textContent;});
  var custom=document.getElementById('custom').value.trim();
  if(custom)sel.push(custom);
  var flags=[];
  if(sw.verify)flags.push('verify=true');
  if(sw.rumors)flags.push('rumors=true');
  var msg='Today news topics: '+sel.join(', ');
  if(flags.length)msg+=' · '+flags.join(' · ');
  sendPrompt(msg);
}
</script>
```

---

## Preview widget HTML

Build fully dynamically: inject actual fetched news into `#digest`. Follow the injection pattern in the comment block. Show verify status (✅ / ⚠️) inline only if verification was on.

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
.section-rumors{background:#fffbeb}
.section-head{font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.5px;color:#888;margin-bottom:10px;display:flex;align-items:center;gap:6px}
.badge{font-size:10px;padding:2px 7px;border-radius:20px;font-weight:600;background:#fef3c7;color:#92400e}
.topic-block{margin-bottom:12px}
.topic-block:last-child{margin-bottom:0}
.topic-name{font-size:12px;font-weight:700;color:#444;margin-bottom:6px;padding-bottom:5px;border-bottom:1px solid rgba(0,0,0,.07)}
.story{margin-bottom:7px;padding-bottom:7px;border-bottom:1px solid rgba(0,0,0,.05)}
.story:last-child{border-bottom:none;margin-bottom:0;padding-bottom:0}
.story-title{font-size:12px;font-weight:600;color:#1a1a1a;line-height:1.4}
.story-meta{font-size:10px;color:#aaa;margin-top:2px}
.story-sum{font-size:11px;color:#666;margin-top:3px;line-height:1.5}
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
    <h2>📰 Today's News</h2>
    <span class="cnt" id="cnt"></span>
  </div>
  <div class="scroll" id="digest">

    <!--
    INJECTION PATTERN — Claude fills in #digest with real content:

    News section:
    <div class="section">
      <div class="section-head">📰 News</div>
      <div class="topic-block">
        <div class="topic-name">AI &amp; Product</div>
        <div class="story">
          <div class="story-title">Headline ✅</div>
          <div class="story-meta">TechCrunch · Jun 26</div>
          <div class="story-sum">2–3 sentence summary.</div>
        </div>
        <div class="story">
          <div class="story-title">Headline ⚠️</div>
          <div class="story-meta">Source · date</div>
          <div class="story-sum">Summary.</div>
        </div>
      </div>
      <div class="topic-block">
        <div class="topic-name">Tech Industry</div>
        ...
      </div>
    </div>

    Rumors section (only if opted in):
    <div class="section section-rumors">
      <div class="section-head">🕵️ Rumors &amp; Leaks <span class="badge">Unverified</span></div>
      <div class="topic-block">
        <div class="topic-name">AI &amp; Product</div>
        <div class="story">
          <div class="story-title">Claim headline</div>
          <div class="story-meta">X post / Reddit · date</div>
          <div class="story-sum">Summary.</div>
        </div>
      </div>
    </div>
    -->

  </div>
  <div class="feedback" id="fb">
    <textarea id="fbtext" placeholder="What should I change?"></textarea>
  </div>
  <div class="btns">
    <button class="btn btn-save" id="btn-save" onclick="doSave()">Save to xTiles</button>
    <button class="btn btn-edit" id="btn-edit" onclick="toggleEdit()">Change something</button>
    <button class="btn btn-cancel" onclick="sendPrompt('Cancel news digest')">Cancel</button>
  </div>
</div>
<script>
var editing=false;
function doSave(){
  var fb=document.getElementById('fbtext').value.trim();
  sendPrompt(fb?'Apply this change then save: '+fb:'Save news digest to xTiles');
}
function toggleEdit(){
  editing=!editing;
  document.getElementById('fb').style.display=editing?'block':'none';
  document.getElementById('btn-save').textContent=editing?'Apply & Save':'Save to xTiles';
  document.getElementById('btn-edit').textContent=editing?'Never mind':'Change something';
}
window.onload=function(){
  var n=document.querySelectorAll('.story').length;
  if(n)document.getElementById('cnt').textContent=n+' stories';
};
</script>
```

---

## CTA widget HTML

Show immediately after a successful write. Replace `{VIEW_URL}` with the real URL.

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

## Schedule widget HTML

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:#f8f8f8;color:#1a1a1a}
.wrap{max-width:480px;margin:0 auto;background:#fff;border-radius:16px;padding:28px;box-shadow:0 2px 12px rgba(0,0,0,.08);text-align:center}
.icon{font-size:32px;margin-bottom:12px}
h2{font-size:17px;font-weight:700;margin-bottom:6px}
.sub{font-size:13px;color:#888;margin-bottom:20px;line-height:1.5}
.time-row{display:inline-flex;align-items:center;gap:8px;background:#f3f3f3;border-radius:10px;padding:8px 16px;font-size:13px;font-weight:600;color:#444;margin-bottom:24px}
.time-row input[type=time]{border:none;background:transparent;font-size:15px;font-weight:700;color:#1a1a1a;outline:none;cursor:pointer}
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
  <p class="sub">I'll fetch fresh news and write Today's News to xTiles automatically.</p>
  <div class="time-row">Every day at <input type="time" id="sched-time" value="09:00"></div>
  <div class="btns">
    <button class="btn btn-yes" onclick="scheduleIt()">Yes, schedule it</button>
    <button class="btn btn-no" onclick="sendPrompt('No schedule needed')">No, thanks</button>
  </div>
</div>
<script>
function scheduleIt(){
  var t=document.getElementById('sched-time').value||'09:00';
  var parts=t.split(':'),h=parseInt(parts[0],10),m=parts[1];
  var label=(h%12||12)+':'+m+' '+(h>=12?'PM':'AM');
  sendPrompt('Yes, schedule my today-news at '+label+' every day (cron: '+t+')');
}
</script>
```

---

## Failure handling

- If WebSearch fails — say search failed, do not fabricate content.
- If xTiles write fails — show the digest in chat so the user still has it.
- If scheduling fails — keep the created digest, explain only recurrence failed.

## How to behave

- Never put invented names, events, or links in the preview widget.
- Never create without preview and explicit approval, except scheduled runs.
- All clarifying questions use `AskUserQuestion`.
- Match the user's language; adapt if they switch.
- Always show the digest preview as the **Preview widget** — never as plain text.