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
  mcp__claude_ai_Granola__list_meetings,
  mcp__claude_ai_Google_Drive__list_recent_files,
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

**For "Other" connectors named by the user** — treat them identically to the known connectors above: attempt detection via available MCP tools; if not detected, walk through connecting via `mcp__mcp-registry__suggest_connectors`. Carry the full list of selected tools through every subsequent step — never discard what the user picked.

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
Call `mcp__claude_ai_Slack__slack_search_channels` with no query to get the full list. From the results, surface up to 8 channels prioritised as follows:
1. Channels whose name contains keywords like `team`, `general`, `product`, `growth`, `sales`, `eng`, `dev`, `design`, `support`, `alert`, `urgent`, `announce` — these are typically high-signal
2. Exclude obviously low-signal channels: `random`, `fun`, `off-topic`, `bots`, `test`, `hiring`, `onboarding`
3. If the list is still long, prefer shorter channel names (usually the main ones)

Present the shortlist via `AskUserQuestion` (multiSelect: true):
"Which channels do you open first each morning?"

**If Newsletters is selected:**
First, silently call `mcp__claude_ai_Gmail__search_threads` with query `from:(*@substack.com OR *@beehiiv.com OR *@convertkit.com OR *@mailchimp.com) newer_than:30d` to discover newsletters already in the inbox. Extract unique sender/publication names from results.

If publications found — present as multi-select via `AskUserQuestion` (multiSelect: true):
"Which newsletters do you want in your Daily? I found these in your inbox:"

If nothing found — ask: "Which newsletters do you want to track? Name the sender or publication — e.g. 'Morning Brew', 'Lenny's Newsletter'."

Add all selected/typed senders to the config. Tip: newsletters typically come from `@substack.com`, `@beehiiv.com`, `@convertkit.com`, `@mailchimp.com`.

**General rule:** if the user writes something custom — add it as-is. Don't reshape it into a predefined option.

---

### 4. Silent data fetch

**Silently, without messaging the user**, pull fresh data from connectors based on selected sections and content choices:

- **Gmail — unread emails**: `mcp__claude_ai_Gmail__search_threads` — query `is:unread is:important in:inbox newer_than:3d`. For each thread call `mcp__claude_ai_Gmail__get_thread` to get sender, subject, and threadId for the direct link (`https://mail.google.com/mail/u/0/#inbox/{threadId}`).
- **Gmail — newsletters**: `mcp__claude_ai_Gmail__search_threads` — query `from:({sender1} OR {sender2} ... OR *@substack.com OR *@beehiiv.com OR *@convertkit.com) is:unread newer_than:1d` — combine user-named senders with common newsletter domains. Fetch each thread with `get_thread` for a one-line summary and `threadId` for the link.
- **Slack**: call `mcp__claude_ai_Slack__slack_read_channel` for each chosen channel (top 50 messages). After reading, **filter to messages from today only** (timestamp ≥ 00:00 local time). Discard older messages. If no messages from today in a channel — skip that channel silently.

  After filtering, analyse all Slack messages across all channels and group them semantically:
  - **Topics** — what was discussed; group by theme, one topic = one line, link to the most relevant message, include channel attribution: `[#channel](url)`
  - **Decisions** — where something was agreed, committed to, or confirmed
  - **Open questions** — where a question was raised but no clear answer came yet; mark as ⏳

- **Calendar (Workload)**: `mcp__claude_ai_Google_Calendar__list_events` — today's events. Compute:
  - **Day shape** (one summary line): meetings count, total hours occupied, longest free focus window with exact times, and an evening density note if the schedule is heavy after 18:00 — e.g. "4 зустрічі, 4 год зайнято. Вільне вікно 13:38–18:00 — найдовший фокус-блок. Вечір щільний (18:00–23:30)."
  - **Conflicts**: overlapping events with exact times; back-to-back with no gap between them
  - **Anomalies**: prefix with ⚠️ — events after 20:00, events without description/agenda, and potential duplicates (two events with similar titles close together, show as "Дубль? [Title A] та [Title B] поспіль — перевір запрошення")
  - **Context per meeting** (prefix with 🧠, only if Granola or Gmail connected): for each notable meeting name the meeting, then — last Granola note involving the same participants, and/or most recent open Gmail thread with the organiser. Format: "до [Meeting name] з [Participant] — [Granola note title] + [Gmail thread subject]"

