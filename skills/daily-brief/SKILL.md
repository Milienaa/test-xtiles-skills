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

  Environment: this is the Claude / Cowork variant — it renders `show_widget`
  and `AskUserQuestion`. In ChatGPT Work, where every form is an inline
  `ask_user_input` / `genui` surface and `show_widget` does not exist, use
  `daily-brief-with-gpt` instead.

  Environment triggers: "Daily Brief in Claude", "the Claude version",
  "Claude Daily Brief".
allowed-tools: >
  mcp__xtiles__xtiles_get_planner_content,
  mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner,
  mcp__xtiles__xtiles_patch_view_content,
  mcp__xtiles__xtiles_get_content_by_link,
  mcp__xtiles__xtiles_create_notification,
  mcp__xtiles__xtiles_list_tasks,
  mcp__xtiles__xtiles_update_task,
  mcp__xtiles__xtiles_get_user_timezone,
  mcp__xtiles__xtiles_get_workflow,
  mcp__xtiles__xtiles_list_calendar_events,
  mcp__claude_ai_Slack__slack_search_channels,
  mcp__claude_ai_Slack__slack_search_public_and_private,
  mcp__claude_ai_Slack__slack_read_channel,
  mcp__claude_ai_Gmail__search_threads,
  mcp__claude_ai_Gmail__list_labels,
  mcp__claude_ai_Gmail__get_thread,
  mcp__claude_ai_Gmail__create_draft,
  mcp__claude_ai_Gmail__unlabel_thread,
  mcp__claude_ai_Google_Calendar__list_events,
  mcp__claude_ai_Granola__list_meetings,
  mcp__claude_ai_Google_Drive__list_recent_files,
  mcp__claude_ai_Linear__list_issues,
  recent_chats,
  conversation_search,
  mcp__mcp-registry__suggest_connectors,
  anthropic-skills:schedule,
  mcp__scheduled-tasks__create-scheduled-tasks
---

# xTiles Daily Planner — Setup & Daily Digest

## Four principles

1. **Survey first, write to xTiles last.** Nothing gets created until the user has seen the preview and said "yes".
2. **Real data, not placeholders.** Pull from connectors before preview so the user sees live content.
3. **Match the user's language** throughout the entire flow — match the language of the user's first message and adapt if they switch. On a **scheduled run** there is no user message: use the language of the scheduled-task config prompt, and if that is ambiguous, the language of the fetched content (emails, Slack). Default to English when still unclear. **Every template, label, and example in this file is written in English as a placeholder** (`Needs action`, `FYI`, `Noise`, `Open email`, `Tasks`, `Decisions`, `Open`, …) — translate all of them into the detected language when composing the preview and the tiles. Never emit a label in a language the user has not used.
4. **Every write is followed by the layout pass.** The moment tiles are created in step 7, re-lay them out into a justified grid via the shared `tile-layout` workflow — automatically, before the CTA, never skipped.
5. **The only deliverable is tiles written to xTiles.** Never render the digest as an HTML artifact, a Cowork canvas, or plain text in chat — and never stop after producing one. An artifact is *never* a substitute for the xTiles write; if xTiles can't be written to, connect it first. The run is complete only when the tiles are in xTiles **and** the full post-write sequence has run: layout pass → CTA button → schedule widget → related-workflows question. Skipping any of these — or ending with an artifact instead — is a failed run, not a shortcut.

---

## Algorithm

**Period is always Daily.** At the start of the flow, tell the user: "I'll set up your **Daily** planner page — a live morning brief from your connected tools." Never ask which period to set up.

**Run mode — detect before step 1:**
- **Scheduled run**: the incoming message contains `role:`, `tools:`, and `daily_content:` (config injected by the `schedule` skill). Do not show the survey. Extract the config from the message, including `carry_over_tasks:` if present (default `false` for older scheduled configs that predate this setting). If a connector from the config is not detected — offer to walk the user through connecting it before continuing. **Skip steps 5 and 6 (preview and approval) — after the fetch, write directly to xTiles. Also skip the schedule widget in step 7 — the task is already scheduled.** Then jump to **step 4 (Silent data fetch)**.
- **Fast-track or fresh manual run**: proceed to step 1.

### 1. Fast-track

If the user is specific ("give me daily for today", "I want to see Slack in the morning") — skip the full survey. Minimum needed: which connectors to pull from — infer from the message, then check the detection table (step 2) to confirm which are actually available. If a required connector is not detected — offer to walk the user through connecting it (see **How to connect connectors**); wait for confirmation before proceeding. Pull only from connectors that are both mentioned and confirmed available. Jump to **step 4**.

If the request is general — run the full flow.
 
---

### 2. Survey — who are you and what's connected

**Before calling `show_widget`**: Make a lightweight test call to each connector's identifying MCP tool (e.g. `list_events` with `maxResults:1` for Calendar, `slack_search_channels` with query `general` for Slack — this is an auth check only, not channel discovery). For any connector that responds without an auth error, pre-select its card in the widget HTML by setting `class="card sel"`. **The Claude card is exempt from this check — it has no auth to test — but it is *not* pre-selected: chat history is pulled only if the user selects it.** **Calendar (xTiles) has no auth check to run either** — `xtiles_list_calendar_events` never returns an auth error for "no calendar linked"; an empty result is indistinguishable from a genuinely empty day, by the tool's own description. Don't fake a pre-selection signal that doesn't exist: leave its card unselected like any untested connector, let the user opt in explicitly, and resolve the "maybe not linked" ambiguity later — see step 4. Generate the widget with those pre-selections applied, then call `show_widget`.

**Show the survey widget** (HTML form) in Cowork. In Claude Code (no Cowork environment), ask the same questions inline as plain text — role, tools, content preferences, schedule.

