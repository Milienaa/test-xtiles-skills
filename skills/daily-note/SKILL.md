---
name: daily-note
description: Convert the current chat into a short note in xTiles daily
  note. Use when user wants to save the chat to daily note,
  add chat summary to diary, capture today's conversation,
  record chat result, write to daily log, save discussion
  to daily note, add chat to today's note.
allowed-tools: mcp__xtiles__create-tiles-from-markdown-in-my-planner
---

# xTiles Chat to a Daily Note

You convert the current chat conversation into a short summary note
and save it to today's daily note in the xTiles workspace.

## Your process

1. **Collect the conversation** — take the current chat from context
2. **Write a summary** — 2-4 sentences capturing the key topic and outcome
3. **Add the chat link** — include the URL to the current Claude conversation
4. **Show the entry** — present what will be added to the daily note
5. **Save to xTiles** — add the entry to today's daily note using mcp__xtiles__create-tiles-from-markdown-in-my-planner

## Output format before saving

Present confirmation like this before calling MCP:

Adding to daily note — [today's date]

💬 [Summary of the chat in 2-4 sentences]
🔗 Chat link: [URL of current Claude conversation]

Saving to xTiles daily note...

## Summary writing rules

- Focus on the main topic and key outcome of the chat
- Keep it short — 2-4 sentences maximum
- Write in past tense — "Discussed...", "Decided...", "Explored..."
- Do not copy messages verbatim — summarize the essence

## Rules

- Always show the entry before saving
- Use today's date automatically — never ask the user for it
- Always include the chat link — it is required, not optional
- Call the tool immediately — no confirmation needed from the user
- If the chat has no clear topic — write a neutral summary of what was covered