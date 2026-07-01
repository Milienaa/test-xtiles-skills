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
- Prefer primary sources and reputable reporting over aggregators.
- Never invent events, dates, quotes, metrics, names, or URLs.

---

## Algorithm

**Run mode — detect before step 1:**
- **Scheduled run**: the incoming message contains `topics:` and `schedule:`. Skip setup. Extract topics, search the web, write the digest to today's xTiles Daily.
- **Fast-track**: if the user names a topic ("news about AI"), skip setup — infer the topic and jump to step 3.
- **Setup**: if the request is general ("set up daily news", "I want morning news"), run the full flow.

---

### 1. Analyze interests

Infer 4–6 likely topics from conversation context — the user's role, companies or products they mentioned, tools they use. If no context is available, default to: AI & product, tech industry, startups, business.

### 2. Suggest topics

Build the **Topic widget HTML** (see below), replacing each `[Topic N]` placeholder with one of the inferred topics (2–4 words each). Then call `show_widget` with the generated HTML.

The widget lets the user select chips and/or type a custom topic — `submit()` sends both via `sendPrompt()`.

### 3. Search

For each selected topic, run `WebSearch` with a query tuned to the domain:
- Tech/AI → `site:techcrunch.com OR site:theverge.com OR site:wired.com`
- Finance → `site:bloomberg.com OR site:ft.com OR site:reuters.com`
- General → `site:reuters.com OR site:apnews.com OR site:bbc.com`

Query pattern: `[topic] news [today OR "last 24 hours"] [domain filters]`

Fetch top 2–3 results per topic with `WebFetch`. Extract:
- Headline + publication + date
- 2–3 sentence summary
- Source URL

Deduplicate by URL. Filter out: opinion without a news hook, PR dressed as news, content older than 48h.

**Cross-day deduplication:** before finalising, call `mcp__xtiles__xtiles_get_planner_content` with `period: "day"` and yesterday's date (ISO 8601). Scan the returned tile content for any hyperlink URLs. Drop any story from today's results whose URL already appeared yesterday — the same news must not repeat on consecutive days.

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
- `markdown`: approved digest

After write: call `mcp__xtiles__xtiles_get_planner_content`, get `view_id`, show the CTA button widget (see **CTA widget HTML**) with `{VIEW_URL}` = `https://xtiles.app/{view_id}`.

### 6. Schedule (optional)

After every successful write — show the schedule widget.

If the user schedules: invoke `anthropic-skills:schedule`, then `mcp__scheduled-tasks__create-scheduled-tasks`:
- `prompt`: `Run today-news — topics: {topics} · schedule: daily-{time}`
- `schedule`: cron from widget (e.g. `30 8 * * *`)
- `timezone`: from `mcp__xtiles__xtiles_get_user_timezone`

---

## Digest format

One tile per topic:

```markdown
### 📰 [Topic]
@colorSize: LIGHTER
@color: [COLOR]

**[Headline — Source](URL)**
[2–3 sentence summary]

**[Headline — Source](URL)**
[2–3 sentence summary]
```

If no news found for a topic:

```markdown
### 📰 [Topic]
@colorSize: LIGHTER
@color: ATHENS_GRAY

No news found in the last 24 hours.
```

- `@colorSize` is always `LIGHTER`
- Pick `@color` per tile from: `GHOST, CUMULUS, GOSSIP, COLDTURKEY, BLUE_CHALK, MILK_PUNCH, HAWKES_BLUE, PATTENS_BLUE, SAIL, ATHENS_GRAY, BERMUDA, PERFUME, SELAGO, RICE_FLOWER, WHITE_LINEN, POLAR`
- Different color per tile — do not repeat the same color twice in a row
- Write all tiles in a **single** xTiles call

---

## Topic widget HTML

Build this HTML dynamically: replace `[Topic 1]` … `[Topic 6]` with the inferred topic labels before calling `show_widget`. Include only as many `.chip` divs as there are inferred topics (4–6).

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
  <button class="btn" id="sub" onclick="submit()" disabled>Get news →</button>
</div>
<script>
function tog(el){el.classList.toggle('sel');chk();}
function chk(){
  var hasSel=document.querySelectorAll('.chip.sel').length>0;
  var hasText=document.getElementById('custom').value.trim().length>0;
  document.getElementById('sub').disabled=!(hasSel||hasText);
}
function submit(){
  var sel=Array.from(document.querySelectorAll('.chip.sel')).map(function(e){return e.textContent;});
  var custom=document.getElementById('custom').value.trim();
  if(custom)sel.push(custom);
  sendPrompt('Today news topics: '+sel.join(', '));
}
</script>
```

---

## Approval widget HTML

Show via `show_widget` after the digest preview in step 4.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:16px;background:transparent}
.btns{display:flex;flex-direction:column;gap:8px}
.btn{width:100%;padding:11px 20px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:background .15s}
.btn-yes{background:#1a1a1a;color:#fff}
.btn-yes:hover{background:#333}
.btn-edit{background:#f0f0f0;color:#1a1a1a}
.btn-edit:hover{background:#e0e0e0}
.btn-cancel{background:transparent;color:#aaa;font-weight:400}
.btn-cancel:hover{color:#666}
</style>
<div class="btns">
  <button class="btn btn-yes" onclick="sendPrompt('Looks good — save it')">✓ Looks good — save it</button>
  <button class="btn btn-edit" onclick="sendPrompt('Change something')">Edit</button>
  <button class="btn btn-cancel" onclick="sendPrompt('Cancel')">Cancel</button>
</div>
```

---

## CTA widget HTML

Show immediately after a successful write. Replace `{VIEW_URL}` with the real URL.

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
    <button class="btn btn-yes" id="btn-yes" onclick="scheduleIt()">Yes, schedule it</button>
    <button class="btn btn-no" id="btn-no" onclick="noThanks()">No, thanks</button>
  </div>
</div>
<script>
function lock(){document.querySelectorAll('.btn').forEach(function(b){b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';});}
function scheduleIt(){
  lock();
  document.getElementById('btn-yes').textContent='⏳ Scheduling…';
  var t=document.getElementById('sched-time').value||'09:00';
  var parts=t.split(':'),h=parseInt(parts[0],10),m=parts[1];
  var label=(h%12||12)+':'+m+' '+(h>=12?'PM':'AM');
  sendPrompt('Yes, schedule my today-news at '+label+' every day (cron: '+t+')');
}
function noThanks(){
  lock();
  document.getElementById('btn-no').textContent='✓ Got it';
  sendPrompt('No schedule needed');
}
</script>
```

---

## Failure handling

- If WebSearch fails — say search failed, do not fabricate content.
- If xTiles write fails — show the digest in chat so the user still has it.
- If scheduling fails — keep the created digest, explain only recurrence failed.

## How to behave

- Never put invented names, events, or links in the preview.
- Never create without preview and explicit approval, except scheduled runs.
- All clarifying questions use `AskUserQuestion`.
- Match the user's language; adapt if they switch.