**Connected tools** (multi select, show all regardless of what's actually detected):
- Claude — your own past chats, read with `recent_chats`, **off by default — pick it to include chat history**
- Calendar (xTiles) — today's events from whatever calendars are connected inside xTiles, read with `mcp__xtiles__xtiles_list_calendar_events`
- Slack
- Gmail
- Calendar — an additional, directly-connected Google account whose events aren't already synced into xTiles
- Other — a free-text field where the user names any connector that isn't on the cards

**Claude is a source, not just the thing reading the sources — but it is off by default.** The user's own past chats hold real signal: what they asked for yesterday, what they said they'd do, what was decided, what was left half-finished. It is *their chat history* that is the source — read through `recent_chats` — not the conversation this run happens to be in; never mine the current session's context as a stand-in. **Leave the Claude card unselected and keep `Claude` out of the initial `tools` set** — chat content reaches xTiles only when the user affirmatively selects it, exactly like any other source. It needs **no connector and no auth**, so once picked it always works: even on a free plan with nothing else connected, a Claude-only digest the user opted into is still a real digest.

**Calendar (xTiles) is a distinct, optional card — not pre-selected, same as Slack or Gmail.** It aggregates whatever Google or Outlook calendars the user connected inside their xTiles account, read via `mcp__xtiles__xtiles_list_calendar_events`. It is **not** exempt like Claude, but it also can't be auth-tested like Slack or Gmail — the tool has no error path for "no calendar linked," so a successful test call proves nothing. Show its card unselected and let the user opt in; if they select it and it turns out nothing is linked, that surfaces downstream (step 4) as an empty result, not as an upfront connection failure. **Calendar stays a separate, optional card, unchanged from before** — that one is for a Google account connected *directly*, outside xTiles, whose events aren't already synced there, and it does get the normal auth-error test. When both are selected, their events are merged into the same single `### 📅 Workload` tile in step 4 — never two separate calendar tiles.

**The "add your own connector" field is mandatory and must never disappear.** When you regenerate the survey HTML to apply pre-selections, keep the `➕ Don't see your tool? Add your own connector` block (`#other-tool` input + `previewCustom()` + `readCustom()`) exactly as written — pre-selection only ever adds ` sel` to card classes, it never removes markup. The catalog is finite and the user's stack is not; this field is the only way someone can bring in a tool we don't ship a card for, and losing it is a setup failure, not a cosmetic one. Same in Claude Code (no widget): after listing the tool cards inline, always ask explicitly — "Any other tool you want me to pull from? Name it and I'll connect it."

**Present the full embedded tool catalog — do not rely on a live registry to know what can be offered.** The complete list of supported tools is baked into this skill below (**Supported tools — embedded catalog**). Always show the full menu regardless of what is detected. This is the whole point on a **free ChatGPT or Claude plan**, where the connector registry (`mcp__mcp-registry__suggest_connectors`) and dynamic detection aren't available at all: the embedded catalog is what still lets the user see every tool and pick sources — it is the **base for the first digest** when no other connectors exist.

After receiving answers — detect which MCP tools are actually available, using the embedded catalog as the source of truth for what exists:

#### Supported tools — embedded catalog

This table is the authoritative, static list of tools this skill supports. It is embedded here on purpose so the skill works even when no registry or detection is available (free ChatGPT / Claude). Treat every row as offerable at all times; use the "Identifying MCP tools" column only to check whether a tool is *currently connected*, never to decide whether it *exists*.

| Connector | Identifying MCP tools | Contributes |
|-----------|-----------------------|-------------|
| **Claude (past chats)** | `recent_chats`, `conversation_search` | **Off by default, no connector needed** — include only if the user picks it; from yesterday's chats: unfinished threads, open questions & decisions |
| Calendar (xTiles) | `mcp__xtiles__xtiles_list_calendar_events` | Aggregates whatever Google/Outlook calendars are connected inside the user's xTiles account: workload analysis, agendas, prep, focus windows |
| Slack     | `mcp__claude_ai_Slack__slack_search_channels`, `mcp__claude_ai_Slack__slack_read_channel` | Channel signals, mentions, action points |
| Gmail     | `mcp__claude_ai_Gmail__search_threads`, `mcp__claude_ai_Gmail__list_labels`, `mcp__claude_ai_Gmail__get_thread` | Unread emails, newsletters, per-topic tiles |
| xTiles    | `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner` | The Daily page itself (required) |
| Calendar | `mcp__claude_ai_Google_Calendar__list_events` | Optional, additional — events from a Google account connected directly (not already synced into xTiles); merged into the same Workload tile as Calendar (xTiles) |
| Granola   | `mcp__claude_ai_Granola__list_meetings` | Meeting notes & summaries |
| Google Drive | `mcp__claude_ai_Google_Drive__list_recent_files` | Recently shared/updated files |
| Linear    | `mcp__claude_ai_Linear__list_issues` | New & updated issues |
| GitHub    | *GitHub MCP tools when connected* | PRs & review requests |
| Gamma     | *Gamma MCP tools when connected* | Presentations updated |
| Figma     | *Figma MCP tools when connected* | Design updates & comments |
| **Your own connector** | *any MCP tools exposed by the tool the user names* | Whatever the user asks for in step 3 — always offerable, never a closed list |

These connectors are external and optional — they are not shipped with this plugin. The user must connect them separately. Keep this catalog in sync with the survey widget's tool cards — the widget renders one card per catalog row.

**For "Other" connectors named by the user** — treat them identically to the known connectors above: attempt detection via available MCP tools; if not detected, walk through connecting via `mcp__mcp-registry__suggest_connectors`. **Before starting the connection flow, say the connector name explicitly** (e.g. "I'll now connect Plaud for you"). After the connection flow completes, explicitly resume: "Plaud connected. Continuing with [full list of tools]…". Carry the full list of selected tools — including every custom connector — through every subsequent step. Never drop a custom connector that the user named, even during multi-step connection flows.

**If xTiles is not connected** — do not continue. Immediately walk the user through connecting xTiles (see **How to connect connectors** below). Wait for confirmation that xTiles is connected before proceeding.

**If a connector the user selected isn't connected** (Gmail, Slack, etc.) — immediately walk them through connecting it step by step. Do not move to the next step until they confirm it's connected or explicitly choose to skip that connector. **Calendar (xTiles) can't be checked this way** — it never fails auth, so "not connected" only shows up later as an empty result (see step 4).

---

### 3. Daily content clarification

Question: "What do you want to see on your Daily each morning?"

Options — include only those relevant to connected tools:
- From our chats — what you left unfinished or unanswered yesterday *(only if the user selected Claude — no connector required)*
- Unread emails that need a reply *(only if Gmail connected)*
- Newsletters — curated summaries from your subscriptions *(only if Gmail connected)*
- Emails by topic — group your inbox into separate thematic tiles *(only if Gmail connected)*
- Emails from key people — a VIP-sender tile *(only if Gmail connected)*
- Follow-ups — threads awaiting your reply *(only if Gmail connected)*
- Slack messages from key channels *(only if Slack connected)*
- Workload — what each meeting is about, what to prepare for it, and where your focus time actually is *(only if Calendar (xTiles) and/or Calendar connected)*
- Other (describe in next message)

Do NOT suggest tasks — they're already in xTiles by default.

**Carry over overdue tasks?** A single-select toggle in the same Survey widget
(Step 2 of 2, below the content checklist; ask the same question inline in
Claude Code): "Carry over tasks still open from the last 2 days to today?" —
**Yes, move them to today** / **No, leave them where they are** (default).
Store the answer as `carry_over_tasks: {true/false}` in the config. This is
about rescheduling existing open tasks, not the digest's own content — a
separate toggle, not one of the options list above.

**Email is often the main source — always ask what the user actually wants from it.** Don't stop at a single yes/no on "unread emails". When Gmail is connected, explicitly ask what they want to pull out of their inbox and how it should be split — a frequent and high-value case is a user whose email is their primary signal, which is best broken down into **several thematic tiles by question/topic** (e.g. one tile per project, client, or recurring subject) rather than one generic Emails tile. If the user names topics/projects/senders, capture each one — they become the tile breakdown in step 7. If they don't, default to the single `### 📩 Emails` tile.

**Never skip this clarification step.** Even on a fast-track or when the user was terse, run it — it's where newsletters and custom apps get scoped. If Gmail is connected, the Newsletters option must be offered explicitly (don't silently omit it); if the user shows interest, run the newsletter-discovery flow below.

**For every custom ("Other") app the user named in step 2 — ask what they want from it, one question per app.** They arrive in the survey response as `daily_content: … · {Name} — custom source` — that marker means "the user wants this tool, content still unknown", so it is a prompt to ask, never a finished answer to write into a tile. **One question per app** (e.g. "From Plaud, what should show up each morning — meeting notes, action points, or both?"). Never assume the content, and never silently drop the app: the #1 setup failure is proceeding without ever asking about a custom app the user typed. Carry each custom app **and** its content choice through the fetch (step 4) and the write (step 7).

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

**Silently, without messaging the user**, pull fresh data from connectors based on selected sections and content choices:

- **Claude chats (yesterday)** — **only when the user selected Claude as a source; no connector needed**: read the user's own past conversations with `recent_chats`. This is the morning mirror of what `evening-reflection` does at night: it looks *forward*, not back — pull only what still needs the user today.
  - **The window.** Call `recent_chats` with `after` set to the previous digest's timestamp (fall back to 24 h ago), `before` set to now, and `sort_order: "desc"`. Both are ISO-8601 in the user's timezone from `xtiles_get_user_timezone` — never in UTC, or a late-evening chat lands on the wrong day. Ask for ~10 chats; if every returned chat sits at the edge of the window, page further back with `before` set to the oldest one you got. Stop as soon as a chat falls outside the window — this is a morning brief, not an archive sweep.
  - **Then go deeper only where it pays.** `recent_chats` returns the conversations themselves — read what comes back and stop there for most of them. Reach for `conversation_search` in exactly one case: a thread ends mid-task and clearly continues an earlier one — search its topic to find where the commitment was actually made, `max_results: 5`, keeping only hits inside the window. Never search speculatively — one query per real gap.
  - **Unfinished threads** — work started and not finished ("we drafted half the launch email"), or a next step the user named but hasn't done yet ("I'll send this to Stefan tomorrow").
  - **Unanswered questions** — something the user asked or was asked in chat that never got resolved.
  - **Decisions worth acting on** — a conclusion reached in chat that implies a concrete step today.
  - Each item is one line, Poke-style and second person, same tone as the 🔴 email bucket: what was left open + what it needs now. Derive one verb-first action item per unfinished thread.
  - **Link back to the chat.** Every item carries the conversation URL returned by the tool, on the item's own title — so the user can reopen the thread and pick up where they left off. Use only URLs that actually came back from `recent_chats` / `conversation_search`; never assemble a chat link by hand. Where a chat has no URL, name it by its title instead.
  - **Exclude the digest's own chats** — any conversation whose content is this skill running (setup, survey, digest writes) is machinery, not signal. They are usually the most recent chats in the window and the easiest to mistake for work. Never let the brief report on itself. Same for `evening-reflection` runs — yesterday's reflection is already on yesterday's page.
  - Ignore purely exploratory or abandoned threads, and anything already closed by an xTiles task.
  - **If `recent_chats` is unavailable in this environment, errors, or returns nothing in the window** — drop this source for the run: no tile, and on a scheduled run no message. On a manual run, note it once in the preview ("Couldn't read chat history in this environment"). Never fabricate chat content, and never substitute this session's own context for it.
