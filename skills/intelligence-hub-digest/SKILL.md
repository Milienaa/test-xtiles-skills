---
name: intelligence-hub-digest
description: >
  Use when the user wants to set up OR run their xTiles Daily planner —
  a Daily page that serves as a live morning brief from connected tools
  (Slack, Gmail, Calendar, Analytics) plus signals that need attention.
  Only the Daily period is supported.

  Setup triggers: "set up my planner", "personalize my workspace",
  "connect my planner to my tools", "create daily",
  "onboard a new xTiles user into the Planner".

  Digest triggers: "show me my morning brief", "what do I need to know today",
  "run my digest". Also runs automatically via scheduled tasks.

  Config is read from the scheduled task prompt — no separate file needed.
  For manual runs: look for config in today's Planner; if there's none, start
  from the survey flow below.
allowed-tools: mcp__xtiles__xtiles_get_planner_content, mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner, AskUserQuestion, slack_search_channels
---

# xTiles Daily Planner — Setup & Daily Digest

## Three principles

1. **Survey first, write to xTiles last.** Nothing gets created until the user has seen the preview and said "yes".
2. **Real data, not placeholders.** Pull from connectors before preview so the user sees live content.
3. **Match the user's language** throughout the entire flow — match the language of the user's first message and adapt if they switch.

---

## Algorithm

**Period is always Daily — never ask the user which planner pages to set up. Skip any period selection step entirely.**

### 1. Fast-track

If the user is specific ("give me daily for today", "I want to see Slack in the morning") — skip the full flow. Collect the minimum needed and jump to Preview (step 4).

If the request is general — run the full flow.
 
---

### 2. Survey — who are you and what's connected

**Show the survey** using AskUserQuestion batches.

