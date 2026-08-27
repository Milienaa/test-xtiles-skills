---
name: reorganize-with-gpt
description: >
  ChatGPT Work version of Reorganize — use this variant in ChatGPT; in
  Claude use `reorganize` instead.

  Take the current xTiles page and produce a new, well-laid-out,
  well-colored page in the same project with the exact same words — not
  an edit of the original, a fresh page next to it. Use when the user
  wants this page to look designed instead of like a wall of text: a
  sensible tile grid, one visually dominant tile for the page's actual
  point, a cohesive color pair, an emoji per tile title. The source
  page's content never changes — nothing is rewritten, shortened,
  merged, or paraphrased; the only generated text is tile titles pulled
  from the page's own wording, plus a scaffold's hint lines on an
  otherwise-empty page.

  One shot: no clarifying questions, ever. If the launch context is
  missing what this needs (see CONTEXT), stop and say so plainly instead
  of guessing or asking.

  Detect the surface, not the request: `show_widget` / `AskUserQuestion`
  mean Claude/Cowork — use `reorganize` instead. This variant has no
  `xtiles_get_tile_styles` / `xtiles_set_tile_styles` available at all —
  every color decision has to be right at creation time.

  Triggers: "format this page for GPT", "reorganize (GPT)", picking
  "Restyle" / "Reorganize" from the Work-with-this-project menu in
  ChatGPT, or asking to make this page look nicer as a new page.
allowed-tools: >
  mcp__xtiles__xtiles_get_view_content,
  mcp__xtiles__xtiles_get_project_content,
  mcp__xtiles__xtiles_create_view_from_markdown,
  mcp__xtiles__xtiles_get_page_layout,
  mcp__xtiles__xtiles_set_page_layout
---

# xTiles Page Reorganize — Restyle a Page into Tiles (ChatGPT Work)

## Principles

1. **Verbatim words, light structural formatting allowed.** Every *word* placed
   on the new page is a word from the source, in the same order — never
   rewritten, shortened, merged, or paraphrased. On top of unchanged words,
   markup may be added to make what's already there easier to scan: **bold** a
   key phrase or label, turn an obviously-enumerated run of prose into a
   bullet list, or wrap a standout line in a `>` quote — see step 3. This is
   presentation, not authorship: it never adds, drops, or reorders a word. The
   only actually generated *text* this workflow ever produces is a short tile
   title pulled from that tile's own wording, and — on a genuinely empty
   source — a short scaffold hint line.
2. **The source page is read-only.** This workflow only ever calls
   `xtiles_get_view_content` on the source `viewId`. Nothing else in this file
   ever writes to, patches, restyles, or repositions the source page.
3. **One shot.** No clarifying question, no form, no confirmation prompt —
   ever. If something required is missing, stop and say so (see CONTEXT); never
   guess and never ask.
4. **Match the source page's language.** Every generated word (tile titles,
   scaffold hints) is written in the same language as the source page's content.
   Never mix languages, never default to English when the source isn't English.
5. **Confirm with the tool's own link.** The final message always uses the
   `resource_url` returned by `xtiles_create_view_from_markdown` — never a
   hand-built URL.
6. **No color correction pass exists in this environment.** Unlike the Claude
   variant, there is no second chance to fix a tile's color after creation —
   `xtiles_get_tile_styles` and `xtiles_set_tile_styles` are not available here.
   Every color decision has to be right in the markdown at creation time.

---

## CONTEXT — required before doing anything

- **`viewId`** — the source page's view id, injected at launch.
- **`projectId`** — the project the source page lives in, injected at launch.
  `xtiles_create_view_from_markdown` requires a `projectId` and there is no tool
  that resolves a project from a bare `viewId`. If `projectId` is not present in
  your launch context, **stop** and tell the user you can't identify the page's
  project — do not call `xtiles_list_projects` to search for it, and do not ask
  the user a clarifying question. This workflow never asks; it either has what
  it needs or it doesn't.

---

## Algorithm

### 1. Read the source

