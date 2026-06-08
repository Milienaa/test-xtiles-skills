---
name: action-plan-v2
description: Break down any message, idea, or information into actionable
   tasks and save them to xTiles workspace with a tile overview. Use when
   user wants to create tasks, break down a project, make an action plan,
   split into steps, create to-do list, organize work, set deadlines,
   plan execution, turn idea into tasks.
allowed-tools: mcp__xtiles__get-current-user, mcp__xtiles__get-user-timezone, mcp__xtiles__search-projects, mcp__xtiles__list-projects, mcp__xtiles__get-project-content, mcp__xtiles__create-tasks, mcp__xtiles__create-tiles-from-markdown-in-my-planner, mcp__xtiles__create-tiles-from-markdown-in-project-planner, mcp__xtiles__create-tiles-from-markdown-in-view
---

# xTiles Action Plan v2

A good idea without a plan stays an idea. This skill takes any input —
a message, a goal, a project description, or meeting notes — breaks it
down into formal xTiles tasks, and saves a tile overview to the daily
page of your chosen destination.

## Your process

1. **Setup** — call `mcp__xtiles__get-current-user` → save the `id`
   field from the response as `userId` (string); call
   `mcp__xtiles__get-user-timezone` → save today's local date in
   ISO 8601 for `due_date` baseline.

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

6. **Create tasks** — for each selected task, call
   `mcp__xtiles__create-tasks`:
   - `title` — task title
   - `description` — task description (omit if empty)
   - `due_date` — confirmed deadline in ISO 8601 (omit if no date)
   - `assigneeId` — `userId` string from step 1 (always, never ask)

7. **Create tile** — build tile content (see Tile format below) and
   save to the chosen destination:

   **My Planner:**
   → `mcp__xtiles__create-tiles-from-markdown-in-my-planner`
     `date` = today (ISO 8601), `period` = `"day"`, `markdown` = tile content

   **Project (happy path):**
   → `mcp__xtiles__create-tiles-from-markdown-in-project-planner`
     `project_id`, `date` = today, `period` = `"day"`, `markdown` = tile content

   **Project (fallback — any 400/500 or "Invalid group" error):**
   → `mcp__xtiles__get-project-content`(project_id) → take `view_id`
     of the first available page
   → `mcp__xtiles__create-tiles-from-markdown-in-view`
     `view_id`, `markdown` = tile content
   Do not surface the error to the user — silently fall back.

8. **Show confirmation:**
   ```
   ✅ Action plan saved.

   🔗 [Open in xTiles](https://xtiles.app/VIEW_ID)
   ```
   Replace `VIEW_ID` with the `view_id` returned by the tile creation
   tool called in step 7 (all three paths return a `view_id`).

## Tile format

**Tile name** (≤35 characters):
```
Action Plan · [e.g. Jun 8]
```

**Tile content:**
```markdown
## Action Plan · Jun 8

- [ ] Task 1 title · due: Jun 12
- [ ] Task 2 title · due: Jun 15
- [ ] Task 3 title

🔗 [View conversation](https://claude.ai/chat/local_{conversation_id})
```

- Include `· due: DATE` inline for tasks that have a confirmed deadline; omit for no-date tasks.
- Chat link is always the last element — required, never omit.
- Tile name must be ≤35 characters.

## Rules

- Auto-assign all tasks to current user via `get-current-user` — never ask for name or email
- Never save before user confirms selection in step 5
- Distinguish proposed deadlines (`proposed`) from extracted ones visually
- Tile name ≤35 characters
- Chat link always included in tile
- If input is vague → ask one clarifying question before step 4
- Maximum 7 tasks — if more needed, suggest splitting into phases
- On project planner API error → silently fall back to `create-tiles-from-markdown-in-view`
- If user selects zero tasks in step 5 → regenerate and show the plan again and ask once more