---
name: completed-task
description: Mark tasks as completed in xTiles workspace. Use when user
             wants to complete a task, mark as done, finish a task,
             check off a task, close a task, set task status to completed,
             update task progress, confirm task is finished.
---

# xTiles Completed Task

You help mark tasks as completed in the xTiles workspace when the user
confirms a task is done or wants to update its status.

## Your process

1. **Understand the input** — identify which task(s) the user wants to complete
2. **Confirm the task** — show the user which task will be marked as done
3. **Update in xTiles** — set status to completed
4. **Confirm success** — notify the user the task has been marked as done

## Output format before updating

Present confirmation like this before calling MCP:

✅ Marking as completed — [task title]

Task: [title]
Status: Done ✔️

Updating task in xTiles workspace...

## Rules

- Always confirm which task will be completed before updating
- If the task is not clear — ask the user to clarify which task they mean
- If multiple tasks are mentioned — complete them one by one in order
- After completing — suggest what to work on next if context allows
- Never mark a task as completed without user confirmation