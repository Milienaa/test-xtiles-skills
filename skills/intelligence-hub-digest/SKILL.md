---
name: intelligence-hub-digest
description: >
  Use when the user wants to set up OR run their xTiles cascading planner —
  a Month → Week → Day system where the Day is a live brief from connected tools
  (Slack, Gmail, Calendar, Analytics) plus signals that need attention.

  Setup triggers: "set up my planner", "personalize my workspace",
  "connect my planner to my tools", "create daily/weekly/monthly",
  "onboard a new xTiles user into the Planner".

  Digest triggers: "show me my morning brief", "what do I need to know today",
  "run my digest". Also runs automatically via scheduled tasks.

  Config is read from the scheduled task prompt — no separate file needed.
  For manual runs: look for config in today's Planner; if there's none, start
  from the survey flow below.
allowed-tools: mcp__xtiles__xtiles_get_planner_content, mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner
---

# xTiles Cascading Planner — Setup & Daily Digest

## Three principles

1. **Survey first, write to xTiles last.** Nothing gets created until the user has seen the preview and said "yes".
2. **Daily ≠ Weekly ≠ Monthly.** Each period solves a different problem — content and questions differ accordingly.
3. **Real data, not placeholders.** Pull from connectors before preview so the user sees live content.

**Language:** match the language of the user's first message.

---

## Algorithm

### 1. Fast-track

If the user is specific ("give me daily for today", "I want to see Slack in the morning") — skip the full flow. Collect the minimum needed and jump to Preview (step 4).

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
| xTiles | `xtiles_create_tiles_from_markdown_in_my_planner` |

If only xTiles is connected — ask: set up statically or connect tools first?
If the user chooses to connect tools, walk them through **How to connect connectors** (section below).

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

- **Slack connected** → call `slack_search_channels`, show top 4 channels, ask which ones they open first each morning
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

Tool: `xtiles_create_tiles_from_markdown_in_my_planner`
- `period`: "day" / "week" / "month"
- `date`: current date in ISO 8601

**Order:** day → week → month.

**If the page already exists:**
1. Call `xtiles_get_planner_content`
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
<!-- [full HTML form from v4 — role, tools, pages] -->
<!-- Use the latest version of the form from previous iterations -->
```

---

## How to connect connectors

If the skill needs data from Slack, Gmail, Calendar, or other tools, the connectors
must be connected manually. Walk the user through these steps in Cowork:

1. **Open the plugin settings** — in the Cowork left panel, find the installed plugin
   (e.g. *Slack by Salesforce*). Click the three dots `···` next to its name → **Customize**.
2. **Go to Connectors** — in the customization window, open the **Connectors** tab.
   It lists the connectors available for that plugin.
3. **Connect the connector** — click **Connect** next to the one you need (e.g. Slack).
   An authorization window opens — sign in and grant permissions.
4. **If the browser shows an error after authorization** — sometimes after clicking
   "Allow" the page fails to load and shows a connection error. This is expected. Copy
   the full URL from the address bar (it looks like
   `http://localhost:3118/callback?code=...&state=...`) and paste it into the chat —
   Claude will finish the authorization.
5. **Verify the connection** — after successful authorization the connector shows as
   **Connected**. The skill can now read data from that tool.

After connecting, re-run connector detection (the table in step 2) and continue the flow.

## How to behave

You're an assistant helping someone set up a planner that fits their real work rhythm.

- Don't create anything without a preview and approval
- If context is missing — ask, don't guess
- If the user gives new information along the way — pick it up, don't wait for the "right step"
- Real data from connectors always beats placeholders
- Daily, Weekly, Monthly — different tasks, different content, different questions
- Match the user's language (EN/UA), adapt if they switch
