---
name: daily-news
description: Generate a fresh morning news digest on a user-specified topic.
  Use when the user asks for daily news, morning news, latest updates, a news
  briefing, newsletter digest, or a recurring digest about a topic, company,
  market, industry, person, place, technology, competitor set, or area of
  interest. The skill can collect newsletters/news digests from the user's
  mailbox, summarize them, write tiles to today's xTiles Daily planner, archive
  processed emails, and optionally schedule itself to run every morning.
allowed-tools: WebSearch, WebFetch, show_widget, mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner, mcp__xtiles__xtiles_get_planner_content, mcp__xtiles__xtiles_get_user_timezone, mcp__claude_ai_Gmail__search_threads, mcp__claude_ai_Gmail__list_labels, mcp__claude_ai_Gmail__get_thread, mcp__claude_ai_Gmail__update_thread, AskUserQuestion, anthropic-skills:schedule, mcp__scheduled-tasks__create-scheduled-tasks
---

# Daily News

Create a useful morning news digest from newsletters and news digests received
in the user's mailbox. Prioritize freshness, editorial value, and clear
relevance over volume.

## Core behavior

- Match the language of the user's request for all questions, previews, and final output.
- Use live mailbox data for every run. Never rely on memory for current news.
- Use web search only as a fallback when mailbox connectors are unavailable or
  when the user explicitly asks for web news.
- Default time window: last 24 hours. Expand only if the user explicitly asks.
- Prefer editorial newsletters, curated digests, primary sources, and reputable reporting.
- Never invent events, dates, quotes, metrics, names, or source links.
- Clearly separate confirmed facts from interpretation.
- If no newsletters are found, create a "no new emails" tile instead of
  fabricating content.

## Algorithm

**Period is always Daily.** At the start of setup, tell the user: "I'll set up
your Daily News page - a fresh morning digest from your newsletters." Never ask
which period to use.

**Run mode - detect before step 1:**
- **Scheduled run**: the incoming message contains `topic:`, `language:`,
  `depth:`, and `schedule:` (config injected by the `schedule` skill). Do not
  show the setup widget. Extract the config from the message, search the mailbox
  for fresh newsletters, and write the result to today's xTiles Daily planner. If xTiles is
  not available or the write fails, surface the connector error and show the
  prepared digest in chat.
- **Fast-track or fresh manual run**: proceed to step 1.

### 1. Fast-track

