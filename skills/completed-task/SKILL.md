---
name: completed-task
description: Convert the current chat into a completed task in xTiles
  workspace. Use when user wants to save chat as completed task,
  log what was done today, create completed task from chat,
  record finished work from discussion, turn conversation into
  a finished task, mark chat as done in xTiles.
allowed-tools: mcp__xtiles__create-tasks, mcp__xtiles__update-task
---

# xTiles Chat to a Completed Task

When a conversation in Claude leads to a result — a decision, a solution,
a finished piece of work — that result deserves to be captured. This skill
takes the current chat and turns it into a completed task in xTiles.

## Your process

1. **Analyze the chat** — extract the main outcome or work done
2. **Derive task title** — verb-first, 60 characters maximum
3. **Write a description** — 2–3 sentences: what was done and the key outcome
4. **Create in xTiles** — call `mcp__xtiles__create-tasks`:
    - `title` — derived title
    - `description` — summary (no chat link; links in descriptions often break)
    - `due_date` — today in ISO 8601
5. **Mark as completed** — call `mcp__xtiles__update-task` with `completed: true`
   using the `taskId` from the create response
6. **Show the confirmation block**

## Confirmation block

---
✅ **[Task title]**
📅 [Today's date, human-readable — e.g. "May 20, 2026"]

[1–2 sentences describing what was done and the key outcome,
written in the same language as the chat]

🔗 [Open in xTiles](https://app.xtiles.app/my/tasks)
---

Always use `https://app.xtiles.app/my/tasks` as the link — this is the
correct stable URL for personal tasks. Never construct or guess other URLs.

## If update fails

Show the confirmation block as usual, then add:
> Warning: Could not mark as completed automatically. Task ID: [taskId]

## Rules

- Derive everything from the chat — never ask the user
- Due date is always today — never ask, never change
- Always mark as completed immediately after creating
- Confirmation block requires all four elements: title, date, description, link