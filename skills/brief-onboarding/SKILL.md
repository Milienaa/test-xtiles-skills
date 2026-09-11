---
name: brief-onboarding
description: >
  Use immediately after a user finishes the xTiles onboarding questionnaire —
  builds and writes their first Daily planner brief directly from their role
  and the tools they said they use, so they see real value before doing
  anything themselves — no preview, no approval gate. This same skill also
  serves every **recurring** run once the user schedules it — there is no
  separate daily-digest skill to hand off to.

  Entry data on a first run (already known, never re-asked with a survey):
  `role:` — the role; `used_connectors:` — the "My connectors:" list (may
  include a custom name, or `other`); `additional:` — set only when News
  was explicitly requested (`News`/`News_yes`, case-insensitive; normalize
  to lowercase `news` when persisting it in step 6). A negative like
  `News_no` never counts, even though it contains "News".

  The real starting message carries none of these labels — it looks like:
  "Role: {role}. My connectors: {tools}. News_yes" (or `News_no`). A
  labeled-field variant or natural bullet list (e.g. "Role: Marketing · My
  connectors: Notion, Google Calendar, Gmail, Other · Additional: News")
  works the same way — parse whichever shape carries these three values,
  never re-derive them from a survey.

  **Connection status is never handed to this skill as data — it determines
  that itself**, with a lightweight live probe per named connector (see step
  2). Gmail and Calendar are probed first, as the highest-value connectors.

  Setup triggers: "start onboarding preview", "onboarding welcome digest", "first-run preview",
  "onboard a new xTiles user into the Planner", "Set workflow of Onboarding
  Brief (brief-onboarding) on xTiles MCP".

  Environment: this is the Claude / Cowork variant — every interactive
  moment (Connector check, Schedule — which itself is up to two sequential
  `AskUserQuestion` calls, see step 6 — and Related workflows) uses
  `AskUserQuestion`, never an HTML form. **The one exception is the CTA
  after the write (step 5) — a single non-interactive `show_widget` button
  linking to the fresh brief** (see "CTA widget HTML" below); this is the
  only `show_widget` call in this skill, it carries no choices to make, and
  it never replaces an `AskUserQuestion` anywhere else in the flow. In
  ChatGPT Work, where every question is an inline `ask_user_input` / `genui`
  surface, use `brief-onboarding-with-gpt` instead.

  Environment triggers: "Brief Onboarding in Claude", "the Claude version",
  "Claude Onboarding Preview".
allowed-tools: >
  mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner,
  mcp__xtiles__xtiles_create_notification,
  mcp__xtiles__xtiles_get_user_timezone,
  mcp__xtiles__xtiles_get_current_user,
  mcp__xtiles__xtiles_list_tasks,
  mcp__xtiles__xtiles_get_workflow,
  mcp__xtiles__xtiles_get_page_layout,
  mcp__xtiles__xtiles_set_page_layout,
  WebSearch,
  WebFetch,
  mcp__mcp-registry__suggest_connectors,
  anthropic-skills:schedule,
  mcp__scheduled-tasks__create-scheduled-tasks,
  AskUserQuestion,
  show_widget
---

# xTiles Onboarding — First & Recurring Daily Brief

## Principles

1. **Never re-ask what onboarding already answered.** Role and the tools the user said they use come in as `role:` and `used_connectors:` (plus an optional `additional:`) — never show a role/tools survey.
2. **Never trust a name as proof of connection.** Whether something is actually connected is never handed to this skill as data — it's determined live, per connector, with a lightweight probe (step 2). A tool the user *said* they use may not be connected yet, or may have been connected since.
3. **Never block on a connector.** Every check that could ask the user to connect something must also offer to skip it, and a way to proceed with nothing resolved at all — see step 2.
4. **Never end a run with nothing to show.** An empty resolved set, or an explicit `additional: news` request, always produces a real Today News tile (step 3) — never an empty digest.
5. **Real data, not placeholders.** Pull from connectors (or the web/inbox, for Today News) before writing so the user sees live content, never invented names, events, or messages.
6. **A distinct tool is always a distinct tile, never merged by category.** Two calendars, or Gmail and Reclaim, never share a tile just because they're both "scheduling" or "inbox" — see step 4.
7. **No preview, no approval gate.** Once step 4's fetch completes, write directly to xTiles — first run or recurring alike. The only interactive moments left in this skill are the Connector check (step 2), the Schedule questions (step 6 — up to two sequential `AskUserQuestion` calls: "Run this every morning?"/"Notify me?" and, if scheduling was chosen, "Add anything to your Daily Brief before I schedule it?"), and the Related-workflows question (step 7). **"No approval gate" applies only to writing the brief itself (step 4 → step 5) — it never means racing past an interactive question.** Every `AskUserQuestion` call in steps 2, 6, and 7 must actually wait for the user's answer before the next step runs; never assume a default and continue in the same turn just because content-generation itself doesn't need approval.
8. **Never surface a third-party connector's own preview in chat.** Read what Todoist, Reclaim, or any other connector returns purely as data to build your own xTiles tile — never let its raw response render as its own card in the conversation, and avoid calls whose only purpose is to produce one.
9. **Never recreate a task that's already open.** Check `xtiles_list_tasks` before writing any `<task>` (step 5) and drop anything that duplicates an already-open task from yesterday or today.
10. **Every write is followed by the layout pass.** The moment tiles are created, re-lay them out into a justified grid via the shared `tile-layout` workflow — automatically, before the CTA, never skipped.
11. **The only deliverable is tiles written to xTiles.** A run — first or recurring — is complete only when the tiles are in xTiles and the full post-write sequence has run: layout pass → Gmail follow-through (if applicable) → CTA button → schedule question → related-workflows question. **On a silent recurring run, only the layout pass, Gmail follow-through, and the opt-in notification still happen** — the CTA, schedule question, and related-workflows question are all skipped (see step 1 and step 7).

---

## Algorithm

**Period is always Daily.** Never ask which period to set up.

### 1. Entry — parse the config

Two ways this skill starts:

- **First run.** The incoming message carries `role:` and `used_connectors:`, and optionally `additional:` — no resolved connector list yet. This is always right after the onboarding questionnaire. **Before anything else, send one short, plain-language line of context** — e.g. "Building your first Daily Brief for a {role} — I'll pull it together from the tools you already use, so you open xTiles to something real, not an empty page." — before any silent probing and before the Connector-check question, if one turns out to be needed. Never let the very first thing the user sees be a bare question with no context. Then go to **step 2**.
- **Recurring run.** The incoming message instead carries the **full config this skill itself wrote at the end of a previous run** (step 6) — `role:`, `tools:` (the already-resolved set), `skipped:` (connectors that weren't connected last time, if any), `news_categories:` (only present if a Today News tile was produced last run — fallback or explicit request), `additional:` (carried forward only when `news` was explicitly requested, so a future recurring run keeps building the tile even once other connectors are resolved), and `notify:`. Its presence (specifically `tools:` alongside `role:`) is the signal — there is no separate scheduled-run skill to hand off to. This runs silently — no intro line, nobody is watching chat. **First, silently re-probe every connector in `skipped:`** (same lightweight probe as step 2, no question shown): if one now succeeds, fold it into today's resolved set and mention it once, briefly, in today's notification ("Nice — Gmail just connected, so I've folded it into today's brief"). This is the only re-check that ever happens on a recurring run. Then **go to step 3** (a mandatory checkpoint — it triggers Today News only if the resolved set is still empty or `additional: news` carried forward, otherwise it's a no-op) **and then step 4 (Silent data fetch)**, using the resulting resolved set.

  **What actually runs after the write on a recurring run — spelled out exactly, nothing implied:** the layout pass always runs; Gmail follow-through (step 5's item 4) always runs if Gmail is in the resolved set, whether or not anyone is watching chat — it's inbox hygiene, not a chat-visible action; and — only if `notify:true` — the notification (step 6) fires. **The Schedule question and the related-workflows question never fire on a recurring run** — those, and only those, are what "silent" excludes.

### 2. Connector check — detect it yourself, then connect or skip

**There is no fixed catalog of connectors in this skill.** `used_connectors` can name anything — Gmail, Calendar, Slack, a tool invented after this file was written, or `other` with a name typed by the user. Treat every name the same way:

1. For each connector in `used_connectors`, make one lightweight, read-only probe call using whatever MCP tool that connector exposes (a minimal list/search call, never a write). A response with no auth error means it's connected right now; an auth error or a missing tool means it isn't. **This probe result — not the questionnaire answer — is the only source of truth for "connected."**
2. **Gmail and Calendar are probed first**, since they tend to carry the richest everyday signal. Everything else follows.
3. For an unfamiliar connector name, **check the known-bundle table below first** — only once it's confirmed the name isn't a known alias do you fall back to guessing a namespace match (e.g. a connector called `{Name}` would expose `mcp__claude_ai_{Name}__*` tools) and use the least invasive read call available. If no matching tool exists at all *and* no bundle row covers it, treat it as not connected — it becomes a candidate to connect natively or to skip. **Never report "no native connector available" for a name that appears in the table below** — that's a wrong answer, not a missing one.

   **Known connector bundles — probe, offer, and connect once per bundle, never once per name:**

   | Named tool(s) the user may list | Underlying connector / MCP namespace |
   |---|---|
   | Outlook Mail, Outlook Calendar, Teams | Microsoft 365 |
   | Jira, Confluence | Atlassian (Rovo) |

   If two or more of the user's named connectors fall in the same row, they share **one** probe, **one** `suggest_connectors` call, **one** slot of the 2-connector cap (point 6), **one** line in the status message, and **one** `"Skip {bundle}"` option — never a separate one per named tool. Connecting or skipping the bundle resolves every named tool listed in that row at once. **Record it in `tools:`/`skipped:` (step 6) by the bundle's own name (e.g. `Microsoft 365`, `Atlassian`), never by the individual names the user happened to type** — that's what a future recurring run's silent re-probe (step 1) keys off of. **This table is illustrative, not exhaustive** — if the session has a matching MCP tool under a different name for something not listed here, that still counts as a known connector, not an unsupported one.
4. **If `other` is in `used_connectors`** — it's a signal the user's real stack is bigger than what they listed. Beyond probing the named connectors, look at what other connector tools this session actually has available (its own tool/capability list) and quickly probe any that weren't named. Anything that responds successfully becomes an **extra candidate** below — distinct from a missing named connector, since it's already usable and just needs opting in, no auth flow required.
5. **If `used_connectors` is non-empty and every named connector's probe succeeds, and step 4 above found no extra candidates to offer** — there is nothing to ask, but **still send the status message** (point 6's format below, ✅ line only, no 🔌 line) so the user sees confirmation of what's already connected before the flow moves on; then go straight to step 3 (a no-op there, unless `additional: news` was requested) **and then step 4.** No question at all — a status line is not a question and needs no answer. **If `used_connectors` was empty from the very start, this is not that case** — zero probes is not zero failures, treat it exactly like an empty resolved set and go to step 3 (skip the status line too — there's nothing to confirm).

   **Checkpoint before moving on: does the status line you're about to send contain a 🔌 line?** A pure ✅-only (and optionally 📰) line means this point 5 applies — no question, continue straight to step 3. **The instant a 🔌 line is needed, this is point 6 instead, and the `AskUserQuestion` below is mandatory** — never let a status line you're already halfway through writing carry you past a question it should have triggered.