- **Gmail — unread emails**: `mcp__claude_ai_Gmail__search_threads` — query `is:important in:inbox newer_than:1d`. For each thread call `mcp__claude_ai_Gmail__get_thread` to get sender, subject, and threadId for the direct link (`https://mail.google.com/mail/u/0/#inbox/{threadId}`).
- **Gmail — newsletters**: `mcp__claude_ai_Gmail__search_threads` — query `from:({sender1} OR {sender2} ... OR @substack.com OR @beehiiv.com OR @convertkit.com) is:unread newer_than:1d` — combine user-named senders with common newsletter domains. Fetch each thread with `get_thread` for a one-line summary and `threadId` for the link.
- **Slack**: two parallel reads:
  1. `mcp__claude_ai_Slack__slack_read_channel` for each chosen channel (top 50 messages). Filter to last 24 hours (timestamp ≥ now − 24 h). Discard older messages. Skip channels with no messages silently.
  2. `mcp__claude_ai_Slack__slack_search_public_and_private` with query `to:me` to find messages where the user was @mentioned or DM'd. Filter results to last 24 hours. This covers both public and private channels, including ones not in the chosen list.

  After collecting, analyse all messages together and group semantically. For every item include a **direct permalink to the specific message** — extract `permalink` from the message object (or build `https://slack.com/archives/{channel_id}/p{ts_without_dot}`). Never link to the channel homepage — always to the individual message.

  **Slack renders as exactly three tiles** (see step 7): `### 📌 Slack — Action Points`, `### ⚡ Slack — Mentions`, `### 💬 Slack — Topics`. Analyse the messages into the groups below, but only three tiles come out of them — Decisions and Open are folded into the Topics tile, they never get tiles of their own.

  - **Mentions** *(highest priority)* — all messages from the `to:me` search. For each: who mentioned the user, in which channel, what was asked or said — one line per mention, message permalink. If the mention requires a response — flag it as ⚡. → **⚡ Slack — Mentions** tile.
  - **Action Points** — every ⚡-flagged mention also becomes an item here, but **stripped down to just the action — no retelling of who/where/what, that's what Mentions is for.** One line: the verb-first action item itself (e.g. "Reply to Maria in #product") plus the message permalink — the exact same wording as its `<task>` below, not a separate poke-style sentence. → **🔴 Slack — Action Points** tile.
  - **Topics** — what was discussed in channels; group by theme, one topic = one line, permalink to the most relevant message, channel attribution `[#channel](permalink)`. **Exclude any message already surfaced in Mentions or Action Points** — it doesn't get a second, redundant appearance here just because it also touched on a topic; if a topic's only messages are ones already covered there, drop the topic entirely. → **💬 Slack — Topics** tile.
  - **Decisions** — where something was agreed, committed to, or confirmed — include message permalink. Rendered as a `**✅ Decisions**` block inside the **💬 Slack — Topics** tile, not as its own tile.
  - **Open questions** — where a question was raised but no clear answer came yet — include message permalink, mark as ⏳. Rendered as a `**❓ Open**` block inside the **💬 Slack — Topics** tile, not as its own tile.

