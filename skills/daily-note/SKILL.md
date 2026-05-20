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
6. **Save to xTiles** — call `mcp__xtiles__create_tiles_from_markdown_in_my_planner`
   with `date` = today (ISO 8601), `period` = `"day"`, `markdown` = the entry.
7. **Show the confirmation block**.
## Questions to ask

Ask these three questions in a single message before writing anything:

> How would you like the note?
>
> 1. **Depth** — brief (2–4 sentences) or detailed (full breakdown)?
> 2. **Format** — prose, bullet points, or structured (Topic / Outcome / Next steps)?
> 3. **Language** — same as the chat, or a specific language?

If the user already specified any of these in their request, skip that
question and apply what they said. If they answer partially (e.g. only
depth), use defaults for the rest: `brief`, `prose`, chat language.

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
 
🔗 [Full conversation](https://claude.ai/chat/CONVERSATION_ID)
```

Extract the conversation ID from the current context to build the URL.
If unavailable, use `https://claude.ai` as a fallback — never omit the link.

## Confirmation block
 
---
✅ Note saved to today's daily page in xTiles.
🔗 [Open xTiles daily note](https://app.xtiles.app/my/planner/day)
---

## Rules

- Always ask the three questions first — never assume settings silently
- Always show the entry before saving
- Use today's date automatically — never ask
- Always include the chat link in the entry — required, not optional
- Always show the confirmation block after saving
 