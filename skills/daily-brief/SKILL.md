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

  Config (role/tools/daily_content) is read from the scheduled task prompt —
  no separate file needed.
  For manual runs: look for config in today's Planner; if there's none, start
  from the survey flow below.
allowed-tools: >
  mcp__xtiles__xtiles_get_planner_content,
  mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner,
  mcp__xtiles__xtiles_list_tasks,
  mcp__xtiles__xtiles_get_user_timezone,
  mcp__xtiles__xtiles_get_page_layout,
  mcp__xtiles__xtiles_set_page_layout,
  mcp__xtiles__xtiles_get_workflow,
  mcp__claude_ai_Slack__slack_search_channels,
  mcp__claude_ai_Slack__slack_search_public_and_private,
  mcp__claude_ai_Slack__slack_read_channel,
  mcp__claude_ai_Slack__slack_read_user_profile,
  mcp__claude_ai_Gmail__search_threads,
  mcp__claude_ai_Gmail__list_labels,
  mcp__claude_ai_Gmail__get_thread,
  mcp__claude_ai_Gmail__unlabel_message,
  mcp__claude_ai_Google_Calendar__list_events,
  mcp__claude_ai_Granola__list_meetings,
  mcp__claude_ai_Google_Drive__list_recent_files,
  mcp__claude_ai_Linear__list_issues,
  mcp__mcp-registry__suggest_connectors,
  anthropic-skills:schedule,
  mcp__scheduled-tasks__create-scheduled-tasks,
  show_widget,
  AskUserQuestion
---

# xTiles Daily Planner — Setup & Daily Digest

## Principles

1. **Survey first, write to xTiles last.** Nothing is created until the user has seen the preview and approved it.
2. **Real data, not placeholders.** Pull from connectors before preview so the user sees live content.
3. **Match the user's language** throughout — follow the language of the user's first message, adapt if they switch.

## Rules

- **Interactive picks.** For any rich choice (survey, Slack channels, newsletters), call `xtiles_get_workflow("db-survey")` and execute it with the matching mode + input — it owns the widget render and the never-stop fallback. For everything else (approve/confirm, a link, a "done" ack, a ≤ 4-option choice), use the environment's native question mechanism (`AskUserQuestion`) or plain markdown.
- **Never skip a selected connector.** If it is not connected, walk the user through connecting it (see *Connecting connectors*) before continuing — never silently drop it.
- **Preview before writing**, and never write the digest as plain chat text — always write to xTiles (or connect xTiles first).
- **Real data only** — never put example names/events/messages into preview or tiles.
- **Render all tile labels and prose in the user's language** (English tokens below are the source; translate at render time).
- **Daily is the only period.** If the user asks for Weekly or Monthly, say only Daily is currently supported and offer to create a Daily page instead — never silently downscope.
- **If context is missing, ask — don't guess.** If the user volunteers new information mid-flow, use it immediately rather than waiting for the "right" step.

---

## Algorithm

**Period is always Daily.** At the start of the flow, tell the user: "I'll set up your **Daily** planner page — a live morning brief from your connected tools." Never ask which period to set up.

**Run mode — detect before step 1:**
- **Scheduled run**: the incoming message contains `role:`, `tools:`, and `daily_content:` (config injected by the `schedule` skill). Do not show the survey. Extract the config. If a connector from the config is not detected, offer to walk the user through connecting it before continuing. **Skip steps 5 and 6 (preview and approval) — write directly to xTiles after the fetch. Also skip the schedule question in step 8 — the task is already scheduled.** Jump to **step 4**.
- **Fast-track or fresh manual run**: proceed to step 1.

### 1. Fast-track

If the user is specific ("give me daily for today", "I want to see Slack in the morning") — skip the full survey. Minimum needed: which connectors to pull from — infer from the message, then check the connector table (step 2) to confirm availability. If a required connector is not detected, offer to walk the user through connecting it (see **Connecting connectors**); wait for confirmation. Pull only from connectors both mentioned and confirmed available. Jump to **step 4**.

If the request is general — run the full flow.

### 2. Survey — who are you and what's connected

Collect role, tools, and Daily content via `db-survey` (survey mode — pass the detected connectors so it pre-selects their tool cards; receive `{role, tools, daily_content}`). The two-screen form is the only mechanism; `db-survey` supplies the text fallback if the widget can't render.

