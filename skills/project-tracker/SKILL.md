---
name: project-tracker
description: >
  Keep ONE specific xTiles project the user owns honest and moving. Read the
  project as it actually stands — its pages, its tasks, its dated planner
  entries — then leave the work of the conversation behind inside it: finished
  work recorded as such, next steps that someone will actually be reminded of,
  the links and insights worth keeping, and a clear-eyed answer to "how far is
  this from done".

  Four kinds of request, recognised from how the user asks rather than by asking
  them: understanding where the project stands ("how's the project", "what's
  left", "what did we forget", "what's blocking", or a bare launch); recording
  what this session produced ("log what we did", "save this", "keep this link",
  "record this insight"); working out what comes next ("what's next", "plan the
  next steps", "break this down"); and making a project trackable in the first
  place ("set up tracking for this project").

  Triggers: "track my project", "update the project", "log this to <project>",
  "what's the status of <project>", "how far is <project> from done",
  "what's left on <project>".

  The project can be identified however the user happens to have it to hand: its
  name in their own words — partial or approximate is fine — or a pasted xTiles
  link, including one pointing at a page or tile inside it. Never ask them for an
  id.

  Environment: the Claude / Cowork variant. Deliberately light — a question only
  where a choice is genuinely ambiguous or irreversible, everything else said in
  chat with real links. No HTML widgets. There is no ChatGPT variant yet.
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

Work on a project happens in conversation. The project itself rarely catches
up — the decisions, the finished pieces, the link someone found, the thing
everyone forgot — all of it stays in a chat log nobody reopens. This workflow
exists to close that gap: to read a project as it really is right now, and to
leave the substance of the session inside it, in a form that will still be
useful in three weeks.

**How to read this file.** The numbered steps are a map of where to look next —
they tell you what each stage of a run is *for*, and they are worth following in
that order. What they deliberately don't do is script it: inside each step you
will find what a good outcome looks like, not a sequence of calls to make. Tool
names appear only as pointers to where a thing lives, in parentheses, because
you can work out the call yourself from the tool list — and because the project
in front of you may want reaching that outcome a different way. Where this file
is firm, it is firm about **facts and consequences**: what the platform can't do,
what costs the user money, what would mislead them. That is what
**What the platform makes true** collects, and it is the one section written as
rules rather than as intent, because guessing at those costs a user their
content.

Its job is not to report activity. Its job is to move a project towards being
finished.

---

## What a good run leaves behind

Whatever was asked, a run that went well leaves these true:

- **The project is more honest than it was.** Finished work reads as finished.
  Work that was agreed is findable as work, not as a sentence in a paragraph.
  Nothing on the page claims something that is no longer the case.
- **Nothing that mattered got lost.** The link, the insight, the decision and
  the reasoning behind it are in the project, wherever that project already
  keeps such things.
- **The user knows where they stand.** Not a list of what happened — an answer
  to how far this is from done, and what would have to be true to close it.
- **There is a next step, and it is real.** Named, owned, and living somewhere
  that will surface it again — not a bullet in a tile nobody reopens.
- **The user was not surprised by anything.** What you changed was reported with
  links. What you couldn't do was said out loud, including when the reason is
  their plan rather than their project.

---

## The run, end to end

The steps below are where to look next, in order — not a script to recite.
Each one says what it is for; how you get there is yours.

0. **Know which project you are in** — one project, the user's own, given by
   name or by link.
1. **Read the project as it stands** — quietly, before claiming anything.
2. **Build the picture** — goal, progress, what stalled, what was forgotten.
3. **Work out what this run is for, and do it** — read the intent from how they
   asked; don't ask them which mode they want.
4. **Put each thing where it belongs** — the project's own homes, never a
   template of yours.
5. **Tell the user what changed** — with real links, and the next step.

Two sections sit outside that order and are worth having read before your first
write: **What the platform makes true**, which is where the firm constraints
live, and **If something goes wrong**.

---

## Step 0 — know which project you are in

Everything in a run belongs to one project, and it is the user's project — never
their personal planner, which belongs to other workflows.

**Both ways of naming a project are normal, and both are enough to start.** The
user may say what the project is called — in their own words, partially, with a
typo, in whatever language they named it in — or they may paste a link to it.
Neither is a reason to ask them for the other, and neither is a reason to ask for
an id, which is not something a person has.

**Given a name**, search for it rather than browsing everything
(`xtiles_search_projects` matches titles and content, so an approximate or
partial name usually lands). One plausible match is your project. Several — go
ahead and ask, briefly, listing them by title; that is a genuinely ambiguous
choice and the cheapest possible question. None — fall back to the list of the
user's projects (`xtiles_list_projects`) and reconcile from there, since a
project can be titled quite differently from how its owner refers to it.

**Given a link**, pass it whole to the resolver rather than reading ids out of it
by eye (`xtiles_get_content_by_link`, query string included). The same URL can
address a whole page or a single tile, and only the resolver knows which — and
either way the project you want is the one that content belongs to, which comes
back with it. A link that points inside a project is as good as naming the
project.

**Given the launch context** — a project or page arriving with the invocation,
e.g. from a menu inside xTiles — that wins over everything; a page is enough,
since reading it tells you its project.

**Given nothing**, look at what the user has. One project is the answer; several
is worth one short question rather than a guess.

Say which project you landed on in your first line. A wrong match caught in one
sentence costs nothing; a wrong match discovered after you wrote to it costs the
user cleanup.

If the user has no project at all, say so and point at `/create-project`. This
workflow works inside an existing project and never creates one.

---

## Step 1 — read the project as it stands

Before you write anything — and certainly before you claim anything — build a
real picture. This is the part worth spending calls on, and it should be quiet:
no play-by-play of your reads in chat.

What you are trying to come to know:

**The shape of the project.** Which pages exist, how they are grouped, what each
one is *for*. The structure read gives you this with ids
(`xtiles_get_project_structure`), and it is what every placement decision later
rests on. Note which group is the product-managed planner — that one is addressed
by date, not by page id.

**What is actually written in it.** The page bodies, not just the titles
(`xtiles_get_project_content` reads several pages at once;
`xtiles_get_view_content` reads one closely). Read enough to reason honestly and
no more: if you stopped early on a large project, say so rather than implying you
saw everything.

**The state of the work.** Open tasks, finished tasks, and anything overdue
(`xtiles_list_tasks`). This is the backbone of any claim about progress — a
project's tasks are the only place that reliably distinguishes "done" from
"discussed".

**The recent history.** What the dated planner entries say happened lately
(`xtiles_get_planner_content` on this project). A project with tasks open and no
entry for a fortnight is telling you something.

---

## Step 2 — build the picture

Now form a view of the project, and hold yourself to only what the data from
step 1 supports:

- **The goal, and what counts as done.** Often stated somewhere; often not stated
  anywhere. If it isn't, that absence is the most valuable thing you found — say
  so plainly, because without it "how far from done" cannot be answered
  honestly, and offer to capture it.
- **Progress.** Grounded in tasks and milestones. A count you can point at, never
  a percentage you composed to sound precise.
- **What has stalled.** Overdue work; work agreed long ago with no movement; a
  milestone with nothing under it; a project that has gone quiet while still
  carrying open work.
- **What was forgotten.** Intentions visible in the project's own content or its
  planner entries that never became work. This is usually the single most useful
  output of a run — nobody spots their own omissions.

Never infer that something shipped because it was discussed. Never invent a date,
a deadline, or an accomplishment.

---

## Step 3 — work out what this run is for, and do it

Read it from how the user asked. Don't open a question to find out. Where each
thing you decide to record actually lands is step 4's question, not this one.

**They want to understand where things stand** — "how's the project", "what's
left", "what did we forget", or a bare launch with no verb. Read, don't write.
Answer in chat: a one-line verdict, progress you can point at, the few things
that genuinely need attention, and the one next step worth taking. Keep the list
of concerns to the worst handful — a complete list of everything imperfect is
noise, and says nothing about priority. If the honest answer is "nothing needs
attention", say that instead of padding it. End with a single offer, chosen by
what the project most lacks: record the missing goal, create the next step as
real work, or set tracking up properly. One offer, one line, not a menu.

This is the default. When a request is unreadable, this is what to do — it writes
nothing, so it cannot be the wrong choice.

A shape that reads well — an example of the order and the density, not a form to
fill; drop any line the project gives you nothing for:

```
**{Project}** — {one-sentence verdict}

**Progress** — {done}/{total} tasks · {n}/{m} milestones · last entry {date}
**Goal** — {the stated goal, or: no goal recorded in the project}

⚠️ **Needs attention**
- {overdue task} — due {date}
- {stalled item} — {why it reads as stalled}
- {forgotten intention} — mentioned in {where}, no task for it

→ **Next** — {the single most valuable next action}

[{CTA to open the project}]({url})
```

**They want this session recorded** — "log what we did", "save this", "keep this
link". Pull out only what is concrete: work actually finished, decisions actually
made, materials actually found, insights actually reached, steps actually agreed.
If the session produced nothing concrete, say so and write nothing; a tile
recording that a conversation happened is worse than no tile.

Then let each thing become what it really is. Finished work is the *state of a
task*, not a paragraph — so a matching open task gets completed (and gains a
description if the session explains what was actually done), and work that was
never tracked gets created and then completed. Don't duplicate a task that
already exists. Materials and insights go to their home (step 4). The session itself is worth one dated entry in the project's planner —
one per session, not one per item — carrying what got done, what was decided,
what comes next, and anything worth keeping. And if there is a page whose
description carries the project's current state, refresh it: that is the one
surface you can rewrite freely.

A session entry that works, as an example of the shape rather than a template to
fill — omit any section the session gave you nothing for, and never keep a
heading to hold a placeholder:

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

Worth seeing in it: bold lines rather than deeper headings, one line per item, a
blank line between items, real `<task>` for what is actually work, and links on
their own line. The colours are whatever pair the project already uses.

**They want to know what comes next** — "what's next", "plan the next steps".
Derive a handful of steps from the goal, the open work and the gaps you found,
each concrete enough to start on. Make them real tasks, because a plan that
lives only as text is a plan nobody is reminded of. Dates only where the user
actually gave a horizon — convert it properly against their timezone — and where
they gave none, create the work without dates and say so in a line, offering to
add them. Never invent a deadline. Put a short readable version of the plan
where the project would expect it, and name which single step to start with.

**They want the project made trackable** — "set up tracking". Start from what is
already there, not from a template. A project with a page that serves as an
overview and a page that collects links needs nothing created; say that plainly
rather than adding pages to look busy. A home that exists under a different name
is a home. Create only what is genuinely missing. If no goal or definition of
done can be derived, that is the one thing worth asking for — everything else
measures against it. Milestones are worth proposing (three to five, from the
goal) and worth creating only once the user agrees. Leave a rewritable status
line in the overview page's description, so future runs have somewhere to keep
the current picture. Follow the project's own conventions — group pages only if
it already groups them, match its titling and colours rather than introducing
your own.

Afterwards, a recurring read is worth one line of offer: a weekly run that
re-reads the project and reports what stalled. If they want it, set it up through
the host's scheduler (`anthropic-skills:schedule`, then
`mcp__scheduled-tasks__create-scheduled-tasks`), passing enough in the prompt for
an unattended run to know which project it is about, and be clear that the
schedule runs on Claude's side — xTiles itself runs nothing.

**Two of these in one message** — "log what we did and plan what's next" — is one
run: record first, plan second, one report at the end.

**A run that discovers the project has nothing to work with** — no goal, no
tasks, no obvious home for anything — should still do what was asked with what
exists. Then say in one line that a short setup would make every later run
better, and offer it. Don't quietly do a setup nobody asked for.

---

## Step 4 — put each thing where it belongs

**This is judgement, not a map.** The project belongs to the user; you are adding
to it in the shape it already has. There is no page schema in this workflow, and
introducing one would be the main way to get this wrong.

Think about what a thing *is*, and the placement usually follows:

- Something that **happened** — work finished today, a decision made, a milestone
  reached — is a fact about a moment. The project's planner is addressed by date
  and is where those belong.
- Something that describes **how things are** — the goal, what counts as done,
  milestones, current status, risks — belongs where someone opening the project
  looks first.
- Something that **accumulates** — links, materials, insights, references — wants
  one findable place it keeps piling up in.
- Something **actionable** is work. It becomes a task. Never a bullet, never a
  checkbox: a line describing work is work nobody will be reminded of.

Then two rules of restraint, which matter more than the categories above:

**An existing home always beats a new one, and homes are matched by meaning, not
by name.** A page that collects links is the home for a link whatever it is
called and whatever language it is written in. Read the project's own page titles
and content and recognise its homes; don't go looking for the names you would
have chosen.

**Create a page only when both are true: there is genuinely no home for this, and
more of it is coming.** One-off content goes into an existing page. A page created
for a single insight is clutter the user has to clean up later.

And leave alone what isn't yours: the planner group (write to it by date, never by
page id), archived pages unless you were asked to bring one back, collection
pages (they cannot be written to at all — see below), and any page this request
didn't involve, however untidy it looks.