**Connected tools** (multi select, show all regardless of what's actually detected):
- Slack
- Gmail
- Google Calendar
- PostHog
- Amplitude
- Other (describe in next message)

If "Other" is selected — ask a follow-up: "Which tool(s)? Just name them — I'll figure out what's available."

After receiving answers — detect which MCP tools are actually available:

| Connector | Identifying MCP tools |
|-----------|----------------------|
| Slack | `slack_send_message`, `slack_read_channel` |
| Gmail | `search_threads`, `list_labels`, `get_thread` |
| Calendar | `list_events`, `create_event` |
| PostHog | `query_chart` + `get_from_url` + `get_events` |
| Amplitude | `query_chart` + `get_experiments` without `get_from_url` |
| xTiles | `xtiles_create_tiles_from_markdown_in_my_planner` |

These connectors are external and optional — they are not shipped with this plugin. The user must connect them separately.

If a tool the user selected isn't actually connected — note it and offer to walk them through connecting it (see **How to connect connectors** below).

If only xTiles is connected — ask: set up statically or connect tools first?

---

### 3. Daily content clarification

Question: "What do you want to see on your Daily each morning?"

Options — include only those relevant to connected tools:
- Unread emails that need a reply *(only if Gmail connected)*
- Slack messages from key channels *(only if Slack connected)*
- Today's meetings *(only if Calendar connected)*
- Key metrics *(only if PostHog or Amplitude connected)*
- Evening reflection prompts
- Other (describe in next message)

Do NOT suggest tasks — they're already in xTiles by default.

**If Slack is selected — always ask, without exception:**
Call `slack_search_channels`, show up to 6 channel names. Ask (single select, multi allowed):
"Which channels do you open first each morning? Pick all that matter."
Also offer: "Other / I'll type the names"

**If metrics are selected and PostHog/Amplitude is connected:**
Ask for chart links or metric names to pull.
- PostHog: use `get_from_url` + `query_chart`
- Amplitude: `get_from_url` unavailable — save URL as text, fetch via `query_chart` / `get_experiments`
  **General rule:** if the user writes something custom — add it as-is. Don't reshape it into a predefined option.

---

### 4. Silent data fetch

**Silently, without messaging the user**, pull fresh data from connectors based on selected pages and content choices:

- **Calendar**: events for today / this week / this month as needed
- **Gmail**: unread important messages (`is:unread is:important in:inbox newer_than:3d`)
- **Slack**: recent messages from the user's chosen channels (top 20–30)

Analyze what you get. Classify each signal:
- 🔴 needs a decision, reply, or action today
- 🟡 informational / can wait

Use only real data from connectors. Do not invent names, events, or messages.
All names, meeting titles, and message content must come directly from API responses — never from examples in this skill file.

---

### 5. Preview — show content in chat

Show real content with real data. Not structure, not headings with "(TBD)" — actual text.

Format (adapt to selected pages):

```
Here's what I've prepared:

---
📅 DAILY — [actual date]

### [Section based on user's choices]
🔴 [Real signal from connector] | [link if available]
🟡 [Real signal from connector]

### Today's meetings
[Real events from Calendar with actual titles and times]

### Gmail
[Real unread threads or "Inbox clear — no important unread email"]

---
```

**Rules:**
- Show only selected pages and sections the user asked for
- If a connector returned no data for a section — write exactly that ("No unread emails", "No meetings today") — don't skip silently
- No placeholder names, example events, or invented data — ever
- After the preview, ask: "Does this look right? Anything to change?"
---

### 6. Approval

`AskUserQuestion` (single select):
- **"Looks good — create it"** → proceed to write
- **"Change something"** → ask what, update only that section, show preview again
- **"Cancel"** → stop
  If the user asks for a change — clarify exactly what, update only that section, re-show preview, ask again.

---

### 7. Write to xTiles

**Only after explicit approval.**

Tool: `xtiles_create_tiles_from_markdown_in_my_planner`
- `period`: "day"
- `date`: current date in ISO 8601
  **Always write only the Daily page.**

**If the page already exists:**
1. Call `xtiles_get_planner_content`
2. Compare existing H3 headers (`###`) with what you're about to add
3. Append only sections whose headers don't exist yet
4. If everything already exists — ask: replace all, append anyway, or cancel?
   **After each successful write:**
   Call `xtiles_get_planner_content` with the same `date` and `period`.
   Extract the `view_id` from the response and include a link in the confirmation:

```
✅ [Page name] created.
🔗 [Open in xTiles](https://xtiles.app/{view_id})
```

Translate the link label ("Open in xTiles") into the user's language.

**If an error occurs:** briefly say what went wrong, offer to retry or skip that page.

---

### 8. Schedule (optional)

Ask with `AskUserQuestion` (single select):
- Daily at 9:00 AM
- No schedule needed

Only show if Daily was created. After confirming schedule, run the `schedule` skill (host-provided capability — not shipped with this plugin).
---

## How to connect connectors

If a tool the user selected isn't connected, walk them through:

1. **Open plugin settings** — in the Cowork left panel, find the plugin. Click `···` → **Customize**.
2. **Go to Connectors tab** — lists all available connectors for that plugin.
3. **Click Connect** next to the tool you need. Sign in and grant permissions.
4. **If the browser shows an error after authorization** — this is expected sometimes. Copy the full URL from the address bar (looks like `http://localhost:3118/callback?code=...&state=...`) and paste it into chat — Claude will finish the authorization.
5. **Verify** — connector shows as **Connected**. Re-run the flow.
---

## How to behave

- Use `AskUserQuestion` for all survey and approval steps — no HTML widgets, no ad-hoc UI
- Never create anything without preview and explicit approval
- Never put example names, example events, or example messages into the preview — only real data from connectors
- If context is missing — ask, don't guess
- If the user gives new information along the way — pick it up, don't wait for the "right step"
- Real data from connectors always beats placeholders
- Daily is the only period. If the user asks for Weekly or Monthly, tell them only the Daily planner is currently supported and offer to create a Daily page instead — never silently downscope.
- Match the user's language (EN/UA), adapt if they switch
 