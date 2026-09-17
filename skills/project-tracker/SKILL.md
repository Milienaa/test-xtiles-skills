---
name: project-tracker
description: >
  Track, advance and close ONE specific xTiles project the user owns. Reads that
  project's own pages, tasks and project planner, then records what was done,
  what comes next, what has stalled or been forgotten, and the links and
  insights worth keeping — as real tasks and tiles inside that project, not as a
  chat summary.

  Five modes, detected from the request rather than asked about:
  `pulse` (the default) — "how is the project doing", "where are we", "what did
  we forget", "what's blocking", or an empty launch;
  `capture` — "log what we did", "save this to the project", "record this
  insight", "keep this link";
  `plan` — "what's next", "plan the next steps", "break this down";
  `setup` — "set up tracking for this project", "start tracking it";
  `close` — "we're closing this project", "wrap it up", "final retro".

  Triggers: "track my project", "update the project", "log this to <project>",
  "what's the status of <project>", "how far is <project> from done",
  "what's left on <project>", "close the project".

  Not for: turning a chat into a brand-new project (use `create-project`), the
  user's personal day or week (use `daily-brief` / `weekly-review`), or
  restyling one page (use `reorganize`).

  Environment: this is the Claude / Cowork variant. It is deliberately light —
  `AskUserQuestion` only where a choice is genuinely ambiguous or irreversible,
  everything else reported in chat with markdown links. No HTML widgets. There
  is no ChatGPT variant yet.
allowed-tools: >
  mcp__xtiles__xtiles_list_projects,
  mcp__xtiles__xtiles_search_projects,
  mcp__xtiles__xtiles_get_project_structure,
  mcp__xtiles__xtiles_get_project_content,
  mcp__xtiles__xtiles_get_view_content,
  mcp__xtiles__xtiles_get_collection_content,
  mcp__xtiles__xtiles_get_content_by_link,
  mcp__xtiles__xtiles_get_planner_content,
  mcp__xtiles__xtiles_list_tasks,
  mcp__xtiles__xtiles_create_tasks,
  mcp__xtiles__xtiles_update_task,
  mcp__xtiles__xtiles_delete_tasks,
  mcp__xtiles__xtiles_create_view_from_markdown,
  mcp__xtiles__xtiles_create_tiles_from_markdown_by_view,
  mcp__xtiles__xtiles_create_tiles_from_markdown_in_project_planner,
  mcp__xtiles__xtiles_patch_view_content,
  mcp__xtiles__xtiles_set_page_description,
  mcp__xtiles__xtiles_update_page,
  mcp__xtiles__xtiles_create_page_group,
  mcp__xtiles__xtiles_move_pages,
  mcp__xtiles__xtiles_get_page_layout,
  mcp__xtiles__xtiles_set_page_layout,
  mcp__xtiles__xtiles_get_tile_styles,
  mcp__xtiles__xtiles_create_notification,
  mcp__xtiles__xtiles_get_current_user,
  mcp__xtiles__xtiles_get_user_timezone,
  mcp__xtiles__xtiles_get_docs,
  mcp__xtiles__xtiles_get_workflow,
  AskUserQuestion,
  anthropic-skills:schedule,
  mcp__scheduled-tasks__create-scheduled-tasks
---

# xTiles Project Tracker

Work on a project happens in conversation; the project page rarely catches up.
This workflow closes that gap: it reads one project as it actually stands —
pages, tasks, dated planner entries — and leaves the work behind in it. Finished
work becomes completed tasks with real descriptions. Next steps become real
tasks with dates. Links and insights land where that project already keeps such
things. And every run ends with the same two facts: **what moves this project
forward next, and how far it still is from done.**

Its job is not to report activity. Its job is to move a project to its close.

---

## Principles

1. **One project, resolved before anything else.** Every read and every write in
   a run carries the same `projectId`. Never split a run across two projects, and
   never write to the user's personal planner — that is `daily-brief`'s and
   `take-note`'s territory, not this workflow's.