**Pre-selection probe (before rendering):** make a lightweight test call to each connector's Detect tool below (auth check only — e.g. `list_events` with `maxResults:1`, `slack_search_channels` with query `general`). For any connector that responds without an auth error, pre-select its card by setting `class="card sel"` **and the matching `data-tool`** in the fetched HTML before rendering — both must be set, or a user who deselects it will have the click silently do nothing and it'll leak back in. Change nothing else in the HTML.

**The submitted `tools` list is authoritative from this point on.** If the user deselects a pre-selected/technically-connected tool, it must not reappear anywhere downstream — not as a content option in step 3, not in the fetch in step 4, not as a tile in step 7 — regardless of what the probe found. "Detected" only controls whether a tool is *offered*; the user's actual selection controls whether it's *used*.

| Connector | Detect tool | Fetch (tool · query) | Tile(s) | Window |
|-----------|-------------|----------------------|---------|--------|
| Gmail | `search_threads` | `search_threads` · `is:important in:inbox newer_than:1d`; newsletters via sender/domain query | Emails; Newsletters | 24h |
| Slack | `slack_search_channels` | `slack_read_channel` per chosen channel + `slack_search_public_and_private` · `to:me` | Needs action / Mentions / Topics / Decisions / Open | 24h |
| Calendar | `list_events` | `list_events` · today | Workload | today |
| Linear | `list_issues` | `list_issues` · assigned/updated | Linear | 24h |
| Google Drive | `list_recent_files` | `list_recent_files` · modified | Google Drive | 24h |
| Granola | `list_meetings` | `list_meetings` · recent (context for Workload 🧠) | (context only) | 24h |
| Other | `ToolSearch` by name | connector's "recent items" tool · content-pref from survey | connector name + emoji | ~24h |

These connectors are external and optional — the user connects each one separately. xTiles itself (`xtiles_create_tiles_from_markdown_in_my_planner`) is required, not optional — see below.

**For "Other" connectors named by the user** — detect via `ToolSearch` with the connector's name; if not detected, walk through connecting via `mcp__mcp-registry__suggest_connectors` (say the connector's name explicitly before starting and after it connects). Carry every custom connector through every subsequent step — never drop one the user named. Right after naming/connecting it, ask (per Rules) what content it should contribute — 2–3 concrete options plus free text, e.g. for Plaud: meetings / meetings + action points / other. Store the answer as that connector's config entry (e.g. `Plaud:meetings+action_points`) — step 4's generic fetch uses this. A custom connector is only "connected" once it can actually produce a tile in step 4/7 — one that authenticates but never fetches must be surfaced as an error there, never silently dropped.

**If xTiles is not connected** — do not continue; walk the user through connecting it (see **Connecting connectors**) and wait for confirmation. **If a connector the user selected isn't connected** — walk them through connecting before moving on (per Rules); do not move on until they confirm it's connected or explicitly skip it.

### 3. Daily content — read from the survey (do not re-ask)

The survey's second screen already collects what to show each morning (`daily_content:` in the submit string) — never restate this as a separate question. Custom connectors contribute content via their own question above, not here. Tasks are never a content option — they're already in xTiles by default.

**If "Unread emails" is among `daily_content`** — ask (per Rules, ≤ 4 options): "Mark ⚪ Noise emails (notifications, nothing to act on) as read automatically?" Store as `mark_noise_read: yes`/`no` (default `no` if unanswered — never mark emails read without explicit opt-in). See step 4 for how it's applied.

**Slack channel discovery** (only if Slack selected and channels not already named):
1. **Anchor:** call `slack_read_user_profile` (no `user_id`) for the user's own title/department; combine with the survey role.
2. **Keywords:** expand the role into 3–4 distinctive terms (e.g. AI Engineer → ai, ml, llm, agent). Add any topics the user named.
3. **Search (in parallel):** `slack_search_channels` for the keywords + universal names (general, team, announcements, product); `slack_search_public_and_private` with `from:<@USER_ID>` and `to:me` (one page each) for where the user is actually active.
4. **Rank & pre-select:** rank by recent activity first, then keyword/created-by-user relevance; drop noise (random, fun, bots, test, hiring, onboarding) and DMs; pre-select up to 5 (activity + relevance first, ≤ 2 universal).
5. Hand the discovered channels to `db-survey` (channels mode), passing the up-to-5 to pre-mark; receive the selected channels. It renders the picker and supplies the text fallback.

