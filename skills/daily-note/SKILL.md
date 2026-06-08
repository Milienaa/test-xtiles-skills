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
   `https://xtiles.app/{view_id}`
7. **Show the confirmation block**.
## Questions to ask

Detect the dominant language of the current conversation. Ask all three questions in that language using `AskUserQuestion` in a single call before writing anything.

- **Depth** — "How detailed should the note be?" → options: `Brief` (2–4 sentences, main topic and outcome) / `Detailed` (full breakdown: topics, decisions, next steps)
- **Format** — "How should the note be structured?" → options: `Narrative` (flowing text, one or two paragraphs) / `Bulleted list` (3–6 key points, quick to scan) / `Structured` (three sections: Topic · Outcome · Next steps)
- **Language** — "What language for the note?" → options: `Auto` (matches the conversation language) / `English` (always write in English)

If the user already specified any of these in their request, skip that question.
If they answer partially, use defaults for the rest: `Brief`, `Narrative`, `Auto`.

## Summary writing rules

**Brief:** 2–4 sentences covering the main topic and key outcome.
Write in past tense: "Discussed…", "Decided…", "Explored…"
Do not copy messages verbatim — summarize the essence.

**Detailed:** Full breakdown including:
- What was discussed
- Decisions made
- Open questions or next steps
  **Narrative:** Flowing sentences, no bullet points.
  **Bulleted list:** 3–6 bullet points covering the key moments.
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
 