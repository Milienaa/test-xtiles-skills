---
name: daily-brief
description: >
  Use when the user wants to set up OR run their xTiles Daily planner —
  a Daily page that serves as a live morning brief from connected tools
  (Slack, Gmail) plus signals that need attention.
  Only the Daily period is supported.

  Setup triggers: "set up my planner", "personalize my workspace",
  "connect my planner to my tools", "create daily",
  "onboard a new xTiles user into the Planner".

  Digest triggers: "show me my morning brief", "what do I need to know today",
  "run my digest". Also runs automatically via scheduled tasks.

  Config is read from the scheduled task prompt — no separate file needed.
  For manual runs: look for config in today's Planner; if there's none, start
  from the survey flow below.

  The self-tuning feedback cycle (the "Tune your digest", "Important keywords",
  and weekly-recap tiles, plus all the config fields they build up) lives in the
  shared digest-tuning sub-workflow — invoked via xtiles_get_workflow("digest-tuning")
  in steps 4 and 7, never reimplemented here.
allowed-tools: >
  mcp__xtiles__xtiles_get_planner_content,
  mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner,
  mcp__xtiles__xtiles_get_user_timezone,
  mcp__xtiles__xtiles_get_workflow,
  mcp__claude_ai_Slack__slack_search_channels,
  mcp__claude_ai_Slack__slack_search_public_and_private,
  mcp__claude_ai_Slack__slack_read_channel,
  mcp__claude_ai_Gmail__search_threads,
  mcp__claude_ai_Gmail__list_labels,
  mcp__claude_ai_Gmail__get_thread,
  mcp__claude_ai_Google_Calendar__list_events,
  mcp__claude_ai_Granola__list_meetings,
  mcp__claude_ai_Google_Drive__list_recent_files,
  mcp__claude_ai_Linear__list_issues,
  mcp__mcp-registry__suggest_connectors,
  anthropic-skills:schedule,
  mcp__scheduled-tasks__create-scheduled-tasks
---

# xTiles Daily Planner — Setup & Daily Digest

## Four principles

1. **Survey first, write to xTiles last.** Nothing gets created until the user has seen the preview and said "yes".
2. **Real data, not placeholders.** Pull from connectors before preview so the user sees live content.
3. **Match the user's language** throughout the entire flow — match the language of the user's first message and adapt if they switch.
4. **Every write is followed by the layout pass.** The moment tiles are created in step 7, re-lay them out into a justified grid via the shared `tile-layout` workflow — automatically, before the CTA, never skipped.

---

## Algorithm

**Period is always Daily.** At the start of the flow, tell the user: "I'll set up your **Daily** planner page — a live morning brief from your connected tools." Never ask which period to set up.

**Run mode — detect before step 1:**
- **Scheduled run**: the incoming message contains `role:`, `tools:`, and `daily_content:` (config injected by the `schedule` skill). Do not show the survey. Extract the config from the message. If a connector from the config is not detected — offer to walk the user through connecting it before continuing. **Skip steps 5 and 6 (preview and approval) — after the fetch, write directly to xTiles. Also skip the schedule widget in step 7 — the task is already scheduled.** Then jump to **step 4 (Silent data fetch)**.
- **Fast-track or fresh manual run**: proceed to step 1.

### 1. Fast-track

If the user is specific ("give me daily for today", "I want to see Slack in the morning") — skip the full survey. Minimum needed: which connectors to pull from — infer from the message, then check the detection table (step 2) to confirm which are actually available. If a required connector is not detected — offer to walk the user through connecting it (see **How to connect connectors**); wait for confirmation before proceeding. Pull only from connectors that are both mentioned and confirmed available. Jump to **step 4**.

If the request is general — run the full flow.
 
---

### 2. Survey — who are you and what's connected

**Before calling `show_widget`**: Make a lightweight test call to each connector's identifying MCP tool (e.g. `list_events` with `maxResults:1` for Calendar, `slack_search_channels` with query `general` for Slack — this is an auth check only, not channel discovery). For any connector that responds without an auth error, pre-select its card in the widget HTML by setting `class="card sel"`. Generate the widget with those pre-selections applied, then call `show_widget`.

**Show the survey widget** (HTML form) in Cowork. In Claude Code (no Cowork environment), ask the same questions inline as plain text — role, tools, content preferences, schedule.