**Newsletter discovery** (only if Newsletters selected, even if pre-picked in the survey): silently call `search_threads` with `from:(@substack.com OR @beehiiv.com OR @convertkit.com OR @mailchimp.com) newer_than:30d`; extract unique sender/publication names. If found, hand them to `db-survey` (newsletters mode) to render the picker; if none found, `db-survey`'s fallback asks for the newsletter names as free text. Add all selected/typed senders to the config. If the user writes something custom, add it as-is.

### 4. Silent data fetch

Silently, without messaging the user, pull fresh data per the connector table above:

- **Gmail — unread emails**: `search_threads` per the table's query. For each thread, `get_thread` for sender/subject and the direct link `https://mail.google.com/mail/u/0/#inbox/{threadId}`; also capture each individual message's `messageId` (a thread can hold several), needed for `mark_noise_read` below.
- **Gmail — newsletters**: `search_threads` with `from:({senders} OR @substack.com OR @beehiiv.com OR @convertkit.com) is:unread newer_than:1d`; `get_thread` per result for a one-line summary + link.
- **Slack**: in parallel, `slack_read_channel` per chosen channel (top 50, filter to last 24h, discard older, skip empty channels silently) and `slack_search_public_and_private` with `to:me` (last 24h) for mentions/DMs across all channels. Every item gets a direct message permalink (`permalink` field, or build `https://slack.com/archives/{channel_id}/p{ts_without_dot}`) — never link to the channel homepage.
  - **Mentions** *(highest priority)* — every `to:me` result: who mentioned the user, where, what was asked, one line + permalink. Flag ⚡ if it needs a response.
  - **Needs action** — every ⚡ mention becomes a poke-style line here too, plus one verb-first action item per item — these join the same task-dedup pool as the email action items below (collect together, dedup once).
  - **Topics** — group by theme, one line each, permalink to the most relevant message, `[#channel](permalink)` attribution.
  - **Decisions** — where something was agreed/committed/confirmed, with permalink.
  - **Open** — unanswered questions, permalink, mark ⏳.
- **Calendar**: `list_events` for today. Per event: time range, title, participants, meeting link. Compute a bold summary line (event count, total hours occupied, longest free focus window), one bold line per event, 🧠 context (only if Granola/Gmail connected — last relevant Granola note or open Gmail thread with the organiser, one sentence, omit silently if none found), and ⚠️ anomalies collected at the bottom (overlaps, back-to-back with no gap, events after 20:00, missing agenda, likely duplicate titles).
- **Linear**: `list_issues` assigned to or updated by the user in the last 24h. Two lines: **New** and **Updated**, each `[Title](url) — status`. Always create the tile — write "No updates today." if empty (its total absence would look like a connector failure).
- **Google Drive**: `list_recent_files` modified in the last 24h — one line per file, `[File name](url) — edited by Name`, newest first. Omit the tile entirely if nothing changed (a quiet Drive day is unremarkable).
- **Custom ("Other") connectors** — this skill can't hardcode every tool, so each one gets this generic path: use the MCP tools discovered in step 2 (search again with `ToolSearch` if needed); pull data matching the stored content preference (e.g. `Plaud:meetings+action_points`), ~24h window unless the connector only supports a longer one. If the fetch succeeds, it gets its own `###` tile in step 7 (connector name + fitting emoji). **If the fetch fails, or no working tool can be found at all** — never drop it silently: surface "Could not fetch [Connector] data — connector error" in both the step 5 preview and the step 7 tile.

Classify emails into three buckets (newsletters are fetched separately — exclude them here entirely):
- 🔴 **Needs action** — user must reply, decide, act, or log in.
- 🟡 **FYI** — confirmed/settled items — past or present tense, nothing to do.
- ⚪ **Noise** — notifications, automated alerts, service emails — count only, do not describe individually.

