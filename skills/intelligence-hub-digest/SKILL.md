---
name: intelligence-hub-digest
description: >
  Fetches data from connected sources (Gmail, Slack, Google Calendar, Notion, GitHub, Figma, Analytics, and others),
  classifies items by priority, formats a digest, and writes it to the user's xTiles Planner.

  Runs automatically via scheduled tasks. Also trigger when the user says:
  "show me my morning brief", "what do I need to know today", "run my digest", "what happened today".

  Config is read from the scheduled task prompt — no separate file needed.
  For manual runs: look for config in today's Planner. If not found — suggest running intelligence-hub-setup first.
allowed-tools: mcp__xtiles__xtiles_get_planner_content, mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner
---

# Intelligence Hub — Digest

Runs on schedule or on demand. No questions — just fetch, classify, write.

---

## Config

Read from the scheduled task prompt body:

```json
{
  "user": { "timezone": "...", "language": "detect from conversation" },
  "sources": {
    "gmail":     { "enabled": true,  "watchlist": [], "blacklist": [] },
    "slack":     { "enabled": true,  "watchlist": ["#general"], "blacklist": [] },
    "calendar":  { "enabled": true,  "upcoming_days": 1 },
    "notion":    { "enabled": false, "watchlist": [] },
    "github":    { "enabled": false, "watchlist": [] },
    "figma":     { "enabled": false, "watchlist": [] },
    "analytics": { "enabled": false, "watchlist": [] }
  }
}
```

**Manual trigger:** call `xtiles_get_planner_content` for today and look for a config block. If not found — tell the user to run intelligence-hub-setup first.

**Preview from setup:** use sources and watchlists passed as context. Skip deduplication.

---

## Deduplication

Before fetching, call `xtiles_get_planner_content` for today. If the relevant `###` sections already exist — stop silently. Re-run only if the user explicitly asks.

---

## Fetch

Don't narrate the fetching process.

**Gmail** (if enabled):
`search_threads` with `query="is:unread is:important in:inbox newer_than:1d"` → `get_thread` for each result.
Boost watchlist senders, filter blacklist.

**Slack** (if enabled):
`slack_read_channel` for each channel in watchlist.
Today only — discard older messages.
Flag: direct mentions, open questions, decisions, deadlines.

**Google Calendar** (if enabled):
`list_events` — today + tomorrow for morning brief, today only for evening.
Extract: time, title, participants, link.

**Notion** (if enabled):
`notion-search` filtered to watchlist workspaces/pages. Today only.

**GitHub** (if enabled):
GitHub MCP tools. Fetch: open PRs assigned to user, review requests, @mentions in issues.
Filter by watchlist repos if set.

**Figma** (if enabled):
`get_metadata` for watchlist files. Flag new comments and recent changes.

**Analytics** (if enabled):
`get_context` or `query_chart` for watchlist dashboards/metrics.
Flag changes >10% vs yesterday.

**Chrome MCP sources** (if enabled and `mcp__Control_Chrome__get_page_content` available):
LinkedIn, X, Reddit, YouTube, Hacker News, Substack, Product Hunt.
Fetch via `open_url` + `get_page_content`. Today only. Filter by watchlist.

---

## Classify

- 🔴 high — needs decision, reply, or action today
- 🟡 medium — useful context, no urgency
- ⚪ low — omit

---

## Formats

### daily-morning-brief
```markdown
## Morning Brief — [date]

### 📅 Today
[time · title · participants · link]

### 🔔 Needs attention
[all 🔴 items across sources]

### 📬 Gmail
[top 5 — subject · summary · link]

### 💬 Slack
[by channel — key messages only]

### 📋 Notion
[recently updated pages]

### 🐙 GitHub
[open PRs · review requests · mentions]

### 🎨 Figma
[new comments · recent changes]

### 📊 Analytics
[key metrics · delta vs yesterday]
```
Include only enabled sources. Skip empty sections silently.

### daily-evening-digest
Same structure, shorter. Only 🔴 and 🟡. Skip what was already in the morning brief.

### weekly-pulse
```markdown
## Weekly Pulse — [Mon–Fri]

### Key highlights
### Recurring themes
### Important communications
### Next week
```
Reads daily Planner tiles Mon–Fri via `xtiles_get_planner_content`. Does not call source APIs.

### monthly-recap
```markdown
## Monthly Recap — [Month Year]

### Progress on goals
### Key insights
### Important decisions
### Month's trends
```
Reads weekly tiles via `xtiles_get_planner_content`. Does not call source APIs.

---

## Write

Call `xtiles_create_tiles_from_markdown_in_my_planner` (period: `"day"` / `"week"` / `"month"`, date: ISO 8601).

Before writing: check existing `###` headers via `xtiles_get_planner_content` — skip sections already present.
If write fails — tell the user briefly and offer to retry.

---

