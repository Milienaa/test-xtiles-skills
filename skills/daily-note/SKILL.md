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

Every meaningful conversation deserves to be captured. This skill takes
the current chat, distills it into a short summary, and saves it to
today's daily note in xTiles — so the context and outcome of the
discussion are always easy to find later.

## What this skill does

Reads the current chat, writes a brief summary of what was discussed
and decided, and adds it to today's daily note in xTiles together with
a link back to the full conversation. No manual input needed.

## The result the user gets

A concise note in today's xTiles daily page that contains:
- A short summary of what the chat was about and what was achieved
- A direct link to the full Claude conversation for reference

## Your process

1. **Collect the conversation** — take the current chat from context
2. **Write a summary** — 2-4 sentences capturing the key topic and outcome
3. **Add the chat link** — include the URL to the current Claude conversation
4. **Show the entry** — present what will be added to the daily note
5. **Save to xTiles** — add the entry to today's daily note using mcp__xtiles__create-tiles-from-markdown-in-my-planner
6. **Confirm to the user** — show what was saved

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