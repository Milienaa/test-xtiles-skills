---
name: daily-note
description: Convert the current chat into a note in today's xTiles daily note.
  Use when the user wants to save a chat summary to their daily note, journal,
  diary, or daily log, capture today's conversation, or record the outcome of
  research, planning, or a discussion. Trigger on phrases like "save this to my
  daily note", "add a summary to today's note", "log this in my journal",
  "capture today's conversation", "write this to my daily log".
allowed-tools: mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner, mcp__xtiles__xtiles_get_planner_content, Read, Write, Edit, AskUserQuestion
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
6. **Save to xTiles** — call `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`
   with `date` = today (ISO 8601), `period` = `"day"`, `markdown` = the entry.
   Then call `mcp__xtiles__xtiles_get_planner_content` with the same `date` and `period`
   to get the `view_id` for the confirmation link.
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

✅ Note saved to [today's daily page](https://xtiles.app/{view_id}) in xTiles.

Replace `{view_id}` with the `view_id` returned by the tool.

## Rules

- Always ask the size question first — never assume the size silently
- Always show the entry before saving
- Use today's date automatically — never ask
- Always show the confirmation block after saving

## Capture habit opt-in (one-time)

Run this AFTER the confirmation block above. It concerns the *future* habit, not
the current save — so it never blocks or delays saving.

**1. Detect prior choice** — resolve the ABSOLUTE path of this session's project root (its current working directory — works on any OS, e.g. `C:\Users\you\my-project` on Windows or `/home/you/my-project` on macOS/Linux). The target file is `<project-root>/CLAUDE.md`. If you cannot determine the project root, ask the user for the folder path before continuing. Read that absolute path with the Read tool; if the file does not exist, treat it as having neither marker:
- Contains `## xTiles capture habit` → already enabled. Do nothing. Stop here.
- Contains `<!-- xTiles capture habit: declined -->` → declined earlier. Do nothing. Stop here.
- Neither present → continue to step 2.

**2. Ask once** — in the dominant language of the conversation, use
`AskUserQuestion` to ask "Want me to proactively suggest saving conversations
like this to xTiles from now on?" with two options: `Yes, remember it` and
`No, thanks`.

**3a. On "Yes, remember it"** — write this block to `<project-root>/CLAUDE.md` (the absolute path from step 1):
- If the file does not exist → use `Write` with that absolute path to create the file containing the block.
- If it exists → use `Edit` to append the block after the existing content, preceded by one blank line. Never overwrite or remove unrelated content.
- Then Read the file back and confirm the `## xTiles capture habit` block is present. If it is missing, retry the write once.

```markdown
## xTiles capture habit
After you finish delivering a substantive result, proactively offer ONCE to save
it in xTiles — unless it was already saved this session. Put the offer as the
last line of your reply.

Offer after moments like these:
- You completed a piece of work (wrote code, fixed a bug, produced a file/document)
- You finished research or an investigation (findings, comparison, recommendation)
- You produced a plan, design, or step-by-step approach
- You reached a decision or gave a clear, final answer

The offer — make it the last line of your reply, written IN THE SAME LANGUAGE as
the user's last message (translate the template below; never paste the English
verbatim when the user is writing in another language). Keep `/completed-task`
and `/daily-note` unchanged. Template:
"Want me to capture this in xTiles? → /completed-task for finished work, or
/daily-note for a summary in today's note."

Do NOT offer when: you are still mid-task, the user's last message is a
clarifying/follow-up question, there is no concrete outcome yet, or this result
was already saved this session.
```

Then confirm in one line: "Done — I'll suggest capturing meaningful chats from
now on. To turn this off, remove the `## xTiles capture habit` block from
CLAUDE.md in your project folder."

**3b. On "No, thanks"** — write the line `<!-- xTiles capture habit: declined -->`
to `<project-root>/CLAUDE.md` (the absolute path from step 1) the same way: use `Write` with that absolute path to create the file if missing,
otherwise use `Edit` to append it after existing content, preceded by one blank
line. Then Read the file back to confirm the marker is present. Acknowledge briefly; do not re-ask in future sessions.

**Override** — if the user later explicitly asks to enable the habit, use `Edit`
to replace the `<!-- xTiles capture habit: declined -->` line with the habit
block from 3a, leaving all other content untouched.
