---
name: brief-onboarding
description: >
  Use immediately after a user finishes the xTiles onboarding questionnaire —
  builds their first Daily planner **preview** from their role and the tools
  they said they use, so they see the value of the service before doing
  anything themselves. This same skill also serves every **recurring** run
  once the user schedules it — there is no separate daily-digest skill to
  hand off to.

  Entry data on a first run (already known, never re-asked with a survey):
  `role:` — the role from the questionnaire; `used_connectors:` — the tools
  the user said they use there (may include a custom name, or `other`).
  **Connection status is never handed to this skill as data — it determines
  that itself**, with a lightweight live probe per named connector (see step
  2). Gmail and Calendar are probed first, as the highest-value connectors.

  Setup triggers: "start onboarding preview", "show me what my Daily could
  look like", "onboarding welcome digest", "first-run preview",
  "onboard a new xTiles user into the Planner".

  Environment: this is the Claude / Cowork variant — it renders `show_widget`
  and `AskUserQuestion`. In ChatGPT Work, where every form is an inline
  `ask_user_input` / `genui` surface and `show_widget` does not exist, use
  `brief-onboarding-with-gpt` instead.

  Environment triggers: "Brief Onboarding in Claude", "the Claude version",
  "Claude Onboarding Preview".
allowed-tools: >
  mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner,
  mcp__xtiles__xtiles_create_notification,
  mcp__xtiles__xtiles_get_user_timezone,
  mcp__xtiles__xtiles_get_workflow,
  mcp__xtiles__xtiles_get_page_layout,
  mcp__xtiles__xtiles_set_page_layout,
  WebSearch,
  WebFetch,
  mcp__mcp-registry__suggest_connectors,
  anthropic-skills:schedule,
  mcp__scheduled-tasks__create-scheduled-tasks
---

# xTiles Onboarding — First & Recurring Daily Preview

## Seven principles

1. **Never re-ask what onboarding already answered.** Role and the tools the user said they use come in as `role:` and `used_connectors:` — never show a role/tools survey.
2. **Never trust a name as proof of connection.** Whether something is actually connected is never handed to this skill as data — it's determined live, per connector, with a lightweight probe (step 2). A tool the user *said* they use may not be connected yet, or may have been connected since.
3. **Never block on a connector.** Every check that could ask the user to connect something must also offer to skip it, and a way to proceed with nothing resolved at all — see step 2.
4. **Never end a run with nothing to show.** If zero connectors end up usable, don't ship an empty digest — research and build a News tile instead (step 3), so the user gets real value from the very first run.
5. **Real data, not placeholders.** Pull from connectors (or the web, for the News fallback) before the preview so the user sees live content, never invented names, events, or messages.
6. **Every write is followed by the layout pass.** The moment tiles are created, re-lay them out into a justified grid via the shared `tile-layout` workflow — automatically, before the CTA, never skipped.
7. **The only deliverable is tiles written to xTiles.** A run — first or recurring — is complete only when the tiles are in xTiles and the full post-write sequence has run: layout pass → Gmail follow-through (if applicable) → CTA button → schedule widget → related-workflows question. **On a silent recurring run, only the layout pass, Gmail follow-through, and the opt-in notification still happen** — the CTA widget, schedule widget, and related-workflows question are all skipped (see step 1 and step 8).

---

## Algorithm

**Period is always Daily.** Never ask which period to set up.

### 1. Entry — parse the config

Two ways this skill starts:

- **First run.** The incoming message carries `role:` and `used_connectors:` only — no resolved connector list yet. This is always right after the onboarding questionnaire. **Before anything else, send one short, plain-language line of context** — e.g. "Setting up your first Daily preview for a {role}, based on the tools you said you use." — before any silent probing and before the Connector-check widget, if one turns out to be needed. Never let the very first thing the user sees be a bare widget with no context. Then go to **step 2**.
- **Recurring run.** The incoming message instead carries the **full config this skill itself wrote at the end of a previous run** (step 8) — `role:`, `tools:` (the already-resolved set), `skipped:` (connectors that weren't connected last time, if any), and `notify:`. Its presence (specifically `tools:` alongside `role:`) is the signal — there is no separate scheduled-run skill to hand off to. This runs silently — no intro line, nobody is watching chat. **First, silently re-probe every connector in `skipped:`** (same lightweight probe as step 2, no widget): if one now succeeds, fold it into today's resolved set and mention it once, briefly, in today's preview or notification ("Gmail just connected — added to today's brief"). This is the only re-check that ever happens on a recurring run. Then **go to step 3** (a mandatory checkpoint — it triggers the News fallback only if the resolved set is still empty, otherwise it's a no-op) **and then step 4 (Silent data fetch)**, using the resulting resolved set.

  **What actually runs after the write on a recurring run — spelled out exactly, nothing implied:** the layout pass always runs; Gmail follow-through (step 7's item 5) always runs if Gmail is in the resolved set, whether or not anyone is watching chat — it's inbox hygiene, not a chat-visible action; and — only if `notify:true` — the notification (step 8) fires. **The CTA widget, the schedule widget, and the related-workflows question never fire on a recurring run** — those three, and only those three, are what "silent" excludes.

### 2. Connector check — detect it yourself, then connect or skip

**There is no fixed catalog of connectors in this skill.** `used_connectors` can name anything — Gmail, Calendar, Slack, a tool invented after this file was written, or `other` with a name typed by the user. Treat every name the same way:

1. For each connector in `used_connectors`, make one lightweight, read-only probe call using whatever MCP tool that connector exposes (a minimal list/search call, never a write). A response with no auth error means it's connected right now; an auth error or a missing tool means it isn't. **This probe result — not the questionnaire answer — is the only source of truth for "connected."**
2. **Gmail and Calendar are probed first**, since they tend to carry the richest everyday signal. Everything else follows.
3. For an unfamiliar connector name, look for MCP tools whose namespace matches it (e.g. a connector called `{Name}` would expose `mcp__claude_ai_{Name}__*` tools) and use the least invasive read call available. If no matching tool exists at all, treat it as not connected — it becomes a candidate to connect natively (see **How to connect connectors**) or to skip.
4. **If `other` is in `used_connectors`** — it's a signal the user's real stack is bigger than what they listed. Beyond probing the named connectors, look at what other connector tools this session actually has available (its own tool/capability list) and quickly probe any that weren't named. Anything that responds successfully becomes an **Add {name}** candidate in the widget below — distinct from **Connect {name}**, since it's already usable and just needs opting in, no auth flow required.
5. **If `used_connectors` is non-empty and every named connector's probe succeeds, and step 4 above found no extra candidates to offer — skip straight to step 3** (a no-op there, since the resolved set isn't empty) **and then step 4.** No widget, no question. This is the best-case path. **If `used_connectors` was empty from the very start, this is not that case** — zero probes is not zero failures, treat it exactly like an empty resolved set and go to step 3.
6. **Otherwise** (something failed to connect, or there's an extra candidate to offer):
   - **Immediately call `mcp__mcp-registry__suggest_connectors`**, passing the names of every connector whose probe failed. This renders real, native connect buttons directly in the Cowork UI, right away — the platform already knows how to offer a connect flow the instant it's asked to; **never gate that behind a custom "Connect" button of our own that the user has to click first just to unlock the real one.** Do this proactively, in the same turn, before or alongside the widget below.
   - **In that same turn**, show the **Connector check widget** (below) alongside it. It has two things the native buttons don't: an explicit **"✓ Already connected"** list (so the user isn't left guessing what's already fine — Gmail and Calendar first if present), and an explicit, visible **Skip** control on every still-missing connector — a real button/toggle labeled "Skip," never just "leave it unselected and hope that reads as skip." Any extra candidate from point 4 gets an **Add** toggle instead. A single **Continue** always proceeds with whatever's resolved at that moment. **Never require every connector to be resolved before continuing.**
   - Say, in one line before showing either: "I've also opened the connect flow for {missing connectors} above — connect what you want there, or skip below and hit Continue."
7. When the user connects something through the native buttons from `suggest_connectors`, follow **How to connect connectors**'s Done-widget step to confirm and resume — never restart the whole check. Clicking **Skip** just marks that connector done-for-this-run, no further nagging. Clicking **Add** needs no connect flow at all — it's already usable, just fold it straight into the resolved set.
8. **On Continue, re-probe anything still marked "not yet connected" that wasn't explicitly skipped** — cheap, and it's the only way to catch a connection the user just made through the native buttons from `suggest_connectors`, since that flow doesn't report back into our widget directly. The **resolved set** = every connector whose probe succeeded (original pass or this re-probe), plus any added, minus anything explicitly skipped. **Track the skipped list too** — carried forward as `skipped:` into the schedule config in step 8, so a future recurring run can quietly notice if one of them gets connected later (see step 1) without ever asking again. `xTiles` itself is required, not optional — if it's not connected, this skill isn't reachable at all; connect it first.
9. In Claude Code (no Cowork), do the same thing as plain text: name which of the connectors mentioned actually responded, mention any extra connectors found via point 4, offer "connect"/"add"/"skip" as appropriate, and an explicit "or say 'skip all' to continue now."

**When to ask — and when never to ask again automatically.** This widget fires **at most once per run**, right after the intro line in step 1, and only on a first run. It is not a recurring nag:
- On a **recurring run**, this widget never appears at all — step 1 handles it with a silent re-probe of `skipped:` instead.
- The user can always trigger a fresh check by asking directly at any time ("connect my Slack now") — that re-enters this step for just the named connector, regardless of run mode.

**No content-preference questions.** Every connector in the resolved set contributes its own default content (step 4) — the user can still ask to change anything at the preview step.

**Two worked examples of the probe — not an exhaustive list, the pattern is the same for anything else the user names:**
- **Gmail** — call `mcp__claude_ai_Gmail__list_labels` (cheap, no query needed). Success = connected.
- **Calendar** — call `mcp__claude_ai_Google_Calendar__list_events` with `maxResults:1`. Success = connected.

### 3. Fallback check — a News tile when nothing is usable

**This is a mandatory checkpoint, not an optional detour — every path from step 1 or step 2 passes through here, on a first run and on a recurring run alike. It is never valid to route straight from step 1 or step 2 to step 4.**

Check the resolved set:
- **Non-empty** — nothing to do here. Continue straight to step 4.
- **Empty** — whether because `used_connectors` was empty from the very start, every named connector's probe failed, or the user skipped everything in the widget — don't ship an empty run. Build a News tile instead, right here, before step 4:
  1. From `role:`, infer 2–4 topic categories this person would plausibly care about right now (the same judgment `today-news` uses — e.g. a Product Manager cares about product/UX trends, competitor moves, and AI tooling news; an Engineer cares about dev tooling, major tech news, and security). If nothing about the role narrows it down, default to broadly useful categories: industry news, productivity/tools, and a general "worth knowing" pick.
  2. Use `WebSearch` and `WebFetch` to find real, current items from the last 24–48 hours per category, verified against reputable sources. **Never invent an item, a date, or a link.** If a category genuinely yields nothing, drop that category rather than force it; if literally every category comes back empty, say so plainly in the preview instead of writing an empty tile.
  3. Build **one** tile, `### 📰 News for you`, with one labeled sub-section per category and 2–3 real items each — one line per item, source linked inline.
  4. This is a fallback, not a permanent feature — the moment even one connector is usable (this run or a future one), skip it entirely and go back to normal per-connector tiles.

---

### 4. Silent data fetch

**Silently, without messaging the user**, pull fresh data from every connector in the resolved set, and — if step 3 triggered — research the News tile instead.

**There is no single grouping that fits every connector — the right shape follows the nature of the data itself, never a template repeated for each one.** Before building a tile, ask what *this specific kind of data* actually needs, not "which of the usual three buckets does this go in."

**And separately — whether that shape becomes one tile or several is a question of volume, not a fixed rule per connector.** A handful of items reads fine inside one tile with labeled internal sections; it's genuine volume in one of those sections that earns it a tile of its own. Never split into several thin, mostly-empty tiles just because a connector "usually" gets split, and never cram a genuinely large volume into one dense, hard-to-scan tile either — let what was actually pulled decide, each time.

- **Email arrives as a firehose that needs triage — the natural question is "do I have to act on this."** That's why it splits by urgency: 🔴 needs a concrete next step, 🟡 informational only, ⚪ automated noise. **With real volume in each bucket**, this becomes three tiles (step 7): `### 📩 Email — Action Points` (🔴, plus real `<task>`s), `### 📩 Email — Key People` (🟡, grouped by *sender* — the "who's actually writing to you" view, a completely different axis from urgency), `### 📩 Email — Noise` (⚪, one rollup line, never itemized). **With only a handful of relevant emails**, keep it all in one `### 📩 Email` tile instead, with the same three labeled blocks inside it — same content, less scaffolding. Tone either way: retell the email in second person, action + consequence, don't copy the subject line — "Google shut down your ad account yesterday — log in and appeal, the window is limited," not "Your account closed."
- **Slack already arrives grouped — by channel and by thread — so urgency isn't the useful axis there.** The real question is "was this addressed to me, or is it ambient discussion I can skim." **With enough real activity**, that becomes two tiles: mentions/DMs that need a reply as real `<task>`s, and a short topics rollup for what's being discussed in the channels you follow. **On a quiet Slack day**, fold both into one `### 💬 Slack` tile instead, with the same two labeled sections inside it. Either way, never reshuffle it into an urgency/FYI/noise split — that would throw away the channel structure that's the whole point of Slack, regardless of how many tiles it ends up as.
- **Calendar isn't a triage problem at all — there's no "noise" in someone's schedule, and a single day only ever needs one tile.** The natural shape is chronological and forward-looking: what does today look like, where's the free time, what's worth preparing for. One `### 📅 Workload` tile — never a second copy of the schedule the user can already see, and never split by volume the way Email or Slack might be. Compute event count, hours occupied, and the longest free focus window; write one concrete 🎯 focus-recommendation sentence; for each event, a one-sentence agenda (from the event description, or the most recent related email/meeting note — never invented) and, only where genuinely implied, one prep `<task>`. Collect anomalies (overlaps, back-to-back runs, meetings with no agenda) at the bottom, never inline.

**Apply the same kind of thinking to anything else in the resolved set** — Linear, Google Drive, GitHub, or a connector this file has never heard of. Read what it actually returns, decide the natural axis (urgency, person, project, time, or something else entirely), and only split it into more than one tile when there's genuinely enough volume on that axis to earn separate scanning — never reuse Email's or Calendar's exact shape, or their tile count, for a different kind of data just because it's already written down here.

Never fabricate a category that has nothing in it for that connector — omit an empty block rather than write "none" three times over. A tile from a connector that returned genuinely nothing still gets written, with one line saying so — its total absence looks like a bug, not a quiet day.

**If Slack is in the resolved set and the user hasn't named channels**, pick a small, sensible set yourself before reading: favor channels the user has posted or been mentioned in recently, plus any that are obviously company-wide (general/announcements), capped at around five. This is a judgment call, not a fixed algorithm — the goal is real signal, not exhaustive coverage; the user can always ask to add more later.

**If Gmail is in the resolved set**, also check for unread newsletters (senders like `@substack.com`, `@beehiiv.com`, or ones the user is clearly subscribed to) and — if any are found — collect them for a single `### 📧 Newsletters` tile (one line per publication, title as the hyperlink, no separate "Open" link). Skip the tile entirely if none are found.

**Cross-source overlap.** If the same real ask clearly shows up in two different connectors (e.g. a Slack message and a follow-up email about the same specific thing), don't produce two separate action items for it — treat one as primary, and add a short cross-reference on the other's line (e.g. "also emailed"). Only merge when it's genuinely the same ask; an unrelated message from the same person is not a duplicate.

Use only real data. Never invent names, events, messages, or links. If a connector call fails outright (error, timeout, 401) — record that as a failure, surfaced explicitly in the preview, never silently written as "no data."

---

### 5. Preview — show content in a widget

**Show it with the Preview widget HTML** (see below) via `show_widget` — never as a plain-text chat dump. Real content with real data, never "(TBD)" placeholders. Build it fully dynamically: one `.section` card per tile that's actually going to be written (whatever the resolved set and step 4 produced, in the order they were fetched), following the injection pattern in the widget's own comment block.

**Rules:**
- One section per tile that will actually be written — never a section for a connector that isn't in the resolved set.
- If a connector returned no data, say so exactly ("No unread emails", "No updates today") — never skip the section silently.
- If a connector call failed, say so exactly ("Could not fetch Calendar — connector error"), never "No data."
- No placeholder names, example events, or invented data — ever.
- If Gmail is in the resolved set, add one line inside the widget noting that approval also marks ⚪ Noise/newsletter threads as read (see step 7's Gmail follow-through).
- After showing the widget, **stop and wait**. Nothing is written yet.

### 6. Approval

**Mandatory. Never skip this step — there is nothing extra to do here.** The Preview widget's own **Looks good — create it** / **Change something** / **Cancel** buttons *are* the approval. Never call a separate `AskUserQuestion` or a second widget for this.

Do not call `xtiles_create_tiles_from_markdown_in_my_planner` until the user explicitly clicks **"Looks good — create it."**

If the user asks for a change — clarify exactly what if it's ambiguous, update only that part, then show the Preview widget again with the revised content.

### 7. Write to xTiles

**Only after explicit approval** (or immediately, on a recurring run — step 1).

Tool: `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`
- `period`: "day"
- `date`: current date in ISO 8601

**Write all sections in a single call.** Combine every tile from step 4 into one markdown string and call the tool once — never split into separate calls per connector.

**Write content tiles only** — exactly what was approved in the preview. No date/header tile, no meta or self-tuning tiles.

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

<task dueDate="2026-08-11">Restore the Google ad account</task>

<task priority="high" dueDate="2026-08-10">Sign the contract</task>
```

- Never `- [ ]` for an action item.
- One `<task>` per line, blank line between each; never nested inside a list item; never carries a link.
- `dueDate="YYYY-MM-DD"` — always set, defaulting to today; use a later date only when the source states a real deadline.
- `priority` only when the source itself signals urgency — omit otherwise; at most a third of a run's tasks should be `high`.
- Never `completed="true"`.

**Links are always inline hyperlinks inside a sentence — never a link alone on its own line.** `… → [Open email](url)` renders as a normal hyperlink; a line containing *only* a link renders as a big block-link card instead, which is never what's wanted here.

**Worked tile examples, in full markdown — the same two connectors as step 4's Email and Calendar, each shown in its split (high-volume) case. Everything else follows step 4's reasoning (the shape *and the tile count* follow the data), never a layout copied from these:**

```
### 📩 Email — Action Points
@colorSize: LIGHTER
@color: SAIL

- [Poke-style description — action + consequence, second person] → [Open email](url)

---

**Action items**

<task dueDate="2026-08-11">Restore the Google ad account</task>
```

```
### 📅 Workload
@colorSize: LIGHTER
@color: BERMUDA

**N events · ~X h occupied · longest focus window HH:MM–HH:MM (X h)**

🎯 [Focus recommendation — one concrete sentence]

**HH:MM–HH:MM · Meeting name** — Participant · [Google Meet](url)

📋 [Agenda — one sentence, from a real source]

<task dueDate="2026-08-11">[Prep task, only if genuinely implied]</task>
```

`### 📩 Email — Key People` (🟡, grouped by sender) and `### 📩 Email — Noise` (⚪, one rollup line) follow the same pattern as Action Points, scoped to their own bucket. `### 📧 Newsletters` (if any were found) is one line per publication: `**[Publication](url)** — one-line summary.` `### 📰 News for you` (the step 3 fallback, if it ran) is one labeled sub-section per category with 2–3 linked items each.

**The combined (low-volume) case** uses the exact same blocks, just inside one tile instead of several:

```
### 📩 Email
@colorSize: LIGHTER
@color: SAIL

🔴 **Needs action**

- [Poke-style description] → [Open email](url)

---

**Action items**

<task dueDate="2026-08-11">Restore the Google ad account</task>

🟡 **Key people**

**Name (context)**

- [One-line item — no link]

⚪ **Noise**

- N notifications — nothing urgent
```

Same idea for Slack — a quiet day gets one `### 💬 Slack` tile with a `**Mentions**` block (real `<task>`s) and a `**Topics**` block underneath, instead of two separate tiles.

**Every run creates fresh tiles — this skill has no way to update an existing tile.** Write the full approved set every time, even if a same-titled tile already exists on the page from an earlier run today.

**After each successful write — run in order, no exceptions:**

1. Write `✅ Daily created.`
2. **Layout pass — mandatory, silent, never asked about.** Read `view_id` and `tile_ids` from the write response (never re-derive them). Call `mcp__xtiles__xtiles_get_workflow` with id `tile-layout` and follow it exactly, **passing `tile_ids` as its "added tiles" and the markdown just written as their content** — those are required inputs the workflow itself expects, not optional context — plus these **layout hints**: 1–4 tiles, default 2 per row, give a heavy tile its own full-width row. This workflow is the one that actually calls `xtiles_get_page_layout`/`xtiles_set_page_layout` — skipping the input handoff here is why it can silently do nothing.
3. **Non-scheduled runs only:** call `show_widget` with the **CTA widget HTML**, using the `resource_url` of the first tile in the write response (fall back to the page URL only if that's missing).
4. **Non-scheduled runs only:** immediately call `show_widget` with the **Schedule widget HTML**. Never substitute `AskUserQuestion` here.
5. **If Gmail is in the resolved set — mandatory, silent, every run:** mark every ⚪ Noise and newsletter thread as read with `mcp__claude_ai_Gmail__unlabel_thread` (remove `UNREAD`). Never touch 🔴 or 🟡 threads, and never draft or send anything on the user's behalf — this only marks threads read.
6. **Recurring runs only, and only if the config's `notify:` is `true`:** call `mcp__xtiles__xtiles_create_notification` — `url` the tile-focused deep link, `text` the fixed string `"Your Daily Digest is ready — see what matters today in 2 min."` translated into the user's language (never customized beyond translation), `agent_source` `"Claude"`.

### 8. Schedule (optional)

The schedule widget is shown in step 7. This step handles the response. **A schedule set up here re-invokes this same skill, `brief-onboarding`, every morning — there is no separate daily-digest skill.** The whole point of resolving everything in steps 1–4 once is that the recurring run can skip straight past all of it.

In Claude Code (no Cowork): ask inline after writing — "Want me to run this every morning automatically? What time? (default: 9:00 AM)"

- If **"Yes, schedule it"** — invoke `anthropic-skills:schedule`, then `mcp__scheduled-tasks__create-scheduled-tasks`:
  - **`prompt`**: the full recurring-run config, assembled from this run's resolved state —
    ```
    Run brief-onboarding — role: {role} · tools: {resolved connector set} · skipped: {connectors whose probe failed or that the user skipped, or "none"} · notify: {true/false} (if true, call xtiles_create_notification at the very end of that run — mandatory, do not skip) · schedule: daily-{HH:MM} days:{days}
    ```
    **No `daily_content:` field** — step 4 always re-derives each connector's content fresh from what it actually returns that day, so a snapshot from setup time would only ever be stale; there's nothing to gain from persisting one. This is exactly the config step 1 looks for to detect a recurring run — assembling it correctly here is what lets tomorrow's run skip the connector check entirely, while `skipped:` still lets it quietly notice a later connection (step 1).
  - **`schedule`**: cron built from the widget the same way as any other scheduled task — `HH:MM days:1-5` → `M H * * 1-5`, `HH:MM days:*` → `M H * * *`. Default `0 9 * * 1-5`.
  - **`timezone`**: from `mcp__xtiles__xtiles_get_user_timezone`.
  - **If `notify:true`** — the digest already on the page right now is worth notifying about too; call `mcp__xtiles__xtiles_create_notification` immediately with the same fixed text as step 7's item 6.
  - Confirm: "Done — your Daily will be ready in xTiles every morning at [time]." (append ", and I'll notify you in xTiles each time" if `notify:true`). Don't show the CTA widget again. **Continue to step 9 in the same turn.**
- If **"No, thanks"** — acknowledge briefly, **continue to step 9 in the same turn.**
- **If `anthropic-skills:schedule` or `mcp__scheduled-tasks__create-scheduled-tasks` genuinely isn't available in this environment — this is not a dead end.** Say so in one plain line ("I can't set up automatic scheduling in this environment — but your Daily is ready, and I'll build a fresh one anytime you ask.") and, if `notify:true` was requested, still send today's notification per the bullet above — that part never depended on the schedule actually being created. **Then continue to step 9 in the same turn, exactly as if the user had said "No, thanks."** A missing scheduling tool skips the schedule itself, never the mandatory closing step.

### 9. Related workflows

**Mandatory closing step of every manual run** (skip only on a silent recurring run, which ends after step 7's notification). Ask via `AskUserQuestion` (single select): "Want to set up anything else on xTiles?"
- 🌙 Evening Reflection — an end-of-day synthesis seeded for tomorrow
- 📰 Today News — a daily news digest on topics you care about
- 📊 Weekly Review — a weekly summary of what moved forward this week
- Nothing else, thanks

On selection, send the exact matching phrase to hand off (never run it yourself):
- Evening Reflection → `Set workflow of Evening Reflection (evening-reflection) on xTiles MCP`
- Today News → `Set workflow of Today News (today-news) on xTiles MCP`
- Weekly Review → `Set workflow of Weekly Review (weekly-review) on xTiles MCP`
- "Nothing else" — acknowledge briefly and stop.

---

## How to connect connectors

Do not send the user to settings manually and do not give a URL to follow. Call `mcp__mcp-registry__suggest_connectors` — it renders interactive connect buttons directly in the Cowork UI.

**Flow:**
1. Call `mcp__mcp-registry__suggest_connectors` passing the name(s) of the connector(s) to connect — the user may pick more than one at once in the widget.
2. Show the **Done widget** (below) directly under the connector form.
3. The user clicks the connect button(s) — the auth flow runs natively for each. When finished, they click **"Done."**
4. Confirm briefly and resume the flow from where it was interrupted.

**Fallback when the registry is unavailable (free plans).** Explain that connecting more tools requires upgrading the plan / enabling connectors, and proceed with whatever's already usable — the run still produces a real preview.

**This connect flow is always optional here.** Every connector offered in step 2 already has a skip path. Never let a stalled or failed connect attempt block the run — if it doesn't complete, drop that connector and continue.

---

## Preview widget HTML

Show via `show_widget` in step 5, on every non-scheduled run. Build fully dynamically: inject **one `.section` card per tile step 4 actually produced**, in the order fetched, with real, already-composed content into `#digest`, following the injection pattern below. Never inject a section for a connector that isn't in the resolved set.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:16px;background:#f8f8f8;color:#1a1a1a}
.wrap{max-width:560px;margin:0 auto;background:#fff;border-radius:16px;padding:24px;box-shadow:0 2px 12px rgba(0,0,0,.08)}
.header{display:flex;align-items:center;justify-content:space-between;margin-bottom:16px}
h2{font-size:16px;font-weight:700}
.cnt{font-size:12px;color:#888;background:#f5f5f5;padding:2px 9px;border-radius:20px}
.scroll{max-height:420px;overflow-y:auto;margin-bottom:16px;display:flex;flex-direction:column;gap:12px}
.section{border-radius:12px;padding:14px;background:#f7f7f7}
.section-head{font-size:12px;font-weight:700;color:#1a1a1a;margin-bottom:10px;padding-bottom:8px;border-bottom:1px solid rgba(0,0,0,.07)}
.sub-head{font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.4px;color:#888;margin:10px 0 6px}
.sub-head:first-of-type{margin-top:0}
.item{font-size:12px;color:#333;line-height:1.5;margin-bottom:6px}
.item:last-child{margin-bottom:0}
.item-title{font-weight:600;color:#1a1a1a}
.feedback{display:none;margin-bottom:12px}
textarea{width:100%;padding:10px;border:1.5px solid #e0e0e0;border-radius:10px;font-size:13px;outline:none;resize:none;height:68px;font-family:inherit}
.btns{display:flex;flex-direction:column;gap:8px}
.btn{width:100%;padding:11px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:all .15s}
.btn-save{background:#eef6ff;color:#1a5fb4;border:1.5px solid #c3d9f7}
.btn-save:hover{background:#deeeff;border-color:#a8c8f0}
.btn-edit{background:#f0f0f0;color:#444}
.btn-edit:hover{background:#e0e0e0}
.btn-cancel{background:transparent;color:#bbb;font-size:13px;font-weight:400}
</style>
<div class="wrap">
  <div class="header">
    <h2>✨ Your Daily, previewed</h2>
    <span class="cnt" id="cnt">[actual date]</span>
  </div>
  <div class="scroll" id="digest">

    <!--
    INJECTION PATTERN — one .section per tile actually produced, real content only. Examples:

    <div class="section">
      <div class="section-head">📩 Email — Action Points</div>
      <div class="item">Poke-style description, second person, action + consequence.</div>
      <div class="sub-head">Action items</div>
      <div class="item">Verb-first task.</div>
    </div>

    <div class="section">
      <div class="section-head">📅 Workload</div>
      <div class="item">N events · ~X h occupied · longest focus window HH:MM–HH:MM.</div>
      <div class="item">🎯 Focus recommendation — one concrete sentence.</div>
      <div class="item"><span class="item-title">HH:MM–HH:MM · Meeting name</span> — Participants.<br>📋 Agenda sentence.</div>
    </div>

    <div class="section">
      <div class="section-head">📰 News for you</div>
      <div class="sub-head">Category name</div>
      <div class="item">One-sentence item — source linked inline.</div>
    </div>
    -->

  </div>
  <div class="feedback" id="fb">
    <textarea id="fbtext" placeholder="What should I change?"></textarea>
  </div>
  <div class="btns">
    <button class="btn btn-save" id="btn-save" onclick="doSave()">Looks good — create it</button>
    <button class="btn btn-edit" id="btn-edit" onclick="toggleEdit()">Change something</button>
    <button class="btn btn-cancel" id="btn-cancel" onclick="cancelIt()">Cancel</button>
  </div>
</div>
<script>
var editing=false;
function collapse(msg){document.querySelector('.btns').innerHTML='<p style="font-size:13px;color:#aaa;text-align:center;padding:4px 0">'+msg+'</p>';}
function doSave(){
  var fb=document.getElementById('fbtext').value.trim();
  collapse(fb?'⏳ Applying & creating…':'⏳ Creating…');
  sendPrompt(fb?'Apply this change then save: '+fb:'Looks good — create it');
}
function cancelIt(){collapse('✓ Cancelled');sendPrompt('Cancel');}
function toggleEdit(){
  editing=!editing;
  document.getElementById('fb').style.display=editing?'block':'none';
  document.getElementById('btn-save').textContent=editing?'Apply & Create':'Looks good — create it';
  document.getElementById('btn-edit').textContent=editing?'Never mind':'Change something';
}
</script>
```

---

## Done widget HTML

Show via `show_widget` immediately after calling `mcp__mcp-registry__suggest_connectors`. The user clicks **Done** when finished connecting — this resumes the flow.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:16px;background:transparent}
.btn{width:100%;padding:11px 20px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;background:#1a1a1a;color:#fff;transition:background .15s}
.btn:hover{background:#333}
</style>
<button class="btn" id="btn-done" onclick="doneIt()">✓ Done</button>
<script>
function doneIt(){var b=document.getElementById('btn-done');b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';b.textContent='⏳…';sendPrompt('Done — connectors connected, continue the flow');}
</script>
```

---

## Connector check widget HTML

Show via `show_widget` in step 2, **only when at least one probe failed, or step 2's point 4 found an extra candidate** — nothing to show when every named connector is already connected and there's nothing extra to offer. Show it **alongside** the native connect buttons from `mcp__mcp-registry__suggest_connectors` (called in the same turn, per step 2 point 6) — this widget never offers its own "Connect" action, it only shows what's already connected and lets the user explicitly skip what isn't. Build the lists dynamically: `#connected` from every connector whose probe succeeded, `#missing` from connectors whose probe failed (Gmail and Calendar first, if present), `#extras` from anything found via the `other` self-check (step 2, point 4) — omit any section's heading entirely when it has nothing in it.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:#f8f8f8;color:#1a1a1a}
.wrap{max-width:480px;margin:0 auto;background:#fff;border-radius:16px;padding:26px;box-shadow:0 2px 12px rgba(0,0,0,.08)}
h2{font-size:17px;font-weight:700;margin-bottom:4px}
.sub{font-size:12px;color:#888;margin-bottom:18px;line-height:1.5}
.sec-title{font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.4px;color:#888;margin:14px 0 8px}
.sec-title:first-of-type{margin-top:0}
.tags{display:flex;flex-wrap:wrap;gap:8px}
.tag{padding:6px 12px;border-radius:20px;background:#eaf7ee;color:#1a7a3c;font-size:13px;font-weight:600}
.rows{display:flex;flex-direction:column;gap:8px}
.row{display:flex;align-items:center;justify-content:space-between;padding:9px 13px;border-radius:10px;border:1.5px solid #e0e0e0;font-size:13px;transition:all .15s}
.row.skipped{opacity:.5}
.row.skipped span{text-decoration:line-through}
.skip-btn{padding:5px 12px;border-radius:20px;border:1.5px solid #e0e0e0;background:#fff;font-size:12px;font-weight:600;cursor:pointer;color:#555;transition:all .15s}
.row.skipped .skip-btn{background:#1a1a1a;color:#fff;border-color:#1a1a1a}
.cards{display:flex;flex-wrap:wrap;gap:8px}
.card{display:flex;align-items:center;gap:7px;padding:8px 14px;border-radius:20px;border:1.5px solid #e0e0e0;font-size:13px;cursor:pointer;background:#fff;user-select:none;transition:all .15s}
.card:hover{border-color:#aaa}
.card.sel{background:#1a1a1a;color:#fff;border-color:#1a1a1a}
.btn-row{display:flex;gap:10px;margin-top:22px}
.btn{padding:11px 18px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:all .15s}
.btn-p{background:#1a1a1a;color:#fff;flex:1}
.btn-p:hover{opacity:.9}
</style>
<div class="wrap">
  <h2>Here's what's connected so far</h2>
  <p class="sub">I've also opened the connect flow above for anything missing — connect what you want there, or hit Skip below and Continue.</p>

  <!-- inject only if at least one probe succeeded -->
  <div class="sec-title">✓ Already connected</div>
  <div class="tags" id="connected">
    <!-- inject: one per connector whose probe succeeded, plain info tag, e.g.
    <div class="tag">Gmail</div>
    <div class="tag">Slack</div>
    -->
  </div>

  <!-- inject only if at least one probe failed -->
  <div class="sec-title">Not yet connected</div>
  <div class="rows" id="missing">
    <!-- inject: one row per connector whose probe failed, e.g.
    <div class="row" data-v="Calendar"><span>Calendar</span><button class="skip-btn" onclick="tog(this)">Skip</button></div>
    <div class="row" data-v="Figma"><span>Figma</span><button class="skip-btn" onclick="tog(this)">Skip</button></div>
    -->
  </div>

  <!-- inject only if step 2's point 4 found extra candidates -->
  <div class="sec-title">Also add to my Daily?</div>
  <div class="cards" id="extras">
    <!-- inject: one per extra candidate found via the `other` self-check, e.g.
    <div class="card" data-kind="add" data-v="Linear" onclick="togAdd(this)">Add Linear</div>
    -->
  </div>

  <div class="btn-row">
    <button class="btn btn-p" id="btn-continue" onclick="submit()">Continue</button>
  </div>
</div>
<script>
function tog(btn){
  var row=btn.closest('.row');
  row.classList.toggle('skipped');
  btn.textContent=row.classList.contains('skipped')?'Skipped':'Skip';
}
function togAdd(el){el.classList.toggle('sel')}
function submit(){
  var b=document.getElementById('btn-continue');b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';b.textContent='⏳…';
  var skip=[],add=[];
  document.querySelectorAll('#missing .row.skipped').forEach(function(el){skip.push(el.dataset.v)});
  document.querySelectorAll('#extras .card.sel').forEach(function(el){add.push(el.dataset.v)});
  sendPrompt('Connector check — skip: '+(skip.join(', ')||'none')+' · add: '+(add.join(', ')||'none')+' · continue');
}
</script>
```

---

## CTA widget HTML

Show immediately after a successful write. Replace `{VIEW_URL}` with the tile-level link resolved in step 7.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
html,body{overflow:hidden;height:auto}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:12px;background:transparent}
.btn{display:block;width:100%;padding:12px 20px;border-radius:10px;font-size:15px;font-weight:700;color:#fff;background:#1a1a1a;text-align:center;text-decoration:none;transition:background .15s}
.btn:hover{background:#333}
</style>
<a class="btn" href="{VIEW_URL}" target="_blank">Open in xTiles →</a>
```

---

## Schedule widget HTML

Show via `show_widget` after a successful write.

```html
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:#f8f8f8;color:#1a1a1a}
.wrap{max-width:480px;margin:0 auto;background:#fff;border-radius:16px;padding:28px;box-shadow:0 2px 12px rgba(0,0,0,.08);text-align:center}
.icon{font-size:36px;margin-bottom:12px}
h2{font-size:17px;font-weight:700;margin-bottom:6px}
.sub{font-size:13px;color:#888;margin-bottom:20px;line-height:1.5}
.time-row{display:inline-flex;align-items:center;gap:8px;background:#f3f3f3;border-radius:10px;padding:8px 16px;font-size:13px;font-weight:600;color:#444;margin-bottom:16px}
.time-row select,.time-row input[type=time]{border:none;background:transparent;font-size:15px;font-weight:700;color:#1a1a1a;outline:none;cursor:pointer}
.notify-row{display:flex;align-items:center;justify-content:space-between;text-align:left;padding:12px 14px;border-radius:10px;background:#f8f8f8;border:1.5px solid #eee;cursor:pointer;user-select:none;margin-bottom:24px}
.notify-info{flex:1;margin-right:12px}
.notify-label{font-size:13px;font-weight:600}
.notify-desc{font-size:11px;color:#888;margin-top:2px;line-height:1.4}
.sw{width:40px;height:22px;background:#e0e0e0;border-radius:11px;position:relative;transition:background .2s;flex-shrink:0}
.sw.on{background:#1a1a1a}
.knob{width:18px;height:18px;background:#fff;border-radius:50%;position:absolute;top:2px;left:2px;transition:left .2s;box-shadow:0 1px 3px rgba(0,0,0,.2)}
.sw.on .knob{left:20px}
.btns{display:flex;flex-direction:column;gap:10px}
.btn{padding:11px 20px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:all .15s}
.btn-yes{background:#1a1a1a;color:#fff}
.btn-yes:hover{background:#333}
.btn-no{background:#f0f0f0;color:#555}
.btn-no:hover{background:#e0e0e0}
</style>
<div class="wrap">
  <div class="icon">⏰</div>
  <h2>Run this every morning?</h2>
  <p class="sub">I'll fetch your signals and write your Daily to xTiles automatically — no need to ask each time.</p>
  <div class="time-row">
    📅 Every
    <select id="sched-days">
      <option value="1-5" selected>Weekdays</option>
      <option value="*">Day</option>
    </select>
    at <input type="time" id="sched-time" value="09:00">
  </div>
  <div class="notify-row" onclick="togNotify()">
    <div class="notify-info">
      <div class="notify-label">🔔 Notify me in xTiles</div>
      <div class="notify-desc">Get pinged the moment each morning's digest is ready</div>
    </div>
    <div class="sw on" id="notify-sw"><div class="knob"></div></div>
  </div>
  <div class="btns">
    <button class="btn btn-yes" id="btn-yes" onclick="scheduleIt()">Yes, schedule it</button>
    <button class="btn btn-no" id="btn-no" onclick="noThanks()">No, thanks</button>
  </div>
</div>
<script>
var notify=true;
function togNotify(){notify=!notify;document.getElementById('notify-sw').classList.toggle('on',notify);}
function collapse(msg){document.querySelector('.btns').innerHTML='<p style="font-size:13px;color:#aaa;text-align:center;padding:4px 0">'+msg+'</p>';}
function scheduleIt(){
  var days=document.getElementById('sched-days').value;
  var t=document.getElementById('sched-time').value||'09:00';
  var parts=t.split(':'),h=parseInt(parts[0],10),m=parts[1];
  var label=(h%12||12)+':'+m+' '+(h>=12?'PM':'AM');
  var dLabel=days==='1-5'?'weekdays':'every day';
  collapse('⏳ Scheduling…');
  sendPrompt('Yes, schedule my daily digest at '+label+' '+dLabel+' (cron: '+t+' days:'+days+') · notify:'+notify);
}
function noThanks(){collapse('✓ Got it');sendPrompt('No schedule needed');}
</script>
```

---

## How to behave

- **Never show a role/tools survey.** `role:` and `used_connectors:` are always already in the incoming message.
- **Never trust the questionnaire alone for connection status.** Probe live, every time (step 2).
- **Never block on a connector.** Every missing connector has an explicit, visible **Skip** button in the Connector check widget — never an implicit "leave it unselected" — and Continue always works regardless.
- **Never gate the real connect flow behind a custom button of our own.** Call `mcp__mcp-registry__suggest_connectors` proactively the moment there's something missing (step 2, point 6) — the native connect buttons render immediately, in the same turn as the widget, not after an extra click.
- **Never end a run with nothing in it.** Zero usable connectors triggers the News fallback (step 3) — never ship an empty digest. **Step 3 is a mandatory checkpoint on every path, first run or recurring** — never route directly from step 1 or step 2 to step 4 and skip it, even when it turns out to be a no-op.
- **If `other` was named, actively look for extra connectors beyond what was listed** (step 2, point 4) — offer them as **Add**, not **Connect**, since they're already usable. This is what "other" is for; don't let it go unanswered.
- **Never output the preview as plain text in chat.** Always write to xTiles directly, or connect it first.
- Never create anything without preview and explicit approval on a manual run.
- Never put example names, events, or messages into the preview — only real data.
- **Every clarifying moment** (the Connector check, approval, change requests) uses `show_widget` with HTML, never `AskUserQuestion` or plain text — `AskUserQuestion` is for the Related-workflows question only.
- If context is missing — ask, don't guess.
- Real data always beats placeholders.
- Daily is the only period. If asked for Weekly or Monthly, say only Daily is supported and offer a Daily page instead.
- Match the user's language, adapt if they switch. Every label in this file is an English placeholder — translate it, and never carry over a language from an example.
- **Gmail follow-through (mark-as-read) is mandatory whenever Gmail is in the resolved set** — not a nice-to-have, and not conditional on anyone watching chat: it runs on a silent recurring run exactly the same as a manual one. Never draft or send emails on the user's behalf.
- **A tile for a connector in the resolved set is never silently blank.** If it returned nothing, write a one-line placeholder instead of omitting the tile.
- **No self-tuning cycle.** The digest writes content tiles only — never a "Tune your digest" or feedback-checkbox tile.
- **A recurring run always re-invokes this same skill** — never hand off to a different skill for the daily digest. The whole point of resolving everything once is that the recurring run can skip straight to the fetch.
- **Never force one connector's grouping onto another.** Email's urgency split, Slack's mentions/topics split, and Calendar's schedule-shape are three genuinely different structures because the data is genuinely different — decide the shape from what a connector actually returns, every time (step 4).
- **Never treat "how many tiles" as fixed per connector either.** A quiet day keeps Email or Slack in one combined tile; real volume is what earns a split into several — decide that from the actual pull, every time, not from what a connector "usually" gets.
- **The Connector-check widget asks at most once, right after the first message, on a first run only.** Never repeat it automatically on a recurring run — silently re-probe `skipped:` instead, and only mention a newly-connected tool once, briefly, if it succeeds.
- **A missing scheduling tool is not a dead end.** If `anthropic-skills:schedule` or `mcp__scheduled-tasks__create-scheduled-tasks` isn't available, say so in one line and still continue straight to step 9 (Related workflows) — the tiles are already written either way, and the run is not complete without the closing question.