2. **Real data only.** Progress, stalled items, decisions and accomplishments
   come from what the tools returned. Never infer that something shipped because
   it was discussed, never invent a percentage that no task or milestone backs,
   and never fabricate a date.

3. **Append is the contract; updating in place is a privilege.** Adding content
   always works. Rewriting existing content needs
   `mcp__xtiles__xtiles_patch_view_content`, which is a paid-plan capability — so
   appending is the default path and updating is attempted only where it is
   genuinely the right answer. When it is refused, the user is **told**, not
   quietly given something else. See **Updating vs appending**.

4. **Fit into the user's project — never impose a template on it.** This
   workflow has no fixed page schema. It reads what the project already has and
   puts content where that project already keeps that kind of content. A new page
   is created only when there is genuinely no home for something and more of it
   is coming. See **Placement heuristics** — those are judgement rules, not a map
   to apply literally.

5. **Progress over activity.** A run that lists what happened and stops has
   failed. It ends with the next step (offered as a real task) and an honest
   statement of the distance to done — including "I can't tell, because the
   project has no stated goal", when that is the truth.

6. **Light touch.** Ask only when a choice is genuinely ambiguous or
   irreversible. Recommend in chat rather than opening forms. Notify only about
   things a person would want a ping for — a new page, milestones, a closed
   project — never about one appended tile.

7. **Two languages, two rules.** Everything written **into** the project follows
   the language of the project's own content. Everything said **in chat** follows
   the language the user wrote in. They usually agree; when they don't, each keeps
   its own rule. Every template and label in this file is an English placeholder —
   translate it, never emit it verbatim into another language.

---

## Modes — detect, don't ask

| Mode | Recognise it by | Writes |
|---|---|---|
| **pulse** *(default)* | "how's the project", "where are we", "what's left", "what did we forget", "what's blocking", or a bare launch with no verb | nothing |
| **capture** | "log what we did", "save this", "fix this as done", "keep this link", "record this insight" | tasks + tiles + planner entry |
| **plan** | "what's next", "plan the next steps", "break this down", "what should I do this week" | tasks + a plan tile |
| **setup** | "set up tracking", "start tracking this project", "make this project trackable" | pages + milestones + status line |
| **close** | "we're closing this", "the project is done", "wrap it up", "final retro" | retro page + task cleanup + archiving |

**Resolution rules:**

- No recognisable verb, or only a question → **pulse**. Pulse writes nothing, so
  it is the safe default.
- Two modes in one message ("log what we did and plan what's next") → run them in
  order, `capture` then `plan`, in one turn, with one combined report at the end.
- `setup` is implied when a `capture` or `plan` run finds the project has no
  goal, no tasks and no obvious home for anything. Don't switch modes silently:
  do what was asked with what exists, then say in one line that a short setup
  would make the next runs much better, and offer it.
- A mode is never chosen with `AskUserQuestion`. If the request is truly
  unreadable, run `pulse` and show what you found.

---

## Step 0 — resolve the project

In this order, stopping at the first that works:

1. **Launch context.** A `projectId` or `viewId` injected at launch (e.g. from a
   project menu in xTiles) wins over everything. With only a `viewId`, call
   `mcp__xtiles__xtiles_get_view_content(viewId)` — its response carries
   `project_id`.
2. **A link the user pasted.** Pass it whole, query string included, to
   `mcp__xtiles__xtiles_get_content_by_link`. Never parse an xTiles URL by hand —
   the same URL can address a page or a single tile, and the response says which.
3. **A name the user said.** `mcp__xtiles__xtiles_search_projects(text)`. Exactly
   one plausible match → use it. Several → **ask** (`AskUserQuestion`, the
   matches as options). None → fall through to 4.
4. **Nothing at all.** `mcp__xtiles__xtiles_list_projects`. Exactly one project →
   use it. Several → **ask**, offering the most plausible few by name.

