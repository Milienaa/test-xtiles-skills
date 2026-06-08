---
name: completed-task
description: Convert the current chat into a completed task in xTiles
  workspace. Use when user wants to save chat as completed task,
  log what was done today, create completed task from chat,
  record finished work from discussion, turn conversation into
  a finished task, mark chat as done in xTiles.
allowed-tools: mcp__xtiles__create-tasks, mcp__xtiles__update-task, mcp__xtiles__get-current-user, mcp__xtiles__get-user-timezone
---

# xTiles Chat to a Completed Task

When a conversation in Claude leads to a result — a decision, a solution,
a finished piece of work — that result deserves to be captured. This skill
takes the current chat and turns it into a completed task in xTiles.

## Your process

1. **Get current user and timezone** — call `mcp__xtiles__get-current-user` → save `userId`; call `mcp__xtiles__get-user-timezone` → save today's local date in ISO 8601 format for use as `due_date`
2. **Analyze the chat** — extract the main outcome; derive title (verb-first, ≤60 chars) and description (2–3 sentences) in the dominant language of the conversation (match the user's input language)
3. **Show preview** — present the title and description before saving (see Preview block below); wait for user confirmation
4. **Create in xTiles** — call `mcp__xtiles__create-tasks`:
    - `title` — derived title
    - `description` — summary (no chat link; links in descriptions often break)
    - `due_date` — today in ISO 8601
    - `assigneeId` — `userId` from step 1 (pass automatically, never ask the user)
5. **Mark as completed** — call `mcp__xtiles__update-task` with `completed: true`
   using the `taskId` from the create response
6. **Show the confirmation block**

## Preview block

```
Here's what will be saved:

**[Task title]**
[Task description]

Save it?
```

Claude waits for explicit confirmation before proceeding to step 4.

## Confirmation block

✅ **[Task title]** marked as completed.
[Task description]

🔗 [Open in xTiles](https://xtiles.app/VIEW_ID)

Replace `VIEW_ID` with the `view_id` returned by the tool.

## If update fails

Show the confirmation block as usual, then add:
> Warning: Could not mark as completed automatically. Task ID: [taskId]

## Rules

- Derive title, description, and language from the chat — never ask the user
- Assign to the current user automatically — never ask the user
- Due date is always today — never ask, never change
- Always show preview before creating; wait for confirmation
- Always mark as completed immediately after creating
- Confirmation block requires: title, description, link