Never write an empty or placeholder tile or page. No content, no tile.

Worth seeing the reasoning end to end:

| What came out of the session | Where it lands | Why |
|---|---|---|
| "We finished the auth integration" | the matching task, completed · one line in today's planner entry | it's the state of a piece of work, and a fact about a day — neither is page prose |
| Three competitor links | appended to the page that already collects links | an existing home, recognised by what it holds |
| "The onboarding funnel is the real bottleneck" | the insights home; a new page only if there is none *and* more insights are coming | it accumulates |
| "Let's ship the beta by 30.09" | a milestone task with that date, and a line where the project states its milestones | it is work with a deadline, so it has to be work |
| A sharper restatement of the goal | rewritten into the page description | the one thing you can update in place for free |

---

## Step 5 — tell the user what changed

Short, plain, and linked. Say what changed and where, each with a link a person
can click, and end with the one thing worth doing next. No account of your own
process, no list of the tools you called, no summary of the file you just read.

Which usually comes out looking like this — again a shape, not a form:

```
Recorded in **{Project}**:
- {what} → [{CTA for where it landed}]({url})
- {what} → [{CTA for where it landed}]({url})

→ Next: {the one thing worth doing next}
```

Link labels are for people: "Open the overview page", "See today's entry" —
translated into the user's language, never a bare URL pasted into a sentence.

