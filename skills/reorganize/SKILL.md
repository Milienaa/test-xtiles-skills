---
name: reorganize
description: >
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

  Triggers: picking "Restyle" / "Reorganize" from the Work-with-this-
  project menu, or asking in the user's own words to make this page look
  nicer, format it, give it real structure, turn a wall of text into
  tiles — as a new page, not changing the one they're looking at.

  Environment: this is the Claude / Cowork variant — it has
  `xtiles_get_tile_styles` / `xtiles_set_tile_styles` available, so it colors
  the *new* page with those tools right after creation instead of inline
  markdown color directives. In ChatGPT Work, where those two tools don't
  exist, use `reorganize-with-gpt` instead, which sets color inline at
  creation time.

  Environment triggers: "Reorganize in Claude", "the Claude version",
  "Claude Reorganize".
allowed-tools: >
  mcp__xtiles__xtiles_get_view_content,
  mcp__xtiles__xtiles_get_project_content,
  mcp__xtiles__xtiles_create_view_from_markdown,
  mcp__xtiles__xtiles_get_page_layout,
  mcp__xtiles__xtiles_set_page_layout,
  mcp__xtiles__xtiles_get_tile_styles,
  mcp__xtiles__xtiles_set_tile_styles
---

# xTiles Page Reorganize — Restyle a Page into Tiles

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
3. **One shot.** No `AskUserQuestion`, no widget, no clarifying message — ever.
   If something required is missing, stop and say so (see CONTEXT); never guess
   and never ask.
4. **Match the source page's language.** Every generated word (tile titles,
   scaffold hints) is written in the same language as the source page's content.
   Never mix languages, never default to English when the source isn't English.
5. **Confirm with the tool's own link.** The final message always uses the
   `resource_url` returned by `xtiles_create_view_from_markdown` — never a
   hand-built URL.

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

**Color — plan it now, apply it with a tool, not markdown.** This variant has
`xtiles_get_tile_styles` / `xtiles_set_tile_styles` available, so color is set
by calling those tools right after creation (step 4), not with inline
`@color`/`@colorSize` directives in the markdown — the markdown for this
variant only ever carries `@position`. Decide the plan here so step 4 has
something to apply. Pick one thematic pair that fits the page's subject and
use only those two colors across every tile, alternating for variety:

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

Give the **primary tile** a bolder style — `HEADER` or `HEADER_LARGE` — from
its pair color. Give every other tile a lighter, consistent style from the same
pair — `LIGHTER` is the default choice; `LIGHTER_HEADER` or
`LIGHTER_CONTOUR_LINE_BORDER` also work. Never use a third color. Keep this
title → color/style plan (e.g. as a short list matched by tile title) — step 4
needs it to call `xtiles_set_tile_styles`.

### 4. Create the page, then apply color

Call `xtiles_create_view_from_markdown(projectId, markdown)` with the structure
from step 3 as one `##`-titled page (reuse the source page's own title
verbatim) containing the `###` tiles built above, positioned with `@position`
but **without** `@color`/`@colorSize` — this variant colors tiles with a tool
call, never inline directives. This creates a brand-new page in the same
project — it never touches the source page.

**Apply color — mandatory for this variant, right after creation.** Call
`xtiles_get_tile_styles(new_view_id)` to get the `tile_id` of every tile on the
new page (match each one to your plan by its title — the same titles you just
wrote). Then call `xtiles_set_tile_styles(new_view_id, tiles: [...])` once,
setting `color` and `color_size` for every tile per the plan from step 3. This
is the primary coloring mechanism for this variant, not optional polish — do
it every run. **If `xtiles_get_tile_styles`/`xtiles_set_tile_styles` are
unavailable (e.g. plan gating) or the call errors**, the page still exists
without palette colors — don't block on it; note this plainly in the step 6
confirmation instead of failing the run.

**Optional correction pass — layout only.** Call
`xtiles_get_page_layout(new_view_id)` to see what actually landed. If the
primary tile didn't end up visually dominant, fix it with
`xtiles_set_page_layout` — always targeting the **new** view id, never the
source `viewId`. This pass is optional polish, not a required step, and the
tool is gated to team plans — if it's unavailable or errors, skip this pass
silently and continue to step 6.

Skip to step 6.

### 5. Empty-page scaffold

Only reached when the source page has zero real content. Call
`xtiles_get_project_content(projectId)` to see the project's title and
sibling pages/content — use this to theme the scaffold (a project called
"Miami Trip" with a sibling page "Packing" implies a travel-shaped scaffold)
without asking the user to confirm it. Build 4–6 tiles, never more, each with:

- a title pulled from the project/sibling theme, with a fitting emoji
- a color planned from one thematic pair (table above) — applied the same way
  as step 4: create with `@position` only, then call
  `xtiles_get_tile_styles`/`xtiles_set_tile_styles` on the new page
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
- **Color is set via `xtiles_set_tile_styles` after creation, not via
  `@color`/`@colorSize` in the markdown** — this is the primary mechanism for
  this variant, run every time, not optional polish.
- `xtiles_get_tile_styles` / `xtiles_set_tile_styles` / `xtiles_set_page_layout`
  may only ever target the newly created page, never the source `viewId`. They
  may be unavailable depending on the workspace's plan — if `set_tile_styles`
  fails, the page still exists without palette colors; note that in the step 6
  confirmation rather than failing the run. If `set_page_layout` (optional
  correction) fails, just skip it silently.
- A tile holds at most 40 blocks — split content across more tiles instead of
  dropping or condensing it.
- Confirm with the tool's own returned `resource_url`, never a constructed one.