- **Calendar → the `### 📅 Workload` tile**: build one merged event list, then run the analysis below over it.
  - **Calendar (xTiles), if selected in step 2.** Call `mcp__xtiles__xtiles_list_calendar_events` for today — it aggregates whatever Google/Outlook calendars the user connected inside xTiles.
  - **Calendar (the existing Google Calendar connector), if also selected/connected.** Call `mcp__claude_ai_Google_Calendar__list_events` for today and add its events to the same list.
  - **Dedup across the two sources.** An event from the Calendar connector is a duplicate — and gets dropped — when an xTiles-calendar event already has the same start time and the same title (case-insensitive); this is the common case where the user's xTiles account is already synced to the same Google account they also connected directly. Keep every event that doesn't match one already in the list. Never show the same meeting twice.
  - For each surviving event extract: start/end time, title, participant names (first name + last name or company), the event description, and the meeting link (Google Meet, Zoom, or other video URL from event data).
  - **If Calendar (xTiles) was selected and contributed zero events, don't assume the day is simply free.** The tool can't distinguish "nothing scheduled" from "no calendar linked" (see step 2) — so if the Calendar connector also contributed nothing (or wasn't selected), flag this once in the preview (step 5) instead of silently treating it as an empty day: "No calendar events found today — this could mean nothing's scheduled, or that no calendar is linked inside xTiles yet. Want help connecting one?" Skip this note if the Calendar connector *did* contribute at least one event — that confirms the day genuinely has nothing from xTiles specifically, so there's nothing to flag.

  **This tile must earn its place — it is not a second copy of the calendar.** The user can already see their schedule; what they cannot see is *what each meeting is about*, *what they have to prepare*, and *where the day's real work fits*. An event row with no agenda and no prep is the weakest thing in the tile — the analysis below is the point, the timetable is just its scaffolding. Compute:
  - **Summary line**: event count, total hours occupied, longest free focus window (HH:MM–HH:MM, duration in hours)
  - **🎯 Focus recommendation — one sentence, always present.** Read the day as a whole and say what to do with it: which window to protect for deep work and for what, or which meeting decides the day. Base it on the real shape — longest free window, where the heavy prep sits, what's stacked. One concrete sentence, second person ("Your only real block is 09:00–11:30 — spend it on the pricing spec before the client call eats the afternoon"). Never generic advice ("plan your day carefully").
  - **📋 Agenda — one sentence per meeting.** For every event, find what it's actually about, in this source order: (1) a Granola or other meeting-notes entry with the same participants or title, (2) the most recent Gmail thread with the organiser or attendees on that subject, (3) the event's own description. Write one sentence — what will be decided or discussed, or where the last conversation left off. **Never invent an agenda**: if none of the three sources yields anything, omit the line for that event rather than paraphrasing the title back.
  - **Prep task — one per meeting that needs it.** From the agenda and its sources, derive the single most useful thing to prepare beforehand ("Pull the Q3 retention numbers before the Acme review"). One per meeting at most, and only where preparation is genuinely implied — a recurring standup usually needs none. Write it as a `<task>` (see step 7); these count as action items like any other, so include them in the flat action-item list.
  - **Grouping by purpose**: cluster the events into 2–4 groups derived from the actual day, not a fixed taxonomy — e.g. ⭐ Important · 🤝 Client & external · 🔁 Recurring syncs · 🧑‍🤝‍🧑 1:1s · 🧠 Focus blocks. Derive each group's name from what's actually in the day and the user's role. **Skip grouping entirely when there are fewer than 4 events** — splitting three meetings into groups is noise, not structure.
  - **⚠️ anomalies** — collect all, show at the bottom of the tile (not inline): overlapping events, back-to-back with no gap, events after 20:00, events without description/agenda, potential duplicate titles close together

**Cross-source dedup (Slack ↔ Gmail) — run once, after both are fetched, before either is classified into tiles or turned into action items.** The same ask sometimes arrives twice — a Slack message and a follow-up (or lead-in) email about the same specific thing, from the same person, close together in time. When a Slack message and a Gmail email clearly describe the same underlying ask:
- **Slack is primary** — it gets the full treatment in Mentions / Action Points as usual.
- **The matching email does not get its own 🔴/🟡 bucket entry or its own action item/task.** Fold it into the Slack item instead: append a short cross-reference to that Mentions/Action Points line, e.g. `(also emailed — [Open email](url))`. It never appears a second time in the Emails tile.
- **Only one `<task>` per duplicated pair** — never derive a second task for the email side of a duplicate.
- **Don't over-merge.** Fold only when it's genuinely the same ask (same person, same specific thing) — a Slack ping and an unrelated email from the same person are two separate items, not a duplicate.

Classify emails into three buckets. **Newsletters are fetched separately — exclude them here entirely and do not count them in any bucket.** Any email already folded into a Slack duplicate per the dedup above is excluded from these buckets — it was handled above, not here.

- 🔴 **Needs action** — emails where the user must take a concrete next step (reply, decide, act, log in)
- 🟡 **FYI** — FYI only: confirmed meetings, signed documents, payments, status updates — past/present tense, nothing to do
- ⚪ **Noise** — notifications, automated alerts, service emails — do not describe individually; count only

**Tone for 🔴 and 🟡 — Poke-style, capitalized:**
- Retell the email, do not copy the subject line. Subject → action → consequence in second person: not "Your account closed" but "Google shut down your ad account yesterday"
- For 🔴: weave the next step into the sentence: "Log in and restore it — the appeal window is limited"
- Use people's names, not email addresses. Context in parentheses if needed: "Stefan (influencers.club)"
- Telegraphic, conversational. First letter capitalized, no bureaucratic language.
- 🟡 items are one-liners — no link needed.

For every 🔴 email, derive one verb-first action item (e.g. "Restore the Google ad account") — these go into the Emails tile's Action items block. Likewise, every Slack ⚡-flagged mention yields a verb-first action item (e.g. "Reply to Maria in #product") — these go into the `### 📌 Slack — Action Points` tile's Tasks block (see step 7). Likewise, every unfinished thread from the chats yields a verb-first action item (e.g. "Finish the launch email draft") — these go into the `### 🤖 Claude — From our chats` tile's Tasks block, and every meeting that needs preparation yields one (e.g. "Pull the Q3 retention numbers before the Acme review") — that one sits inline under its meeting in the `### 📅 Workload` tile. Collect all as a flat list — used in preview and tiles. While deriving each one, also capture whether the source stated a **real deadline** and whether it carries **genuine urgency**. A stated deadline overrides the default date, and genuine urgency sets `priority`, when the item is written as a `<task>` in step 7. If the source states neither, the task still takes the page's day as its `dueDate` (see step 7) and gets no `priority` — do not infer urgency.

Use only real data from connectors. Do not invent names, events, or messages.
All names and message content must come directly from API responses — never from examples in this skill file.

If a connector call fails (error, timeout, 401) — record the failure. Do not write "No data" for a failed call — surface the error explicitly in step 5 so the user knows the connector did not respond.

- **Overdue tasks, only if `carry_over_tasks: true`.** Call
  `mcp__xtiles__xtiles_list_tasks` with `completed: "false"`, `due_date_after`:
  2 days ago (00:00, user's timezone), `due_date_before`: today (00:00) —
  today's own tasks are already on the page and are never touched. Collect the
  matches as candidates to carry forward; do not reschedule anything yet — that
  only happens after approval, in step 7.

---

### 5. Preview — show content in chat

Show real content with real data. Not structure, not headings with "(TBD)" — actual text.

Format (adapt to selected pages):

```
Here's what I've prepared:

---
📅 DAILY — [actual date]

### 📩 Emails
🔴 Needs action (N)
- [Poke-style description — 1–2 sentences, second person, action + consequence] → [Open email](https://mail.google.com/mail/u/0/#inbox/{threadId})

- [Next 🔴 email, same format] → [Open email](https://mail.google.com/mail/u/0/#inbox/{threadId})

🟡 FYI (N)
- [One-line item — no link]
- [One-line item]

⚪ Noise
- N notifications (sources) — nothing urgent

**Action items:**

<task dueDate="YYYY-MM-DD">[verb-first task from 🔴 email 1 — dueDate defaults to today]</task>

<task priority="high" dueDate="YYYY-MM-DD">[verb-first task from 🔴 email 2 — use the email's real deadline if it named one]</task>

*(omit Action items entirely if no 🔴 emails)*

### 📧 Newsletters

**[Newsletter Name](https://mail.google.com/mail/u/0/#inbox/{threadId})** — one-line summary.

**[Another Newsletter](https://mail.google.com/mail/u/0/#inbox/{threadId})** — one-line summary.

### 🤖 Claude — From our chats
- [What you left open yesterday + what it needs today — 1 sentence, second person] — [chat title](conversation URL from recent_chats)

- [Next unfinished thread, same format] — [chat title](conversation URL from recent_chats)

**Tasks**

<task dueDate="YYYY-MM-DD">[verb-first task — dueDate defaults to today]</task>

### 📌 Slack — Action Points
- [Verb-first action item, same wording as its task — no who/where/what retelling] — [#channel](url)

**Tasks**

<task dueDate="YYYY-MM-DD">[verb-first task — dueDate defaults to today]</task>

### ⚡ Slack — Mentions
- **@Name** in [#channel](url) — what they asked/said ⚡

### 💬 Slack — Topics
**Channels:** #channel1 (N) · #channel2 (N)
- **[Topic name]** — [one-sentence summary] — [#channel](url)

**✅ Decisions**
- [Decision] — [#channel](url)

**❓ Open**
- [Question] — [#channel](url) ⏳

### 📅 Workload
**N events · ~X h occupied · longest focus window HH:MM–HH:MM (X h)**

🎯 [Focus recommendation — one concrete sentence about this specific day]

**⭐ [Group name]**

**HH:MM–HH:MM · Meeting name** — Participant1, Participant2 · [Google Meet](url)

📋 [Agenda — one sentence: what will be decided, or where the last conversation left off]

<task dueDate="YYYY-MM-DD">[What to prepare before this meeting — dueDate is today's date]</task>

**🤝 [Second group name]**

**HH:MM–HH:MM · Meeting name** — Participant from Company · [Google Meet](url)

📋 [Agenda]

**HH:MM–HH:MM · Meeting name**

⚠️ [anomaly — e.g. two external calls back-to-back in the evening, 30 min gap between them]

---
```

Each 🔴 email uses the real `threadId` from `get_thread` for the [Open email] link. 🟡 items are one-liners with no link.
Each newsletter is shown as its own named section in the preview — never mixed into the Emails section.
Separate each item with a blank line for readability.

**Rules:**
- Show only selected sections the user asked for
- If a connector returned no data — write exactly that ("No unread emails", "No newsletters today", "No Slack updates today") — never skip the section silently; its absence looks like a bug
- If a connector call failed — write "Could not fetch [connector] data — connector error" (not "No data")
- No placeholder names, example events, or invented data — ever
- In one line, tell the user that after they approve you'll prepare Gmail draft replies for the 🔴 Needs action emails and mark the ⚪ Noise emails and newsletters as read — so the approval in step 6 covers those actions (see step 7·A)
- **If `carry_over_tasks: true` and step 4 found overdue open tasks**, list them in one line before the approval: `🔁 Will carry over N task(s) to today: "[title 1]", "[title 2]"…` — so the approval in step 6 covers this too. If none were found, omit the line entirely (nothing to carry over is not worth mentioning).
- After the preview, **stop and wait**. Do not write anything to xTiles yet.
---

### 6. Approval

**Mandatory. Never skip this step.** After showing the preview, call `show_widget` with the **Approval widget HTML** (see below).

Do not call `xtiles_create_tiles_from_markdown_in_my_planner` until the user explicitly clicks **"Looks good — create it"**.

If the user asks for a change — clarify exactly what, update only that section, re-show preview, ask again.

---

### 7. Write to xTiles

**Only after explicit approval.**

**If `carry_over_tasks: true` and step 4 found overdue tasks — reschedule them first,** before the tile write below: call `mcp__xtiles__xtiles_update_task` once per task from step 4's list, setting `due_date` to today (yyyy-MM-dd, user's timezone). Do not mark them complete and do not recreate them — this moves the existing task forward, it never duplicates it. If none were found, skip this silently (nothing to do, nothing to mention).

Tool: `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`
- `period`: "day"
- `date`: current date in ISO 8601

**Write all sections in a single call.** Combine all selected connectors (Gmail, Slack, etc.) into one markdown and call the tool once — never split into separate calls per connector.

**Write content tiles only.** The digest is exactly what the user approved in the preview — no meta, feedback, or self-tuning tiles are composed or appended.

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
- **The title emoji names the tile; it never doubles as a status marker.** 🔴 🟡 ⚪ ⚡ ⏳ ✅ ❓ are *item-level* markers — they belong on individual lines inside a tile (`🔴 **Needs action (3)**`, `— [#channel](url) ⚡`), never in a `###` heading. A heading takes a subject emoji that says what the tile *is*: 📩 Emails · 📧 Newsletters · 📌 Slack — Action Points · ⚡ Slack — Mentions · 💬 Slack — Topics · 📅 Workload · 🤖 Claude — From our chats. (⚡ in `### ⚡ Slack — Mentions` is the exception that proves the rule — there it names the subject, mentions, not a status.)

**Action items are real tasks, not checkboxes.** Every action item the digest derives — from a 🔴 email, from a ⚡ Slack mention, from an unfinished chat thread, from a meeting that needs preparation — is written with the `<task>` tag so it becomes a first-class xTiles task with its own due date and priority, not a checkbox that only lives inside the tile's text:

```
**Action items**

<task dueDate="2026-08-11">Restore the Google ad account</task>

<task priority="high" dueDate="2026-08-10">Sign the contract</task>
```

- **Never `- [ ]` for action items.** Plain `- [ ]` checkboxes stay reserved for checklists that are not tasks — if such a checklist ever appears on the page, never convert it into `<task>`.
- **One `<task>` per line, blank line between each.** A `<task>` must never be nested inside a list item (`- <task>…</task>` does not parse) and never carries a link.
- **`dueDate="YYYY-MM-DD"` — always set it, defaulting to the task's own day** (today's date for a Daily page), resolved against the user's timezone from `xtiles_get_user_timezone`. If the source states a **later real deadline** ("by Friday", "before the 10th", "appeal window closes Tuesday"), use that date instead; when the source also gives a specific time, use the `dueDate="YYYY-MM-DD HH:MM"` form. Never derive the date from when the email was sent or the message was posted.
- **`priority` — only when the source itself signals it.** `high` for a hard deadline inside 24 h, a blocker, an explicitly urgent ask, or something with a real cost of missing it (suspended account, expiring window); `medium` when it matters but nothing forces it today; omit otherwise. Do not stamp `high` on everything just because it came from the 🔴 bucket — if every task is high, the field carries no information. As a sanity cap: at most a third of a morning's tasks should be `high`.
- **Never `completed="true"`** — a morning brief describes work still to do.
- Task titles stay verb-first and in the user's language, same as before.