Ask only where a choice is genuinely ambiguous or genuinely irreversible — which
project when several match, whether a new page or an existing one should take
something when the difference will be visible to them. Everywhere else,
recommend in one line and proceed. A question you could have answered yourself
is a question that costs the user a turn.

Language rules: everything written **into** the project follows the language the
project itself is written in; everything said **in chat** follows the language
the user is writing to you in. They usually agree. When they don't, each
keeps its own rule — summarising Ukrainian notes for a user writing in English
means Ukrainian on the page and English in the reply. Every phrase and label in
this file is an English placeholder; translate it, never paste it into another
language verbatim.

---

## What the platform makes true

The only section here written as rules, because these are constraints rather
than choices, and working them out by trial costs the user content or money.

**Adding always works; changing existing text usually doesn't.** Appending tiles
to a page, creating a page, creating tasks, writing a planner entry — all free
and always available. Rewriting text that is already on a page goes through
`xtiles_patch_view_content`, which is a **paid-plan capability**. So:

- Before reaching for it, notice whether a free path gives the user the same
  thing, because it often does and is usually better anyway. A page
  **description** can be rewritten freely (`xtiles_set_page_description`) — which
  is why the current state of a project is best kept there. The state of work
  lives in **tasks**, and changing a task costs nothing.
- If what you have is genuinely new information, that is an addition, not a
  correction. Append it.