If the user is specific ("give me morning news about AI", "daily news for
fintech in Europe"), skip the setup widget. Infer topic, region, language, and
depth from the request where possible. If the user gives no topic, use
`all newsletters` and summarize all valuable newsletters from the last 24 hours.

If the request is general ("set up daily news", "I want morning news"), show the
setup widget in Cowork. In Claude Code, ask the same questions inline as plain
text: optional topic filter, region or market, language, and depth.

After receiving setup answers, parse the single config string and continue with
the research process without re-asking answered questions.

## Digest process

### 1. Find newsletters

Search the user's mailbox for newsletters and news digests received in the last
24 hours.

Use two searches when Gmail tools are available:

1. Search for newsletter/news digest emails from the last 24 hours. Look for:
   - sender platforms: Substack, Beehiiv, ConvertKit, Mailchimp, Revue, Ghost,
     Buttondown, Paragraph
   - recurring editorial digests or curated content emails
   - emails that contain an unsubscribe link and look editorial, curated, or
     informational
2. Search Gmail Updates/category-style results from the last 24 hours as an
   additional source, then filter them with the same rules.

If the user specified a topic, keep newsletters that match the topic directly or
contain substantial related coverage. If the user did not specify a topic,
summarize all valuable newsletters found in the last 24 hours.

Deduplicate by `thread_id`. Count each email only once.

### 2. Filter results

Keep only editorial newsletters and news digests. Filter out:

- transactional emails: order confirmations, receipts, shipping updates
- service notifications: GitHub, Jira, Slack, calendar, security alerts
- marketing and sales promos: discounts, limited offers, flash sales
- automated SaaS reports: weekly stats, usage summaries, unless they include
  real editorial commentary

Newsletter indicators:

- unsubscribe link in the body
- sender from a known newsletter platform domain
- structured editorial format with multiple topics or sections
- recurring curated content or author commentary

### 3. Read content

For each candidate email, call `mcp__claude_ai_Gmail__get_thread`.

If parallel `get_thread` calls fail with stream/timeout errors, retry
sequentially one by one.

After reading each email body, do a final classification:

- If it is editorial, create a tile.
- If it is transactional, promotional, or a service notification, skip the tile.
- Still archive processed emails after the write step, including emails skipped
  after reading.

Ignore tracking links, pixel links, unsubscribe links, and platform CDN image
links. Keep only useful article, podcast, resource, discussion, or source links.

If more than 10 newsletters are found, process only the 10 most valuable by
editorial depth. Archive the rest without creating tiles.

### 4. Sort

Sort tiles by content depth:

1. longform author articles
2. multi-topic digests
3. short announcements

### 5. Web fallback

Use web search only if Gmail tools are unavailable, Gmail search fails, or the
user explicitly asks for web news. In that case, search live sources for the
topic and use the same xTiles tile style, but do not archive anything.

## Digest format

For longform newsletters with one deep topic:

```markdown
### [Source name] - [Email subject]
@colorSize: LIGHTER
@color: [COLOR]

**Topic:** [short description, including guest/author when relevant]

---

**Main idea:** [1-2 sentences]

---

**Key insights:**
- [Insight 1, 2-4 sentences]
- [Insight 2, 2-4 sentences]
- [Insight 3, 2-4 sentences]

---

**Links:** [useful links from the email]
```

For digests and short newsletters with multiple topics:

```markdown
### [Source name] - [Email subject]
@colorSize: LIGHTER
@color: [COLOR]

**From:** [sender] | **Date:** [date]

---

**[Product/topic 1]** - [2-3 sentences]

**[Product/topic 2]** - [2-3 sentences]

**Additional:** [interesting discussions, comments, forums, or resources]

---

**Links:** [useful links]
```

If no newsletters are found:

```markdown
### Newsletters - no new emails
@colorSize: LIGHTER
@color: ATHENS_GRAY

No newsletters were received in the last 24 hours.
```

Language:

- Write all tile content in the user's language.
- Keep specific technical terms in English where appropriate, for example
  pre-training, agent, SDK, open-source, human-in-the-loop.

## xTiles write flow

Daily News writes to xTiles Daily by default.

1. Prepare the digest.
2. Show a preview in chat.
3. Ask for explicit approval unless this is a scheduled run.
4. Call `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner` with:
   - `period`: `"day"`
   - `date`: today's local date in ISO 8601
   - `markdown`: the approved digest
5. Archive each processed email by calling `mcp__claude_ai_Gmail__update_thread`
   with `thread_id`, `last_message_id`, and `mark_done: true`. Archive both
   emails used for tiles and emails skipped after reading because they were not
   editorial newsletters.
6. Call `mcp__xtiles__xtiles_get_planner_content` for the same date and period.
7. Confirm with a link: `[Open in xTiles](https://xtiles.app/{view_id})`.

Archive only after the xTiles write succeeds. If the xTiles write fails, do not
archive emails; explain the write error and show the prepared digest in chat.

Tile formatting for xTiles:

```markdown
### Morning News: [Topic]
@colorSize: LIGHTER
@color: [COLOR]

[digest content]
```

- Each `###` section must include color and style annotations immediately after
  the heading, with no blank line between the heading and annotations.
- `@colorSize` is always `LIGHTER`.
- Pick `@color` for each section from:
  `GHOST, CUMULUS, GOSSIP, COLDTURKEY, BLUE_CHALK, MILK_PUNCH, HAWKES_BLUE, PATTENS_BLUE, SAIL, ATHENS_GRAY, BERMUDA, PERFUME, SELAGO, RICE_FLOWER, WHITE_LINEN, POLAR`
- Each section gets a different color. Do not repeat the same color twice in a row.
- Separate each news item with a blank line. Never write items as a continuous block.
- Write all selected newsletter tiles in a single xTiles call. Do not split into
  one call per newsletter.

Use one tile per valuable newsletter, up to 10 tiles. If a newsletter has very
little editorial content, merge it into a short digest tile or skip it.

## Scheduling

After every successful write, always show the schedule widget. In Claude Code
(no Cowork), ask inline: "Want me to run this every morning automatically? What
time should I run it? (default: 9:00 AM)"

- If the user selects **"Yes, schedule it"**, first invoke
  `anthropic-skills:schedule`, then call
  `mcp__scheduled-tasks__create-scheduled-tasks`. Pass to both:
  - **`prompt`**: the full config string assembled from values collected during
    setup:

```text
Run daily news - topic: {topic} . language: {language} . depth: {depth} . destination: xTiles Daily . schedule: daily-{time}
```

    Replace every placeholder with real values. Do not leave placeholders.
  - **`schedule`**: cron expression derived from the time the user selected in
    the widget. The widget sends `cron: HH:MM` in the message - parse that value
    and build the cron: `M H * * *` where H = hour, M = minute. Example: user
    picks 08:30 -> `30 8 * * *`. If no time is found, default to `0 9 * * *`.
  - **`timezone`**: the user's local timezone - call
    `mcp__xtiles__xtiles_get_user_timezone` before scheduling if it has not been
    fetched yet.

  This prompt fires each morning and triggers `daily-news` in scheduled-run
  mode. The full config must be embedded so the setup widget is skipped
  automatically.

  After scheduling succeeds, confirm: "Done - your Daily News will be ready in
  xTiles every morning at [chosen time]." Then show the link to today's already
  created page again using the `view_id` from the write step.
- If the user selects **"No, thanks"**, acknowledge briefly and stop.

## Setup widget HTML

Show this form via `show_widget` at the start of setup in Cowork. In Claude Code
(no Cowork environment), ask the same questions inline. After Submit, the widget
sends a string of answers to chat - process it and continue the flow.

```html
<style>
:root{--bg:#f8f8f8;--card:#fff;--text:#1a1a1a;--muted:#777;--line:#dedede;--soft:#f1f1f1}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:var(--bg);color:var(--text)}
.wrap{max-width:560px;margin:0 auto;background:var(--card);border-radius:16px;padding:26px;box-shadow:0 2px 12px rgba(0,0,0,.08)}
h2{font-size:18px;font-weight:700;margin-bottom:5px}
.sub{font-size:13px;color:var(--muted);line-height:1.45;margin-bottom:20px}
.sec{margin-bottom:18px}
.label{font-size:13px;font-weight:650;margin-bottom:8px}
.hint{font-size:12px;color:var(--muted);margin-top:6px}
input[type=text],input[type=time]{width:100%;padding:10px 12px;border:1.5px solid var(--line);border-radius:10px;font-size:14px;outline:none;background:#fff;color:var(--text)}
input[type=text]:focus,input[type=time]:focus{border-color:#999}
.pills,.cards{display:flex;flex-wrap:wrap;gap:8px}
.pill,.card{padding:8px 13px;border:1.5px solid var(--line);border-radius:10px;background:#fff;font-size:13px;cursor:pointer;user-select:none;transition:all .15s}
.pill:hover,.card:hover{border-color:#aaa}
.pill.sel,.card.sel{background:#1a1a1a;color:#fff;border-color:#1a1a1a}
.card{display:flex;align-items:center;gap:7px}
.chk{width:15px;height:15px;border-radius:4px;border:1.5px solid #aaa;display:flex;align-items:center;justify-content:center;font-size:9px;flex-shrink:0}
.card.sel .chk{background:#fff;border-color:#fff;color:#1a1a1a}
.row{display:grid;grid-template-columns:1fr 1fr;gap:10px}
.btn-row{display:flex;gap:10px;margin-top:22px}
.btn{padding:11px 18px;border-radius:10px;border:none;font-size:14px;font-weight:650;cursor:pointer}
.btn-p{background:#1a1a1a;color:#fff;flex:1}
.btn-p:disabled{opacity:.45;cursor:not-allowed}
.btn-s{background:#eee;color:#444}
@media(max-width:520px){.row{grid-template-columns:1fr}.wrap{padding:22px}}
</style>

<div class="wrap">
  <h2>Daily News</h2>
  <p class="sub">Set up a fresh morning digest from newsletters received in the last 24 hours.</p>

  <div class="sec">
    <div class="label">Topic filter</div>
    <input id="topic" type="text" placeholder="Optional: AI product launches, fintech in Europe, climate policy...">
    <div class="hint">Leave empty to summarize all valuable newsletters from the last 24 hours.</div>
  </div>

  <div class="row">
    <div class="sec">
      <div class="label">Region or market</div>
      <input id="region" type="text" placeholder="Global, US, Europe, Ukraine...">
    </div>
    <div class="sec">
      <div class="label">Language</div>
      <input id="lang" type="text" placeholder="Same as chat">
    </div>
  </div>

  <div class="sec">
    <div class="label">Depth</div>
    <div class="pills" id="depth">
      <div class="pill sel" onclick="pickOne('depth',this,'quick')">Quick brief</div>
      <div class="pill" onclick="pickOne('depth',this,'detailed')">Detailed brief</div>
    </div>
  </div>

  <div class="btn-row">
    <button class="btn btn-s" onclick="sendPrompt('Cancel Daily News setup')">Cancel</button>
    <button class="btn btn-p" id="submit" onclick="submitNews()">Create news brief</button>
  </div>
</div>

<script>
var state={depth:'quick'};
function pickOne(group,el,val){
  document.querySelectorAll('#'+group+' .pill,#'+group+' .card').forEach(function(x){x.classList.remove('sel')});
  el.classList.add('sel');
  state[group]=val;
}
function submitNews(){
  var topic=document.getElementById('topic').value.trim()||'all newsletters';
  var region=document.getElementById('region').value.trim()||'not specified';
  var lang=document.getElementById('lang').value.trim()||'same as chat';
  var msg='Daily News setup - topic: '+topic+' . region: '+region+' . language: '+lang+' . depth: '+state.depth+' . destination: xTiles Daily';
  sendPrompt(msg);
}
</script>
```

## Schedule widget HTML

Show this widget via `show_widget` after a successful write in Cowork. After the
user clicks a button, the widget calls `sendPrompt()` and the response lands in
chat.

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
  <div class="icon">Daily</div>
  <h2>Run this every morning?</h2>
  <p class="sub">I'll fetch fresh news and write your Daily News to xTiles automatically.</p>
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
  sendPrompt('Yes, schedule my daily news at '+label+' every day (cron: '+t+')');
}
</script>
```

## Failure handling

- If Gmail search fails, say mailbox search failed and do not fabricate a digest.
- If an email cannot be opened, skip that email and continue with the remaining
  candidates.
- If xTiles is not connected or returns an error, explain that saving to Daily
  failed and show the prepared digest in chat so the user still has the brief.
- If scheduling fails, keep the created digest and explain that only recurrence
  was not created.

## How to behave

- Use the setup widget for setup; ask inline for approval and any follow-up
  clarifications.
- Show the setup widget in Cowork only. In Claude Code, ask the same questions
  inline.
- After every successful write, show the schedule widget in Cowork. In Claude
  Code, ask the scheduling question inline.
- Never ask the user to copy the digest manually into xTiles. Write to xTiles
  directly, or explain the connector/write error.
- Never create anything without preview and explicit approval, except scheduled
  runs.
- Never put example names, example events, or example links into the preview.
  Only use real data from live web research.
- All clarifying questions after the setup widget should use `AskUserQuestion`,
  not plain text, when that tool is available.
- If context is missing, ask; do not guess.
- If the user gives new information along the way, pick it up immediately.
- Daily is the only period. If the user asks for Weekly or Monthly, tell them
  this skill writes Daily News only and offer to create a Daily page instead.
- Match the user's language and adapt if they switch.

## Quality rules

- Keep the digest scannable; the user should understand the morning picture in
  under 2 minutes.
- Prefer fewer stronger items over many weak mentions.
- Include source links for every substantive item.
- Mention the freshness window, especially when there are no recent updates.
- Avoid sensational language.
- Do not include raw search result dumps.