Call `xtiles_get_view_content(viewId)`. This is the only source of truth for the
page's words — every block placed on the new page has to be a verbatim copy of
something read here. Note the language the content is written in; every piece of
text generated later (tile titles, scaffold hints) has to match it.

**If the response describes a collection (database) page** — a schema of
attributes/columns instead of a markdown body — this workflow does not apply.
Stop and tell the user Reorganize works on canvas pages, not database/collection
pages.

### 2. Decide: reorganize, or scaffold a genuinely empty page

Always reorganize when there is any real content to work with — however
little. The scaffold branch (**step 5**) exists only for the true edge case:
the source page has **zero real content** (just a title, no body text, no
blocks at all). In that case there is nothing verbatim to place on tiles, so
skip to step 5 instead of steps 3–4. Any actual content at all, even a single
sentence, goes through steps 3–4 as a real (if small) reorganize.

### 3. Plan the new page's structure

Group the source content into thematic tiles. For each tile:

- **Title** — a short phrase pulled from that tile's own wording, never
  invented. Prefix it with one fitting emoji.
- **Body** — the exact source blocks that belong to it, copied verbatim; moving
  *which* tile a block sits under is fine, touching its wording is not. A tile
  holds at most 40 blocks — if a group's content would exceed that, split it
  into more tiles rather than dropping or condensing anything.
- **Light structural formatting, on top of the unchanged words.** Where it
  makes the tile easier to scan, add markup the source didn't already have:
  wrap a label or key phrase in `**bold**`, turn a clearly-enumerated run of
  prose ("first... then... finally...") into a `-` bullet list, or wrap a
  standout line (the page's own stated takeaway) in a `>` quote. Every word
  stays exactly as written, in the same order — this only adds punctuation
  and layout around them, never a new word, never a reordering. Skip it
  wherever the source is already reasonably scannable — don't format for the
  sake of it.

Identify the one tile that holds the page's actual point (its decision, goal,
or summary) — this is the **primary tile**.

**Grid.** Canvas pages are a 48-column-wide grid; every tile needs `x + w ≤ 48`,
minimum size 8 wide × 2 tall (default 16×12), and tiles may never overlap.

**Position** with the inline directive, on its own line directly under the
tile's `###` heading:

```
@position:x,y,w,h
```

Give the primary tile a wider `w` than the rest. The importer may still
normalize positions on its side (e.g. shifting everything so the topmost tile
starts at row 0) — that's expected, not an error.

**Color — get this right here; there is no fixing it after creation.** Pick
one thematic pair that fits the page's subject and use only those two colors
across every tile, alternating for variety:

| Theme | Pair |
|---|---|
| Business | `GHOST` + `CUMULUS` |
| Fitness / sport / gym | `GOSSIP` + `COLDTURKEY` |
| Idea generation | `BLUE_CHALK` + `CUMULUS` |
| Travel / culture | `MILK_PUNCH` + `HAWKES_BLUE` |
| Task planning | `COLDTURKEY` + `PATTENS_BLUE` |
| Learning / studying / personal growth | `SAIL` + `ATHENS_GRAY` |
| Students | `BERMUDA` + `BLUE_CHALK` |
| Pets | `PERFUME` + `SELAGO` |
| Home | `COLDTURKEY` + `RICE_FLOWER` |
| Mental health / psychology | `SELAGO` + `WHITE_LINEN` |
| Money / finances / budgeting | `POLAR` + `HAWKES_BLUE` |

For a theme not listed, pick the closest pair from the full palette:
`BOTTICELLI`, `MANDYSPINK`, `RAJAH`, `PALECANARY`, `GOSSIP`, `BERMUDA`,
`ANAKIWA`, `SAIL`, `PERFUME`, `MAUVE`, `GHOST`, `COLDTURKEY`, `PIPPIN`,
`MILK_PUNCH`, `CUMULUS`, `RICE_FLOWER`, `POLAR`, `PATTENS_BLUE`, `HAWKES_BLUE`,
`SELAGO`, `BLUE_CHALK`, `ATHENS_GRAY`, `WHITE_LINEN`.

