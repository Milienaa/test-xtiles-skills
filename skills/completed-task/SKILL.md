---
name: completed-task
description: Convert the current chat into a completed task in xTiles. Use
  when the user wants to save or log finished work, record what was done or
  accomplished, or turn a discussion, research result, executed plan, or solved
  problem into a done task in xTiles. Trigger on phrases like "save this as a
  completed task", "log what I did", "mark this done", "add this to my completed
  tasks", "record this finished work", "track today's accomplishment".
allowed-tools: mcp__xtiles__create-tasks, mcp__xtiles__update-task, mcp__xtiles__get-current-user, mcp__xtiles__get-user-timezone, mcp__xtiles__get-planner-content, Read, Write, Edit, AskUserQuestion
---

# xTiles Chat to a Completed Task

When a conversation in Claude leads to a result — a decision, a solution,
a finished piece of work — that result deserves to be captured. This skill
takes the current chat and turns it into a completed task in xTiles.

**Invoking this skill IS the confirmation.** The user already decided to save
the chat as a completed task — create it immediately, then show the result.
Don't ask "Save it?" or wait for approval.

## Your process

1. **Get current user and timezone** — call `mcp__xtiles__get-current-user` → save `userId`; call `mcp__xtiles__get-user-timezone` → save today's local date in ISO 8601 format for use as `due_date`
2. **Analyze the chat** — extract the main outcome; derive title (verb-first, ≤60 chars) and description (2–3 sentences) in the dominant language of the conversation (match the user's input language)
3. **Create in xTiles** — call `mcp__xtiles__create-tasks`:
    - `title` — derived title
    - `description` — summary (no chat link; links in descriptions often break)
    - `due_date` — today in ISO 8601
    - `assigneeId` — `userId` from step 1 (pass automatically, never ask the user)
4. **Mark as completed** — call `mcp__xtiles__update-task` with `completed: true`
   using the `taskId` from the create response
5. **Show the result block** — call `mcp__xtiles__get-planner-content` with `date` = today (ISO 8601), `period` = `"day"` → use the returned `view_id` in the confirmation link; the block shows the saved title and description to the user
6. **Capture habit opt-in** — after the result block, run the
   "Capture habit opt-in (one-time)" procedure below. Skip silently if it is
   already enabled or was previously declined.

## Result block

✅ **[Task title]** marked as completed.

[Task description]

🔗 [Open in xTiles](https://xtiles.app/VIEW_ID)

Replace `VIEW_ID` with the `view_id` returned by `mcp__xtiles__get-planner-content`.

## Capture habit opt-in (one-time)

Run this AFTER the result block above. It concerns the *future* habit, not the
current save — so it never blocks or delays saving.

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
`~/.claude/CLAUDE.md`."

**3b. On "No, thanks"** — add the line `<!-- xTiles capture habit: declined -->`
to `~/.claude/CLAUDE.md` the same way: use `Write` to create the file if missing,
otherwise use `Edit` to append it after existing content, preceded by one blank
line. Acknowledge briefly; do not re-ask in future sessions.

**Override** — if the user later explicitly asks to enable the habit, use `Edit`
to replace the `<!-- xTiles capture habit: declined -->` line with the habit
block from 3a, leaving all other content untouched.

## If update fails

Show the confirmation block as usual, then add:
> Warning: Could not mark as completed automatically. Task ID: [taskId]

## Rules

- Derive title, description, and language from the chat — never ask the user
- Assign to the current user automatically — never ask the user
- Due date is always today — never ask, never change
- Never ask for confirmation; create immediately
- The capture-habit opt-in runs only AFTER the task is saved — it asks about the
  future habit, not the current task, so it does not violate "create immediately"
- Always mark as completed immediately after creating
- Result block requires: title, description, link
