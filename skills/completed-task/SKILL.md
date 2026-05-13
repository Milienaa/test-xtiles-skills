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
takes the current chat and turns it into a completed task in xTiles, so
the work done is recorded and visible in the workspace.

## What this skill does

Reads the current chat, understands what was accomplished, and creates
a completed task in xTiles with today's date. No manual input needed —
the task title and description are derived automatically from the conversation.

## The result the user gets

A completed task in xTiles that reflects what was done in this chat:
- A clear title that describes the work
- A short description of what was accomplished
- Due date set to today
- Status set to completed

## Your process

1. **Analyze the chat** — extract the main goal or outcome from the conversation
2. **Derive task title** — short verb-first title based on what was done
3. **Write a description** — 1-2 sentences summarizing what was accomplished
4. **Create in xTiles** — call mcp__xtiles__create-tasks with due date today
5. **Mark as completed** — call mcp__xtiles__update-task with status completed
6. **Confirm to the user** — show what was saved

## If update fails

If mcp__xtiles__create-tasks succeeds but mcp__xtiles__update-task fails:
- Notify the user that the task was created but could not be marked as completed
- Provide the task ID so the user can update it manually
- Do not retry automatically

## Rules

- Always derive title and description from the chat — never ask the user
- Due date is always today — never ask, never change
- Always mark as completed immediately after creating — never leave as open
- If update fails — report it with the task ID, do not silently ignore
- If the chat covers multiple outcomes — create one task for the most significant one
- Keep the title under 60 characters
- Do one thing only — create and complete the task, nothing else