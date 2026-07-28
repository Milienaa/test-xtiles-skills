---
name: chats-to-xtiles
description: >
  Capture the user's work from their GPT threads into xTiles — either as one
  structured project or as short per-thread daily notes. Use when the user says
  "Turn chats into xTiles" (or its equivalent in their language), opens with an
  empty or vague prompt, or wants their recent GPT conversations turned into
  xTiles content. Compiles and structures the themes actually present
  across threads, or one dated knowledge note per thread, and writes it to xTiles.
---

# xTiles Chats to Projects, Digests & Notes

## Purpose
Capture the user's work from their GPT threads into xTiles — as one structured project, as a morning digest on today's planner page, or as short per-thread daily notes.

## Trigger recognition
Treat any of these as a request to RUN this workflow:
- the phrase "Turn chats into xTiles" (and its natural equivalent in the user's language, e.g. "Перетворити чати на xTiles" / "Зібрати чати в xTiles")
- an empty or vague opening, or an explicit ask to turn recent GPT chats into an xTiles project, a morning digest, or notes.
  The trigger's literal meaning IS the action — so the only correct response is to start STEP 1 → STEP 2. Do not answer with generated content, a fun fact, or an artifact instead of running the workflow. If you're unsure whether it's the trigger, assume it is and run the workflow.

## CRITICAL — ALWAYS ASK FIRST (do not skip)
The trigger STARTS this workflow — it is NOT permission to create anything on its own. Do not treat it as a command to build a project, a digest, or notes. You MUST run STEP 1 → STEP 2 and WAIT for the user's explicit selection before creating or writing anything. Never generate a project, a digest, or notes straight from the trigger. If you are about to write before the user has picked — stop and show the options first.

## LANGUAGE
Respond in the user's language — detect and mirror it. Labels below are examples; translate them.

## INPUT SOURCE
Content to structure comes FROM the user's GPT conversations/threads — what they discussed and did. Read across available threads to find themes and daily activity. You MAY read the list of the user's existing xTiles projects to offer them as candidate targets (low-hanging fruit), but the material itself is sourced from threads, never invented.

## COMPILE, DON'T INTERPRET (applies to everything you build)
Never invent new concepts, frameworks, categories or tasks. Build the project (and the notes) only from information explicitly present in the selected chats. Your job is to organize, deduplicate and structure — not to expand the content. When grouping into themes/sections, you are naming what is already there, not generating anything new.

## STEP 1 — ANALYZE (silently)
From the threads, extract:
- PROJECT candidates: the 2–3 strongest themes (not just one). For each: a real title + a one-line reason ("spans N threads about X"). Also surface any of the user's existing xTiles projects that match a theme.
- NOTE candidates: for today's activity, one real one-line summary per thread.
- DIGEST candidates: whether there is enough real material for a morning digest, roughed out across the six sections in STEP 3 DIGEST. Read recent activity first, and reach further back when a thread was left unfinished and still matters — so "Continue today" is not empty after a quiet evening. How far back to reach is a judgement call, not a fixed window.
  Use only real content — no filler or placeholders.
  If a saved preference exists in memory (Step 5), lead with that option first (but still ask and wait).

## STEP 2 — PROPOSE (plain text, wait for choice)
Show ONE plain-text message with the real, detected options and STOP for the user's choice. Do not use any visual/interactive form — just a clear numbered list. Make no write and no create call until the user replies. Never run more than one use case.

Give it a warm, contextual framing (not a dry menu): greet, say you looked through their recent threads, then offer to organize those multiple chats into projects, then offer today's morning digest, and separately offer to capture the notes you found. Name the actual notes you detected, not just a count.

**Number EVERY main category — not just the projects.** Each of the three use cases (project proposals, morning digest, notes) is its OWN top-level number (1, 2, 3), so the user cannot miss the digest or the notes among plain prose. The individual project themes are nested UNDER category 1 as lettered sub-options (a, b, c …). Never leave the digest or notes as an unnumbered paragraph — if a category is offered at all, it gets a number. Numbering stays contiguous: if the digest is not offered (no real material), don't leave a gap — renumber so notes become 2.

Structure (fill `<...>` with real content; translate to the user's language):

    Hi! I went through the threads you've been working in lately. Here's what I can do — just tell me the number:

    1. 📁 Turn these chats into a project — which theme should we set up first?
       a) "<theme A>" — <N threads>
       b) "<theme B>" — <N threads>
       c) "<theme C>" — <N threads>

    2. 💡 **Morning digest** — start your day clear: one glance shows what to continue and your three focuses, right on today's planner page.

    3. 📝 Capture <N> notes with the key takeaways from today's chats:
       — <found note 1 — short summary>
       — <found note 2 — short summary>
       — <found note 3 — short summary>

    Tell me what we do next (e.g. "1b" or "2")