**Content formatting inside each tile:**
- **All links must be inline hyperlinks — never link-only lines.** Always `[text](url)`, never a bare URL, and always **on the same line as surrounding text**. This is a rendering rule, not a style preference: xTiles turns a line that contains *only* a link into a big block-link card, and turns a link sitting inside a line of text into an ordinary hyperlink. Users want the hyperlink.

  ```
  Reply to Stefan before the window closes → [Open email](url)     ← hyperlink ✅

  [Open email](url)                                                ← block-link card ❌
  ```

  So: never break a link onto its own line, never start a line with a link, and never leave a link as the only content of a paragraph. Attach it to the end of the sentence it belongs to (`… → [Open email](url)`, `— [#channel](permalink)`, `· [Google Meet](url)`). If a link has nothing to attach to, that's a sign the item is missing its description — write the description, don't ship a naked link.

  > Note: `xtiles://guide/markdown/blocks` says to put each link on its own line after a blank line. That produces the block-link card described above. This skill deliberately does the opposite — do not "fix" it back when consulting the guide.
- Separate each item with a blank line — never write items as a continuous block
- **Emails**: the tile is titled **`### 📩 Emails`** — always keep the 📩 envelope in the title so it's clear the content comes from email. If email content is ever split across more than one tile (e.g. a separate "Needs action" tile), **every email-derived tile keeps the 📩 prefix** (`### 📩 Needs action`).
  - **Email-as-primary-source → thematic breakdown.** When the user asked to split email by topic in step 3 (e.g. named projects, clients, or recurring subjects), do **not** produce one generic Emails tile — produce **one `### 📩 [Topic]` tile per topic** (e.g. `### 📩 Acme deal`, `### 📩 Hiring`), each keeping the 📩 prefix and each using the same three-block 🔴/🟡/⚪ structure below, scoped to that topic. Anything that doesn't match a named topic goes into a catch-all `### 📩 Emails` tile. This is the "one source, many question-tiles" case — a common and high-value setup when email is the user's main signal.
  - Otherwise, a single `### 📩 Emails` tile. Structure it in three labeled blocks followed by action items:
  ```
  🔴 **Needs action (N)**

  - [Poke-style description — 1–2 sentences, second person, action + consequence] → [Open email](https://mail.google.com/mail/u/0/#inbox/{threadId})

  - [Next 🔴 item, same format] → [Open email](url)

  🟡 **FYI (N)**

  - [One-line item — no link, never]

  - [One-line item]

  ⚪ **Noise**

  - N notifications (sources) — nothing urgent

  ---

  **Action items**

  <task dueDate="2026-08-11">[Verb-first task from 🔴 email 1 — dueDate defaults to today]</task>

  <task priority="high" dueDate="2026-08-10">[Verb-first task from 🔴 email 2 — priority only when the email signalled it; a stated deadline overrides today's date]</task>
  ```
  Omit `Action items` section entirely if no 🔴 emails. Newsletters are in the separate `### 📧 Newsletters` tile — never include them here.
- **Newsletters**: ALL newsletters go in a **single `### 📧 Newsletters` tile** — never create a separate tile per newsletter. Structure:
  - Each newsletter is **one line**: a bold hyperlink title, then an em dash, then the one-line summary — all on the same line, so the title renders as a hyperlink and not as a block-link card:
    ```
    **[Newsletter Name](https://mail.google.com/mail/u/0/#inbox/{threadId})** — one-line summary.
    ```
  - Blank line between entries.
  - The link IS the title — no separate "Open" button or link at the bottom of each entry, and never the title alone on its own line.
  - Omit the entire tile only if there are no unread newsletters at all.
- **Claude chats**: a single `### 🤖 Claude — From our chats` tile. One line per unfinished thread / unanswered question / actionable decision, Poke-style and second person, blank line between items. Each line ends with the chat link in the shape `— [<chat title>](<conversation URL from the tool>)`, so the user can jump back into the thread; use only URLs returned by `recent_chats` / `conversation_search`, and where one is missing write the chat title in plain bold instead of a broken link. Below them a `**Tasks**` block with one `<task>` per item (see **Action items are real tasks** above). **Omit the tile entirely if nothing was left open** — unlike the Slack Topics tile, an empty chat day is normal and doesn't look like a failure.
- **Slack**: split into **exactly three tiles** — `### 📌 Slack — Action Points`, `### ⚡ Slack — Mentions`, `### 💬 Slack — Topics` — never one big tile, and never more than these three (Decisions and Open are blocks inside the Topics tile, not tiles of their own). Each tile uses `###` as its header. All Slack links must point to the specific message permalink, never to the channel homepage.
  - `### 📌 Slack — Action Points` — the actionable subset, **narrowed to just the action, never a retelling of who/where/what** (that belongs in Mentions, not here): one line per ⚡-flagged mention: `- [Verb-first action item — same wording as its <task>] — [#channel](message_permalink)`, plus `(also emailed — [Open email](url))` appended when the cross-source dedup above folded a matching email into this item. Below that, a `**Tasks**` block with one `<task>` per item (e.g. `<task>Reply to Maria in #product</task>`) — see **Action items are real tasks** above. **Omit tile entirely if no ⚡ mentions today.** This tile is a rollup, not a replacement — the same messages still appear in `### ⚡ Slack — Mentions` below for full context.
  - `### ⚡ Slack — Mentions` — one line per mention: `- **@Name** in [#channel](message_permalink) — what they asked/said`. Add ` ⚡` if a response is needed, and `(also emailed — [Open email](url))` when the dedup above folded a matching email in. **Omit tile entirely if no mentions.**
  - `### 💬 Slack — Topics` — the discussion rollup, **excluding any message already shown in Mentions or Action Points** (a topic whose only messages are already covered there is dropped entirely, not repeated here). First line: `**Channels:** #channel1 (N) · #channel2 (N)`. Then one line per topic: `- **Topic name** — one-sentence summary — [#channel](message_permalink)`. Then fold decisions and open questions into this same tile as labeled blocks (never separate tiles):
    - a `**✅ Decisions**` block — one line per decision: `- Decision made — [#channel](message_permalink)`; omit the block if there are no decisions.
    - a `**❓ Open**` block — one line per unanswered question: `- Question — [#channel](message_permalink) ⏳`; omit the block if there are none.
    **Always create this Topics tile** — if no messages today, write a single line: `No updates today.` Its absence looks like a connector failure.