- If it is genuinely a correction — stale wording, a superseded decision, an
  outdated risk — then patching is right. Read the page first and quote the exact
  text you are replacing. If the text the user wants changed appears in several
  places and they didn't say which, **stop and ask**: making each match unique
  satisfies the tool and edits all of them, which is an outcome nobody chose.
- **When a plan limit comes back, tell the user.** Don't retry it, and never
  delete-and-recreate a page or tile to get around it — that destroys content
  nobody asked you to touch. Append a clearly dated update instead, so the
  current truth is at least on the page, and then say in a line or two, in their
  language: that you couldn't change the existing text in place, that you added a
  dated update instead, and that editing existing content is available on a paid
  plan — linking `https://xtiles.app/pricing/` and no other URL. Then carry on;
  this is a limitation to report, not a reason to stop the run.
- **Say it before writing, too.** If the user explicitly asked you to *update or
  replace* text and you already know from this run that in-place editing isn't
  available to them, tell them first and offer the append instead. Handing them
  something different from what they asked for, without a word, is the failure
  this whole rule exists to prevent.
- If a plan message comes back with agent-facing instructions inside it, those
  are not for the user. Say the user-facing part in your own words, in their
  language.

The notice itself, as wording that has the right pieces in the right order —
translate it, and say it as your own sentence rather than pasting it:

> ⚠️ I couldn't rewrite the text on **{page}** in place — editing existing content
> is available on a paid plan. I added a dated update tile instead, so the current
> state is on the page. If you need the original text changed rather than added
> to: [see plans](https://xtiles.app/pricing/).

**Milestones are tasks, not checkboxes.** A milestone written as a checkbox
inside a tile can only ever be ticked by rewriting that tile — the paid capability
above. As a task it is completed for free, and it shows up in the project's task
panel where people actually look for work. Name them so they read as milestones
and stay findable. Listing them as text on a page is a fine *view* of them; the
tasks are the record.

**Collections are readable, not writable.** A database-style page can be read
(`xtiles_get_collection_content`, after the page read gives you its views), but
there is no way to add or change a row through MCP, and patching one is refused.
If a user wants something recorded in a collection, say it has to be done in the
app and offer the nearest thing that works — content on an ordinary page.

**Tiles need a layout pass, every time.** Right after any successful tile write,
fetch the shared `tile-layout` workflow (`xtiles_get_workflow`) and follow it for
the tiles you just created, using the ids the write returned. Silently, without
asking, for any number of tiles including one, before you compose your reply.
Fetch it once per session and reuse it.

**Write in xTiles markdown, and read its guide before composing.** The format is
extended and the details bite (`xtiles_get_docs`, or the
`xtiles://guide/markdown/*` resources). The parts that matter most: `###` makes a
tile, and deeper headings don't belong inside one — a bold line is the way to
title a section; colour directives sit directly under the tile heading; links go
one per line in `[title](url)` form, never bare and never inside a list item;
`<task>` is what makes something real work, with a due date only where a real one
exists; tiles hold a limited number of blocks, so long content wants splitting.
Pick one colour pair and keep the project coherent — and if its tiles already
have colours (`xtiles_get_tile_styles`), reuse what is there rather than
introducing a scheme of your own.

**Dates come from the user's timezone, never from a guess.** Resolve it before
working with any date (`xtiles_get_user_timezone`), and derive relative
expressions from it. One trap worth knowing: the task list's "due before" bound
is **exclusive**, so "due before today" is exactly the overdue set and correctly
leaves out what is due today.

**Tasks want an assignee.** An unassigned task exists on its page but appears in
nobody's task list, so default to the current user (`xtiles_get_current_user`)
rather than leaving it to nobody.

**A planner period nobody has touched may answer with template content** — the
tiles it *would* be created with. That is not the user's work: never report it as
recorded progress, and never treat that period as an existing entry.

**Links you report come from the tools' own responses** — the tile or page url a
write returns. Never assemble an xTiles URL by hand, and never link to a tile you
didn't just create.

**Notifications are for things worth a ping.** A page appearing in someone's
project, milestones created or reached, tracking newly set up, or anything an
unattended scheduled run wrote — nobody is watching chat for that last one. Not
for an appended tile, not for tasks ticked while the user is sitting right there,
not for a read-only answer. One short sentence, well under a hundred characters,
pointing at a page rather than a tile (`xtiles_create_notification`).

**Nothing gets deleted here.** Not a page, not a group, not a task. Removing
content is not this workflow's job, and archiving is the reversible answer when a
user wants something out of the way.

---

## If something goes wrong

A failed call is worth one honest line and then carrying on with everything else
— never a fabricated version of what it would have returned. A project too large
to read fully is worth acting on partially *and saying so*, rather than implying
a complete picture. A plan limit is covered above: report, append, continue. A
stale id from an earlier run is worth re-reading rather than retrying.

The through-line: a run that delivers less than hoped but says so plainly is a
good run. A run that quietly delivers something other than what was asked is not.