- Each main category is numbered (1/2/3); project themes are lettered sub-options under 1. If a category is missing (e.g. no digest material), renumber the rest so the list stays 1, 2 with no gap.
- Include a matching existing xTiles project as one of the theme options when one fits.
- If fewer than 3 strong themes exist, show only the real ones — never pad with generic entries.
- List the real notes you found in STEP 1 (title/summary each), so the user sees what would be captured.
- Offer the digest only when STEP 1 found real material for it. If it would be empty, leave the offer out rather than promising a digest you cannot fill.
- If the user picks a project theme (e.g. "1b") → STEP 3 PROJECT. If they pick the digest (2) → STEP 3 DIGEST. If they pick notes (3) → STEP 3 NOTES.

## STEP 3 — PROJECT (if chosen)
- Confirm which theme (from the 2–3 offered) the user wants.
- Build one rich, comprehensive project from ALL the relevant content around that theme. Do not artificially constrain its size — cover the material as fully as it deserves; a real project can be large. Typical structure (add or expand sections as the content warrants):
    - Title
    - Context / description
    - Key sections
    - Tasks / next steps — create a REAL, actionable task collection (using the xTiles task-creation tool), so the project opens with a working task list, not just a tile labeled "Tasks". Include only tasks explicitly stated or clearly agreed in the threads; do not invent next steps.
    - Open questions / risks
    - Sources (which threads it's based on)
- Apply real tile styles so the project doesn't look pale/empty: after creating the tiles, use the xTiles tile-style tool to set colors (and color size) on the tiles, giving adjacent tiles distinct colors. This styling must actually be applied, not just described.
- Thorough over terse — this is a working document, err on the side of more depth.
- Create as one project in xTiles; confirm and share the link.

## STEP 3 — MORNING DIGEST (if chosen)
A digest is a picture of where the user's work actually stands this morning, written onto today's planner page. It is built from their conversation history and nothing else — no connector is read, no external source is consulted, nothing is searched on the web. Connectors appear exactly once, at the end, as a recommendation.

If the conversation history is not reachable in this environment, say so plainly and stop. Never reconstruct a plausible morning out of nothing — a fabricated digest is worse than no digest.

**The six sections**, in this order, each becoming its own tile. Titles carry an emoji and are written in the user's language:

1. **Morning context** — what they have been working on lately
2. **Continue today** — unfinished work, open questions, logical next steps. This tile MUST carry a direct link back to the ChatGPT thread each item came from, so the user can jump straight back into the conversation and pick up where they left off. Put the link on its own line under the item it belongs to, in the shape `[Open the <topic> thread](https://chatgpt.com/c/...)` (see **Links back to source** below). Use only real thread URLs from the history — never invent one; where no URL exists, name the conversation so the user can find it.
3. **New decisions and agreements** — what was actually decided, planned or fixed
4. **Ideas and insights** — hypotheses, conclusions, connections between conversations
5. **Three focuses of the day** — at most three priorities, each with a concrete next action
6. **Extend your digest** — 1–3 connectors that would close this digest's real gaps

Sections 1–5 are written only where they carry a new, practically useful signal. An empty section is not an empty tile — it is simply absent. Section 6 accompanies real content and never stands alone.

**Source discipline** — COMPILE, DON'T INTERPRET applies in full, and on top of it:
- Keep three registers visibly apart: a **fact from a chat** (what the user actually said or decided), a **conclusion** (a careful reading of the available context), and a **recommendation** (a suggested next action).
- Never invent people, tasks, decisions, dates, numbers or deadlines.
- Merge duplicates across threads into a single item.
- Never promote a passing conversation into a project or a priority.
- Prefer actions and decisions over retelling what was discussed.
- Where the material is thin, say so plainly instead of padding it out.

**Links back to source** — each block carries labelled links to the threads it came from, in the shape `[Open the launch thread](https://chatgpt.com/c/...)`. Links go on their own line under the block they belong to, never inside a list item — the format does not allow a link inside a bullet. Use only URLs that genuinely exist in the history; never invent a thread identifier. Where no URL is available, name the conversation and give a phrase the user can search their history for.

**The three focuses** — real tasks, written inline in the tile with the `<task>` tag so they live next to the reasoning behind them. Attach a due date only when the source states an actual deadline; the day page already supplies "today", and an invented deadline breaks the rule above.

**The connector tile** — chosen for this digest, not boilerplate. One to three connectors, each named together with the specific gap it would close this particular morning: meetings and focus time, mail that needs an answer, team decisions and blockers, changes in working documents, what was agreed in a call. Recommend only connectors this environment actually offers — a suggestion the user cannot act on is worse than none. Write them as checkbox items so the tile reads as something the user can act on. It recommends only — nothing is connected, configured or fetched in this run.

**Preview first.** Show the whole digest in chat before anything is written — every tile, in full, not a summary of it. Then wait for approval. If the user wants changes, revise and show it again. Nothing reaches the planner before an explicit yes.

**Then write it.** One write, all tiles at once, onto the personal planner page for today — period `day`, today's date resolved in the user's own timezone. Compose the tiles in xTiles Markdown; check the format guide before writing if anything about the syntax is uncertain.

Do not create a title tile. The day page is already the date, and a heading-only tile is dead weight. Instead the digest names itself in one italic line at the top of the first tile — a line that also says it was built from chat history and that no connector data was used, so the user can see at a glance what they are reading and how far to trust it.

Give the page one calm palette: a complementary pair of colours from the format guide, alternating between tiles, set inline as part of the write itself so the page can never end up unstyled. This is deliberately not random per-tile colouring — the digest should read as one object, not six unrelated cards.

After the write: confirm briefly, then run the shared `tile-layout` pass over the new tiles as a best effort — five or six tiles, roughly two per row, with the focuses tile given the full width. Finish by giving the user a clickable link to the day page. If the layout pass cannot run, carry on regardless: it must never cost the user their link.

**When there is nothing to say.** If no section carries a real signal, write nothing to the planner at all. A page holding a single "connect some tools" tile is noise on the user's day. Say so directly in chat instead, and make the connector recommendation there — the same one to three connectors with the same reasoning, spoken rather than written to the page.

## STEP 3 — NOTES (if chosen)
Notes are KNOWLEDGE notes, not an activity log. Do NOT retell the thread as "what happened / what I did". Instead, extract the useful, reusable knowledge from the thread that's worth remembering.

- Confirm the date in plain text (default today; resolve to YYYY-MM-DD in the user's timezone).
- Treat EACH thread as its OWN note (card):
    - Title = the topic / the takeaway, not "did X".
    - Content = the distilled knowledge: rules, principles, patterns, gotchas, checklists, prompt formulas/templates, product conclusions — whatever the thread actually yielded.
    - Self-sufficient: a reader must get value from the card a week later WITHOUT opening the thread. No dependence on the thread's context.
    - Always carry over any resources, links, or images that appeared in the discussion — place them inline where they support the knowledge. If a link was present in the chat, it MUST appear in the note.
    - Source thread link may go at the END, as a reference — it is not the point of the card.
    - Tone: lively but not a story retelling — more knowledge note, less activity log.
- REVIEW BEFORE WRITE: first show all the drafted note cards in chat, then ask for approval to add them to xTiles. Do NOT write to xTiles until the user approves. If they want changes, revise and show again.
- After approval, write each as a SEPARATE note on that day's daily page (multiple notes if multiple threads); confirm and share links.

## STEP 4 — EXECUTE
Act only on the ONE selected use case — never two, never all three. Use xTiles only to WRITE (input is sourced from threads). No speculative calls.

## STEP 5 — REMEMBER PREFERENCE (MANDATORY — always ask)
This step is REQUIRED and must run on every successful run — never skip it, never finish without it. After confirming what was created, always ask ONE plain-text yes/no question: whether to remember this choice so next time you default to the same action (creating a project, or adding notes to the planner — matching what they just picked).
- Yes → save preferred default to memory (<notes | project | digest>).
- No → change nothing.
  A saved preference only changes which option is shown first in STEP 2 — it never skips the STEP 2 ask.
  If you completed a run without asking this question, you have not finished the workflow.

## RULES
- Always ask before creating — the trigger alone is never enough.
- One question at a time.
- Project = detailed (with real tasks and clear formatting); notes = self-sufficient knowledge cards (reusable takeaways, not an activity log), with the source link at the end.
- Digest = today's working picture drawn from chat history alone; connectors are recommended in it, never read for it.
- The digest goes on today's planner page, never as a separate project page.
- Always tie notes to a concrete date.
- STEP 5 (remember preference) is mandatory on every run — never end without asking it.
- After finishing, confirm what was produced and where, then offer one next step.
