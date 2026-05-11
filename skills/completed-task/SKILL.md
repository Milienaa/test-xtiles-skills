---
name: completed-task
description: Convert the current chat into a completed task in xTiles
  workspace. Use when user wants to mark chat as done, save
  chat result as completed task, turn conversation into a
  finished task, log what was done today, create completed
  task from chat, record finished work from discussion.
allowed-tools: mcp__xtiles__list-tasks, mcp__xtiles__update-task, mcp__xtiles__create-tasks
---

# xTiles Chat to a Completed Task

You convert the current chat conversation into a completed task in
the xTiles workspace with today's date as the due date.

## Your process

1. **Analyze the chat** — extract the main task or goal from the conversation
2. **Derive task title** — short verb-first title based on what was done
3. **Search in xTiles** — use mcp__xtiles__list-tasks to find tasks with similar titles
4. **If one match** — update it with mcp__xtiles__update-task (status: completed, due date: today)
5. **If multiple matches** — show the list to the user and ask which one to update
6. **If no match** — create a new task with mcp__xtiles__create-tasks, then mark it completed with mcp__xtiles__update-task
7. **Show the result** — confirm to the user what was saved

## Output format before saving

Present confirmation like this before calling MCP:

✅ Saving as completed task — [today's date]

Task: [derived title]
Description: [what was done]
Due date: [today]
Status: Done ✔️

Saving to xTiles workspace...

## If multiple matches found

Show the list like this and wait for user input:

Found multiple matching tasks — which one should be marked as completed?

1. [task title]
2. [task title]
3. [task title]

Or create a new task instead?

## Task title rules

- Start with a verb (Researched, Built, Discussed, Reviewed, Configured...)
- Keep it short — one line, under 60 characters
- Reflect the actual outcome, not just the topic

## Rules

- Always derive the task title from the chat — never ask the user to name it
- Due date is always today — never ask, never change
- Always show the task preview before saving
- Search for existing task first — create new only if nothing matches
- If multiple similar tasks found — always ask the user which one to update, never guess
- If the chat covers multiple distinct outcomes — create one task for the most significant one
- Call the tool immediately after showing the preview — no extra confirmation needed