## After writing

Briefly confirm what was written. Let the user know they can adjust anything anytime.---
name: intelligence-hub-digest
description: >
Fetches data from connected sources (Gmail, Slack, Google Calendar, Notion, GitHub, Figma, Analytics, and others),
classifies items by priority, formats a digest, and writes it to the user's xTiles Planner.

Runs automatically via scheduled tasks. Also trigger when the user says:
"show me my morning brief", "what do I need to know today", "run my digest", "what happened today".

Config is read from the scheduled task prompt — no separate file needed.
For manual runs: look for config in today's Planner. If not found — suggest running intelligence-hub-setup first.
---

# Intelligence Hub — Digest

Runs on schedule or on demand. No questions — just fetch, classify, write.

---

## Config

Read from the scheduled task prompt body:

```json
{
  "user": { "timezone": "...", "language": "detect from conversation" },
  "sources": {
    "gmail":     { "enabled": true,  "watchlist": [], "blacklist": [] },
    "slack":     { "enabled": true,  "watchlist": ["#general"], "blacklist": [] },
    "calendar":  { "enabled": true,  "upcoming_days": 1 },
    "notion":    { "enabled": false, "watchlist": [] },
    "github":    { "enabled": false, "watchlist": [] },
    "figma":     { "enabled": false, "watchlist": [] },
    "analytics": { "enabled": false, "watchlist": [] }
  }
}
```

**Manual trigger:** call `xtiles_get_planner_content` for today and look for a config block. If not found — tell the user to run intelligence-hub-setup first.

**Preview from setup:** use sources and watchlists passed as context. Skip deduplication.

---

## Deduplication

Before fetching, call `xtiles_get_planner_content` for today. If the relevant `###` sections already exist — stop silently. Re-run only if the user explicitly asks.

---

## Fetch

Don't narrate the fetching process.

**Gmail** (if enabled):
`search_threads` with `query="is:unread is:important in:inbox newer_than:1d"` → `get_thread` for each result.
Boost watchlist senders, filter blacklist.

**Slack** (if enabled):
`slack_read_channel` for each channel in watchlist.
Today only — discard older messages.
Flag: direct mentions, open questions, decisions, deadlines.

**Google Calendar** (if enabled):
`list_events` — today + tomorrow for morning brief, today only for evening.
Extract: time, title, participants, link.

**Notion** (if enabled):
`notion-search` filtered to watchlist workspaces/pages. Today only.

**GitHub** (if enabled):
GitHub MCP tools. Fetch: open PRs assigned to user, review requests, @mentions in issues.
Filter by watchlist repos if set.

**Figma** (if enabled):
`get_metadata` for watchlist files. Flag new comments and recent changes.

**Analytics** (if enabled):
`get_context` or `query_chart` for watchlist dashboards/metrics.
Flag changes >10% vs yesterday.

**Chrome MCP sources** (if enabled and `mcp__Control_Chrome__get_page_content` available):
LinkedIn, X, Reddit, YouTube, Hacker News, Substack, Product Hunt.
Fetch via `open_url` + `get_page_content`. Today only. Filter by watchlist.

---

## Classify

- 🔴 high — needs decision, reply, or action today
- 🟡 medium — useful context, no urgency
- ⚪ low — omit

---

## Formats

### daily-morning-brief
```markdown
## Morning Brief — [date]

### 📅 Today
[time · title · participants · link]

### 🔔 Needs attention
[all 🔴 items across sources]

### 📬 Gmail
[top 5 — subject · summary · link]

### 💬 Slack
[by channel — key messages only]

### 📋 Notion
[recently updated pages]

### 🐙 GitHub
[open PRs · review requests · mentions]

### 🎨 Figma
[new comments · recent changes]

### 📊 Analytics
[key metrics · delta vs yesterday]
```
Include only enabled sources. Skip empty sections silently.

### daily-evening-digest
Same structure, shorter. Only 🔴 and 🟡. Skip what was already in the morning brief.

### weekly-pulse
```markdown
## Weekly Pulse — [Mon–Fri]

### Key highlights
### Recurring themes
### Important communications
### Next week
```
Reads daily Planner tiles Mon–Fri via `xtiles_get_planner_content`. Does not call source APIs.

### monthly-recap
```markdown
## Monthly Recap — [Month Year]

### Progress on goals
### Key insights
### Important decisions
### Month's trends
```
Reads weekly tiles via `xtiles_get_planner_content`. Does not call source APIs.

---

## Write

Call `xtiles_create_tiles_from_markdown_in_my_planner` (period: `"day"` / `"week"` / `"month"`, date: ISO 8601).

Before writing: check existing `###` headers via `xtiles_get_planner_content` — skip sections already present.
If write fails — tell the user briefly and offer to retry.

---

## After writing

Briefly confirm what was written. Let the user know they can adjust anything anytime.