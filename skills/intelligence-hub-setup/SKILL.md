---
name: intelligence-hub-setup
description: >
  One-time setup wizard for Intelligence Hub — a personal awareness system built on xTiles Planner.
  Detects connected sources (Gmail, Slack, Calendar, Notion, GitHub, Figma, Analytics, and Chrome-based sources),
  personalizes filters per source, fetches a live preview digest, writes it to xTiles Planner, and creates scheduled tasks.

  Trigger when user says: "set up my morning digest", "connect my sources to xTiles", "I want a daily brief",
  "automate my digest", "set up Intelligence Hub", or any first-time setup intent.

  Do NOT trigger for changing existing settings (use intelligence-hub-tune) or running a digest (use intelligence-hub-digest).
allowed-tools: mcp__xtiles__xtiles_get_planner_content, mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner
---

# Intelligence Hub — Setup

One-time wizard. Keep the tone conversational but precise — brief updates between steps, name what you found, explain what you're about to do. Don't jump silently between steps.

---

## Step 1 — Check for Planner

Call `xtiles_get_planner_content` silently.

- **Found** → "Your Planner is ready — let's go." Continue.
- **Not found** → "You'll need an xTiles Planner for this — setting one up now." Run `planner-setup`. When done: "Planner created — continuing." Continue.

---

## Step 2 — Detect connected sources

Probe all available tools silently. Two extra checks before showing results:

- **Analytics:** call `get_context` on analytics MCP. Use the actual platform name (PostHog, Amplitude, etc). Skip if not connected.
- **Slack channels:** if Slack is connected, ask the user which channels to monitor — don't try to auto-detect. `slack_read_channel` requires a known channel name, so let the user name them first.

Sources to probe:

| Source | Tools |
|--------|-------|
| Gmail | `search_threads`, `get_thread` |
| Slack | `slack_read_channel` |
| Google Calendar | `list_events`, `create_event` |
| Notion | `notion-search`, `notion-fetch` |
| GitHub | any GitHub MCP tool |
| Figma | `get_design_context`, `get_metadata` |
| Analytics | `get_context`, `query_chart` |

If `mcp__Control_Chrome__get_page_content` is available — also offer: LinkedIn, X, YouTube, Reddit, Hacker News, Substack, Product Hunt.

Tell the user which sources are connected and which aren't — offer to add missing ones or skip. Show connected sources pre-checked, unavailable ones grayed out.

If nothing found — show popular list and call `suggest_connectors`. After connecting, re-probe and confirm what was added.

---

## Step 3 — Personalize each source

Go through selected sources one at a time. For each — briefly explain why you're asking before the question. Keep it one sentence. Don't script the exact wording — say it naturally based on what the user has selected.

**Slack:** Ask which channels to monitor. AskUserQuestion multiselect — let the user name them. If user says "all" — note that you'll need specific channel names to read from Slack, ask to list the most important ones.

**Gmail:** Ask if there are priority senders or labels to focus on. If "no" / "all" — pull everything marked important, skip filter.

**Calendar:** Skip — all events are relevant by default.

**Notion:** Ask if there are specific workspaces or pages that matter most. If "everything" — skip.

**GitHub:** Ask whether to focus on PRs and mentions only, or include specific repos and issues. If "everything" — skip.

**Figma:** Ask whether to watch specific files/projects or everything active. If "everything" — skip.

**Analytics:** Ask which dashboards or metrics to surface daily. If "everything" — skip.

**Chrome MCP sources:** For each enabled source ask what to follow — accounts, hashtags, subreddits, channels. Be specific to the platform.

When all sources are done, briefly confirm what's been configured and move on.

---

## Step 4 — Preview

Call `intelligence-hub-digest` with selected sources and watchlists as context. While it runs, let the user know you're fetching.

Show the result in chat.

---

## Step 5 — Tuning

After showing the preview, ask if anything should change. Accept freeform input:

- "remove Slack" → disable, update config
- "only show urgent items" → update priority filter
- "more from Gmail" → adjust query
- "add GitHub" → re-probe, add to config
- "looks good" / "all good" → move on

For each change: update config, re-call `intelligence-hub-digest`, show updated preview, note what changed. Repeat until the user confirms.

---

## Step 6 — Rhythm

After the user is happy with the preview, ask how often they want the digest to run.

AskUserQuestion, single select:
- Morning only — daily at 08:00
- Morning + evening — 08:00 and 17:00 daily
- Weekly only — Friday at 18:00

Also ask separately: "Want a weekly summary every Friday at 18:00? It rolls up the week's highlights without re-fetching sources."

---

## Step 7 — Write to xTiles

Call `xtiles_get_planner_content`, check existing `###` headers — skip sections already present.

Call `xtiles_create_tiles_from_markdown_in_my_planner` (period: `"day"`, date: ISO 8601).

Confirm briefly when done.

---

## Step 8 — Scheduled task setup

Scheduled tasks can't be created programmatically — the user sets them up once in Cowork.

Generate a ready-to-paste prompt based on the user's config and rhythm choice, then tell the user:

> Go to Cowork → Scheduled → New Task, paste this prompt, and set the frequency.

The prompt to generate:

```
/intelligence-hub-digest

CONFIG:
- timezone: [detected from context or ask]
- language: [detected from conversation]
- sources:
    gmail:    enabled: [true/false] | watchlist: [...] | blacklist: [...]
    slack:    enabled: [true/false] | watchlist: [...] | blacklist: [...]
    calendar: enabled: [true/false] | upcoming_days: 1
    notion:   enabled: [true/false] | watchlist: [...]
    github:   enabled: [true/false] | watchlist: [...]
    figma:    enabled: [true/false] | watchlist: [...]
    analytics:enabled: [true/false] | watchlist: [...]
```

If user selected morning + evening — generate two separate prompts.
If user selected weekly — generate one prompt for weekly-pulse.

After showing the prompt, summarize what was set up — what runs when, which sources are active — and let the user know they can adjust anytime with `intelligence-hub-tune`.