**If `mark_noise_read: yes`**: for every thread placed in ⚪ Noise, call `unlabel_message` with `labelIds: ["UNREAD"]` once per individual **message** in that thread (using each message's own `messageId` captured above) — one call per message, or only part of a multi-message thread gets marked read. Do this silently, every run — don't ask again. Mention the count in the preview/tile, e.g. "⚪ Noise — N notifications (sources) — marked as read." If `mark_noise_read` is `no`/unset, leave messages untouched — count only.

**Tone for 🔴 and 🟡 — poke-style, capitalized:** retell the email as a second-person poke — subject → action → consequence — do not copy the subject line; for 🔴, weave the next step in; use people's names, not addresses (context in parentheses if useful); telegraphic, conversational, first letter capitalized, no bureaucratic language. 🟡 items are one-liners — no link needed.

For every 🔴 email, derive one verb-first action item. Collect as a flat list, combined with the verb-first items derived from Slack's ⚡ mentions above — one shared list across both sources. Call `xtiles_list_tasks` **once** with `completed: false` to fetch all open tasks; for each combined action item, drop it if an open task with the same or very similar meaning already exists. Keep only unmatched items — email-derived ones go into the Emails tile's Action items block, Slack-derived ones go into the Needs action tile's Tasks block (step 7).

Use only real data from connectors — never invent names, events, or messages; every name and message must come directly from API responses. If a connector call fails (error, timeout, 401), record the failure and surface it explicitly in step 5 — never write "No data" for a failed call.

### 5. Preview — show content in chat

Show real content with real data — not structure, not "(TBD)" headings. Open with "Here's what I've prepared:" then `📅 DAILY — [actual date]`. Render only the sections the user selected, in the **exact same section formats, `###` headers, and blank-line-between-items spacing as step 7's tile content** below (Emails' 🔴/🟡/⚪ buckets + Action items, Newsletters, Slack's five tiles, Workload, Linear, Drive, custom connectors) — just as plain chat text, without the `@colorSize`/`@color` annotations.

Rules: if a connector returned no data, write exactly that ("No unread emails", "No newsletters today", "No Slack updates today") — never skip the section silently, its absence looks like a bug; if a connector call failed, write "Could not fetch [connector] data — connector error"; no placeholder names, example events, or invented data, ever. After the preview, **stop and wait** — do not write to xTiles yet.

### 6. Approval

**Mandatory. Never skip.** After the preview, ask (per Rules) with three options: **"Looks good — create it"** / **"Edit"** / **"Cancel"**. Do not call the write tool until the user explicitly picks the first option. On "Edit", ask what to change, update only that section, re-show the preview, and ask again.

### 7. Write to xTiles

**Only after explicit approval.** Tool: `xtiles_create_tiles_from_markdown_in_my_planner` — `period: "day"`, `date`: current date in ISO 8601. Write **all** selected sections in a single call — never split into per-connector calls, and never write a date/header-only tile first; start directly with content tiles.

Each `###` section is immediately followed (no blank line) by:
```
@colorSize: LIGHTER
@color: [COLOR]
```
Pick `@color` randomly per section, **exactly** from: `GHOST, CUMULUS, GOSSIP, COLDTURKEY, BLUE_CHALK, MILK_PUNCH, HAWKES_BLUE, PATTENS_BLUE, SAIL, ATHENS_GRAY, BERMUDA, PERFUME, SELAGO, RICE_FLOWER, WHITE_LINEN, POLAR` — never a semantic name (RED, BLUE, GREY, …), they don't render. Don't repeat the same color twice in a row.

All links are Markdown hyperlinks (`[text](url)`, never bare); blank line between items, never a continuous block.

- **Emails**: 🔴 Needs action (N) → poke-style lines with `[Open email](url)`; 🟡 FYI (N) → one-liners, no link; ⚪ Noise → count only; then `---` and **Action items** checkboxes (omit if no 🔴). Newsletters never appear here.
- **Newsletters**: single `### 📧 Newsletters` tile, never one per newsletter — each entry is a bold hyperlinked title + one-line summary, blank line between entries. Omit only if there are no unread newsletters at all.
- **Slack** — separate tiles per category, all links to the message permalink:
  - `### 🔴 Needs action` — one line per ⚡ mention + a **Tasks** checkbox block (same dedup pool as email action items). Omit if no ⚡ mentions. This is a rollup, not a replacement — the same messages still appear in Mentions.
  - `### ⚡ Mentions` — one line per mention, ` ⚡` if a response is needed. Omit if none.
  - `### 💬 Topics` — **Channels:** line then one line per topic. **Always create** — write "No updates today." if empty.
  - `### ✅ Decisions` — omit if none. `### ❓ Open` — `⏳` suffix, omit if none.
- **Calendar**: `### 📅 Workload` — bold summary line first, one bold line per event (+ participants/link), 🧠 only when context found, ⚠️ anomalies collected at the bottom, blank line between items. Omit entirely if no events.
- **Linear**: `### 📌 Linear` — **New** / **Updated** blocks. Always create — "No updates today." if empty.
- **Google Drive**: `### 📁 Google Drive` — one line per file. Omit entirely if nothing changed.
- **Custom connectors**: one `###` tile per connector (name + emoji), structured to match its stored content preference, same `@colorSize`/`@color` annotation. If its fetch failed, still create the tile with "Could not fetch [Connector] data — connector error" — never omit it silently.

**If xTiles is not connected** — never output the digest as plain chat text; walk the user through connecting it (see **Connecting connectors**), wait for confirmation, then write.

**If the page already exists:** call `xtiles_get_planner_content`, compare existing `###` headers with what you're about to add, append only sections whose headers don't exist yet; if everything already exists, ask (per Rules): replace all, append anyway, or cancel.

**After the single successful write:**
1. Say `✅ Daily created.`
2. Read `view_id` and `tile_ids` from the write-call response (do not re-derive via `get_planner_content`). `tile_ids[i]` matches the *i*-th `###` section.
3. **Layout pass — run `tile-layout` once.** Call `xtiles_get_workflow` id `tile-layout`, follow it with these tiles as "added tiles" and the markdown just written as their content. Hints: 1–N tiles · 2 per row · a very heavy tile gets a full-width row. (Scheduled runs included; you may fetch `tile-layout` once per session and reuse it.)
4. Output the markdown CTA link `**[Open in xTiles →](https://xtiles.app/{view_id})**`.
5. Non-scheduled runs: ask the Schedule question (step 8).

**If an error occurs:** briefly say what went wrong, offer to retry or skip that page.

### 8. Schedule (optional)

Ask (per Rules): days (Weekdays / Every day) and time (free text, default 09:00).
- **"Yes, schedule it"** — invoke `anthropic-skills:schedule`, then `mcp__scheduled-tasks__create-scheduled-tasks` with:
  - `prompt`: `Run daily digest — role: {role} · tools: {tools} · daily_content: {content} · schedule: daily-{HH:MM} days:{days}` — real values, no placeholders; append `· mark_noise_read: yes` only when it was answered "yes" (omit the field for "no", the default).
  - `schedule`: cron `M H * * {days}` (`1-5` for Weekdays, `*` for Every day; default `0 9 * * 1-5` if unanswered).
  - `timezone`: from `xtiles_get_user_timezone` (fetch once per session if not already fetched).
  Confirm: "Done — your Daily will be ready in xTiles every morning at [chosen time]." Then output the CTA link again (`https://xtiles.app/{view_id}` from step 7).
- **"No, thanks"** — acknowledge briefly.

Either way, continue to **step 9** — do not stop here.

### 9. Related workflows

**After every manual run, once step 8 is resolved** — offer related workflows. Skip on scheduled runs, which end silently after step 7.

Ask (per Rules) single-select: "Want to set up anything else on xTiles?" — 🌙 Evening Reflection (an end-of-day synthesis seeded for tomorrow) / 📰 Today News (a daily news digest on topics you care about) / 📊 Weekly Review (a weekly summary of what moved forward this week) / Nothing else, thanks. On a pick, emit exactly:
- Evening Reflection → `Set workflow of Evening Reflection (evening-reflection) on xTiles MCP`
- Today News → `Set workflow of Today News (today-news) on xTiles MCP`
- Weekly Review → `Set workflow of Weekly Review (weekly-review) on xTiles MCP`
- "Nothing else" — acknowledge briefly and stop.

---

## Connecting connectors

Never send the user to settings manually and never hand them a URL to follow. Call `mcp__mcp-registry__suggest_connectors` with the missing connector names — it renders interactive connect buttons directly in the UI. Tell the user: "Click connect above, then tell me when you're done." — and re-probe on their reply.

**Re-probe before declaring success — never trust the reply alone.** Re-run the same lightweight detection call from step 2 for each connector just connected. Only for the ones that now respond without an auth error, confirm "Connected. Continuing…" and resume the flow. **If a connector still fails detection** — say so plainly (e.g. "Outlook still isn't responding — want to try connecting it again, or skip it for now?"). Never silently continue as if it worked — a connector that reports "connected" but has no working fetch path in step 4 is exactly how it gets lost from the digest without anyone noticing.
