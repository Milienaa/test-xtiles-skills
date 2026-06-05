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
the current chat, distills it into a summary, and saves it to today's
daily note in xTiles — so the context and outcome of the discussion are
always easy to find later.

## Your process

1. **Ask the user how they want the note** — ask all three questions in one
   message (see Questions to ask below). Wait for the answer before writing.
2. **Collect the conversation** — take the current chat from context.
3. **Write a summary** — follow the Summary writing rules below.
4. **Build the entry** — assemble markdown using the Entry format below.
5. **Show the entry** — present what will be saved before calling the tool.
6. **Save to xTiles** — call `mcp__xtiles__create_tiles-from_markdown-in-my_planner`
   with `date` = today (ISO 8601), `period` = `"day"`, `markdown` = the entry.
   The tool returns `view_id` — use it to build the tile URL:
   `https://app.xtiles.app/{view_id}`
7. **Show the confirmation block**.
## Questions to ask

Use `AskUserQuestion` with all three questions in a single call before writing anything:

- **Depth** — "How detailed should the note be?" → options: `Brief` (2–4 sentences) / `Detailed` (full breakdown)
- **Format** — "What format do you prefer?" → options: `Prose` / `Bullet points` / `Structured (Topic · Outcome · Next steps)`
- **Language** — "Which language for the note?" → options: `Same as chat` / `English` / `Ukrainian`

If the user already specified any of these in their request, skip that question.
If they answer partially, use defaults for the rest: `Brief`, `Prose`, `Same as chat`.

## Summary writing rules

**Brief:** 2–4 sentences covering the main topic and key outcome.
Write in past tense: "Discussed…", "Decided…", "Explored…"
Do not copy messages verbatim — summarize the essence.

**Detailed:** Full breakdown including:
- What was discussed
- Decisions made
- Open questions or next steps
  **Prose:** Flowing sentences, no bullet points.
  **Bullets:** 3–6 bullet points covering the key moments.
  **Structured:** Three sections — **Topic** / **Outcome** / **Next steps**

**Language:** Write in the language the user specified. If not specified,
match the dominant language of the chat.

## Entry format

```
### [Short title derived from chat topic] — [Today's date, e.g. May 20, 2026]
 
[Summary — per depth, language, and format chosen by the user]
 
🔗 [Full conversation](https://claude.ai/chat/local_{conversation_id})
```

Look for the conversation ID in the current URL or system context.
Build the URL as: https://claude.ai/chat/local_{conversation_id}

## Confirmation block

✅ Note saved to [today's daily page](https://xtiles.app/VIEW_ID) in xTiles.

Replace `VIEW_ID` with the `view_id` returned by the tool.

## Rules

- Always ask the three questions first — never assume settings silently
- Always show the entry before saving
- Use today's date automatically — never ask
- Always include the chat link in the entry — required, not optional
- Always show the confirmation block after saving
 