**Hard rules for this step:**

- Never guess a `projectId`, and never carry one over from an earlier
  conversation without re-confirming it exists in the list.
- **This workflow never creates a project.** If the user has none, say so and
  point at `/create-project` — that is the workflow that turns a conversation
  into a new project.
- Confirm the resolved project by name in the first line of your reply
  (`Working on **{project title}**`), so a wrong match is caught immediately.

---

## Step 1 — read the project (silently)

No progress narration in chat while reading. One short line is fine if the reads
are slow ("Reading {project}…"); a play-by-play is not.

**Always:**

- `mcp__xtiles__xtiles_get_user_timezone` → the IANA timezone and today's local
  date. Every date in this run — planner anchors, due dates, "overdue" — is
  derived from this, never from a guess. Skip only if already fetched this
  conversation.
- `mcp__xtiles__xtiles_get_project_structure(projectId)` → every page and group
  with ids, `view_type`, `color`, `is_archived`. This is the map that
  **Placement heuristics** reasons over. Note which group has `system: true` —
  that is the project planner, and it is read-only to every write tool in this
  file.
- `mcp__xtiles__xtiles_list_tasks(projectId, completed: false, sort: "due_date")`
  → open work.
- `mcp__xtiles__xtiles_list_tasks(projectId, completed: true, per_page: 50)` →
  finished work, for the progress ratio and for matching "what we did" against
  tasks that are already closed.
- `mcp__xtiles__xtiles_get_project_content(projectId, limit: 5)` → the actual
  page bodies. Page further with `start_view_id: next_view_id` **only** when the
  mode needs it (`close` reads the whole project; `pulse` and `capture` stop at
  two pages and say so if content was left unread).
- `mcp__xtiles__xtiles_get_planner_content(projectId, period: "week", date: today)`
  and the same for last week → the recent dated history. Add
  `period: "day"` reads only when a specific day matters.

**Overdue** is a separate, cheap read and worth its own call:
`mcp__xtiles__xtiles_list_tasks(projectId, completed: false, due_date_before: today)`.
Note the bound is **exclusive** — `due_date_before: today` returns tasks due
strictly before today, which is exactly "overdue" and correctly excludes
anything due today.

**Conditionally:**

- `mcp__xtiles__xtiles_get_view_content(viewId)` for a page you intend to append
  to, when you need its current wording or its exact tile titles.
- `mcp__xtiles__xtiles_get_collection_content` when a page turned out to be a
  collection (database) and its rows matter. Collections are **read-only** here —
  see **Failure handling**.
- `mcp__xtiles__xtiles_get_tile_styles(viewId)` before writing tiles onto an
  existing page, to reuse the colour pair that page already uses.

---

## Step 2 — build the picture

Derive these from step 1. Each one either has evidence or is reported as
missing — never filled in with a plausible guess.

**Goal and definition of done.** From an overview-style page's text, a page
description, or a stated goal in the planner. If the project states no goal,
that is the single most valuable thing missing from it: say so in one line and
offer to capture it. Without it, "distance to done" cannot be honest.

**Progress.** Completed vs open tasks. Milestone tasks counted separately (see
**Milestones are tasks**). Any explicit progress markers found in the content
(a checked list, a "Phase 2" heading) read as supporting evidence, not as the
number.

**Stalled.** Any of:
- open tasks due before today (the overdue read above);
- open tasks with no due date at all, sitting there since before the last planner
  entry;
- a milestone with no open task under it and no completion — work that was
  planned and then dropped;
- no planner entry at all in the last two weeks while open tasks exist.

**Forgotten.** Intentions visible in the project's own content or planner
entries ("we still need to do X", a decision that implies work) with no matching
task, open or closed. This is the highest-value output of a pulse run — it is
what nobody notices on their own.