6. **Otherwise** (something failed to connect, or there's an extra candidate to offer):
   - **Cap the active connect offer at 2 connectors.** Pick the two highest-priority failed probes (Gmail/Calendar first, then the rest in the order named — a bundle from the table in point 3 counts as one item here regardless of how many of its names were listed) and **immediately call `mcp__mcp-registry__suggest_connectors`** for just those two — this renders real, native connect buttons directly in the Cowork UI right away. Any further missing connector beyond these two is **not** given a button this run — mention it in text only, as something the user can connect themselves later by just asking in chat.
   - **In that same turn**, send one short status message, built dynamically — always sent, whether or not there's a question after it, e.g.:
     ```
     **🔍 Here's what I found:**
     ✅ **Connected:** Gmail, Google Calendar
     🔌 **One click away:** Jira — connect it above
     🚫 **No native connector yet:** Todoist — I'll skip that for this brief
     📰 **Bonus:** I'll also put together a News tile for you
     ```
     — bold both the intro line and every label, and keep each status on its own line so the message scans as a list of badges, not a paragraph. The ✅ **Connected:** line lists everything already connected (Gmail/Calendar first if present). The 🔌 **One click away:** line names the (at most 2) connectors/bundles just offered a connect button. A 🚫 line covers whatever's still missing beyond that — connectors with no matching MCP tool at all (checked against the bundle table in point 3 first) get the label **No native connector yet:** and "I'll skip those for this brief"; connectors that do exist but fell outside the 2-cap get **Also missing:** and "you can connect them later by just asking" instead — use two separate 🚫 lines if both kinds apply. **Double-check every name in the 🚫 line against the bundle table in point 3 before writing "no native connector"** — a name covered by a table row (like Outlook Mail or Jira) always belongs in ✅ or 🔌, never here. The 📰 **Bonus:** line appears only if Today News (step 3) will run this pass (fallback or explicit request). Omit any line with nothing to say. Translate into the user's language, keeping the same bold-label structure.
   - **Immediately after**, ask via `AskUserQuestion` — one call, up to two questions:
     - Always include a `multiSelect` question letting the user explicitly skip anything still pending: `"Skip anything?"` with one option per connector just offered a connect button (`"Skip {name}"`). Leaving all unselected means "still open, don't skip" — but the flow must still literally wait for the `AskUserQuestion` tool call to return an answer; never proceed before that.
     - **Only if step 4 found extra candidates**, add a second `multiSelect` question: `"What should I add to the brief?"` with one option per extra candidate plus a fixed `"No, that's enough"` — matching the pattern "I also see Todoist and Linear connected. What should I add to the brief?".
   - **Never gate the real connect flow behind a question the user has to answer first** — the native connect buttons from `suggest_connectors` are already live in the same turn as the question above.
7. When the user connects something through the native buttons from `suggest_connectors`, that's picked up by the re-probe in point 8 below — never restart the whole check. Answering "Skip {name}" marks that connector done-for-this-run, no further nagging. Picking an "Add" option needs no connect flow at all — it's already usable, just fold it straight into the resolved set.
8. **On the AskUserQuestion response, re-probe every connector still marked "not yet connected" that wasn't explicitly skipped** — cheap, and it's the only way to catch a connection the user just made through the native buttons from `suggest_connectors`, since that flow doesn't report back directly. This includes connectors beyond the 2-cap that were only mentioned in text. The **resolved set** = every connector whose probe succeeded (original pass or this re-probe), plus any added, minus anything explicitly skipped. **Track the skipped list too** — carried forward as `skipped:` into the schedule config in step 6, so a future recurring run can quietly notice if one of them gets connected later (see step 1) without ever asking again. `xTiles` itself is required, not optional — if it's not connected, this skill isn't reachable at all; connect it first.

**When to ask — and when never to ask again automatically.** This question fires **at most once per run**, right after the intro line in step 1, and only on a first run. It is not a recurring nag:
- On a **recurring run**, this step never appears at all — step 1 handles it with a silent re-probe of `skipped:` instead.
- The user can always trigger a fresh check by asking directly at any time ("connect my Slack now") — that re-enters this step for just the named connector, regardless of run mode.

**No content-preference questions.** Every connector in the resolved set contributes its own default content (step 4) — the user can still ask to change anything after the write.

**Two worked examples of the probe — not an exhaustive list, the pattern is the same for anything else the user names:**
- **Gmail** — call `mcp__claude_ai_Gmail__list_labels` (cheap, no query needed). Success = connected.
- **Calendar** — call `mcp__claude_ai_Google_Calendar__list_events` with `maxResults:1`. Success = connected.

### 3. Today News — a standalone tile, by request or fallback

**This is a mandatory checkpoint, not an optional detour — every path from step 1 or step 2 passes through here, on a first run and on a recurring run alike. It is never valid to route straight from step 1 or step 2 to step 4.**

This step produces a `### 📰 Today News` tile in **two independent situations** — check both, every run:

- **Fallback.** The resolved set is empty — whether because `used_connectors` was empty from the very start, every named connector's probe failed, or the user skipped everything in step 2. A run must never ship nothing.
- **Explicit request.** The incoming config's `additional:` field includes `News` — an optional field, often absent. When requested, build this tile **even if other connectors are also resolved** — unlike the fallback case, it does not step aside the moment real per-connector tiles exist.

If **neither** applies, skip this step entirely and continue to step 4.

When either applies:

1. **If the incoming config already carries `news_categories:`** (a recurring run that already built this tile before) — use those exact categories, don't re-derive them; they were chosen deliberately for this person and should stay stable run to run. **Otherwise** (first time this tile is built), infer 2–4 topic categories from `role:` that this person would plausibly care about right now (the same judgment `today-news` uses — e.g. a Product Manager cares about product/UX trends, competitor moves, and AI tooling news; an Engineer cares about dev tooling, major tech news, and security). If nothing about the role narrows it down, default to broadly useful categories: industry news, productivity/tools, and a general "worth knowing" pick. **Whichever way they were obtained, this exact category list is what carries forward into `news_categories:` in step 6** if the user schedules a recurring run.
2. **Sourcing — mail first, then web.** If Gmail is in the resolved set, first check there: search recent (last 24–48h) mail that itself carries real news — subscribed news digests/newsletters, alert-style mail, publications — relevant to the chosen categories. If genuine, current items turn up this way, use them. **Only if Gmail isn't in the resolved set, or that search turns up nothing usable**, fall back to `WebSearch`/`WebFetch`: find real, current items from the last 24–48 hours per category, verified against reputable sources. **Never invent an item, a date, or a link.** If a category genuinely yields nothing either way, drop that category rather than force it; if literally every category comes back empty, say so plainly rather than writing an empty tile.
3. Build **one** tile, `### 📰 Today News`, with one labeled sub-section per category and 2–3 real items each — one line per item, source linked inline.
4. **Persistence.** The category list carries forward into `news_categories:` in step 6, regardless of which trigger produced it. If this run's tile came from an explicit `additional: news` request, also carry `additional: news` forward — that's what tells a future recurring run to keep building it even once other connectors are resolved. **The fallback case has no such persistence flag** — the moment even one connector is usable in a later run, the fallback tile stops appearing on its own.

---

### 4. Silent data fetch

**Silently, without messaging the user**, pull fresh data from every connector in the resolved set, and — if step 3 triggered — research (or read mail for) the Today News tile too.

**There is no single grouping that fits every connector — the right shape follows the nature of the data itself, never a template repeated for each one.** Before building a tile, ask what *this specific kind of data* actually needs, not "which of the usual three buckets does this go in."

**And separately — whether that shape becomes one tile or several is a question of volume, not a fixed rule per connector.** A handful of items reads fine inside one tile with labeled internal sections; it's genuine volume in one of those sections that earns it a tile of its own. Never split into several thin, mostly-empty tiles just because a connector "usually" gets split, and never cram a genuinely large volume into one dense, hard-to-scan tile either — let what was actually pulled decide, each time.

**A distinct tool is always a distinct tile, never merged by category.** Two calendars (e.g. a work Google Calendar and a personal one, or Calendar alongside Reclaim) each get their own `### 📅` tile — never combined into one "Calendar" or "Scheduling" section just because they're the same kind of tool. Gmail and Reclaim never share a tile either, even though both are "inbox-adjacent." The only place multiple sources ever land in one tile is when they're genuinely the *same* connector's own data split by volume (Email's three buckets, Slack's two) — never across two different connectors.

**Never surface a third-party connector's own rendered preview in chat.** Read what Todoist, Reclaim, or any other connector returns purely as data to build your own tile from — never let its raw response render as its own card in the conversation, and avoid calls whose only purpose is to produce one.

- **Email arrives as a firehose that needs triage — the natural question is "do I have to act on this."** That's why it splits by urgency: 🔴 needs a concrete next step, 🟡 informational only, ⚪ automated noise. **With real volume in each bucket**, this becomes three tiles (step 5): `### 📩 Email — Action Points` (🔴, plus real `<task>`s), `### 📩 Email — Key People` (🟡, grouped by *sender* — the "who's actually writing to you" view, a completely different axis from urgency), `### 📩 Email — Noise` (⚪, one rollup line, never itemized). **With only a handful of relevant emails**, keep it all in one `### 📩 Email` tile instead, with the same three labeled blocks inside it — same content, less scaffolding. Tone either way: retell the email in second person, action + consequence, don't copy the subject line — "Google shut down your ad account yesterday — log in and appeal, the window is limited," not "Your account closed."
- **Slack already arrives grouped — by channel and by thread — so urgency isn't the useful axis there.** The real question is "was this addressed to me, or is it ambient discussion I can skim." **With enough real activity**, that becomes two tiles: mentions/DMs that need a reply as real `<task>`s, and a short topics rollup for what's being discussed in the channels you follow. **On a quiet Slack day**, fold both into one `### 💬 Slack` tile instead, with the same two labeled sections inside it. Either way, never reshuffle it into an urgency/FYI/noise split — that would throw away the channel structure that's the whole point of Slack, regardless of how many tiles it ends up as.
- **Calendar isn't a triage problem at all — there's no "noise" in someone's schedule, and a single day only ever needs one tile.** The natural shape is chronological and forward-looking: what does today look like, where's the free time, what's worth preparing for. One `### 📅 Workload` tile per calendar connector — never a second copy of the schedule the user can already see, and never split by volume the way Email or Slack might be. Compute event count, hours occupied, and the longest free focus window; write one concrete 🎯 focus-recommendation sentence; for each event, a one-sentence agenda (from the event description, or the most recent related email/meeting note — never invented) and, only where genuinely implied, one prep `<task>`. Collect anomalies (overlaps, back-to-back runs, meetings with no agenda) at the bottom, never inline.

**Apply the same kind of thinking to anything else in the resolved set** — Linear, Google Drive, GitHub, Reclaim, or a connector this file has never heard of. Read what it actually returns, decide the natural axis (urgency, person, project, time, or something else entirely), and only split it into more than one tile when there's genuinely enough volume on that axis to earn separate scanning — never reuse Email's or Calendar's exact shape, or their tile count, for a different kind of data just because it's already written down here.

Never fabricate a category that has nothing in it for that connector — omit an empty block rather than write "none" three times over. A tile from a connector that returned genuinely nothing still gets written, with one line saying so — its total absence looks like a bug, not a quiet day.

**If Slack is in the resolved set and the user hasn't named channels**, pick a small, sensible set yourself before reading: favor channels the user has posted or been mentioned in recently, plus any that are obviously company-wide (general/announcements), capped at around five. This is a judgment call, not a fixed algorithm — the goal is real signal, not exhaustive coverage; the user can always ask to add more later.

**If Gmail is in the resolved set**, also check for unread newsletters (senders like `@substack.com`, `@beehiiv.com`, or ones the user is clearly subscribed to) and — if any are found — collect them for a single `### 📧 Newsletters` tile (one line per publication, title as the hyperlink, no separate "Open" link). Skip the tile entirely if none are found.

**Cross-source overlap.** If the same real ask clearly shows up in two different connectors (e.g. a Slack message and a follow-up email about the same specific thing), don't produce two separate action items for it — treat one as primary, and add a short cross-reference on the other's line (e.g. "also emailed"). Only merge when it's genuinely the same ask; an unrelated message from the same person is not a duplicate.

Use only real data. Never invent names, events, messages, or links. If a connector call fails outright (error, timeout, 401) — record that as a failure, surfaced explicitly in the write (step 5), never silently written as "no data."

---

### 5. Write to xTiles

**Write immediately once step 4's fetch completes — first run or recurring alike. There is no preview and no approval gate.**

Tool: `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`
- `period`: "day"
- `date`: current date in ISO 8601

**Write all sections in a single call.** Combine every tile from step 4 into one markdown string and call the tool once — never split into separate calls per connector.

**Write content tiles only** — no date/header tile, no meta or self-tuning tiles.

**Before finalizing the task list — dedup against already-open tasks.** Call `mcp__xtiles__xtiles_list_tasks` for tasks that are **not completed** — regardless of due date, not just yesterday/today — and drop any action item about to be written whose text duplicates one that's already open. **A task with a real future deadline (dueDate set days out) is still open today and must be included in this check** — narrowing the window to "due yesterday or today" would let it silently duplicate on a later run once its own due date has passed. Never recreate the same unfinished task twice across two runs.

**Resolve the assignee once per run.** Call `mcp__xtiles__xtiles_get_current_user` and reuse that email for the `assignee` attribute on **every** `<task>` written this run — first or recurring, no exceptions.

**Tile formatting** — each `###` section carries color annotations immediately after the heading, no blank line between:

```
### [emoji] [Title]
@colorSize: LIGHTER
@color: [COLOR]

[content]
```

- `@colorSize` is always `LIGHTER`.
- `@color` — pick randomly per section from this list **exactly as written**: `GHOST, CUMULUS, GOSSIP, COLDTURKEY, BLUE_CHALK, MILK_PUNCH, HAWKES_BLUE, PATTENS_BLUE, SAIL, ATHENS_GRAY, BERMUDA, PERFUME, SELAGO, RICE_FLOWER, WHITE_LINEN, POLAR`. **Never a semantic name (RED, BLUE, GREY…) — it won't render.** Never repeat the same color twice in a row.
- The heading emoji names the tile's *subject*, never a status. Status markers (🔴 🟡 ⚪ ⏳ ✅ ❓) belong on item lines inside a tile, never in a `###` heading.

**Action items are real tasks, not checkboxes:**

```
**Action items**

<task dueDate="2026-08-11" assignee="user@example.com">Restore the Google ad account</task>

<task priority="high" dueDate="2026-08-10" assignee="user@example.com">Sign the contract</task>
```

- Never `- [ ]` for an action item.
- One `<task>` per line, blank line between each; never nested inside a list item; never carries a link.
- `dueDate="YYYY-MM-DD"` — always set, defaulting to today; use a later date only when the source states a real deadline.
- `priority` only when the source itself signals urgency — omit otherwise; at most a third of a run's tasks should be `high`.
- `assignee` — always the current xTiles user's own email, resolved via `xtiles_get_current_user`, on every task, every run. Never omit it, never assign to anyone else in this flow.
- Never `completed="true"`.
- Never a task that duplicates an already-open one — checked against `xtiles_list_tasks` before this call.

**Links are always inline hyperlinks inside a sentence — never a link alone on its own line.** `… → [Open email](url)` renders as a normal hyperlink; a line containing *only* a link renders as a big block-link card instead, which is never what's wanted here.

**Worked tile examples, in full markdown — the same two connectors as step 4's Email and Calendar, each shown in its split (high-volume) case. Everything else follows step 4's reasoning (the shape *and the tile count* follow the data), never a layout copied from these:**

```
### 📩 Email — Action Points
@colorSize: LIGHTER
@color: SAIL

- [Poke-style description — action + consequence, second person] → [Open email](url)

---

**Action items**

<task dueDate="2026-08-11" assignee="user@example.com">Restore the Google ad account</task>
```

```
### 📅 Workload
@colorSize: LIGHTER
@color: BERMUDA

**N events · ~X h occupied · longest focus window HH:MM–HH:MM (X h)**

🎯 [Focus recommendation — one concrete sentence]

**HH:MM–HH:MM · Meeting name** — Participant · [Google Meet](url)

📋 [Agenda — one sentence, from a real source]

<task dueDate="2026-08-11" assignee="user@example.com">[Prep task, only if genuinely implied]</task>
```

`### 📩 Email — Key People` (🟡, grouped by sender) and `### 📩 Email — Noise` (⚪, one rollup line) follow the same pattern as Action Points, scoped to their own bucket. `### 📧 Newsletters` (if any were found) is one line per publication: `**[Publication](url)** — one-line summary.` `### 📰 Today News` (step 3, whether by fallback or explicit request) is one labeled sub-section per category with 2–3 linked items each.

**The combined (low-volume) case** uses the exact same blocks, just inside one tile instead of several:

```
### 📩 Email
@colorSize: LIGHTER
@color: SAIL

🔴 **Needs action**

- [Poke-style description] → [Open email](url)

---

**Action items**

<task dueDate="2026-08-11" assignee="user@example.com">Restore the Google ad account</task>

🟡 **Key people**

**Name (context)**

- [One-line item — no link]

⚪ **Noise**

- N notifications — nothing urgent
```

Same idea for Slack — a quiet day gets one `### 💬 Slack` tile with a `**Mentions**` block (real `<task>`s) and a `**Topics**` block underneath, instead of two separate tiles.

**Every run creates fresh tiles — this skill has no way to update an existing tile.** Write the full set every time, even if a same-titled tile already exists on the page from an earlier run today.

**After the write — run in order, no exceptions:**

1. Write `✅ Your Daily Brief is ready — built just now from real data.` in chat, then immediately call `show_widget` with the **CTA widget HTML** (see below), replacing `{VIEW_URL}` with the `resource_url` of the **first** tile in the write response (fall back to the page URL, `https://xtiles.app/{view_id}`, only if that's missing). **Never leave `{VIEW_URL}` unresolved and never output a markdown link instead of the widget — the button must render every time this item runs.** **Manual (first) runs only — never sent on a silent recurring run** (step 1's recurring path has no chat audience).
2. **Layout pass — mandatory, silent, never asked about.** Read `view_id` and `tile_ids` from the write response (never re-derive them). **If either is genuinely missing from that response** — don't block or retry the write; skip this layout pass for this run only (the tiles remain written and usable, just unarranged) and continue to the next item below. This is the one case where the layout pass is allowed to not run. Otherwise, call `mcp__xtiles__xtiles_get_workflow` with id `tile-layout` and follow it exactly, **passing `tile_ids` as its "added tiles" and the markdown just written as their content** — those are required inputs the workflow itself expects, not optional context — plus these **layout hints**: default 2 tiles per row, give a heavy tile its own full-width row — this holds regardless of tile count; a rich run (e.g. split Email + Slack + Calendar + Notion + News) can easily produce 5+ tiles, and the workflow should still lay all of them out, just across more rows of the same 2-per-row grid. This workflow is the one that actually calls `xtiles_get_page_layout`/`xtiles_set_page_layout` — skipping the input handoff here is why it can silently do nothing.
3. **Non-scheduled runs only:** immediately ask the Schedule question — see step 6. Never a widget.
4. **If Gmail is in the resolved set — mandatory, silent, every run:** mark every ⚪ Noise and newsletter thread as read with `mcp__claude_ai_Gmail__unlabel_thread` (remove `UNREAD`). Never touch 🔴 or 🟡 threads, and never draft or send anything on the user's behalf — this only marks threads read.
5. **Recurring runs only, and only if the config's `notify:` is `true`:** call `mcp__xtiles__xtiles_create_notification` — `url` the tile-focused deep link, `text` the fixed string `"Your Daily Brief is ready — see what matters today in 2 min."` translated into the user's language, with exactly one allowed addendum: if step 1 just silently reconnected something from `skipped:`, append `" Nice — {Name} just connected, so I've folded it into today's brief."` for that one run only — never customized any other way.

### 6. Schedule (optional)

The Schedule question is asked right after the write (step 5). **A schedule set up here re-invokes this same skill, `brief-onboarding`, every morning — there is no separate daily-digest skill.** The whole point of resolving everything in steps 1–4 once is that the recurring run can skip straight past all of it.

Before asking, say one line adapting this to the actual resolved connectors, translated into the user's language:

> Every morning I can check {resolved connectors, e.g. Gmail and Calendar} myself and have your Daily Brief waiting in xTiles — no need to ask each time.

Then ask via `AskUserQuestion` (one call, two questions):

- `"Run this every morning?"` (single_select): `"Yes, 9:00 on weekdays"`, `"Yes, 9:00 every day"`, `"No, thanks"` — the user can also pick "Other" for a custom time/cadence.
- `"Notify me in xTiles each time it's ready?"` (single_select): `"Yes, notify me"`, `"No, thanks"`.

In Claude Code (no Cowork), ask the same two things as plain text if `AskUserQuestion` isn't available.

- If scheduling was chosen (either preset or a parsed custom time/cadence):
  - **First, before creating anything, run a fresh discovery pass** — regardless of whether `other` was in `used_connectors` — using the same technique as step 2 point 4: check this session's own available connector tools for anything that responds successfully and isn't already in the resolved set. **Never rely only on candidates found earlier in step 2** — this is a second, independent check run right before scheduling, specifically so the schedule question always has something real to offer.
  - Build the `AskUserQuestion` (`multiSelect`) options in this order:
    1. **Connected but unused** — one `"Add {name}"` option per connector the discovery pass just found (e.g. Notion responds successfully but wasn't part of today's brief → `"Add Notion"`).
    2. **If that pass finds nothing** — don't leave the question with only "No, that's enough." Propose 1–2 connectors that fit the user's role/interests from the entry data (e.g. a designer → Figma, a support lead → Intercom, someone tracking metrics → Amplitude) as `"Connect {name}"` options — these are genuine suggestions, not confirmed-connected, so picking one runs the normal connect flow (`suggest_connectors`) for it rather than folding it in directly.
    3. Always end with a fixed `"No, that's enough"`.
  - Ask this as `"Add anything to your Daily Brief before I schedule it?"`. **This question always fires when scheduling is chosen — never skip asking it, and never let it resolve to a bare "No, that's enough" without first running both the discovery pass and the role-based fallback above.** The flow must literally wait for the answer before continuing. The user can also type a brand-new tool name via the built-in "Other" option. If they name a new tool (or pick a `"Connect {name}"` suggestion), run it through the same probe/connect flow as step 2 for just that one connector (one `suggest_connectors` call, one re-probe) and fold the result into today's resolved set. **Fold every addition from this question into the resolved set before the config below is assembled** — the recurring config gets created once, already final, never scheduled first and patched afterward.
  - **Then** invoke `anthropic-skills:schedule`, then `mcp__scheduled-tasks__create-scheduled-tasks`:
  - **`prompt`**: the full recurring-run config, assembled from this run's resolved state —
    ```
    Run brief-onboarding — role: {role} · tools: {resolved connector set} · skipped: {connectors whose probe failed or that the user skipped, or "none"} · news_categories: {the exact categories step 3 used this run — only include this field at all if step 3 ran} · additional: {"news" if this run's Today News tile came from an explicit request, otherwise omit} · notify: {true/false} (if true, call xtiles_create_notification at the very end of that run — mandatory, do not skip) · schedule: daily-{HH:MM} days:{days}
    ```
    **`news_categories:` and `additional:` are conditional** — include `news_categories:` only when step 3 ran this pass (fallback or explicit request); include `additional: news` only when the request was explicit, so a future recurring run keeps building the tile even once other connectors are resolved. When there's no Today News tile this run, omit both entirely.

    **No `daily_content:` field** — step 4 always re-derives each connector's content fresh from what it actually returns that day, so a snapshot from setup time would only ever be stale; there's nothing to gain from persisting one. This is exactly the config step 1 looks for to detect a recurring run — assembling it correctly here is what lets tomorrow's run skip the connector check entirely, while `skipped:` still lets it quietly notice a later connection (step 1).
  - **`schedule`**: cron built from the answer the same way as any other scheduled task — `9:00 on weekdays` → `M H * * 1-5`, `9:00 every day` → `M H * * *`, a custom answer → parse it the same way. Default `0 9 * * 1-5`.
  - **`timezone`**: from `mcp__xtiles__xtiles_get_user_timezone`.
  - **If `notify:true`** — the digest already on the page right now is worth notifying about too; call `mcp__xtiles__xtiles_create_notification` immediately with the same fixed text as step 5's item 5.
  - Confirm: "Done — your Daily Brief will be ready in xTiles every morning at [time]." (append ", and I'll notify you in xTiles each time" if `notify:true`). **Continue to step 7 in the same turn.**
- If **"No, thanks"** — acknowledge briefly, **continue to step 7 in the same turn.**
- **If `anthropic-skills:schedule` or `mcp__scheduled-tasks__create-scheduled-tasks` genuinely isn't available here — this is not a dead end.** Say so in one plain line ("I can't set up automatic scheduling here just yet — but your Daily Brief is ready, and I'll build a fresh one anytime you ask.") and, if `notify:true` was requested, still send today's notification per the bullet above — that part never depended on the schedule actually being created. **Then continue to step 7 in the same turn, exactly as if the user had said "No, thanks."** A missing scheduling tool skips the schedule itself, never the mandatory closing step.

### 7. Related workflows

**Mandatory closing step of every manual run** (skip only on a silent recurring run, which ends after step 5's notification). **Branches on step 6's outcome — never the same question regardless of the answer:**

- **If step 6 ended in a schedule actually being created** — the user is in a "yes" mood; ask the full question via `AskUserQuestion` (single select): "Want to set up anything else?"
  - 🌙 Evening Reflection — a quick end-of-day recap that sets tomorrow up for you
  - 📰 Today News — a daily news digest on topics you care about (**omit this option entirely if step 3 already built a Today News tile this run** — offering it again reads as duplicating what they just got)
  - 📊 Weekly Review — a recap of what moved forward this week
  - Nothing else, thanks

  On selection, send the exact matching phrase to hand off (never run it yourself):
  - Evening Reflection → `Set workflow of Evening Reflection (evening-reflection) on xTiles MCP`
  - Today News → `Set workflow of Today News (today-news) on xTiles MCP`
  - Weekly Review → `Set workflow of Weekly Review (weekly-review) on xTiles MCP`
  - "Nothing else" — acknowledge briefly and stop.

- **If step 6 ended in "No, thanks", or the scheduling tool was unavailable** — the user just declined one piece of automation; don't immediately ask them to consider another. No `AskUserQuestion` here. Send one short, low-pressure line instead and stop: "You can also set up Evening Reflection, Today News, or Weekly Review anytime — just ask."

---

## How to connect connectors

Do not send the user to settings manually and do not give a URL to follow. Call `mcp__mcp-registry__suggest_connectors` — it renders interactive connect buttons directly in the Cowork UI.

**Flow:**
1. Call `mcp__mcp-registry__suggest_connectors` passing the name(s) of the connector(s) to connect (capped at 2 per step 2, point 6) — the user may pick more than one at once.
2. The `AskUserQuestion` shown alongside it (step 2) is what resumes the flow once the user is done — there is no separate confirmation step. The user clicks the connect button(s), the auth flow runs natively for each, and answering the question (even with nothing selected) continues from where the run left off.
3. Re-probe as described in step 2, point 8, to pick up anything just connected.

**Fallback when the registry is unavailable (free plans).** Say so in one reassuring line, e.g. "Connecting more tools needs a plan upgrade — for now, I'll build your Daily Brief from what's already connected, and it'll still be a real, useful one." — then proceed with whatever's already usable. Never let this read as a dead end.

**This connect flow is always optional here.** Every connector offered in step 2 already has a skip path. Never let a stalled or failed connect attempt block the run — if it doesn't complete, drop that connector and continue.

---

## CTA widget HTML

Show this via `show_widget` immediately after the write (step 5, item 1) — the one and only `show_widget` call in this skill. Replace `{VIEW_URL}` with the tile-level link resolved in step 5's write sequence (falling back to the page URL only if no tile-level link was available) before calling `show_widget`. Translate the button label into the user's language.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
html,body{overflow:hidden;height:auto}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:12px;background:transparent}
.btn{display:block;width:100%;padding:12px 20px;border-radius:10px;font-size:15px;font-weight:700;color:#fff;background:#1a1a1a;text-align:center;text-decoration:none;transition:background .15s}
.btn:hover{background:#333}
</style>
<a class="btn" href="{VIEW_URL}" target="_blank">Open your Daily Brief in xTiles</a>
```

---

## How to behave

- **Never show a role/tools survey.** `role:`, `used_connectors:`, and `additional:` are always already in the incoming message.
- **Never trust the questionnaire alone for connection status.** Probe live, every time (step 2).
- **Never block on a connector.** Every missing connector has an explicit, visible way to skip it in the step 2 question — never an implicit "leave it unselected" — and answering always proceeds regardless.
- **Named connectors that are really one bundled connector (e.g. Outlook Mail / Outlook Calendar / Teams under Microsoft 365 — table in step 2, point 3) are probed, offered, and resolved once, never once per name.**
- **Always send the step 2 status line, even when nothing needs asking.** If every named connector already works, still confirm what's connected in one short line before moving on (step 2, point 5) — silence there is what makes the flow feel like it skipped past the user.
- **Never gate the real connect flow behind a question the user has to answer first.** Call `mcp__mcp-registry__suggest_connectors` proactively the moment there's something missing (step 2, point 6) — the native connect buttons render immediately, in the same turn as the question.
- **Never end a run with nothing in it.** Zero usable connectors, or an explicit `additional: news` request, triggers Today News (step 3) — never ship an empty digest. **Step 3 is a mandatory checkpoint on every path, first run or recurring** — never route directly from step 1 or step 2 to step 4 and skip it, even when it turns out to be a no-op.
- **If `other` was named, actively look for extra connectors beyond what was listed** (step 2, point 4) — offer them as an "Add" option, not "Connect," since they're already usable. This is what "other" is for; don't let it go unanswered.
- **Never wait for approval before writing.** Write directly to xTiles once step 4's fetch completes — first run or recurring alike. There is no preview and no approval gate anywhere in this skill.
- **That "no approval gate" is about the write, not about the questions.** Every `AskUserQuestion` in steps 2, 6, and 7 must actually wait for the user's answer before the next step runs — don't let "no approval needed to generate the brief" bleed into skipping or auto-advancing past an interactive question.
- **Before scheduling, always run a fresh connected-but-unused discovery pass and ask if the user wants to add anything else** (step 6) — don't rely solely on step 2's `other` candidates; if the pass finds nothing, fall back to role-based suggestions instead of a bare "No, that's enough." Fold any addition into the resolved set first, then create the recurring config once, already final.
- **Never dump the brief's content as plain text in chat.** The tiles in xTiles are the only deliverable — chat only ever carries the short intro line, the Connector-check question, the write confirmation + CTA button, the Schedule question, and the Related-workflows question.
- **A distinct tool is always a distinct tile, never merged by category** (step 4) — two calendars, or Gmail and Reclaim, never share a tile.
- **Never let a third-party connector's own preview render in chat** (step 4) — its data feeds your tile, it never appears as its own card.
- **Check `xtiles_list_tasks` before writing any task** (step 5) — never recreate an already-open action item from yesterday or today.
- **Every `<task>` carries `assignee`, always** (step 5) — the current xTiles user's own email, resolved via `xtiles_get_current_user`, every run, no exceptions.
- Never put example names, events, or messages into a written tile — only real data.
- **Every interactive moment** (the Connector check, Schedule — up to two sequential calls, step 6 — and Related workflows) uses `AskUserQuestion`, never `show_widget` or any HTML form. **The single exception is the CTA button after the write (step 5)** — non-interactive, no choices, the only `show_widget` call in this skill.
- If context is missing — ask, don't guess.
- Real data always beats placeholders.
- Daily is the only period. If asked for Weekly or Monthly, say only Daily is supported and offer a Daily page instead.
- Match the user's language, adapt if they switch. Every label in this file is an English (or transliterated) placeholder — translate it, and never carry over a language from an example.
- **Gmail follow-through (mark-as-read) is mandatory whenever Gmail is in the resolved set** — not a nice-to-have, and not conditional on anyone watching chat: it runs on a silent recurring run exactly the same as a manual one. Never draft or send emails on the user's behalf.
- **A tile for a connector in the resolved set is never silently blank.** If it returned nothing, write a one-line placeholder instead of omitting the tile.
- **No self-tuning cycle.** The digest writes content tiles only — never a "Tune your digest" or feedback-checkbox tile.
- **A recurring run always re-invokes this same skill** — never hand off to a different skill for the daily digest. The whole point of resolving everything once is that the recurring run can skip straight to the fetch.
- **Never force one connector's grouping onto another.** Email's urgency split, Slack's mentions/topics split, and Calendar's schedule-shape are three genuinely different structures because the data is genuinely different — decide the shape from what a connector actually returns, every time (step 4).
- **Never treat "how many tiles" as fixed per connector either.** A quiet day keeps Email or Slack in one combined tile; real volume is what earns a split into several — decide that from the actual pull, every time, not from what a connector "usually" gets.
- **The Connector-check question asks at most once, right after the first message, on a first run only.** Never repeat it automatically on a recurring run — silently re-probe `skipped:` instead, and only mention a newly-connected tool once, briefly, if it succeeds.
- **A missing scheduling tool is not a dead end.** If `anthropic-skills:schedule` or `mcp__scheduled-tasks__create-scheduled-tasks` isn't available, say so in one line and still continue straight to step 7 (Related workflows) — the tiles are already written either way, and the run is not complete without the closing question.
