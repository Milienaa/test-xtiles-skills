---
name: tile-layout
description: >
  INTERNAL shared step — not a user-facing workflow. The justified-grid layout
  pass that every planner-writing workflow runs immediately after creating tiles.
  Invoked only by other workflows (daily-brief, evening-reflection, today-news,
  weekly-review, and any future workflow that writes tiles) via
  xtiles_get_workflow("tile-layout"), right after a successful
  xtiles_create_tiles_from_markdown_* call. Never match this to a user request,
  never list or offer it to a user, never run it standalone.
allowed-tools: mcp__xtiles__xtiles_get_page_layout, mcp__xtiles__xtiles_set_page_layout
---

# Tile Layout — Justified-Grid Layout Pass (shared step)

The shared post-write layout pass reused by every workflow that writes tiles to
an xTiles planner page. A calling workflow invokes this right after a successful
`mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner` (or any
`xtiles_create_tiles_*`) call. This is **not** a standalone workflow — never run
it on its own and never offer it to a user.

## When the caller runs this

Immediately after every successful write of tiles, no exceptions: mandatory,
silent, automatic — scheduled runs included, fast-track included, any tile count
included (even a single tile). Run it before composing the CTA button or any
other post-write step. Never ask the user about it and never skip it because the
content "looks simple enough". The only permitted skip is the documented
last-resort at the end of step 9.

## Inputs the caller passes

- **view_id** — from the write-call response. Do NOT re-derive it via
  `mcp__xtiles__xtiles_get_planner_content`.
- **tile_ids** — from the write-call response, ordered to match the `###`
  sections in the markdown just written (`tile_ids[i]` is the tile for the *i*-th
  section). These are the ONLY tiles this pass repositions ("added tiles"). A
  single id is valid.
- **the markdown just composed** for those tiles — reused as-is to measure
  content; never re-fetch it.
- **layout hints** (optional, caller-specific): default tiles-per-row and which
  tile(s) are "heavy". If absent, use the defaults in the steps below.

## Steps

1. Call `mcp__xtiles__xtiles_get_page_layout` with `view_id`. Mandatory. It
   returns the grid bounds (`max_width`, `max_height`, `min_tile_width`,
   `min_tile_height`) and EVERY tile on the page with `tile_id`/`x`/`y`/`w`/`h`,
   new and pre-existing alike (other workflows may already have written to this
   same page). Coordinates are grid cells: `(x, y)` = top-left corner (0-based),
   `(w, h)` = size. Split the tile list by `tile_id`: those matching `tile_ids`
   are the **added** tiles (the only ones you move/resize); every other tile is a
   fixed **obstacle** — read its `x/y/w/h`, never modify it, never include it in
   step 9's call.

2. **Measure each added tile from the markdown you already have.** Parse each
   added tile's content into blocks, each one of:
   - **header** — a `####`/`#####`/bold subsection label (1 line,
     width-independent)
   - **short-line** — a one-line item that never wraps even at `min_tile_width`
     (checkbox, `**Channels:**` summary, short bullet, divider) — 1 line
   - **paragraph** — a longer block that wraps; its line count depends on width

   If unsure, compare the item's character length to the line capacity at
   `min_tile_width`: fits in one line even there → short-line, else → paragraph.

3. **Line capacity by width:** `chars_per_line(w) = (w * col_px − padding_px) /
   avg_char_px`, with heuristic constants `col_px ≈ 25`, `padding_px ≈ 48`,
   `avg_char_px ≈ 7.5` (mixed Cyrillic/Latin ~14px; calibrate against a real tile
   if you can). For a paragraph at width `w`: `lines = ceil(char_count /
   chars_per_line(w))`. Header/short-line = 1 line regardless of `w`.

4. **Tile height at a width** (grid rows ≈ text lines):
   `H_tile(w) = Σ lines(block, w) + (#header blocks) + (#paragraph→paragraph gaps) + 2`
   — the `+2` is tile chrome (title + top/bottom padding); the header/gap terms
   are blank lines that don't shrink with width. Round up to whole rows, clamp to
   `≥ min_tile_height`. (Naive "height ∝ 1/width" fails because only paragraphs
   shrink with width.)

5. **Compose rows** from the added tiles in their existing top-to-bottom order
   (never reshuffle content, never pull in an existing tile). Place them in free
   grid space from step 1's full-page layout that doesn't overlap any obstacle —
   typically the first empty `y` below everything already on the page.
   - **Single added tile** → give it a generous width: `max_width` if that row is
     free, else the largest free band next to existing tiles. Skip the
     row-balancing in steps 6–7.
   - **Multiple added tiles** → default to the caller's tiles-per-row hint (fall
     back to **2 per row**). Give a very heavy tile its own full-width row
     (`w = max_width`). Avoid a lonely narrow tile when it can pair with a
     neighbor. If all added tiles' natural widths fit one band, a single row is
     fine.

6. **Distribute width in each row** across the full `max_width` (fill the band —
   no leftover gap), proportional to content weight (≈ paragraph char count):
   light tiles get just above `min_tile_width`; the remaining columns go to the
   heavy tile(s) so they wrap less. Every tile stays `≥ min_tile_width`; widths in
   a row sum to exactly `max_width`.

7. **Equalize height (flush-bottom rule):** compute `H_tile(w)` at each tile's
   assigned width, then set every tile in the row to `h = max(H_tile)` so the row
   ends at one bottom line (empty space under a lighter tile is fine — dashboard
   look). Set `x` left→right (first tile at the row's left edge, next `x` =
   previous `x + w`), same `y` for the row, next row `y` = this row's `y + H`. If
   the max is dominated by one heavy tile, first try widening it (redo step 6) to
   pull `H` down. If a tile still doesn't fit: narrow its neighbor toward
   `min_tile_width` and give it the columns; if the neighbor is already at
   `min_tile_width`, raise `H` by exactly the shortfall.

8. **Self-check before calling:** for every row with more than one tile, widths
   sum to exactly `max_width`; every `w`/`h` is `≥ min_tile_width` /
   `≥ min_tile_height`; no added-tile rectangle overlaps another added tile or any
   obstacle from step 1. Fix any violation yourself — don't rely on the server to
   catch it.

9. Call `mcp__xtiles__xtiles_set_page_layout` once, listing **only the added
   tiles** (`tile_id` + new `x/y/w/h`) — the same set identified in step 1. Never
   include an obstacle; tiles you omit keep their position automatically. **If the
   call is rejected**, retry once with a simpler fallback: rows of 2 added tiles
   at equal width (`max_width / 2` each, or the full `max_width` for a lone
   leftover), and per row a shared height equal to the tallest `H_tile` from
   step 4 for that row. **Only if the retry also fails, skip silently and
   continue** — a slightly-off layout beats a broken flow.

## Return to the caller

Once `mcp__xtiles__xtiles_set_page_layout` succeeds (or the last-resort skip is
reached), return to the calling workflow and continue with its next step (e.g.
the CTA button). Do not message the user about this pass.
