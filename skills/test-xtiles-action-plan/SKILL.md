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

## Emoji selection

Pick one emoji per task based on its content. Examples:
- 🔧 fix, repair, debug, technical issue
- 🧪 test, QA, check, verify
- 📱 mobile, device, screen, UI
- 📝 write, document, content, copy
- 📋 review, audit, plan, organize
- 🚀 launch, deploy, release, ship
- 🎨 design, visual, layout, style
- 📊 analytics, data, metrics, report
- 💬 communication, email, message, notify
- 🔍 research, investigate, find, explore

If nothing fits — use 📌 as a safe default.

## Show the plan before saving

Before calling MCP, present the plan in the user's language and wait for confirmation.

Group tasks by week using this format (shown as a code block to avoid Markdown conflicts):

```
📋 N завдань готові до збереження   ← header in user's language, N = task count

── Цей тиждень ────────────────────   ← or "Наступний тиждень", "Через два тижні" etc.

[emoji] [Title]
   [Short description — 1 sentence max]
   └─ [weekday] [dd/mm]

[emoji] [Title]
   [Short description — 1 sentence max]
   └─ [weekday] [dd/mm]

── Наступний тиждень ──────────────

[emoji] [Title]
   ...

Зберігаємо? Або щось змінити?   ← closing line in user's language
```

### Display rules

- **Language**: write all labels (header, week groups, closing line) in the user's language
- **Week groups**: show only groups that have tasks; skip empty ones
- **Week label logic**:
   - same calendar week as today → "Цей тиждень" / "This week" / etc.
   - next calendar week → "Наступний тиждень" / "Next week" / etc.
   - further out → use the week's Monday date: "Тиждень з 26/05" / "Week of May 26"
- **Description**: compress to 1 sentence if the original is longer
- **Weekday**: always show short weekday before the date (пн, вт, ср, чт, пт / Mon, Tue...)
- **No task numbers** — never use "Task 1:", "Завдання 1:" or any numbered prefix

## Rules

- Always show the plan first and wait for confirmation — never save without review
- Tasks must be actionable — always start with a verb
- Deadlines should be realistic — space tasks 1-3 days apart by default
- If input is vague — ask one clarifying question before planning
- Maximum 7 tasks per plan — if more needed, suggest splitting into phases