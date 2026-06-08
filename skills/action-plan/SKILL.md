---
name: action-plan
description: Break down any message, idea, or information into actionable
   tasks and save them to xTiles workspace. Use when user wants
   to create tasks, break down a project, make an action plan,
   split into steps, create to-do list, organize work, set
   deadlines, plan execution, turn idea into tasks.
allowed-tools: mcp__xtiles__search-users, mcp__xtiles__create-tasks, mcp__xtiles__list-projects, mcp__xtiles__search-project, mcp__xtiles__get-planner-content
---

# xTiles Action Plan

A good idea without a plan stays an idea. This skill takes any input —
a message, a goal, a project description, or meeting notes — and breaks
it down into clear actionable tasks saved directly to the xTiles workspace,
so the work can start immediately.

## What this skill does

Analyzes the input, identifies concrete next steps, enriches each with
a description and a realistic deadline, shows the plan for interactive
review with checkboxes, then saves only the selected tasks to xTiles.

## The result the user gets

A ready action plan in xTiles with 3–7 tasks, each containing:
- A clear title that starts with a verb
- A short description of what exactly needs to be done
- A realistic deadline based on logical sequence and complexity
- The user assigned as the task owner
  After saving — a direct link to the xTiles tasks page.

## Your process

1. **Ask what to create** — first thing, before any tool calls:
   ```
   question: "How would you like to save this plan?"
   options:
     - Note in Planner (a tile on the daily page)
     - Page in Project (a new view inside the project)
   ```
   Wait for the answer. Everything below depends on this choice.
2. **Find relevant projects** — call `mcp__xtiles__search-projects` with
   2–3 keywords from the user's input.
   - If results found: take top 2–3 matching projects.
   - If no results: call `mcp__xtiles__list-projects` and pick 2–3 most
     plausibly related projects by name.
   - If still nothing: offer only "My Planner" as the destination.

3. **Ask where to save** — only if "Page in Project" was chosen in step 1.
   Use `AskUserQuestion` with single select:
   ```
   question: "Where would you like to save this plan?"
   options:
     - [Project name 1]
     - [Project name 2]
     - [Project name 3]
   ```
     If "Note in Planner" was chosen — skip this step entirely,
     destination is always My Planner.
   Wait for the answer before proceeding.
4. **Resolve the current user** — call `mcp__xtiles__search-users` with the
   user's name or email from context. If nothing is known, ask via
   `AskUserQuestion`: "What is your name or email in xTiles?"
   Extract `id` and `email` for `assignees`.
5. **Ask about deadlines** — use `AskUserQuestion` with single select:
   ```
   question: "Do these tasks have a deadline?"
   options:
     - No deadline
     - In 3 days  (YYYY-MM-DD)
     - In 7 days  (YYYY-MM-DD)
     - In 14 days (YYYY-MM-DD)
     - Other
   ```
   Compute and show actual dates for the relative options.
   If "Other" — follow up via `AskUserQuestion` with a date input.
   If "No deadline" — create tasks without `due_date`.
   Never infer or guess dates.
6. **Analyze the input** — understand the goal and scope.
7. **Break into steps** — identify 3–7 concrete actionable items.
   Each title starts with a verb (Create, Write, Review, Set up…).
8. **Show the plan** — use `AskUserQuestion` with `multiSelect: true`:
   ```
   question: "Which items would you like to save to xTiles?"
   multiSelect: true
   options: [
     { label: "[Item title]", description: "[What to do][ · due: DATE]" },
     ...
     { label: "Save all", description: "Save the entire plan to xTiles" },
   ]
   ```
   If the user selects "Save all" — save everything regardless of other
   selections. Wait for the answer before saving anything.
9. **Save** — depends on the format chosen in step 1:
   **"Note in Planner"**
   - Create a tile with:
      - Title: derived from topic, max 35 characters, title case
      - Content: list of selected items in Markdown
         + `🔗 [View conversation](https://claude.ai/chat/local_{conversation_id})` at the bottom
   - Destination:
      - "My Planner" → `mcp__xtiles__create-tiles-from-markdown-in-my-planner`
      - Project selected → `mcp__xtiles__create-tiles-from-markdown-in-project-planner`
        **"Page in Project"**
   - Requires a project selected in step 3. If "My Planner" was chosen,
     ask again via `AskUserQuestion` to pick a project.
   - Call `mcp__xtiles__create-view-from-markdown` with `projectId`.
   - Structure the Markdown as:
     ```
     ## [Plan title]
     ### [Section or task group]
     [Items as bullet list]
     🔗 [View conversation](https://claude.ai/chat/local_{conversation_id})
     ```

## Assignee rules — CRITICAL

Always assign every task to the current user. Tasks without an assignee
do not appear on daily pages. Call `mcp__xtiles__search-users` before
creating tasks and pass both `id` and `email` in `assignees`.

## Show the plan before saving

Use `AskUserQuestion` with `multiSelect: true`:

```
question: "Which tasks would you like to save to xTiles?"
multiSelect: true
options: [
  { label: "[Task title]", description: "[What to do] · due: [date]" },
  { label: "[Task title]", description: "[What to do] · due: [date]" },
  ...
  { label: "All tasks", description: "Save the entire plan to xTiles" },
]
```

Generate one option per task before "All tasks".
If the user selects "All tasks" — save everything regardless of other selections.
The built-in "Other" option lets the user type a custom reply if needed.

## After saving

---
Build the link as https://xtiles.app/ + the view_id returned by the tool
and render it as: 🔗 [Open action plan](https://xtiles.app/VIEW_ID)
---

## Rules

- Never save tasks before the user confirms the selection
- All tasks start with a verb (Create, Write, Review, Set up…)
- If input is vague — ask one clarifying question before planning
- Maximum 7 tasks per plan — if more needed, suggest splitting into phases