Analyze what you get. Classify each email/Slack signal:
- 🔴 needs a decision, reply, or action today
- 🟡 informational / can wait

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
🔴 [Subject — from Sender](https://mail.google.com/mail/u/0/#inbox/{threadId})

🟡 [Subject — from Sender](https://mail.google.com/mail/u/0/#inbox/{threadId})

*(emoji always before the `[`, never inside)*

### Newsletter: [Name]
[One-line summary]
[Open](https://mail.google.com/mail/u/0/#inbox/{threadId})

### 💬 Slack
**Channels:** #channel1 (N) · #channel2 (N)

#### 💬 Topics
- **[Topic name]** — [one-sentence summary] — [#channel](url)

#### ✅ Decisions
- [Decision] — [#channel](url)

#### ❓ Open
- [Question] — [#channel](url) ⏳

### ⚡ Workload
[N] meetings · [X]h occupied · focus window [HH:MM]–[HH:MM]
⚠️ [Conflict if any]

---

**[Meeting name]**
[Calendar event](url)
🧠 [context if available]

---

---
```

Each email entry must be a Markdown hyperlink using the real `threadId` from `get_thread`.
Each newsletter is shown as its own named section in the preview.
Separate each item with a blank line for readability.

**Rules:**
- Show only selected sections the user asked for
- If a connector returned no data — write exactly that ("No unread emails", "No newsletters today") — don't skip silently
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

**Write all sections in a single call.** Combine all selected connectors (Gmail, Slack, etc.) into one markdown and call the tool once — never split into separate calls per connector.

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
- **Emails**: each entry is a Markdown hyperlink — `🔴 [Subject — from Sender](https://mail.google.com/mail/u/0/#inbox/{threadId})` — the priority emoji goes BEFORE the `[`, never inside the brackets
- **Newsletters**: one separate `###` tile per newsletter; tile title = newsletter name. Body: short 2–3 sentence summary. Last line: **`[Open in Gmail](https://mail.google.com/mail/u/0/#inbox/{threadId})`** — wrap the link text in bold so it renders as a visible hyperlink, not plain text: `**[Open in Gmail](url)**`. **Skip the tile entirely if there are no unread issues from that newsletter.**
- **Slack**: **ALL Slack channels go in a SINGLE `### 💬 Slack` tile** — never split channels into separate tiles. Structure the tile content using semantic `####` subheadings:
  - First line (no subheading): channel activity summary — `**Channels:** #channel1 (N) · #channel2 (N)`
  - `#### 💬 Topics` — one line per topic: `- **Topic name** — one-sentence summary — [#channel](url)`
  - `#### ✅ Decisions` — one line per decision: `- Decision made — [#channel](url)`. Omit subheading if no decisions.
  - `#### ❓ Open` — one line per unanswered question: `- Question — [#channel](url) ⏳`. Omit subheading if no open questions.
  - Omit the entire tile if no messages from today.
- **Workload**: single tile titled `⚡ Workload`. First line: day shape summary (meetings count, hours occupied, longest focus window, evening density if applicable). Then `⚠️` conflict/anomaly lines if any — each as its own paragraph. Then one block per meeting, separated by `---` dividers:

  ```
  ---

  **[Meeting name]**
  [Calendar event](url)
  🧠 [context — Granola note or Gmail thread with organiser, only if available]

  ---
  ```

  Use the event's `htmlLink` from Calendar as the URL. Omit the 🧠 line if no Granola or Gmail context is available. Omit Workload tile entirely if Calendar returned no events.
- This ensures the tile is scannable, not a wall of text

**If xTiles is not connected** — do not output the digest as plain text in chat. Walk the user through connecting xTiles (see **How to connect connectors**), wait for confirmation, then write.

**If the page already exists:**
1. Call `mcp__xtiles__xtiles_get_planner_content`
2. Compare existing H3 headers (`###`) with what you're about to add
3. Append only sections whose headers don't exist yet
4. If everything already exists — ask: replace all, append anyway, or cancel?
   **After each successful write — run these three steps in order, no exceptions:**

   1. Write `✅ Daily created.`
   2. Call `mcp__xtiles__xtiles_get_planner_content` with the same `date` and `period`. Extract `view_id`. Call `show_widget` with the **CTA widget HTML** (see below), replacing `{VIEW_URL}` with `https://xtiles.app/{view_id}`. Translate the button label into the user's language. **Never output a markdown link instead of the widget.**
   3. Immediately call `show_widget` with the **Schedule widget HTML** (see below). Do not skip this step, do not ask first — just show it.

**If an error occurs:** briefly say what went wrong, offer to retry or skip that page.

---

### 8. Schedule (optional)

The schedule widget is shown in step 7 above. This step handles the user's response.

In Claude Code (no Cowork): after writing, ask inline: "Want me to run this every morning automatically? What time? (default: 9:00 AM)"

- If the user selects **"Yes, schedule it"** — first invoke `anthropic-skills:schedule`, then call `mcp__scheduled-tasks__create-scheduled-tasks`. Pass to both:
  - **`prompt`**: the full config string assembled from values collected during setup —
    ```
    Run daily digest — role: {role} · tools: {tools} · daily_content: {content} · schedule: daily-9am
    ```
    Replace `{role}`, `{tools}`, `{content}` with the actual values. Do not leave placeholders.
  - **`schedule`**: cron derived from the widget. The widget sends `cron: HH:MM days:1-5` (weekdays) or `cron: HH:MM days:*` (every day) — parse both values and build: `M H * * 1-5` for weekdays, `M H * * *` for every day. Default `0 9 * * 1-5` (9:00 AM on weekdays) if not found.
  - **`timezone`**: the user's local timezone — call `mcp__xtiles__xtiles_get_user_timezone` to get it before scheduling if it hasn't been fetched yet.

  This prompt fires each morning and triggers `daily-brief` in scheduled-run mode — the full config must be embedded so the survey is skipped automatically.

  After scheduling succeeds, confirm: "Done — your Daily will be ready in xTiles every [weekday morning / morning] at [chosen time]." (say "weekday morning" if `days:1-5`, "every morning" if `days:*`) Then call `show_widget` with the **CTA widget HTML**, replacing `{VIEW_URL}` with `https://xtiles.app/{view_id}` (the same `view_id` from step 7). Never output a markdown link here — always the button widget.
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
  var r=role==='__other__'?document.getElementById('role-other-in').value.trim():role;
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
    <button class="btn btn-yes" onclick="scheduleIt()">Yes, schedule it</button>
    <button class="btn btn-no" onclick="sendPrompt('No schedule needed')">No, thanks</button>
  </div>
</div>
<script>
function scheduleIt(){
  var days=document.getElementById('sched-days').value;
  var t=document.getElementById('sched-time').value||'09:00';
  var parts=t.split(':'),h=parseInt(parts[0],10),m=parts[1];
  var label=(h%12||12)+':'+m+' '+(h>=12?'PM':'AM');
  var dLabel=days==='1-5'?'weekdays':'every day';
  sendPrompt('Yes, schedule my daily digest at '+label+' '+dLabel+' (cron: '+t+' days:'+days+')');
}
</script>
```

---

## How to behave

- Use the survey widget for setup; ask inline for approval and any follow-up clarifications
- **Never output the digest as plain text in chat** and ask the user to copy it manually — always write to xTiles directly, or walk through connecting xTiles first
- **Never skip a connector** the user selected — if it's not connected, walk through the connection before continuing, don't silently drop it
- Never create anything without preview and explicit approval
- Never put example names, example events, or example messages into the preview — only real data from connectors
- **All clarifying questions after the main survey form** (channel selection, newsletter names, approval, change requests) — use `AskUserQuestion`, not plain text
- If context is missing — ask, don't guess
- If the user gives new information along the way — pick it up, don't wait for the "right step"
- Real data from connectors always beats placeholders
- Daily is the only period. If the user asks for Weekly or Monthly, tell them only the Daily planner is currently supported and offer to create a Daily page instead — never silently downscope.
- Match the user's language, adapt if they switch
- Show the survey widget in Cowork only — in Claude Code, ask the same questions inline
 