**Distance to done.** One or two plain sentences grounded in the counts:
"3 of 5 milestones done, 2 tasks overdue, 6 open" — plus what would have to be
true to close it. Never a percentage that tasks and milestones don't support, and
never a date estimate the user didn't give.

---

## Step 3 — Placement heuristics

**This is the part that must not become a template.** The project belongs to the
user; this workflow adds to it in the shape it already has. These are the signals
to reason over, in priority order — not a schema to reproduce.

1. **An existing home beats a new one, always.** Match the pages from step 1 **by
   meaning, not by name** — `Resources`, `Links`, `Sources`, `Reference`, or any
   of those words in the project's own language, are all the same home for a
   saved link. A page whose content is clearly this kind of content **is** the
   home, whatever it is called and whatever language it is written in.
2. **Dated, event-shaped content → the project planner.** What happened in this
   session, a decision made today, a milestone reached, progress made — these are
   facts about a moment, and the planner is what is addressed by date:
   `mcp__xtiles__xtiles_create_tiles_from_markdown_in_project_planner(projectId, period: "day", date: today, markdown)`.
3. **State-shaped content → the overview-ish page.** The goal, definition of
   done, milestones, current status, risks — these describe how things *are*, and
   belong where someone opening the project looks first.
4. **Accumulating reference content → its own page, appended to.** Links,
   materials, insights, snippets: content whose value is that it piles up in one
   findable place.
5. **Anything actionable → a real task.** Never a bullet, never a `- [ ]`
   checkbox inside a tile. A bullet that describes work is work nobody will be
   reminded of.
6. **Create a page only on two conditions together:** there is genuinely no home
   for this content, *and* more of it is coming. One-off content goes into an
   existing page as a tile. A page created for a single insight is clutter the
   user has to clean up.
7. **Follow the project's own conventions.** If its pages are grouped, a new page
   joins a group; if they aren't, don't introduce groups. If its tiles use a
   colour pair, reuse that pair. If its page titles carry emoji, match that; if
   they don't, don't add any.
8. **Never touch:** pages inside a `system: true` group (address the planner by
   date instead), archived pages unless explicitly unarchiving, collection pages
   (they cannot be written through MCP at all), and any page the user didn't
   involve in this request just because it looked untidy.
9. **Never write an empty or placeholder tile or page.** No content → no tile.
10. **When two homes are equally reasonable and the difference is visible to the
    user** — appending to an existing page versus creating a new one — that is one
    of the few `AskUserQuestion` moments. One question, two options, then proceed.

**Worked examples** — the reasoning this should produce:

| What came out of the session | Where it goes | Why |
|---|---|---|
| "We finished the auth integration" | completed task + one line in today's planner entry | it's finished work (a task's state) and a dated fact (the log) — not page content |
| Three competitor links | appended tile on the page that already holds links | rule 1 — an existing home, matched by meaning |
| "Realised the onboarding funnel is the real bottleneck" | appended tile on the insights home; if there is none and more insights are coming, a new page | rule 4 + rule 6 |
| "Let's ship the beta by 30.09" | a milestone task, `dueDate` set, plus a line on the overview page | rule 5 — it is work with a deadline, so it has to be a task |
| A goal restated more precisely | the page description, rewritten | the one surface that can be **updated** without a paid plan |

---

## Step 4 — write primitives

| Intent | Tool | Notes |
|---|---|---|
| New page in the project | `xtiles_create_view_from_markdown(projectId, markdown)` | first `##` = page title, each `###` = a tile |
| Add tiles to an existing page | `xtiles_create_tiles_from_markdown_by_view(viewId, markdown)` | appends; returns `tiles[]` with `id` + `resource_url`, and `parent_resource_url` |
| Dated entry | `xtiles_create_tiles_from_markdown_in_project_planner(projectId, period, date, markdown)` | `period`: `"day"` \| `"week"` \| `"month"` |
| A status line under a page title | `xtiles_set_page_description(viewId, description)` | inline markdown, may span lines; **free and rewritable** — the one update-in-place surface that needs no paid plan; empty string clears it |
| Create tasks | `xtiles_create_tasks(projectId, tasks[])` | call `xtiles_get_current_user` first and assign the user by default — an unassigned task appears in nobody's task list |
| Complete a task or milestone | `xtiles_update_task(projectId, taskId, completed: true)` | free; the reason milestones are tasks |
| Rename / recolour / archive a page | `xtiles_update_page(projectId, viewId, …)` | tab colours: `DEFAULT, LAVENDER, RED, ORANGE, YELLOW, GREEN, DARK_GREEN, LIGHT_BLUE, BLUE, PURPLE, PINK, GRAY, BEIGE` |
| Group pages | `xtiles_create_page_group(projectId, title, view_ids[])` / `xtiles_move_pages` | `move_pages`' `group_id` is the **destination**: `null` means top level, so passing `null` to reorder pages inside a group moves them out of it and still reports success |
| Rewrite existing tile text | `xtiles_patch_view_content(viewId, replacements[])` | paid-plan capability — see **Updating vs appending** |
| Fix the grid after writing tiles | `xtiles_get_workflow("tile-layout")` | **mandatory after every tile write** — see below |

### Markdown rules

Before composing markdown for any `*_from_markdown` tool, read the format guide —
resource `xtiles://guide/markdown/overview` (also `/canvas`, `/blocks`) or
`mcp__xtiles__xtiles_get_docs`. The rules that matter most here:

- `###` is a tile. Never `####`/`#####` inside one — use a **bold line** for a
  sub-section instead.
- Metadata goes immediately under the `###` line, no blank line between:
  `@color:` and `@colorSize:`, then one blank line, then the body.
- **One colour pair per project**, alternating between tiles. If the project's
  tiles already have colours (`xtiles_get_tile_styles`), reuse that pair. If not,
  pick one and keep it for every future run: `COLDTURKEY` + `PATTENS_BLUE` for
  delivery/tracking work, `GHOST` + `CUMULUS` for business, `BLUE_CHALK` +
  `CUMULUS` for research/ideas. `@colorSize:LIGHTER` unless the page uses
  something else.
- Links: `[title](url)`, one per line, each after a blank line. Never a bare URL,
  never a link inside a list item, never a URL as its own label.
- Actionable items: `<task priority="high" dueDate="2026-09-30">Title</task>`.
  Only add `dueDate` when a real deadline exists — never estimate one. `- [ ]`
  checkboxes are for a throwaway checklist only, and ticking one later needs a
  paid plan, which is why work belongs in `<task>`.
- A tile holds at most 40 blocks — split long content across tiles.
- Separate items with a blank line; one line per item. Dense and scannable, not
  paragraphs.

### Layout pass — mandatory, silent, every time

Immediately after **every** successful tile write (`_by_view`,
`_in_project_planner`, or a `create_view_from_markdown` whose page you then want
laid out), call `mcp__xtiles__xtiles_get_workflow` with id `tile-layout` and
follow it exactly: the `tile_ids` from the write response are its "added tiles",
the markdown you just composed is their content, and the `view_id` comes from the
same response. Any tile count, including one. Never ask about it, never mention
it, never skip it because the content "looks simple". Run it before composing the
reply. (Fetch `tile-layout` once per session and reuse it.)

---

## Milestones are tasks, not checkboxes

A milestone written as `- [ ] M2 · Beta ready` inside a tile can only be ticked
by rewriting that tile — a paid-plan capability. The same milestone as a task is
completed with `xtiles_update_task(completed: true)`, which every plan can do, and
it additionally shows up in the project's task panel where people look for work.

So: **milestones are project tasks.** Name them so they read as milestones and
stay findable — `M1 · `, `M2 · ` prefix, `priority: "high"`, a `dueDate` only when
the user named one. Listing them on the overview page as text is fine and useful;
that text is a view of the milestones, not the record of them.

When a milestone is reached: complete the task, log it in the planner entry for
that date, and — because this is one of the few genuinely significant events —
send a notification.

---

## Updating vs appending — and when to say an upgrade is needed

Adding content always works. **Rewriting existing content needs
`xtiles_patch_view_content`, which is a paid-plan capability.** This workflow
therefore appends by default — and when the user actually wants an *update*, it
says so plainly instead of silently substituting an append.

**The order to try:**

1. **Is there a free update path?** Often yes, and it is the better answer
   anyway: a status/goal line lives in the **page description**
   (`xtiles_set_page_description` — rewritable, no plan limit), and the state of
   work lives in **tasks** (`xtiles_update_task` — no plan limit). Reach for
   these before patching.
2. **Is appending the honest answer?** New facts, a new session's log, another
   insight, more links — these are additions, not corrections. Append.
3. **Only a real correction left** — stale wording on a page, an outdated risk,
   a superseded decision — attempt `xtiles_patch_view_content`. Read the page
   first (`xtiles_get_view_content`), copy the exact substring, and make each
   `old_str` unique. If the text the user wants changed appears more than once and
   they didn't say which, **stop and ask** — never widen each match into its own
   replacement, which edits them all without anyone choosing.

**When the tool answers with a plan limit:**

- **Do not retry. Do not delete-and-recreate** the page or tile to work around
  it — that destroys content the user did not ask you to touch.
- Fall back to appending a clearly-marked, dated update tile
  (`### 🔄 Update — DD.MM.YYYY`), so the new truth is at least on the page.
- **Tell the user, in their language, in two lines:** what you could not do
  (change the existing text in place), what you did instead (added a dated
  update tile), and that editing existing content is a paid-plan feature — with
  the link. Then finish the run normally; this is a limitation to report, not a
  failure to stop on.
- Show only the user-facing half of any plan-limit message the tool returns.
  Never paste its agent-only instructions into chat, and never quote the raw
  block — rewrite it as your own sentence in the user's language.
- Always link to `https://xtiles.app/pricing/` — never a URL assembled from
  anywhere else.

Wording to translate, not to paste verbatim:

> ⚠️ I couldn't rewrite the text on **{page}** in place — editing existing content
> is available on a paid plan. I added a dated update tile instead, so the current
> state is on the page. If you want the original text changed rather than added
> to: [see plans](https://xtiles.app/pricing/).

**Warn before writing, not only after.** If the user's request is explicitly
"update / replace / fix the text on that page" and this run has already hit the
plan limit once, say so **before** writing anything else, and offer the append
alternative. Producing something different from what was asked, without saying
so, is the one failure mode this section exists to prevent.

---

## Mode playbooks

### pulse — read the project, write nothing

The default, and the one that has to be genuinely useful on its own.

Report in chat, in this order, short lines only:

```
**{Project}** — {one-sentence verdict}

**Progress** — {done}/{done+open} tasks · {n}/{m} milestones · last entry {date}
**Goal** — {the stated goal, or: no goal recorded in the project}

⚠️ **Needs attention**
- {overdue task} — due {date}
- {stalled item} — {why it reads as stalled}
- {forgotten intention} — mentioned in {where}, no task for it

→ **Next step** — {the single most valuable next action}

[Open {Project} in xTiles →]({url})
```

At most five items under "Needs attention" — the worst five, not all of them.
Say "nothing needs attention" when that is true, rather than padding it.

End with **one** offer, chosen by what the project lacks most: create the next
step as a task, record the missing goal, or run a short setup. One offer, one
line, no menu.

Pulse writes nothing. If the user asks for the state to be recorded on the page,
that is a `capture` run — say so and do it.

### capture — fix what this session produced

1. **Extract from the conversation**, and only what is concrete: work actually
   finished, decisions actually made, links actually found, insights actually
   reached, next steps actually agreed. **If the session produced nothing
   concrete, say so and write nothing** — a tile that says "we discussed the
   project" is worse than no tile.
2. **Finished work → tasks.** For each item, look for a matching open task from
   step 1: found → `xtiles_update_task(completed: true)`, adding a `description`
   when the session explains what was actually done. No match → create it, then
   complete it (`xtiles_create_tasks` has no `completed` field, so it is two
   calls). Never create a duplicate of a task that already exists.
3. **Links, insights, materials →** per **Placement heuristics**, appended to
   their home.
4. **The session itself → one planner entry** for today's date (`period: "day"`),
   using the session-log format below. One tile per session, not one per item.
5. **Status line** — if an overview-style page exists, refresh its page
   description with the current shape of things (free, rewritable).
6. **Layout pass**, then report: what was recorded, each with its link, and the
   next step last.
7. **Notification** only if something significant happened (a page was created,
   milestones were created or reached). Not for appended tiles or completed
   tasks in a manual run.

Session-log tile format:

```
### 📌 {What this session was about} — DD.MM.YYYY
@color:COLDTURKEY
@colorSize:LIGHTER

**Done**

- {finished item} — {one line on the outcome}

**Decided**

- {decision} — {the reasoning in one line}

**Next**

<task dueDate="2026-09-30">{next step}</task>

**Materials**

[{title}]({url})
```

Omit any section with no real content. Never keep a heading to hold a placeholder.

### plan — turn the state into next steps

1. Derive 3–7 next steps from the goal, the open and overdue work, and the gaps
   found in step 2. Each one concrete enough to start: a verb, an object, an
   outcome.
2. **Dates.** If the user named a horizon ("next week", "by the end of the
   month"), convert it with the timezone from step 1 and set real `dueDate`s. If
   they named none, create the tasks **without** due dates and say so in one
   line, offering to add them — never invent a deadline.
3. Create them as project tasks (`xtiles_create_tasks`, assigned to the current
   user by default), and put a short plan tile where **Placement heuristics**
   says — usually the overview page, or the planner's current week when the plan
   is explicitly for this week.
4. Layout pass, then report the plan as a list with the project link, and name
   which single step to start with.

### setup — adapt the project, don't scaffold it

1. **Inventory first** (step 1 already did). Decide, per kind of content, whether
   a home exists: overview-ish, materials, insights, and the planner (always
   present, system-managed). **A home that exists under a different name is a
   home** — do not create a second one beside it.
2. **The one thing worth asking:** if no goal or definition of done can be
   derived, ask for it — one `AskUserQuestion`, or accept it from the
   conversation if it is already there. It is the input everything else measures
   against.
3. **Create only what is missing**, per rule 6 of the heuristics. A project that
   already has an overview and a links page needs nothing created; say that
   plainly rather than adding pages to look busy.
4. **Milestones.** Propose 3–5 derived from the goal, in chat, and create them as
   tasks once the user agrees. Don't create milestones nobody confirmed.
5. **Status line** on the overview page via `xtiles_set_page_description`, so
   there is one rewritable surface carrying the current state from here on.
6. **Grouping** only if the project already uses groups.
7. **Layout pass**, then report what now exists, with links.
8. **Notification** — yes. A structure appearing in someone's project is exactly
   the kind of change worth a ping.
9. **Offer a recurring pulse** in one line: a weekly run that re-reads the
   project and reports what stalled. If the user says yes, invoke
   `anthropic-skills:schedule` then `mcp__scheduled-tasks__create-scheduled-tasks`
   with a prompt of the shape
   `Run project-tracker — mode: pulse · project: {projectId} ({title}) · schedule: weekly-monday-9am`,
   the cron, and the timezone from step 1. Be explicit that the schedule runs on
   Claude's side — xTiles itself runs nothing.

### close — finish the project properly

1. **Read everything**: page through `xtiles_get_project_content` to the end, all
   tasks including completed, and the planner across the project's span. A retro
   built on a partial read is worse than none.
2. **Compose a retro page** (`xtiles_create_view_from_markdown`) from real data
   only: the goal as stated · what was delivered · what was dropped and why · the
   decisions that shaped it · an index of the materials collected · what to do
   differently. Quote the project's own wording where you can.
3. **Open tasks — ask.** One `AskUserQuestion`: complete them as done, leave them
   open, or delete them. This is genuinely ambiguous and partly irreversible, so
   it is never decided for the user. **`xtiles_delete_tasks` runs only on an
   explicit yes**, and never on tasks the user hasn't seen listed.
4. **Archiving**, only if asked: `xtiles_update_page(is_archived: true)` hides a
   page and keeps it findable in `xtiles_get_project_structure`. **This workflow
   never calls `xtiles_delete_page` or `xtiles_delete_page_group`** — deletion is
   irreversible and closing a project is not a reason for it.
5. Layout pass, then a short closing summary with the retro link, and a
   **notification**.

---

## Reporting back in chat

Light, short, and linked. After any writing run:

```
Recorded in **{Project}**:
- {what} → [{where}]({url})
- {what} → [{where}]({url})

→ Next: {the one thing to do next}
```

- Every link comes from the tool's own response — `resource_url` for a tile,
  `parent_resource_url` or `link` for a page or project. **Never assemble an
  xTiles URL by hand**, and never link to a tile you didn't just create.
- Link labels are CTAs a person can act on — "Open the Overview page →",
  "See today's log →" — translated into the user's language, never a bare URL.
- One next-step line at the end. No summary of your own process, no list of the
  tools you called.

---

## Notifications — what earns one

`mcp__xtiles__xtiles_create_notification` — `text` is one short sentence, **100
characters maximum** (longer is rejected), `url` is an absolute page URL on the
xTiles host (a page, never a tile deep link), `agent_source` is `"Claude"`.

**Yes:** a page was created · milestones were created or one was reached · setup
completed · the project was closed · any scheduled run that wrote something
(nobody is watching chat for those).

**No:** a tile was appended · tasks were completed in a manual run · a pulse run ·
anything the user is looking at right now anyway.

---

## Failure handling

- **A tool call fails** → say what failed in one line, continue with everything
  else, and never fabricate what it would have returned.
- **A plan limit on `patch_view_content`** → **Updating vs appending**. Report it,
  append instead, carry on.
- **The project is too big to read fully** → read two pages of content, act on
  what you have, and say explicitly that part of the project wasn't read rather
  than implying a complete picture.
- **A collection (database) page** → readable (`xtiles_get_collection_content`),
  not writable: MCP has no add-row or update-row tool, and patching a collection
  is refused. If the user wants something recorded in one, say it has to be done
  in the app, and offer the nearest thing that works — a tile on a canvas page.
- **A planner period with no page** → the read may answer with the **template**
  content that period would be created with. That is not the user's own work:
  never report template tiles as recorded progress, and never treat a templated
  period as an existing entry.
- **`system: true` group** → the planner. Writing to it by page id answers 409;
  address it by date instead.
- **A task that no longer exists** (a stale id from an earlier run) → re-read the
  task list rather than retrying the id.

---

## How to behave

- One project per run, named back to the user in the first line.
- Real data only. No invented percentages, dates, accomplishments or decisions.
- Append by default; update only where it is right, and **say so when the plan
  blocks it**.
- Fit into the project's existing structure. Create a page only when there is no
  home and more content is coming.
- Anything actionable is a task, never a bullet or a checkbox.
- Milestones are tasks.
- The layout pass runs after every tile write, silently, always.
- Ask only when genuinely ambiguous or irreversible; otherwise recommend in one
  line and proceed.
- Notify only what a person would want a ping about.
- Never delete a page or page group. Never delete tasks without an explicit yes.
- Project content in the project's language; chat in the user's.
- Every run ends with the next step and an honest distance to done.
