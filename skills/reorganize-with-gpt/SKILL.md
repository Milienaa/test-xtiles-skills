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
  from the page's own wording, a short suffix on the new page's title
  marking it as the reorganized copy, plus a scaffold's hint lines on an
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
   title pulled from that tile's own wording, a short suffix on the new page's
   title marking it as the reorganized copy, and — on a genuinely empty
   source — a short scaffold hint line.
2. **The source page is read-only.** This workflow only ever calls
   `xtiles_get_view_content` on the source `viewId`. Nothing else in this file
   ever writes to, patches, restyles, or repositions the source page.
3. **One shot.** No clarifying question, no form, no confirmation prompt —
   ever. If something required is missing, stop and say so (see CONTEXT); never
   guess and never ask.
4. **Match the source page's language.** Every generated word (tile titles,
   the title suffix, scaffold hints) is written in the same language as the
   source page's content. Never mix languages, never default to English when
   the source isn't English.
5. **Confirm with the tool's own link.** The final message always uses the
   `resource_url` returned by `xtiles_create_view_from_markdown` — never a
   hand-built URL.
6. **No color correction pass exists in this environment.** Unlike the Claude
   variant, there is no second chance to fix a tile's color after creation —
   `xtiles_get_tile_styles` and `xtiles_set_tile_styles` are not available here.
   Every color decision has to be right in the markdown at creation time.

---

## CONTEXT — required before doing anything

- **`viewId`** — the source page's view id, injected at launch. If it's not
  present, **stop** and tell the user you can't identify the page — do not ask
  a clarifying question. This workflow never asks; it either has what it needs
  or it doesn't.
- **`projectId`** — not a separate launch input. `xtiles_get_view_content`
  (step 1) returns `project_id` in its response alongside the page's content —
  that is where `projectId` comes from, every time, for both the reorganize
  and scaffold branches. Never call `xtiles_list_projects` to search for it;
  it's already in the step 1 response.

---

## Algorithm

### 1. Read the source

Call `xtiles_get_view_content(viewId)`. This is the only source of truth for the
page's words — every block placed on the new page has to be a verbatim copy of
something read here. Note the language the content is written in; every piece of
text generated later (tile titles, scaffold hints) has to match it. **Capture
`project_id` from this same response** — this is the `projectId` used in every
`xtiles_create_view_from_markdown` / `xtiles_get_project_content` call below.

**Take a block inventory — this is the conservation target.** As you read,
count every block by type: paragraphs, list items, checkboxes / `<task>` items,
images (`![alt](url)`), tables, quotes, code blocks. Write these counts down
(mentally or in your plan). **Every single block must reappear on the new page,
verbatim, exactly once** — the same number of images, the same number of tasks,
the same list items. This inventory is what step 4 verifies against; nothing is
"reorganized away".

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
- **Tasks and images are first-class blocks — never lost.** Every checkbox /
  `<task>...</task>` item and every image (`![alt](url)`) from the source is
  carried onto exactly one tile, verbatim — copy the task line as a task and the
  image as the same `![alt](url)` block. Never turn an image into a text caption,
  never summarize a task away, never drop either to make a tile fit. If carrying
  them would exceed the 40-block cap, split into more tiles.
- **Don't restate the title inside the body.** When a tile's title is pulled
  from a heading/label the source itself used for that section, that heading
  line becomes the tile's `###` title and stops there — it is not also
  repeated as a bolded first line of the body. Putting those words in the
  title slot is where they now live, not a deletion; only keep the wording in
  the body too if the source itself repeats it a second time as running text,
  not just as that section's own heading.
- **Light structural formatting, on top of the unchanged words.** Where it
  makes the tile easier to scan, add markup the source didn't already have:
  wrap a label or key phrase in `**bold**`, turn a clearly-enumerated run of
  prose ("first... then... finally...") into a `-` bullet list, or wrap a
  standout line (the page's own stated takeaway) in a `>` quote. Every word
  stays exactly as written, in the same order — this only adds punctuation
  and layout around them, never a new word, never a reordering. Skip it
  wherever the source is already reasonably scannable — don't format for the
  sake of it.

**Page title.** The new page reuses the source page's title, but never as an
exact duplicate — append a short suffix marking it as the reorganized copy
(the sense of "(Reorganized)"), translated into the source page's language
like every other generated word. This keeps the new page distinguishable from
the source in project listings and search.

Identify the one tile that holds the page's actual point (its decision, goal,
or summary) — this is the **primary tile**.

**Grid.** Canvas pages are a 48-column-wide grid; every tile needs `x + w ≤ 48`,
minimum size 8 wide × 2 tall (default 16×12), and tiles may never overlap.

**Size tiles to their content, not the default.** The 8×2 floor and the 16×12
default both exist only for a tile with a single short line to show — most
content-bearing tiles need to grow past the *default*, not just past the
floor. Scale `h` up with block count: a short paragraph can stay near the
default, but a multi-row table or a list of several items needs roughly
double that (`h=24` or more), and a long table or dense list needs more
still. Never leave a table, image, or multi-item list resting at the 16×12
default — that default is a starting point for a near-empty tile, not a
target size for real content.

**Images get room to show in full.** A tile whose body includes an
`![alt](url)` image block needs `w`/`h` sized so the image isn't cropped —
give it at least `h=24`, more for a tall/portrait image. The 16×12 default
crops most images and is never an acceptable resting size for an image tile.

