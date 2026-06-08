---
name: action-plan-v2
description: Break down any message, idea, or information into actionable
   tasks and save them to xTiles workspace with a tile overview. Use when
   user wants to create tasks, break down a project, make an action plan,
   split into steps, create to-do list, organize work, set deadlines,
   plan execution, turn idea into tasks.
allowed-tools: mcp__xtiles__get-current-user, mcp__xtiles__get-user-timezone, mcp__xtiles__search-projects, mcp__xtiles__list-projects, mcp__xtiles__get-project-content, mcp__xtiles__create-tasks, mcp__xtiles__create-tiles-from-markdown-in-my-planner, mcp__xtiles__create-tiles-from-markdown-in-project-planner, mcp__xtiles__create-tiles-from-markdown-by-view
---

# xTiles Action Plan v2

A good idea without a plan stays an idea. This skill takes any input —
a message, a goal, a project description, or meeting notes — breaks it
down into formal xTiles tasks, and saves a tile overview to the daily
page of your chosen destination.

## Your process

1. **Setup** — call `mcp__xtiles__get-current-user` → save the `id`
   field from the response as `userId` (string); call
   `mcp__xtiles__get-user-timezone` → save today's local date as
   `yyyy-MM-dd` (e.g. `2026-06-08`) for `due_date` baseline.

2. **Find destination options** — call `mcp__xtiles__search-projects`
   with 2–3 keywords from the user's input:
   - If results found → take top 3 matching projects.
   - If 0 results → call `mcp__xtiles__list-projects` → pick 3 most
     plausibly related by name.
   - If still 0 → offer only "My Planner".

3. **Ask destination** — use `AskUserQuestion` (single select):
   ```
   question: "Where would you like to save this plan?"
   options:
     - My Planner
     - [Project name 1]
     - [Project name 2]
     - [Project name 3]
   ```
   Wait for answer before proceeding.

4. **Analyze input** — extract 3–7 concrete actionable tasks:
   - Title: verb-first, ≤60 characters
   - Description: 1–2 sentences (omit if obvious from title)
   - Deadline: extract from context if stated; propose a logical date
     if not stated — space tasks 2–3 days apart starting from today,
     assigning earlier dates to simpler or prerequisite tasks. Mark
     all proposed deadlines so they can be confirmed in step 5.

5. **Show plan for review** — use `AskUserQuestion` (multiSelect):
   ```
   question: "Which tasks would you like to save to xTiles?"
   multiSelect: true
   options:
     - { label: "[Task title]", description: "[desc] · due: Jun 12" }
     - { label: "[Task title]", description: "[desc] · due: Jun 15 (proposed)" }
     - { label: "[Task title]", description: "[desc] · no date" }
     - { label: "All tasks", description: "Save the entire plan to xTiles" }
   ```
   - Extracted deadlines: shown as `due: DATE`
   - Proposed deadlines: shown as `due: DATE (proposed)`
   - Selecting a task = confirming its deadline.
   - "All tasks" → save everything regardless of other selections.
   - If user replies via "Other" with deadline changes (e.g. "change
     Task 2 to Jun 20") → apply before saving, then proceed.

6. **Create tasks** — call `mcp__xtiles__create-tasks` once with all
   selected tasks in the `tasks` array:
   ```
   tasks: [
     {
       title:       "<task title>",
       description: "<task description>",   // omit if empty
       due_date:    "<yyyy-MM-dd>",          // omit if no date
       assignees:   [{ "id": "<userId>" }]
     },
     ...
   ]
   ```

7. **Create tile** — compose the `markdown` from the confirmed task
   list (see Tile format below), then save to the chosen destination:

   **My Planner:**
   → `mcp__xtiles__create-tiles-from-markdown-in-my-planner`
     `date` = today as `yyyy-MM-dd`, `period` = `"day"`,
     `markdown` = tile markdown

   **Project (happy path):**
   → `mcp__xtiles__create-tiles-from-markdown-in-project-planner`
     `projectId` = project id, `date` = today as `yyyy-MM-dd`,
     `period` = `"day"`, `markdown` = tile markdown

   **Project (fallback — any 400/500 or "Invalid group" error):**
   → `mcp__xtiles__get-project-content`(projectId) → take `viewId`
     of the first available page
   → `mcp__xtiles__create-tiles-from-markdown-by-view`
     `viewId` = viewId from above, `markdown` = tile markdown
   Do not surface the error to the user — silently fall back.

8. **Show confirmation:**
   ```
   ✅ Action plan saved.

   🔗 [Open in xTiles](https://xtiles.app/VIEW_ID)
   ```
   Replace `VIEW_ID` with the `view_id` returned by the tile creation
   tool called in step 7 (all three paths return a `view_id`).

## Tile format

Build the `markdown` string dynamically from the confirmed task list.
The `### heading` becomes the tile name; content beneath it becomes
the body.

```
### Action Plan

- <Title of confirmed task 1> 
- <Title of confirmed task 2>
- <Title of confirmed task 3>

🔗 [View conversation](https://claude.ai/chat/local_<conversation_id>)
```

- List every confirmed task as a `- ` bullet in the same order as step 5.
- Chat link is always the last line — required, never omit. Replace
  `<conversation_id>` with the UUID from the current session URL
  (last path segment of `https://claude.ai/chat/...`).

## Rules

- Auto-assign all tasks to current user via `get-current-user` — never ask for name or email
- Never save before user confirms selection in step 5
- Distinguish proposed deadlines (`proposed`) from extracted ones visually
- Tile name ≤35 characters
- Chat link always included in tile
- If input is vague → ask one clarifying question before step 4
- Maximum 7 tasks — if more needed, suggest splitting into phases
- On project planner API error → silently fall back to `create-tiles-from-markdown-by-view`
- If user selects zero tasks in step 5 → regenerate and show the plan again and ask once more