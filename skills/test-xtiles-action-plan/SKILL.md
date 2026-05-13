---
name: test-xtiles-action-plan
description: Break down any message, idea, or information into actionable
   tasks and save them to xTiles workspace. Use when user wants
   to create tasks, break down a project, make an action plan,
   split into steps, create to-do list, organize work, set
   deadlines, plan execution, turn idea into tasks.
allowed-tools: mcp__xtiles__create-task
---

# xTiles Action Plan

A good idea without a plan stays an idea. This skill takes any input —
a message, a goal, a project description, or meeting notes — and breaks
it down into clear actionable tasks saved directly to the xTiles workspace,
so the work can start immediately.

## What this skill does

Analyzes the input, identifies concrete next steps, enriches each with
a description and a realistic deadline, shows the plan for review, and
saves all tasks to xTiles in order.

## The result the user gets

A ready action plan in xTiles with 3-7 tasks, each containing:
- A clear title that starts with a verb
- A short description of what exactly needs to be done
- A realistic deadline based on logical sequence and complexity

## Your process

1. **Analyze the input** — understand the goal and scope
2. **Break into tasks** — identify 3-7 concrete actionable steps
3. **Enrich each task** — add description and suggested deadline
4. **Show the plan** — present tasks to the user for review before saving
5. **Save to xTiles** — call mcp__xtiles__create-task for each task in order,
   wait for each to complete before creating the next

## Task structure

For each task define:
- **Title** — short, starts with a verb (Create, Write, Review, Set up...)
- **Description** — 1-2 sentences what exactly needs to be done
- **Deadline** — suggested date based on logical sequence and complexity

## Show the plan before saving

Before calling MCP present the plan like this and wait for user confirmation:

Action Plan — [goal title]

Task 1: [Title]
Description: [what to do]
Deadline: [date]

Task 2: [Title]
Description: [what to do]
Deadline: [date]

Saving [N] tasks to xTiles workspace...

## Rules

- Always show the plan first and wait for confirmation — never save without review
- Tasks must be actionable — always start with a verb
- Deadlines should be realistic — space tasks 1-3 days apart by default
- If input is vague — ask one clarifying question before planning
- Maximum 7 tasks per plan — if more needed, suggest splitting into phases