**Position** with the inline directive, on its own line directly under the
tile's `###` heading:

```
@position:x,y,w,h
```

Give the primary tile a wider `w` than the rest. The importer may still
normalize positions on its side (e.g. shifting everything so the topmost tile
starts at row 0) — that's expected, not an error.

**Pack the grid with no gaps.** Lay tiles left-to-right, top-to-bottom in reading
order. Within a row, the tiles' widths should fill the full 48 columns (no empty
horizontal band); start the next row at the bottom edge of the tallest tile in
the row above (`y = that row's y + its max h`), with no empty rows between. Two
tiles per row is a good default; a single wide tile (often the primary) can take
a full-width row. Don't leave a lone half-width tile with 24 empty columns beside
it when it could be widened or paired — the finished page should read as a filled
grid, not scattered tiles with holes.

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

**Conservation gate — check the markdown before you send it.** This workflow has
no tool to patch content into a page after creation, so the new page can only be
as complete as the markdown you pass here. Before calling the creation tool,
verify the assembled markdown contains **every block from the step 1 inventory** —
the same number of images (`![alt](url)`), the same number of tasks/checkboxes,
and the same list items and paragraphs. If the draft is short even one block, add
the missing block to the tile it belongs to before creating. Never send a draft
that lost content to gain tidiness.

Call `xtiles_create_view_from_markdown(projectId, markdown)` with the structure
from step 3 as one `##`-titled page (reuse the source page's own title plus
the reorganized-copy suffix from step 3) containing the `###` tiles built
above. This creates a brand-new page
in the same project — it never touches the source page.

**Completeness read-back — verify what actually landed.** Call
`xtiles_get_view_content(new_view_id)` and compare against the step 1 inventory:
same number of images, same number of tasks/checkboxes, same list items and
paragraphs. If the read-back shows a block was dropped despite a complete draft,
say so plainly in the step 6 reply (name what's missing) — this variant has no
content-patch tool, so don't silently pretend the page is complete.

**Layout check — always run it.** Call `xtiles_get_page_layout(new_view_id)`
to see what actually landed on the new page — do this every run, don't skip
it just because the plan from step 3 looks fine on paper. Fix with
`xtiles_set_page_layout` — always targeting the **new** view id, never the
source `viewId` — when: the primary tile isn't visually dominant; anything
overlaps; **or there are gaps** — an empty horizontal band beside a tile (widen
it or its neighbor to fill the 48 columns) or an empty row between tiles (pull
the lower tiles up so rows are contiguous). The page should read as a filled
grid. The tool is gated to team plans — only skip the whole pass if
`xtiles_get_page_layout` itself is unavailable or errors; in
that case skip silently and continue to step 6. **Do not attempt to call
`xtiles_get_tile_styles` or `xtiles_set_tile_styles` — they do not exist in
this environment.** If a color needs fixing, it was a creation-time mistake,
not something to patch after the fact — there is no color-correction step in
this workflow.

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
- if a tile's theme is visual (a "Photos", "Gallery", or similarly
  picture-driven tile), don't just describe the picture in the hint line —
  add an `@mediaKeyword:VALUE` directive line under that tile's `###` heading
  (same line group as `@position`/`@color`) so it ships with an actual sourced
  image, keyworded to the tile's theme, alongside the hint text

Create it the same way, via `xtiles_create_view_from_markdown(projectId,
markdown)`, as a new page.

### 6. Confirm

**Always post the link — the run is not finished without it.** The reply's first
line is a clickable link to the new page, using the exact `resource_url` the
creation tool returned (never a hand-built one). Never end the run without this
link in chat. Keep the rest of the reply a short, human summary, not a technical
report:

- Say plainly this is a new page in the same project and the original is untouched.
- List what landed on the page, one line per tile: its emoji + title with a few
  words on what it holds — enough that the user knows where to find things
  without re-reading the page. Note which one is the primary tile if that's not
  obvious from its position.
- Never dump technical details in the reply — no color names, style codes, tile
  ids, or grid coordinates. If the optional layout pass failed or was skipped
  (see Hard rules), say so in plain language instead of naming the tool.
- Close with a short, low-pressure offer to keep going — e.g. continuing to
  develop the project, or turning the page into an action plan — phrased as a
  question, not another workflow launch.

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
  titles (from the tile's own wording), the new page's title suffix, and, on
  a genuinely empty page only, short scaffold hint lines.
- No clarifying questions, ever. Missing `viewId` is a stop condition, not a
  question. `projectId` is never a launch input — it always comes from
  `xtiles_get_view_content`'s own response (step 1), never from
  `xtiles_list_projects`.
- All generated text matches the source page's language.
- **Never call `xtiles_get_tile_styles` or `xtiles_set_tile_styles`** — they
  are not available in this environment. Color must be correct via
  `@color`/`@colorSize` at creation time; there is no fixing it afterward in
  this workflow.
- `xtiles_get_page_layout` / `xtiles_set_page_layout` may only ever target the
  newly created page, never the source `viewId`. `xtiles_get_page_layout` is
  always called (not optional polish) — only skip the layout pass if that
  call itself is unavailable or errors (e.g. plan gating), and do so silently.
- A tile holds at most 40 blocks — split content across more tiles instead of
  dropping or condensing it.
- Confirm with the tool's own returned `resource_url`, never a constructed one.