- **Calendar**: tile titled `### 📅 Workload` — **never `### 📅 Calendar`.** The name is the promise: this is an analysis of the day, not a reprint of the schedule. Use this exact structure:
  ```
  ### 📅 Workload
  @colorSize: LIGHTER
  @color: [pick randomly from the color list]

  **N events · ~X h occupied · longest focus window HH:MM–HH:MM (X h)**

  🎯 [Focus recommendation — one sentence]

  **⭐ [Group name]**

  **HH:MM–HH:MM · Meeting name** — Participant1, Participant2 · [Google Meet](url)

  📋 [Agenda — one sentence]

  <task dueDate="YYYY-MM-DD">[What to prepare before this meeting — dueDate is today's date]</task>

  **🤝 [Second group name]**

  **HH:MM–HH:MM · Meeting name** — Participant from Company · [Zoom](url)

  📋 [Agenda]

  **HH:MM–HH:MM · Meeting name**

  ⚠️ [anomaly]
  ```
  Rules:
  - Summary line is bold, always first; the 🎯 focus recommendation goes directly under it and is **never omitted** — a Workload tile without it is just a schedule
  - **Group headings** are bold lines with a leading emoji, derived from the day per step 4 (⭐ Important · 🤝 Client & external · 🔁 Recurring syncs · 🧑‍🤝‍🧑 1:1s · 🧠 Focus blocks are suggestions, not a fixed set). Events sit under their group in chronological order. **With fewer than 4 events, drop the group headings** and list events flat
  - Each event on its own bold line: `**HH:MM–HH:MM · Title**` — append ` — Participants · [Link label](url)` if participants or meeting link exist
  - 📋 agenda goes on the next paragraph directly under its event — one sentence, from Granola / meeting notes / Gmail / the event description. Omit the line when no source yielded anything; never paraphrase the meeting title back as an agenda. When citing a source, weave the hyperlink into the agenda sentence itself — `Continues the pricing thread from [Monday's note](url)` — never append it as a separate line
  - `<task>` prep item goes directly under its agenda, at most one per meeting, only where preparation is genuinely implied. Same attribute rules as every other task (see **Action items are real tasks**) — the prep is for today's meeting, so its `dueDate` is today (the page's day) unless the source states an earlier real deadline
  - All ⚠️ anomalies collected at the bottom, one per line
  - Blank line between every item (group heading, event, 📋, `<task>`, ⚠️) for readability
  - **Never silently omit the tile if Calendar or Granola was a selected source** — same rule as every other section: an empty Workload tile reads as a connector failure, not a quiet day. If the merged event list is empty, still write the summary line (`**0 events · 0 h occupied**`) and the 🎯 focus recommendation, then a single line `No meetings scheduled today.` in place of the event list — this is also where a zero-result Granola fetch becomes visible, since Granola has no tile of its own. Only omit the tile entirely if neither Calendar nor Granola was selected at all.
- This ensures the tile is scannable, not a wall of text

**If xTiles is not connected** — do not output the digest as plain text in chat. Walk the user through connecting xTiles (see **How to connect connectors**), wait for confirmation, then write.

**If the page already exists — update matching tiles in place, never duplicate the user's template:**
1. Call `mcp__xtiles__xtiles_get_planner_content` and list the existing `###` tile headings on the page.
2. Split the sections you're about to write by comparing headings — match on the heading text, ignoring any trailing date suffix:
   - **Heading already on the page** → update that tile in place with `mcp__xtiles__xtiles_patch_view_content`: one search-and-replace that swaps the tile's current body (everything under its `###` heading and colour annotations, up to the next `###`) for the freshly composed body. **Keep the `###` heading line and the `@colorSize`/`@color` annotations exactly as they are** — never change the user's colour, title wording, or the tile's position. This is what lets a user's own saved template be refreshed each morning instead of duplicated.
   - **Heading not on the page** → include it in the single `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner` call (append), as usual.
3. **Never create a second tile whose heading already exists on the page.** If the in-place update capability can't target this page, leave the existing tile untouched (skip it) rather than writing a duplicate.
4. The layout pass and CTA below apply only to tiles you **newly created**; tiles updated in place keep their position and are not re-laid-out. If you created nothing new, skip the layout pass and reuse the `view_id` from the `get_planner_content` call you already made. **While matching headings in step 2, note the first composed section's own link/`resource_url`** if that `get_planner_content` call returns one alongside the matched tile — step 3 below needs it for the CTA button when that tile is only patched, not created.

This runs silently on a scheduled run; on a manual run it needs no extra confirmation — the refreshed content is exactly what the user approved in the preview.

**After each successful write — run these steps in order, no exceptions (Cowork included — steps 3–4 are chat widgets/questions, never replaced by an artifact; step 6 is scheduled-run-only and replaces them with a real notification):**

1. Write `✅ Daily created.`

2. **Layout pass — mandatory stage after adding tiles. Runs on every write (scheduled runs included); never skipped, never deferred, never asked about.** Freshly written tiles land in a default stack — re-lay them out *now*, before the CTA and schedule widgets below:
   - Read `view_id` and `tile_ids` straight from the `xtiles_create_tiles_from_markdown_in_my_planner` response (`tile_ids` is ordered to match the `###` sections you just wrote). Keep `view_id` — step 3 reuses it, do not re-fetch it.
   - Call `mcp__xtiles__xtiles_get_workflow` with id `tile-layout` and follow it exactly: pass `tile_ids` as its "added tiles", the markdown you just wrote as their content, and these **layout hints** — 1–4 tiles · default 2 per row · give a heavy tile (usually 💬 Slack — Topics, 📩 Emails, or 📅 Workload) its own full-width row.
   - Apply the layout silently — no message, no confirmation. Only once it is applied, continue to step 3.

3. **For non-scheduled runs only: link to the first tile, not the page**, so the user lands right on the brief. (On a scheduled run, nobody is watching chat — skip this widget entirely and use step 6 below instead.)
   - **If step 2's layout pass ran (something was newly created)** — take the `resource_url` of the **first** entry in the create call's response `tiles` array (a deep link that opens the Daily page focused on that tile) and use it as `{VIEW_URL}` directly.
   - **If nothing was newly created (the first composed section already existed and was only patched, so step 2 was skipped) — the common case on a recurring Daily page.** Take the link/`resource_url` you noted for it while matching headings, and resolve it once with `mcp__xtiles__xtiles_get_content_by_link` — if the response's `resource_type` is `TILE`, use that same URL as `{VIEW_URL}`. This confirms it addresses the tile itself, not the whole page.
   - **Only if no tile-level link was available at all, or it resolves as `PAGE` rather than `TILE`** — fall back to `https://xtiles.app/{view_id}`, reusing the `view_id` from the `get_planner_content` call already made when checking existing headings (no extra call needed).
   Call `show_widget` with the **CTA widget HTML** (see below) using that `{VIEW_URL}`. Translate the button label into the user's language. **Never leave `{VIEW_URL}` unresolved and never output a markdown link instead of the widget — the button must render on every non-scheduled run, whether tiles were created or only patched.**