**Connected tools** (multi select, show all regardless of what's actually detected):
- Slack
- Gmail
- Other 


After receiving answers — detect which MCP tools are actually available:

| Connector | Identifying MCP tools                                                                                           |
|-----------|-----------------------------------------------------------------------------------------------------------------|
| Slack     | `mcp__claude_ai_Slack__slack_search_channels`, `mcp__claude_ai_Slack__slack_read_channel`                      |
| Gmail     | `mcp__claude_ai_Gmail__search_threads`, `mcp__claude_ai_Gmail__list_labels`, `mcp__claude_ai_Gmail__get_thread`|
| xTiles    | `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`                                                  |
| Calendar  | `mcp__claude_ai_Google_Calendar__list_events`                                                                   |
| Granola   | `mcp__claude_ai_Granola__list_meetings`                                                                         |
| Google Drive | `mcp__claude_ai_Google_Drive__list_recent_files`                                                             |
| Linear    | `mcp__claude_ai_Linear__list_issues`                                                                            |

These connectors are external and optional — they are not shipped with this plugin. The user must connect them separately.

**For "Other" connectors named by the user** — treat them identically to the known connectors above: attempt detection via available MCP tools; if not detected, walk through connecting via `mcp__mcp-registry__suggest_connectors`. **Before starting the connection flow, say the connector name explicitly** (e.g. "I'll now connect Plaud for you"). After the connection flow completes, explicitly resume: "Plaud connected. Continuing with [full list of tools]…". Carry the full list of selected tools — including every custom connector — through every subsequent step. Never drop a custom connector that the user named, even during multi-step connection flows.

**If xTiles is not connected** — do not continue. Immediately walk the user through connecting xTiles (see **How to connect connectors** below). Wait for confirmation that xTiles is connected before proceeding.

**If a connector the user selected isn't connected** (Gmail, Slack, etc.) — immediately walk them through connecting it step by step. Do not move to the next step until they confirm it's connected or explicitly choose to skip that connector.

---

### 3. Daily content clarification

Question: "What do you want to see on your Daily each morning?"

Options — include only those relevant to connected tools:
- Unread emails that need a reply *(only if Gmail connected)*
- Newsletters — curated summaries from your subscriptions *(only if Gmail connected)*
- Slack messages from key channels *(only if Slack connected)*
- Workload — calendar analysis: day shape, conflicts, focus windows *(only if Calendar connected)*
- Other (describe in next message)

Do NOT suggest tasks — they're already in xTiles by default.

**If Slack is selected and the user has not already named their channels:**

Use the role captured in the form as the anchor for this whole discovery — it drives both the interest search (Step C) and how specialized channels are scored (Step D). The goal is to surface *this specific user's* channels, not a generic company list.

**Step A — universal channels (every role).** Call `mcp__claude_ai_Slack__slack_search_channels` for each of these names: `general`, `all`, `team`, `company`, `announcements`, `product`. Collect every channel that actually exists — these are candidates for the shared/general slot, never for the specialized slot.

**Step B — activity signal (the strongest relevance signal).** Call `mcp__claude_ai_Slack__slack_search_public_and_private` twice: once with query `from:me` (channels where the user actually posts) and once with query `to:me` (channels where the user is @mentioned or replied to). For every result, record the channel and the message timestamp. Per channel, track two things: **recency** — the timestamp of the user's most recent post or mention there, and **frequency** — total hit count across both queries. A channel the user posted or was mentioned in yesterday is more relevant right now than one with more total hits but nothing in weeks — recency is the primary activity signal, frequency only breaks ties between channels with similarly recent activity.

**Step C — role & interest/affinity search.** Reason from the user's role: what does this person actually write and receive in Slack day-to-day? Derive 2–3 short phrases that would naturally appear in messages in their active channels and search for them. Then run a second, broader pass for interest and affinity-group channels that may exist regardless of role — e.g. terms like `women`, `parents`, `wellness`, `book club`, `volunteering`, `pride`, `remote`, `pets`, or other hobby/interest terms suggested by the role context. Do not use a fixed table for either pass — think from context. If the user explicitly named topics or interests, search those first.

**Step D — merge, rank, and select.**
1. Merge every channel found in Steps A–C, removing duplicates (a channel found in more than one step counts once, keeping its highest score).
2. Drop low-signal: name contains `random`, `fun`, `off-topic`, `bots`, `test`, `hiring`, `onboarding`.
3. Score each channel, in this order: **Step B activity ranks highest** — sort by recency of the user's last post/mention there first, then by frequency as the tiebreaker among similarly-recent channels; **then** role/interest/affinity matches from Step C; **then** bare universal presence from Step A alone. A channel where the user was just mentioned yesterday outranks a role-matched channel they never actually post in.
4. Show every remaining discovered channel as a selectable card — cast a wide net across interests and affinity groups so the user has real options to add, not just a token list.
5. **Pre-select (mark active) up to 5 total, no more:** at most 2 from the universal slot (highest-scoring first, e.g. general → all → team → announcements → company → product), and the rest from the highest-scoring specialized channels in Step D's ranking — the channels this specific user actually writes in, is mentioned in, or that match their role/interests. Every other discovered channel stays visible but unchecked — the user decides what else matters to them each morning.

**Fallback:** if Steps B and C both return zero results — call `mcp__claude_ai_Slack__slack_search_public_and_private` with query `team update` and extract channels from those results.

Generate an HTML multi-select widget with the discovered channels as selectable cards — mark the up-to-5 chosen in Step D.5 with the pre-selected `sel` state — and call `show_widget`. Include a free-text input for unlisted channels. Use `sendPrompt()` to submit. Template (inject one card per discovered channel; add `sel` to the class and keep `data-v` in sync for the pre-selected ones):

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:#f8f8f8}
.wrap{max-width:480px;margin:0 auto;background:#fff;border-radius:16px;padding:24px;box-shadow:0 2px 12px rgba(0,0,0,.08)}
h2{font-size:15px;font-weight:700;margin-bottom:14px;color:#1a1a1a}
.cards{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:12px}
.card{padding:6px 14px;border-radius:20px;border:1.5px solid #e0e0e0;font-size:13px;cursor:pointer;background:#fff;user-select:none;transition:all .15s}
.card:hover{border-color:#aaa}
.card.sel{background:#1a1a1a;color:#fff;border-color:#1a1a1a}
input{width:100%;padding:8px 12px;border:1.5px solid #e0e0e0;border-radius:8px;font-size:13px;margin-bottom:14px;outline:none}
input:focus{border-color:#aaa}
.btn{width:100%;padding:11px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;background:#1a1a1a;color:#fff}
</style>
<div class="wrap">
  <h2>Which channels do you open first each morning?</h2>
  <p style="font-size:12px;color:#888;margin:-8px 0 12px">Pre-selected: your most active and relevant channels. Add any others you want to see.</p>
  <div class="cards" id="ch">
    <!-- inject: <div class="card[ sel]" data-v="#channelname" onclick="tog(this,'#channelname')">#channelname</div> — add " sel" to class for the up-to-5 pre-selected channels from Step D.5 -->
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

**If Newsletters is selected:**
**Important:** if the user selected "Newsletters" in the survey widget (step 2), this discovery flow must still run — do not skip it because newsletters was pre-selected there. The survey captures the preference; this step discovers the actual sources.

First, silently call `mcp__claude_ai_Gmail__search_threads` with query `from:(@substack.com OR @beehiiv.com OR @convertkit.com OR @mailchimp.com) newer_than:30d` to discover newsletters already in the inbox. Extract unique sender/publication names from results.

If publications found — call `show_widget` with an HTML multi-select listing the discovered newsletters as selectable cards. Template (inject one card per found publication):

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:#f8f8f8}
.wrap{max-width:480px;margin:0 auto;background:#fff;border-radius:16px;padding:24px;box-shadow:0 2px 12px rgba(0,0,0,.08)}
h2{font-size:15px;font-weight:700;margin-bottom:14px;color:#1a1a1a}
.cards{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:12px}
.card{padding:6px 14px;border-radius:20px;border:1.5px solid #e0e0e0;font-size:13px;cursor:pointer;background:#fff;user-select:none;transition:all .15s}
.card:hover{border-color:#aaa}
.card.sel{background:#1a1a1a;color:#fff;border-color:#1a1a1a}
input{width:100%;padding:8px 12px;border:1.5px solid #e0e0e0;border-radius:8px;font-size:13px;margin-bottom:14px;outline:none}
input:focus{border-color:#aaa}
.btn{width:100%;padding:11px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;background:#1a1a1a;color:#fff}
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

If nothing found — call `show_widget` with a simple text input widget asking the user to name the newsletters they want to track.

Add all selected/typed senders to the config. Tip: newsletters typically come from `@substack.com`, `@beehiiv.com`, `@convertkit.com`, `@mailchimp.com`.

**General rule:** if the user writes something custom — add it as-is. Don't reshape it into a predefined option.

---

### 4. Silent data fetch

**4.0 Digest-tuning — read & apply yesterday's feedback (invoke the sub-workflow).** Before fetching today's data, call `mcp__xtiles__xtiles_get_workflow` with id `digest-tuning` and follow its **Stage A** exactly. Pass it: the current config (all fields from the scheduled-task prompt), yesterday's and today's dates plus today's ISO week (resolve via `xtiles_get_user_timezone`), and whether this is the first-ever digest. **Determine `is_first_ever` = true when the config carries no tuning fields (only role/tools/daily_content, or empty) AND yesterday's planner page has no `### 🎛️ Tune your digest` tile** — i.e. this user has never had a tuned digest before. If either is present, it's not first-ever. Stage A reads yesterday's page, applies the ticked feedback and keyword checkboxes to the config, and hands back the **patched config** and the **applied-changes list**. Carry both forward — the patched config drives today's fetch and pinned-topic tiles; the applied-changes list feeds the feedback tile in step 7. Never reimplement this logic here.

**Silently, without messaging the user**, pull fresh data from connectors based on selected sections and content choices:

- **Gmail — unread emails**: `mcp__claude_ai_Gmail__search_threads` — query `is:important in:inbox newer_than:1d`. For each thread call `mcp__claude_ai_Gmail__get_thread` to get sender, subject, and threadId for the direct link (`https://mail.google.com/mail/u/0/#inbox/{threadId}`).
- **Gmail — newsletters**: `mcp__claude_ai_Gmail__search_threads` — query `from:({sender1} OR {sender2} ... OR @substack.com OR @beehiiv.com OR @convertkit.com) is:unread newer_than:1d` — combine user-named senders with common newsletter domains. Fetch each thread with `get_thread` for a one-line summary and `threadId` for the link.
- **Slack**: two parallel reads:
  1. `mcp__claude_ai_Slack__slack_read_channel` for each chosen channel (top 50 messages). Filter to last 24 hours (timestamp ≥ now − 24 h). Discard older messages. Skip channels with no messages silently.
  2. `mcp__claude_ai_Slack__slack_search_public_and_private` with query `to:me` to find messages where the user was @mentioned or DM'd. Filter results to last 24 hours. This covers both public and private channels, including ones not in the chosen list.

  After collecting, analyse all messages together and group semantically. For every item include a **direct permalink to the specific message** — extract `permalink` from the message object (or build `https://slack.com/archives/{channel_id}/p{ts_without_dot}`). Never link to the channel homepage — always to the individual message.

  **Slack renders as exactly three tiles** (see step 7): `### 🔴 Action Points`, `### ⚡ Mentions`, `### 💬 Topics`. Analyse the messages into the groups below, but only three tiles come out of them — Decisions and Open are folded into the Topics tile, they never get tiles of their own.

  - **Mentions** *(highest priority)* — all messages from the `to:me` search. For each: who mentioned the user, in which channel, what was asked or said — one line per mention, message permalink. If the mention requires a response — flag it as ⚡. → **⚡ Mentions** tile.
  - **Action Points** — every ⚡-flagged mention also becomes an item here: a short poke-style line (what's being asked, second person, same tone as email's 🔴 bucket) plus the message permalink. For each, derive one verb-first action item (e.g. "Відповісти Марії в #product"). → **🔴 Action Points** tile.
  - **Topics** — what was discussed in channels; group by theme, one topic = one line, permalink to the most relevant message, channel attribution `[#channel](permalink)`. → **💬 Topics** tile.
  - **Decisions** — where something was agreed, committed to, or confirmed — include message permalink. Rendered as a `**✅ Decisions**` block inside the **💬 Topics** tile, not as its own tile.
  - **Open questions** — where a question was raised but no clear answer came yet — include message permalink, mark as ⏳. Rendered as a `**❓ Open**` block inside the **💬 Topics** tile, not as its own tile.

- **Calendar**: `mcp__claude_ai_Google_Calendar__list_events` — today's events. For each event extract: start/end time, title, participant names (first name + last name or company), and meeting link (Google Meet, Zoom, or other video URL from event data). Compute:
  - **Summary line**: event count, total hours occupied, longest free focus window (HH:MM–HH:MM, duration in hours)
  - **Per-event row**: time range · title — participants list · [meeting link label](url) if present
  - **🧠 context** (only if Granola or Gmail connected): for each meeting, find the last Granola note involving the same participants and/or the most recent open Gmail thread with the organiser — write one sentence summarising what the meeting is about or what was discussed last time. Only include if relevant context is found; skip silently otherwise.
  - **⚠️ anomalies** — collect all, show at the bottom of the tile (not inline): overlapping events, back-to-back with no gap, events after 20:00, events without description/agenda, potential duplicate titles close together

Classify emails into three buckets. **Newsletters are fetched separately — exclude them here entirely and do not count them in any bucket.**

- 🔴 **Потребує дії** — emails where the user must take a concrete next step (reply, decide, act, log in)
- 🟡 **До уваги** — FYI only: confirmed meetings, signed documents, payments, status updates — past/present tense, nothing to do
- ⚪ **Шум** — notifications, automated alerts, service emails — do not describe individually; count only

**Tone for 🔴 and 🟡 — Poke-style, capitalized:**
- Retell the email, do not copy the subject line. Subject → action → consequence in second person: not "Your account closed" but "Google закрив твій рекламний акаунт учора"
- For 🔴: weave the next step into the sentence: "Залогінься і віднови — вікно на апеляцію обмежене"
- Use people's names, not email addresses. Context in parentheses if needed: "Стефан (influencers.club)"
- Telegraphic, conversational. First letter capitalized, no bureaucratic language.
- 🟡 items are one-liners — no link needed.

For every 🔴 email, derive one verb-first action item (e.g. "Відновити рекламний акаунт Google") — these go into the Emails tile's Action items block. Likewise, every Slack ⚡-flagged mention yields a verb-first action item (e.g. "Відповісти Марії в #product") — these go into the `### 🔴 Action Points` tile's Задачі block (see step 7). Collect all as a flat list — used in preview and tiles.

Use only real data from connectors. Do not invent names, events, or messages.
All names and message content must come directly from API responses — never from examples in this skill file.

If a connector call fails (error, timeout, 401) — record the failure. Do not write "No data" for a failed call — surface the error explicitly in step 5 so the user knows the connector did not respond.

---

### 5. Preview — show content in chat

Show real content with real data. Not structure, not headings with "(TBD)" — actual text.

Format (adapt to selected pages):

```
Here's what I've prepared:

---
📅 DAILY — [actual date]

### Emails
🔴 Потребує дії (N)
- [Poke-style description — 1–2 sentences, second person, action + consequence]
  → [Відкрити лист](https://mail.google.com/mail/u/0/#inbox/{threadId})

- [Next 🔴 email, same format]
  → [Відкрити лист](https://mail.google.com/mail/u/0/#inbox/{threadId})

🟡 До уваги (N)
- [One-line item — no link]
- [One-line item]

⚪ Шум
- N сповіщень (sources) — нічого термінового

**Action items:**
- [ ] [verb-first task from 🔴 email 1]
- [ ] [verb-first task from 🔴 email 2]

*(omit Action items entirely if no 🔴 emails)*

### 📧 Newsletters

**[Newsletter Name](https://mail.google.com/mail/u/0/#inbox/{threadId})**
One-line summary.

**[Another Newsletter](https://mail.google.com/mail/u/0/#inbox/{threadId})**
One-line summary.

### 🔴 Action Points
- [Poke-style one-liner of what's being asked] — [#channel](url)

**Задачі**
- [ ] [verb-first task]

### ⚡ Mentions
- **@Name** in [#channel](url) — what they asked/said ⚡

### 💬 Topics
**Channels:** #channel1 (N) · #channel2 (N)
- **[Topic name]** — [one-sentence summary] — [#channel](url)

**✅ Decisions**
- [Decision] — [#channel](url)

**❓ Open**
- [Question] — [#channel](url) ⏳

### 📅 Calendar
**N events · ~X h occupied · longest focus window HH:MM–HH:MM (X h)**

**HH:MM–HH:MM · Meeting name** — Participant1, Participant2 · [Google Meet](url)

🧠 [one sentence: meeting point / what will be discussed]

**HH:MM–HH:MM · Meeting name**

**HH:MM–HH:MM · Meeting name** — Participant from Company · [Google Meet](url)

🧠 [meeting point]

⚠️ [anomaly — e.g. two external calls back-to-back in the evening, 30 min gap between them]

---
```

Each 🔴 email uses the real `threadId` from `get_thread` for the [Відкрити лист] link. 🟡 items are one-liners with no link.
Each newsletter is shown as its own named section in the preview — never mixed into the Emails section.
Separate each item with a blank line for readability.

**Rules:**
- Show only selected sections the user asked for
- If a connector returned no data — write exactly that ("No unread emails", "No newsletters today", "No Slack updates today") — never skip the section silently; its absence looks like a bug
- If a connector call failed — write "Could not fetch [connector] data — connector error" (not "No data")
- No placeholder names, example events, or invented data — ever
- Include the digest-tuning meta tiles in the preview too — compose them by following digest-tuning Stage B, which defines which tiles appear and their order. Don't fabricate them here.
- After the preview, **stop and wait**. Do not write anything to xTiles yet.
---

### 6. Approval

**Mandatory. Never skip this step.** After showing the preview, call `show_widget` with the **Approval widget HTML** (see below).

Do not call `xtiles_create_tiles_from_markdown_in_my_planner` until the user explicitly clicks **"Looks good — create it"**.

If the user asks for a change — clarify exactly what, update only that section, re-show preview, ask again.

---

### 7. Write to xTiles

**Only after explicit approval.**

Tool: `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`
- `period`: "day"
- `date`: current date in ISO 8601

**Write all sections in a single call.** Combine all selected connectors (Gmail, Slack, etc.) into one markdown and call the tool once — never split into separate calls per connector.

**Digest-tuning — compose the meta tiles (invoke the sub-workflow).** The meta tiles come from `digest-tuning` Stage B: call `mcp__xtiles__xtiles_get_workflow` with id `digest-tuning` and follow its Stage B exactly, passing the config (as patched in step 4.0), today's composed content tiles, the dates/ISO week, and the applied-changes list from Stage A. It returns the markdown for the meta tiles it decides to emit, plus config patches to persist. **Manual runs** already ran Stage B at preview time (step 5) — reuse that output here, don't re-run it. **Scheduled runs** (no preview) invoke Stage B here, right before the write. Either way, include the returned tiles in the same single write as the content tiles — never a separate call or a chat widget. Their order and placement are digest-tuning's concern, not this skill's.

**Do NOT create a date/header tile.** Never write `### [date]` or any title-only tile as the first item — start directly with content tiles.

**Tile formatting** — each `###` section must include color and style annotations immediately after the heading (no blank line between):

```
### [emoji] [Title]
@colorSize: LIGHTER
@color: [COLOR]

[content]
```

- `@colorSize` is always `LIGHTER`
- `@color` — pick randomly for each section from this list **exactly as written**:
  `GHOST, CUMULUS, GOSSIP, COLDTURKEY, BLUE_CHALK, MILK_PUNCH, HAWKES_BLUE, PATTENS_BLUE, SAIL, ATHENS_GRAY, BERMUDA, PERFUME, SELAGO, RICE_FLOWER, WHITE_LINEN, POLAR`
  **CRITICAL: never use semantic color names (RED, BLUE, GREY, ORANGE, YELLOW, GREEN, etc.) — they will not render. Only the exact names from the list above.**
- Each section gets a different color — do not repeat the same color twice in a row

**Content formatting inside each tile:**
- **All links must be Markdown hyperlinks** — always `[text](url)`, never a bare URL. If you include a link, it must have a label.
- Separate each item with a blank line — never write items as a continuous block
- **Emails**: structure the tile in three labeled blocks followed by action items:
  ```
  🔴 **Потребує дії (N)**

  - [Poke-style description — 1–2 sentences, second person, action + consequence]
    → [Відкрити лист](https://mail.google.com/mail/u/0/#inbox/{threadId})

  - [Next 🔴 item, same format]
    → [Відкрити лист](url)

  🟡 **До уваги (N)**

  - [One-line item — no link, never]

  - [One-line item]

  ⚪ **Шум**

  - N сповіщень (sources) — нічого термінового

  ---

  **Action items**

  - [ ] [Verb-first task from 🔴 email 1]
  - [ ] [Verb-first task from 🔴 email 2]
  ```
  Omit `Action items` section entirely if no 🔴 emails. Newsletters are in the separate `### 📧 Newsletters` tile — never include them here.
- **Newsletters**: ALL newsletters go in a **single `### 📧 Newsletters` tile** — never create a separate tile per newsletter. Structure:
  - Each newsletter as a bold hyperlink title followed by a one-line summary on the next line:
    ```
    **[Newsletter Name](https://mail.google.com/mail/u/0/#inbox/{threadId})**
    One-line summary.
    ```
  - Blank line between entries.
  - The link IS the title — no separate "Open" button or link at the bottom of each entry.
  - Omit the entire tile only if there are no unread newsletters at all.
- **Slack**: split into **exactly three tiles** — `### 🔴 Action Points`, `### ⚡ Mentions`, `### 💬 Topics` — never one big tile, and never more than these three (Decisions and Open are blocks inside the Topics tile, not tiles of their own). Each tile uses `###` as its header. All Slack links must point to the specific message permalink, never to the channel homepage.
  - `### 🔴 Action Points` — the actionable subset: one line per ⚡-flagged mention: `- [Poke-style one-liner of what's being asked] — [#channel](message_permalink)`. Below that, a `**Задачі**` block with one verb-first checkbox per item: `- [ ] [verb-first task]` (e.g. "Відповісти Марії в #product"). **Omit tile entirely if no ⚡ mentions today.** This tile is a rollup, not a replacement — the same messages still appear in `### ⚡ Mentions` below for full context.
  - `### ⚡ Mentions` — one line per mention: `- **@Name** in [#channel](message_permalink) — what they asked/said`. Add ` ⚡` if a response is needed. **Omit tile entirely if no mentions.**
  - `### 💬 Topics` — the discussion rollup. First line: `**Channels:** #channel1 (N) · #channel2 (N)`. Then one line per topic: `- **Topic name** — one-sentence summary — [#channel](message_permalink)`. Then fold decisions and open questions into this same tile as labeled blocks (never separate tiles):
    - a `**✅ Decisions**` block — one line per decision: `- Decision made — [#channel](message_permalink)`; omit the block if there are no decisions.
    - a `**❓ Open**` block — one line per unanswered question: `- Question — [#channel](message_permalink) ⏳`; omit the block if there are none.
    **Always create this Topics tile** — if no messages today, write a single line: `No updates today.` Its absence looks like a connector failure.
- **Calendar**: tile titled `### 📅 Calendar`. Use this exact structure:
  ```
  ### 📅 Calendar
  @colorSize: LIGHTER
  @color: [pick randomly from the color list]

  **N events · ~X h occupied · longest focus window HH:MM–HH:MM (X h)**

  **HH:MM–HH:MM · Meeting name** — Participant1, Participant2 · [Google Meet](url)

  🧠 [one sentence: meeting point / what will be discussed]

  **HH:MM–HH:MM · Meeting name**

  **HH:MM–HH:MM · Meeting name** — Participant from Company · [Zoom](url)

  🧠 [meeting point]

  ⚠️ [anomaly]
  ```
  Rules:
  - Summary line is bold, always first
  - Each event on its own bold line: `**HH:MM–HH:MM · Title**` — append ` — Participants · [Link label](url)` if participants or meeting link exist
  - 🧠 goes on the next paragraph directly under its event — only if Granola or Gmail context was found; omit otherwise. Use Markdown hyperlinks for Granola note and Gmail thread: `[Note title](url)` + `[Thread subject](gmail-url)`
  - All ⚠️ anomalies collected at the bottom, one per line
  - Blank line between every item (event, 🧠, ⚠️) for readability
  - Omit tile entirely if Calendar returned no events
- This ensures the tile is scannable, not a wall of text

**If xTiles is not connected** — do not output the digest as plain text in chat. Walk the user through connecting xTiles (see **How to connect connectors**), wait for confirmation, then write.

**If the page already exists:**
1. Call `mcp__xtiles__xtiles_get_planner_content`
2. Compare existing H3 headers (`###`) with what you're about to add
3. Append only sections whose headers don't exist yet
4. If everything already exists — ask: replace all, append anyway, or cancel?

**After each successful write — run these four steps in order, no exceptions:**

1. Write `✅ Daily created.`

2. **Layout pass — mandatory stage after adding tiles. Runs on every write (scheduled runs included); never skipped, never deferred, never asked about.** Freshly written tiles land in a default stack — re-lay them out *now*, before the CTA and schedule widgets below:
   - Read `view_id` and `tile_ids` straight from the `xtiles_create_tiles_from_markdown_in_my_planner` response (`tile_ids` is ordered to match the `###` sections you just wrote). Keep `view_id` — step 3 reuses it, do not re-fetch it.
   - Call `mcp__xtiles__xtiles_get_workflow` with id `tile-layout` and follow it exactly: pass `tile_ids` as its "added tiles", the markdown you just wrote as their content, and these **layout hints** — 1–4 tiles · default 2 per row · give a heavy tile (usually 💬 Topics or Emails) its own full-width row.
   - Apply the layout silently — no message, no confirmation. Only once it is applied, continue to step 3.

3. Reuse the `view_id` from the write response (step 2) — **do not call `get_planner_content` to re-derive it.** Call `show_widget` with the **CTA widget HTML** (see below), replacing `{VIEW_URL}` with `https://xtiles.app/{view_id}`. Translate the button label into the user's language. **Never output a markdown link instead of the widget.**
4. **For non-scheduled runs only**: Immediately call `show_widget` with the **Schedule widget HTML** (see below). Do not skip this step, do not ask first — just show it.

**If an error occurs:** briefly say what went wrong, offer to retry or skip that page.

---

### 8. Schedule (optional)

The schedule widget is shown in step 7 above. This step handles the user's response.

In Claude Code (no Cowork): after writing, ask inline: "Want me to run this every morning automatically? What time? (default: 9:00 AM)"

- If the user selects **"Yes, schedule it"** — first invoke `anthropic-skills:schedule`, then call `mcp__scheduled-tasks__create-scheduled-tasks`. Pass to both:
  - **`prompt`**: the full config string assembled from values collected during setup —
    ```
    Run daily digest — role: {role} · tools: {tools} · daily_content: {content} · schedule: daily-{HH:MM} days:{days}
    ```
    Replace `{role}`, `{tools}`, `{content}`, `{HH:MM}`, and `{days}` with the actual values parsed from the widget response. Do not leave placeholders. Once digest-tuning starts producing config patches, append whichever of its fields (`pinned_topics`, `trial_topics`, `detail_sections`, `secondary_channels`, `digest_format`, `tone_style`, `signal_trail`, `question_history`, `highlighted_keywords`, `keyword_cooldown`, `tuning_log`, `last_recap`) are non-empty to this same string when re-creating the scheduled task — this is how those fields persist across runs. Use `;` to separate multiple entries within a single field (names may contain commas).
  - **`schedule`**: cron expression derived from the widget. The widget sends `cron: HH:MM days:1-5` or `cron: HH:MM days:*` — parse both values: time gives H and M, days gives the weekday field. Build: `M H * * {days}`. Examples: `cron: 08:30 days:1-5` → `30 8 * * 1-5` · `cron: 08:30 days:*` → `30 8 * * *`. Default if missing: `0 9 * * 1-5`.
  - **`timezone`**: the user's local timezone — call `mcp__xtiles__xtiles_get_user_timezone` to get it before scheduling if it hasn't been fetched yet.

  This prompt fires each morning and triggers `daily-brief` in scheduled-run mode — the full config must be embedded so the survey is skipped automatically.

  After scheduling succeeds, confirm: "Done — your Daily will be ready in xTiles every morning at [chosen time]." Then call `show_widget` with the **CTA widget HTML**, replacing `{VIEW_URL}` with `https://xtiles.app/{view_id}` (the same `view_id` from step 7). Never output a markdown link here — always the button widget.
- If the user selects **"No, thanks"** — acknowledge briefly.

Either way, continue to **step 9 (Related workflows)** — do not stop here.

---

### 9. Related workflows

**After every manual run, once step 8 is resolved** (scheduled or declined) —
offer related workflows. Skip this on scheduled runs, which end silently
after step 7.

Ask via `AskUserQuestion` (single select): "Want to set up anything else on
xTiles?"
- 🌙 Evening Reflection — an end-of-day synthesis seeded for tomorrow
- 📰 Today News — a daily news digest on topics you care about
- 📊 Weekly Review — a weekly summary of what moved forward this week
- Nothing else, thanks

**Never list these as plain text requiring the user to retype a choice —
always use the interactive question.**

On selection, send the exact matching phrase to hand off to that skill (do
not attempt to run it yourself):
- Evening Reflection → `Set workflow of Evening Reflection (evening-reflection) on xTiles MCP`
- Today News → `Set workflow of Today News (today-news) on xTiles MCP`
- Weekly Review → `Set workflow of Weekly Review (weekly-review) on xTiles MCP`
- "Nothing else" — acknowledge briefly and stop.

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

## Approval widget HTML

Show this via `show_widget` after the preview in step 6. If the user clicks "Change something" — ask what to change in plain text, update that section, re-show the preview, then show this widget again.

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
  <button class="btn btn-yes" id="btn-yes" onclick="approve()">✓ Looks good — create it</button>
  <button class="btn btn-edit" id="btn-edit" onclick="edit()">Edit</button>
  <button class="btn btn-cancel" id="btn-cancel" onclick="cancel()">Cancel</button>
</div>
<script>
function collapse(msg){document.querySelector('.btns').innerHTML='<p style="font-size:13px;color:#aaa;text-align:center;padding:4px 0">'+msg+'</p>';}
function approve(){collapse('⏳ Creating…');sendPrompt('Looks good — create it');}
function edit(){collapse('✓ Got it');sendPrompt('Change something');}
function cancel(){collapse('✓ Cancelled');sendPrompt('Cancel');}
</script>
```

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
<button class="btn" id="btn-done" onclick="doneIt()">✓ Done</button>
<script>
function doneIt(){var b=document.getElementById('btn-done');b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';b.textContent='⏳…';sendPrompt('Done — connectors connected, continue the flow');}
</script>
```

---

## Survey widget HTML

Show this form via `show_widget` at the start of setup in Cowork.
After Submit, the user sends a string of answers to chat — process it and continue the flow.

```html
<style>
    :root{--color-background-primary:#fff;--color-background-secondary:#f5f5f5;--color-background-tertiary:#f8f8f8;--color-text-primary:#1a1a1a;--color-text-secondary:#888;--color-border-secondary:#aaa;--color-border-tertiary:#e0e0e0}
    *{box-sizing:border-box;margin:0;padding:0}
    body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:var(--color-background-tertiary);color:var(--color-text-primary)}
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
        <div class="card" onclick="togTool(this,'Slack')"><div class="chk">✓</div>Slack</div>
        <div class="card" onclick="togTool(this,'Gmail')"><div class="chk">✓</div>Gmail</div>
        <div class="card" onclick="togTool(this,'Calendar')"><div class="chk">✓</div>Calendar</div>
        <div class="card" onclick="togTool(this,'Granola')"><div class="chk">✓</div>Granola</div>
        <div class="card" onclick="togTool(this,'Linear')"><div class="chk">✓</div>Linear</div>
        <div class="card" onclick="togTool(this,'GitHub')"><div class="chk">✓</div>GitHub</div>
        <div class="card" onclick="togTool(this,'GoogleDrive')"><div class="chk">✓</div>Google Drive</div>
        <div class="card" onclick="togTool(this,'Gamma')"><div class="chk">✓</div>Gamma</div>
        <div class="card" onclick="togTool(this,'Figma')"><div class="chk">✓</div>Figma</div>
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
  'GitHub':      {daily:['GitHub — PRs & review requests']},
  'GoogleDrive': {daily:['Google Drive — shared files updated']},
  'Gamma':       {daily:['Gamma — presentations updated']},
  'Figma':       {daily:['Figma — design updates & comments']}
};
var AM=[];

var ROLE_DEFAULTS={
  'Product Manager':   ['Slack messages — work chat signals','Important emails — unread inbox','Workload — calendar analysis','Linear issues — new & updated','Granola — meeting notes & summaries'],
  'Designer':          ['Figma — design updates & comments','Slack messages — work chat signals','Important emails — unread inbox','Workload — calendar analysis'],
  'Engineer':          ['GitHub — PRs & review requests','Slack messages — work chat signals','Important emails — unread inbox','Linear issues — new & updated','Workload — calendar analysis'],
  'Growth & Marketing':['Important emails — unread inbox','Newsletters — curated summaries','Slack messages — work chat signals','Gamma — presentations updated','Workload — calendar analysis'],
  'Founder / CEO':     ['Slack messages — work chat signals','Important emails — unread inbox','Newsletters — curated summaries','Granola — meeting notes & summaries','Workload — calendar analysis'],
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

---

## CTA widget HTML

Show this immediately after a successful write. Replace `{VIEW_URL}` with the real xTiles page URL before calling `show_widget`.

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

Show this widget via `show_widget` after a successful write in Cowork.
After the user clicks a button, the widget calls `sendPrompt()` and the response lands in chat.

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
  <div class="icon">⏰</div>
  <h2>Run this every morning?</h2>
  <p class="sub">I'll fetch your signals and write your Daily to xTiles automatically — no need to ask each time.</p>
  <div class="time-row">
    📅 Every
    <select id="sched-days">
      <option value="1-5" selected>Weekdays</option>
      <option value="*">Day</option>
    </select>
    at <input type="time" id="sched-time" value="09:00">
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
  var t=document.getElementById('sched-time').value||'09:00';
  var parts=t.split(':'),h=parseInt(parts[0],10),m=parts[1];
  var label=(h%12||12)+':'+m+' '+(h>=12?'PM':'AM');
  var dLabel=days==='1-5'?'weekdays':'every day';
  collapse('⏳ Scheduling…');
  sendPrompt('Yes, schedule my daily digest at '+label+' '+dLabel+' (cron: '+t+' days:'+days+')');
}
function noThanks(){collapse('✓ Got it');sendPrompt('No schedule needed');}
</script>
```

---

## How to behave

- Use the survey widget for setup; ask inline for approval and any follow-up clarifications
- **Never output the digest as plain text in chat** and ask the user to copy it manually — always write to xTiles directly, or walk through connecting xTiles first
- **Never skip a connector** the user selected — if it's not connected, walk through the connection before continuing, don't silently drop it
- Never create anything without preview and explicit approval
- Never put example names, example events, or example messages into the preview — only real data from connectors
- **All clarifying questions and approvals after the main survey form** (channel selection, newsletter names, approval, change requests) — use `show_widget` with HTML, never `AskUserQuestion` or plain text
- If context is missing — ask, don't guess
- If the user gives new information along the way — pick it up, don't wait for the "right step"
- Real data from connectors always beats placeholders
- Daily is the only period. If the user asks for Weekly or Monthly, tell them only the Daily planner is currently supported and offer to create a Daily page instead — never silently downscope.
- Match the user's language, adapt if they switch
- Show the survey widget in Cowork only — in Claude Code, ask the same questions inline
- The "Tune your digest", "Important keywords", and weekly-recap tiles are owned by the shared **digest-tuning** sub-workflow — invoke it (steps 4.0 and 7), never hand-roll those tiles, their order/placement, or their config logic here. They're written to xTiles as part of the digest's single write, never a chat widget.
 