Set color with inline directives, on their own lines under the tile's `###`
heading (same line group as `@position`):

```
@color:COLOR_NAME
@colorSize:STYLE_NAME
```

Give the **primary tile** a bolder style — `HEADER` or `HEADER_LARGE` — from
its pair color. Give every other tile a lighter, consistent style from the same
pair — `LIGHTER` is the default choice; `LIGHTER_HEADER` or
`LIGHTER_CONTOUR_LINE_BORDER` also work. Never use a third color. Double-check
every tile's `@color`/`@colorSize` line before calling the creation tool — this
is the only chance to get it right.

### 4. Create the page

Call `xtiles_create_view_from_markdown(projectId, markdown)` with the structure
from step 3 as one `##`-titled page (reuse the source page's own title
verbatim) containing the `###` tiles built above. This creates a brand-new page
in the same project — it never touches the source page.

**Optional correction pass — layout only.** Call
`xtiles_get_page_layout(new_view_id)` to see what actually landed on the new
page. If the primary tile isn't visually dominant enough, or something
overlaps or looks misplaced, fix it with `xtiles_set_page_layout` — always
targeting the **new** view id, never the source `viewId`. This pass is
optional polish, not a required step, and it's gated to team plans — if it's
unavailable or errors, skip it silently and continue to step 6. **Do not
attempt to call `xtiles_get_tile_styles` or `xtiles_set_tile_styles` — they do
not exist in this environment.** If a color needs fixing, it was a
creation-time mistake, not something to patch after the fact — there is no
color-correction step in this workflow.

Skip to step 6.

### 5. Empty-page scaffold

Only reached when the source page has zero real content. Call
`xtiles_get_project_content(projectId)` to see the project's title and
sibling pages/content — use this to theme the scaffold (a project called
"Miami Trip" with a sibling page "Packing" implies a travel-shaped scaffold)
without asking the user to confirm it. Build 4–6 tiles, never more, each with:

- a title pulled from the project/sibling theme, with a fitting emoji
- a color from one thematic pair (table above) — set correctly now, there is no
  correction pass for it
- one short instructional hint line as the tile's only body content (e.g. "Add
  your daily itinerary here") — a placeholder prompting the user to fill it in,
  never real content

Create it the same way, via `xtiles_create_view_from_markdown(projectId,
markdown)`, as a new page.

### 6. Confirm

Reply with the exact `resource_url` the creation tool returned — never a
hand-built one — and say plainly that this is a new page and the original is
untouched.

---

## Hard rules

- Never call any tool that edits the source `viewId` — this workflow only ever
  reads it (`xtiles_get_view_content`), never patches, restyles, or repositions
  it.
- Never create a new project — always a new page inside the existing
  `projectId`, via `xtiles_create_view_from_markdown`.
- Never rewrite, shorten, merge, or paraphrase source text, and never add or
  drop a word. Light structural markup (bold, bullets, quote) may be added on
  top of unchanged words per step 3. The only actually generated text is tile
  titles (from the tile's own wording) and, on a genuinely empty page only,
  short scaffold hint lines.
- No clarifying questions, ever. Missing `projectId`/`viewId` is a stop
  condition, not a question — and never worked around by calling
  `xtiles_list_projects` to search for the project.
- All generated text matches the source page's language.
- **Never call `xtiles_get_tile_styles` or `xtiles_set_tile_styles`** — they
  are not available in this environment. Color must be correct via
  `@color`/`@colorSize` at creation time; there is no fixing it afterward in
  this workflow.
- `xtiles_set_page_layout` may only ever target the newly created page, never
  the source `viewId` — and may be unavailable depending on the workspace's
  plan; treat that as a silent skip, not a failure.
- A tile holds at most 40 blocks — split content across more tiles instead of
  dropping or condensing it.
- Confirm with the tool's own returned `resource_url`, never a constructed one.
