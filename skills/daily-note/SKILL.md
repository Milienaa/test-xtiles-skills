---
name: daily-note
description: Convert the current chat into a short note in xTiles daily
  note. Use when user wants to save the chat to daily note,
  add chat summary to diary, capture today's conversation,
  record chat result, write to daily log, save discussion
  to daily note, add chat to today's note.
allowed-tools: mcp__xtiles__create-tiles-from-markdown-in-my-planner, Read, Write, Edit, AskUserQuestion
---

# xTiles Chat to a Daily Note

Every meaningful conversation deserves to be captured. This skill takes
the current chat, distills it into a summary, and saves it to today's
daily note in xTiles — so the context and outcome of the discussion are
always easy to find later.

## Your process

1. **Ask the user how detailed the note should be** — ask the single size
   question (see Questions to ask below). Wait for the answer before writing.
2. **Collect the conversation** — take the current chat from context.
3. **Write a summary** — follow the Summary writing rules below.
4. **Build the entry** — assemble markdown using the Entry format below.
5. **Show the entry** — present what will be saved before calling the tool.
6. **Save to xTiles** — call `mcp__xtiles__create_tiles-from_markdown-in-my_planner`
   with `date` = today (ISO 8601), `period` = `"day"`, `markdown` = the entry.
   The tool returns `view_id` — use it to build the tile URL:
   `https://xtiles.app/{view_id}`
7. **Show the confirmation block**.
8. **Capture habit opt-in** — after the confirmation block, run the
   "Capture habit opt-in (one-time)" procedure below. Skip silently if it is
   already enabled or was previously declined.

## Questions to ask

Detect the dominant language of the current conversation. Ask one question in that language using `AskUserQuestion` before writing anything.

- **Size** — "How detailed should the note be?" → options: `Short` (a summary of the dialogue and the key decisions) / `Detailed` (structured, with more detail from the dialogue)

If the user already specified the size in their request, skip the question and use it. If unspecified and unanswered, default to `Short`. The note is always written in the dominant language of the conversation.

## Summary writing rules

**Short:** A concise summary of the dialogue and the key decisions.
Write in past tense: "Discussed…", "Decided…", "Explored…"
Do not copy messages verbatim — summarize the essence.

**Detailed:** A structured breakdown with more detail from the dialogue, using
three sections — **Topic** / **Outcome** / **Next steps** — covering what was
discussed, the decisions made, and any open questions or next steps.

**Language:** Always write in the dominant language of the conversation.

## Entry format

```
### [Short title derived from chat topic] — [Today's date, e.g. May 20, 2026]
 
[Summary — per the size chosen by the user, in the conversation's language]
 
```

## Confirmation block

✅ Note saved to [today's daily page](https://xtiles.app/VIEW_ID) in xTiles.

Replace `VIEW_ID` with the `view_id` returned by the tool.

## Rules

- Always ask the size question first — never assume the size silently
- Always show the entry before saving
- Use today's date automatically — never ask
- Always include the chat link in the entry — required, not optional
- Always show the confirmation block after saving

## Capture habit opt-in (one-time)

Run this AFTER the confirmation block above. It concerns the *future* habit, not
the current save — so it never blocks or delays saving.

**1. Detect prior choice** — read `~/.claude/CLAUDE.md` (if the file does not
exist, treat it as having neither marker):
- Contains `## xTiles capture habit` → already enabled. Do nothing. Stop here.
- Contains `<!-- xTiles capture habit: declined -->` → declined earlier. Do nothing. Stop here.
- Neither present → continue to step 2.

**2. Ask once** — in the dominant language of the conversation, use
`AskUserQuestion` to ask "Want me to proactively suggest saving conversations
like this to xTiles from now on?" with two options: `Yes, remember it` and
`No, thanks`.

**3a. On "Yes, remember it"** — add this block to `~/.claude/CLAUDE.md`:
- If the file does not exist → use `Write` to create it containing the block.
- If it exists → use `Edit` to append the block after the existing content,
  preceded by one blank line. Never overwrite or remove unrelated content.

```markdown
## xTiles capture habit
When a conversation reaches a concrete result (a decision, finished work, or a
clear answer) AND it has not already been saved this session, proactively offer
ONCE: "Want me to capture this? → completed task (/completed-task) or today's
daily note (/daily-note)?"

Do NOT offer when: the user is mid-task, the message is a clarifying/follow-up
question, there is no concrete outcome yet, or the chat was already saved.
```

Then confirm in one line: "Done — I'll suggest capturing meaningful chats from
now on. To turn this off, remove the `## xTiles capture habit` block from
`~/.claude/CLAUDE.md`."

**3b. On "No, thanks"** — add the line `<!-- xTiles capture habit: declined -->`
to `~/.claude/CLAUDE.md` the same way: use `Write` to create the file if missing,
otherwise use `Edit` to append it after existing content, preceded by one blank
line. Acknowledge briefly; do not re-ask in future sessions.

**Override** — if the user later explicitly asks to enable the habit, use `Edit`
to replace the `<!-- xTiles capture habit: declined -->` line with the habit
block from 3a, leaving all other content untouched.
