---
name: action-plan
description: Break down any message, idea, or information into actionable
   tasks and save them to xTiles workspace. Use when user wants
   to create tasks, break down a project, make an action plan,
   split into steps, create to-do list, organize work, set
   deadlines, plan execution, turn idea into tasks.
allowed-tools: mcp__xtiles__search-users, mcp__xtiles__create-tasks
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

1. **Resolve the current user** — call `mcp__xtiles__search_users` with the
   user's name or email from context. If the user wrote `@username`, strip
   the `@` and use the username as the query. If nothing is known, ask:
   "What is your name or email in xTiles?" Extract `id` and `email` for
   `assignees`. Use whichever field is returned — both are accepted.
2. **Determine deadlines** — read the Deadline rules below before planning.
3. **Analyze the input** — understand the goal and scope.
4. **Break into tasks** — identify 3–7 concrete actionable steps.
5. **Enrich each task** — add description and deadline.
6. **Show the interactive plan** — present tasks as a numbered checklist;
   ask the user to reply with numbers to save (e.g. "1,2,4") or `all` / `none`.
7. **Wait for selection** — do not save anything until the user replies.
8. **Save selected tasks** — call `mcp__xtiles__create-tasks` with all selected
   tasks in a single call; include `assignees` and `due_date` for every task.
9. **Confirm** — show the confirmation block with a link to xTiles.
## Deadline rules

**Step 1 — Extract from user input.**
Look for explicit dates, relative references ("by Friday", "next week",
"until the 25th", "in 3 days"), or an overall project deadline. If found,
distribute tasks evenly up to that date.

**Step 2 — Infer from context.**
If no date is stated but the scope implies one (e.g. "prepare for tomorrow's
meeting"), use that implied deadline.

**Step 3 — Ask if unclear.**
If the input gives no indication of timing and the scope is ambiguous,
ask one question before planning:
> "What is the target date or deadline for this?"

Do not guess arbitrarily. A missing deadline is better caught here than
wrong dates saved to xTiles.

**Formatting:** Always use full ISO 8601: `YYYY-MM-DD`. Space tasks logically
— the exact interval depends on the total timeframe and task complexity.

## Assignee rules — CRITICAL

Always assign every task to the current user. Tasks without an assignee
do not appear on daily pages. Call `mcp__xtiles__search-users` before
creating tasks and pass both `id` and `email` in `assignees`.

## Show the plan before saving
 
---
**Action Plan — [goal title]**

- [ ] 1. **[Title]** — [what to do] *(due: [date])*
- [ ] 2. **[Title]** — [what to do] *(due: [date])*
- [ ] 3. **[Title]** — [what to do] *(due: [date])*

Reply with task numbers to save (e.g. `1,3`) or `all` / `none`.
---

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