4. **For non-scheduled runs only**: Immediately call `show_widget` with the **Schedule widget HTML** (see below). Do not skip this step, do not ask first, and never substitute `AskUserQuestion` here — this is a Cowork chat widget, not a question.
5. **Gmail follow-through — mandatory, silent, every run with Gmail connected; never skipped, never deferred, never left for "later," scheduled runs included.** Immediately after step 4 (do not wait for the schedule widget's response), run both actions from **step 7·A** below: draft replies for every 🔴 Needs action email, and mark every ⚪ Noise/newsletter thread as read. These do not block or reorder steps 1–4 above — they run as their own step, not folded into any of them — but they are just as mandatory as the layout pass. Skipping them silently is a failed run, identical to skipping the CTA button.
6. **Scheduled runs only: notify instead of showing a widget.** Nobody is present to click a `show_widget` CTA during an unattended run — replace it with `mcp__xtiles__xtiles_create_notification` so the user still learns their brief is ready:
   - `url`: the **page** URL, `https://xtiles.app/{view_id}` — the tool requires a page link, never the tile-specific `{VIEW_URL}` from step 3 (which this step skips anyway).
   - `text`: one short, punchy sentence in the user's language, max 100 characters — **written like a good marketer's notification, not a status log.** Give the user an actual reason to tap it *now*: reference something concrete from today's brief (a count, the most pressing item) rather than a bland "Daily created." English placeholder examples to translate, not copy verbatim: `"Your Daily Brief just landed — 3 emails need you today →"`, `"Morning brief's in: your only free block is 10–12 →"`. Never invent a number that isn't real — pull it from what you actually wrote.
   - `agent_source`: `"Claude"`.

**If an error occurs:** briefly say what went wrong, offer to retry or skip that page.

---

### 7·A. Gmail follow-through — draft replies & mark-as-read

This is **step 5** of the post-write sequence above — run it right after the
Schedule widget (step 4), silently, as part of the same run. It is auxiliary
inbox hygiene, not the deliverable, so it never replaces the xTiles write and
never reorders steps 1–4 (layout → CTA → schedule) — but it is not optional
either: a run that writes the digest and stops at step 4 without also doing
this is incomplete.

- **Draft replies for important emails.** For every 🔴 **Needs action** email,
  create a Gmail **draft reply on that thread** with
  `mcp__claude_ai_Gmail__create_draft` — pass the thread's `threadId`, reply to
  the sender, keep the `Re:` subject. Write a short, on-point reply in the
  user's language that they could send after a quick check — **never send it,
  only leave the draft**. Never draft for 🟡 FYI or ⚪ Noise.
- **Mark noise and newsletters as read.** Once the digest has captured them,
  mark every ⚪ **Noise** email and every **newsletter** thread as read by
  removing the `UNREAD` label with `mcp__claude_ai_Gmail__unlabel_thread`.
  **Never touch 🔴 or 🟡 threads** — those stay unread so the user still acts on
  them.

**Surface the outcome by patching the tile you already wrote — on every run,
manual or scheduled.** Both actions are named in advance too (in the preview on
a manual run, step 5; implicitly approved by the schedule config on a scheduled
run), but naming the *intent* beforehand isn't the same as confirming the
*outcome*, and these actions run silently, off-screen, after the tile is
already written (they're step 5 of the post-write sequence, after the draft
URLs and read counts even exist). So once both actions finish, call
`mcp__xtiles__xtiles_patch_view_content` once against the just-written
`### 📩 Emails` tile (or the topic tile it landed in, if email was split by
topic) to:
1. Add `· [Draft reply](draft_url)` inline on each 🔴 email line that got one —
   `… → [Open email](url) · [Draft reply](draft_url)`.
2. Append one summary line at the end of the tile's body:
   `✉️ Auto-actions: prepared N draft replies · marked M Noise/newsletter emails as read`.

State only the real counts; if a count is zero, drop that clause, and if
nothing changed, skip both edits — don't patch for a no-op. **Never skip this
on a manual run** — the user approved the intent, not the result, and this
patch is the only place they see what was actually done.

**Read-only fallback.** If a Gmail write tool is missing or errors, skip that
action, still finish the digest, and note it once on a manual run ("Couldn't
prepare drafts / mark read in this environment"). Never block or fail the run
over it.

---

### 8. Schedule (optional)

The schedule widget is shown in step 7 above. This step handles the user's response.

In Claude Code (no Cowork): after writing, ask inline: "Want me to run this every morning automatically? What time? (default: 9:00 AM)"

- If the user selects **"Yes, schedule it"** — first invoke `anthropic-skills:schedule`, then call `mcp__scheduled-tasks__create-scheduled-tasks`. Pass to both:
  - **`prompt`**: the full config string assembled from values collected during setup —
    ```
    Run daily digest — role: {role} · tools: {tools} · daily_content: {content} · carry_over_tasks: {true/false} · schedule: daily-{HH:MM} days:{days}
    ```
    Replace `{role}`, `{tools}`, `{content}`, `{true/false}`, `{HH:MM}`, and `{days}` with the actual values parsed from the widget response and setup. Do not leave placeholders. Use `;` to separate multiple entries within a single field (names may contain commas).
  - **`schedule`**: cron expression derived from the widget. The widget sends `cron: HH:MM days:1-5` or `cron: HH:MM days:*` — parse both values: time gives H and M, days gives the weekday field. Build: `M H * * {days}`. Examples: `cron: 08:30 days:1-5` → `30 8 * * 1-5` · `cron: 08:30 days:*` → `30 8 * * *`. Default if missing: `0 9 * * 1-5`.
  - **`timezone`**: the user's local timezone — call `mcp__xtiles__xtiles_get_user_timezone` to get it before scheduling if it hasn't been fetched yet.

  This prompt fires each morning and triggers `daily-brief` in scheduled-run mode — the full config must be embedded so the survey is skipped automatically.

  After scheduling succeeds, confirm: "Done — your Daily will be ready in xTiles every morning at [chosen time]." **Do not show the CTA widget again here** — it was already shown once, right after the write in step 7; repeating it after the schedule confirmation is redundant. **This confirmation is not the end of the run — immediately continue to step 9 (Related workflows) in the same turn.**
- If the user selects **"No, thanks"** — acknowledge briefly, then **immediately continue to step 9 (Related workflows) in the same turn.**

**Either way, step 9 still has to run before this turn ends — the schedule confirmation or decline is not a stopping point, do not wait for the user to ask.**

---

### 9. Related workflows

**After every manual run, once step 8 is resolved** (scheduled or declined) —
offer related workflows. **This is a mandatory closing step of every manual run,
not optional — the run is not complete without it.** Skip it only on scheduled
runs, which end after step 7's notification — no widgets, nobody to ask.

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

**Fallback when the registry is unavailable (free ChatGPT / Claude).** On free plans `mcp__mcp-registry__suggest_connectors` often isn't available, so there are no interactive connect buttons. Do not dead-end. Instead, present the **Supported tools — embedded catalog** (step 2) so the user still sees the full menu of what's possible, explain that connecting tools requires upgrading their plan / enabling connectors, and proceed with whatever *is* available so they still get a first digest. The embedded catalog — not the registry — is the source of truth for what the skill can offer.
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
    .tag{font-size:10px;font-weight:600;padding:2px 6px;border-radius:20px;background:var(--color-background-secondary);color:var(--color-text-secondary);text-transform:uppercase;letter-spacing:.03em}
    .card.sel .tag{background:rgba(255,255,255,.22);color:var(--color-background-primary)}
    .chk{width:15px;height:15px;border-radius:4px;border:1.5px solid var(--color-border-secondary);display:flex;align-items:center;justify-content:center;font-size:9px;flex-shrink:0}
    .card.sel .chk{background:var(--color-background-primary);border-color:var(--color-background-primary);color:var(--color-text-primary)}
    .custom-in{margin-top:9px}
    .custom-in input{width:100%;padding:7px 11px;border:1.5px solid var(--color-border-tertiary);border-radius:8px;font-size:13px;outline:none;background:var(--color-background-primary);color:var(--color-text-primary)}
    .custom-in input:focus{border-color:var(--color-border-secondary)}
    .custom-wrap{margin-top:12px;padding:12px;border:1.5px dashed var(--color-border-tertiary);border-radius:10px;background:var(--color-background-tertiary)}
    .custom-wrap:focus-within{border-color:var(--color-text-primary)}
    .custom-label{font-size:13px;font-weight:600;color:var(--color-text-primary);margin-bottom:2px}
    .custom-wrap input{width:100%;padding:7px 11px;border:1.5px solid var(--color-border-tertiary);border-radius:8px;font-size:13px;outline:none;background:var(--color-background-primary);color:var(--color-text-primary)}
    .custom-wrap input:focus{border-color:var(--color-border-secondary)}
    .custom-chips{display:flex;flex-wrap:wrap;gap:6px;margin-top:8px}
    .custom-chips span{padding:4px 10px;border-radius:20px;background:var(--color-text-primary);color:var(--color-background-primary);font-size:12px}
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
        <div class="card" onclick="togTool(this,'Claude')"><div class="chk">✓</div>Claude <span class="tag">no setup</span></div>
        <div class="card" onclick="togTool(this,'Calendar')"><div class="chk">✓</div>Calendar (xTiles)</div>
        <div class="card" onclick="togTool(this,'GoogleCalendar')"><div class="chk">✓</div>Calendar</div>
        <div class="card" onclick="togTool(this,'Slack')"><div class="chk">✓</div>Slack</div>
        <div class="card" onclick="togTool(this,'Gmail')"><div class="chk">✓</div>Gmail</div>
        <div class="card" onclick="togTool(this,'Granola')"><div class="chk">✓</div>Granola</div>
        <div class="card" onclick="togTool(this,'Linear')"><div class="chk">✓</div>Linear</div>
        <div class="card" onclick="togTool(this,'GitHub')"><div class="chk">✓</div>GitHub</div>
        <div class="card" onclick="togTool(this,'GoogleDrive')"><div class="chk">✓</div>Google Drive</div>
        <div class="card" onclick="togTool(this,'Gamma')"><div class="chk">✓</div>Gamma</div>
        <div class="card" onclick="togTool(this,'Figma')"><div class="chk">✓</div>Figma</div>
      </div>
      <div class="custom-wrap">
        <div class="custom-label">➕ Don't see your tool? Add your own connector</div>
        <div class="hint" style="margin-bottom:6px">Type any tool you have connected — separate several with commas</div>
        <input type="text" id="other-tool" placeholder="e.g. Plaud, Notion, Notion Calendar…" oninput="previewCustom()">
        <div class="custom-chips" id="custom-chips"></div>
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

    <div class="sec">
      <div class="sec-title">Carry over tasks still open from the last 2 days?</div>
      <div class="pills" id="carry-pills">
        <div class="pill sel" onclick="pickCarry(this,false)">No, leave them</div>
        <div class="pill" onclick="pickCarry(this,true)">Yes, move to today</div>
      </div>
    </div>

    <div class="btn-row">
      <button class="btn btn-s" onclick="go1()">← Back</button>
      <button class="btn btn-p" onclick="submit()">Set up Daily</button>
    </div>
  </div>
</div>

<script>
// Claude is a source (needs no connector) but is OFF by default — chat history
// reaches xTiles only if the user picks it. Card ships unselected; initial set is empty.
var role=null, tools=new Set(), content=new Set(), carryOver=false;

var TM={
  'Claude':      {daily:['From our chats — unfinished threads & open questions']},
  'Slack':       {daily:['Slack messages — work chat signals']},
  'Gmail':       {daily:['Important emails — unread inbox','Newsletters — curated summaries','Emails by topic — separate thematic tiles','Emails from key people — VIP senders','Follow-ups — threads awaiting your reply']},
  'Calendar':    {daily:['Workload — agendas, prep & focus time']},
  'GoogleCalendar': {daily:['Workload — agendas, prep & focus time']},
  'Granola':     {daily:['Granola — meeting notes & summaries']},
  'Linear':      {daily:['Linear issues — new & updated']},
  'GitHub':      {daily:['GitHub — PRs & review requests']},
  'GoogleDrive': {daily:['Google Drive — shared files updated']},
  'Gamma':       {daily:['Gamma — presentations updated']},
  'Figma':       {daily:['Figma — design updates & comments']}
};
var AM=[];

var ROLE_DEFAULTS={
  'Product Manager':   ['Slack messages — work chat signals','Important emails — unread inbox','Workload — agendas, prep & focus time','Linear issues — new & updated','Granola — meeting notes & summaries'],
  'Designer':          ['Figma — design updates & comments','Slack messages — work chat signals','Important emails — unread inbox','Workload — agendas, prep & focus time'],
  'Engineer':          ['GitHub — PRs & review requests','Slack messages — work chat signals','Important emails — unread inbox','Linear issues — new & updated','Workload — agendas, prep & focus time'],
  'Growth & Marketing':['Important emails — unread inbox','Newsletters — curated summaries','Slack messages — work chat signals','Gamma — presentations updated','Workload — agendas, prep & focus time'],
  'Founder / CEO':     ['Slack messages — work chat signals','Important emails — unread inbox','Newsletters — curated summaries','Granola — meeting notes & summaries','Workload — agendas, prep & focus time'],
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
// Custom connectors typed by the user. Read on every step change so they survive Back/Next
// and so each one gets its own line in the step-2 content list — never silently dropped.
var custom=[];
function readCustom(){
  custom.forEach(function(c){tools.delete(c);});
  AM.forEach(function(i){content.delete(i);});
  var raw=document.getElementById('other-tool').value;
  custom=raw.split(',').map(function(s){return s.trim()}).filter(function(s){return s.length});
  AM=custom.map(function(c){return c+' — custom source'});
  custom.forEach(function(c){tools.add(c);});
}
function previewCustom(){
  var raw=document.getElementById('other-tool').value;
  var list=raw.split(',').map(function(s){return s.trim()}).filter(function(s){return s.length});
  document.getElementById('custom-chips').innerHTML=list.map(function(c){
    return '<span>'+c.replace(/</g,'&lt;')+'</span>';
  }).join('');
}
function go2(){
  readCustom();
  if(!content.size){
    var r=role==='__other__'?null:role;
    (ROLE_DEFAULTS[r]||[]).forEach(function(v){content.add(v);});
  }
  // Always pre-select content for every tool the user explicitly picked
  tools.forEach(function(t){if(TM[t]&&TM[t].daily)TM[t].daily.forEach(function(v){content.add(v);});});
  // Custom connectors are pre-selected too — the user named them, so they're wanted by default
  AM.forEach(function(v){content.add(v);});
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
function pickCarry(el,v){document.querySelectorAll('#carry-pills .pill').forEach(function(p){p.classList.remove('sel')});el.classList.add('sel');carryOver=v;}
function submit(){
  document.querySelectorAll('.btn').forEach(function(b){b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';});
  var r=role==='__other__'?document.getElementById('role-other-in').value.trim():role;
  var tArr=Array.from(tools);
  var valid=[];
  tArr.forEach(function(t){if(TM[t]&&TM[t].daily)TM[t].daily.forEach(function(i){if(valid.indexOf(i)<0)valid.push(i)});});
  AM.forEach(function(i){if(valid.indexOf(i)<0)valid.push(i)});
  var items=Array.from(content).filter(function(i){return valid.indexOf(i)>=0;});
  var parts=['Daily planner setup — role: '+r+' · tools: '+(tArr.join(', ')||'none')+' · daily_content: '+(items.join(', ')||'none')+' · carry_over_tasks: '+carryOver];
  sendPrompt(parts.join(' · '));
}
</script>
```

---

## CTA widget HTML

Show this immediately after a successful write. Replace `{VIEW_URL}` with the tile-level link resolved in step 7's write sequence above (falling back to the page URL only if no tile-level link was available) before calling `show_widget`.

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
- **In Cowork, every terminal-sequence step (CTA, Schedule offer) is a `show_widget` HTML widget, never `AskUserQuestion` and never plain text** — `AskUserQuestion` is for step 9 (Related workflows) only
- If context is missing — ask, don't guess
- If the user gives new information along the way — pick it up, don't wait for the "right step"
- Real data from connectors always beats placeholders
- Daily is the only period. If the user asks for Weekly or Monthly, tell them only the Daily planner is currently supported and offer to create a Daily page instead — never silently downscope.
- Match the user's language, adapt if they switch. All bucket labels, block titles, link texts, and action items in this file are English placeholders — render them in the user's language, and never carry over a language from an example
- Show the survey widget in Cowork only — in Claude Code, ask the same questions inline
- **Gmail follow-through (drafts + mark-as-read, step 5/7·A) is mandatory whenever Gmail is connected — not a nice-to-have.** A run that finishes the digest and the widgets but skips it is an incomplete run, same severity as a missing CTA button.
- **A section representing a selected connector is never silently blank.** If Calendar/Granola/Slack/etc. was selected and returned nothing, write the tile with a one-line text placeholder ("No meetings scheduled today.", "No updates today.") instead of omitting it — its total absence reads as a bug, not a quiet day.
- **No self-tuning cycle.** The digest writes content tiles only — never a "Tune your digest", "Important keywords", or weekly-recap tile, and never a feedback checkbox. Do not invoke the `digest-tuning` workflow and do not hand-roll its tiles here.
 
