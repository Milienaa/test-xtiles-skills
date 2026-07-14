---
name: test-meeting-notes
description: >
  TEST workflow — used to verify that one workflow can invoke another workflow
  (specifically that it calls the `tile-layout` sub-workflow after writing
  tiles). Creates 7 meeting-note tiles on today's daily page, then invokes the
  shared `tile-layout` pass. Trigger only when the user explicitly asks to run
  the meeting-notes test / test workflow chaining. Not a real user-facing
  feature.
allowed-tools: >
  mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner,
  mcp__xtiles__xtiles_get_planner_content,
  mcp__xtiles__xtiles_get_user_timezone,
  mcp__xtiles__xtiles_get_page_layout,
  mcp__xtiles__xtiles_set_page_layout,
  mcp__xtiles__xtiles_get_workflow
---

# Test — Meeting Notes → Daily Page (workflow-calls-workflow)

This is a **test workflow**. Its only purpose is to verify that a workflow can
invoke another workflow — here, that after writing tiles it correctly hands off
to the shared `tile-layout` sub-workflow via `xtiles_get_workflow("tile-layout")`.

> **Design note (7 vs 10):** the spec mentioned both "7 tiles" and "after 10
> tiles." This is resolved: the workflow writes **7** meeting-note tiles and
> calls `tile-layout` right after the write — the same trigger point the real
> workflows use. There is intentionally no "wait until 10 tiles" gate.

## Your process

1. **Resolve today's date.** Call `xtiles_get_user_timezone` and use its
   `timezone` field to resolve the current local date to `yyyy-MM-dd` (ISO 8601).
2. **Build the markdown** for all 7 meeting notes using the block in
   "Meeting notes content" below — verbatim, all in a single string.
3. **Write to xTiles in ONE call.** Call
   `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner` with
   `period: "day"`, `date`: today's date from step 1, and `markdown`: the block
   from step 2. Never split into multiple calls — all 7 tiles go in one write.
4. **Layout pass — invoke the `tile-layout` workflow (this is what the test
   verifies).** Immediately after the successful write:
   - Read `view_id` and `tile_ids` from the write-call response (do NOT re-derive
     via `get_planner_content`). `tile_ids[i]` matches the *i*-th `###` section.
   - Call `mcp__xtiles__xtiles_get_workflow` with id `tile-layout`.
   - Follow that workflow exactly, passing these 7 tiles as the "added tiles" and
     the markdown from step 2 as their content. Hints: 7 tiles · 2 per row · no
     single tile is unusually heavy.
5. **Report the result** using the confirmation block below.

## Meeting notes content

Write this markdown verbatim as the `markdown` argument. Each `###` is one tile;
the content directly follows its heading with no blank line between heading and
body.

```
### Team Standup — Product Sync
**Attendees:** Anna, Bohdan, Chris
- Shipped the new onboarding flow to staging
- Blocker: waiting on design tokens from the design team
- Next: QA pass scheduled for Thursday

### Roadmap Planning — Q3
**Attendees:** Product, Eng leads
- Agreed on 3 headline goals for the quarter
- Deprioritized the reporting revamp to Q4
- Action: PM to draft the one-pager by Friday

### 1:1 — Career Growth
**Attendees:** Manager, Dana
- Discussed path toward a senior role
- Focus areas: system design, mentoring juniors
- Action: pick one project to lead next sprint

### Design Review — Mobile App
**Attendees:** Design, Eng, PM
- Reviewed the new navigation prototype
- Feedback: reduce taps to reach search
- Action: update prototype, re-review next week

### Customer Call — Acme Corp
**Attendees:** Sales, CS, Acme team
- Renewal likely; wants SSO before signing
- Raised a bug in bulk export — logged as P2
- Action: send SSO timeline by end of week

### Retro — Sprint 24
**Attendees:** Whole squad
- Went well: faster PR reviews, fewer regressions
- Improve: flaky CI, unclear ticket acceptance criteria
- Action: add a CI stabilization task to the backlog

### Budget Review — Marketing
**Attendees:** Finance, Marketing
- Q2 spend is 8% under budget
- Reallocating saved budget to paid search
- Action: Finance to update the forecast sheet
```

## Confirmation block

Report to the user with:

```
✅ Test complete — 7 meeting-note tiles written and `tile-layout` invoked.
**[Open in xTiles →](https://xtiles.app/{view_id})**
```

Replace `{view_id}` with the `view_id` from the write-call response.

## Rules

- Write all 7 tiles in a single `create_tiles_from_markdown_in_my_planner` call.
- Always invoke `tile-layout` after the write — that handoff is the whole point
  of this test. Never skip it.
- Use today's date automatically — never ask.