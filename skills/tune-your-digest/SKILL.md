---
name: tune-your-digest
description: >
  INTERNAL shared step — not a user-facing workflow. The self-tuning feedback
  cycle for planner digests: reads yesterday's feedback tiles, applies the user's
  answers to the digest config, then proposes today's questions and composes the
  three "meta" tiles (📈 weekly recap, 🔑 Important keywords, 🎛️ Tune your digest).
  Invoked only by a digest-writing workflow (daily-brief, and any future digest
  workflow) via xtiles_get_workflow("tune-your-digest") — never matched to a user
  request, never listed or offered to a user, never run standalone. The caller
  owns the actual tile write and config persistence; this workflow only reads,
  decides, and returns markdown + config patches.
user-invocable: false
allowed-tools: mcp__xtiles__xtiles_get_planner_content
---

# Tune Digest — Self-Tuning Feedback Cycle (shared step)

The shared self-tuning cycle reused by every workflow that writes a recurring
digest to an xTiles planner page. It turns the digest into a feedback loop: the
user ticks checkboxes on yesterday's page, and today's digest adapts. This is
**not** a standalone workflow — never run it on its own and never offer it to a
user.

The caller runs this in **two stages** at two points in its flow, and does the
actual tile write itself (this workflow never writes tiles — it only reads
yesterday's page, decides, and hands back markdown + config patches).

---

## Inputs the caller passes

- **config** — the full digest config carried in the scheduled-task prompt:
  `role`/`tools`/`daily_content` plus every tuning field listed under "Config
  fields" below (all optional/empty until first used).
- **yesterday's date** and **today's date** (ISO `yyyy-MM-dd`), plus **today's
  ISO week** (e.g. `2026-W29`) — the caller resolves these from
  `xtiles_get_user_timezone`.
- **is_first_ever** — true only on the very first digest this user has ever had
  (no prior page exists).
- **Stage B only: today's composed content tiles** — the digest content the
  caller has already assembled (Emails, Slack, Calendar, pinned-topic tiles,
  etc.), as markdown. Used for structural analysis and keyword detection. Never
  treated as an answer source.

## Outputs the caller uses

- **Stage A** → a **patched config** (the caller persists it in its
  scheduled-task prompt when re-creating the schedule) and an **applied-changes
  list** (surfaced as the applied-line on the feedback tile in Stage B).
- **Stage B** → the **markdown for up to three meta tiles**, in this exact
  order, to append **after** all content tiles in the caller's single write:
  1. `### 📈 Digest tuning this week` — optional (weekly, only if tuned)
  2. `### 🔑 Important keywords` — optional (only if a real candidate exists)
  3. `### 🎛️ Tune your digest` — **always present**, the last tile of the whole
     digest
  Plus further config patches (`question_history`, `tuning_log`, `last_recap`,
  keyword fields) for the caller to persist.

The caller appends these tiles to its own `xtiles_create_tiles_*` call, then runs
the `tile-layout` pass over the full write. **Layout is not this workflow's job**
— tile placement lives entirely in `tile-layout`; the meta tiles participate in
that pass like any other tile, via the caller's `tile_ids`.

---

## Config fields

Carried in the caller's scheduled-task prompt, all optional/empty until first
used:

- `pinned_topics` — topics confirmed to always get their own tile, stored with the date they last had real content, optionally followed by a `weekday_only` flag (category 4), e.g. `TopicA(2026-07-05), TopicB(2026-07-08,weekday_only)`. The caller keeps splitting each of these out into its own `###` tile automatically — unless it's flagged `weekday_only` and today is Saturday/Sunday, then skip it for today only. A pinned topic whose stored date is more than 3 days old is gone quiet — the caller skips its tile today, and that feeds a category-2 question here. **Soft cap: 6** (see the config-bloat guardrail).
- `trial_topics` — topics proposed via category 2, split into a one-off tile once, awaiting the tick that promotes them into `pinned_topics` (or drops them if left unticked). **At most 2 pending at once.**
- `detail_sections` — sections given extra detail, stored with the date added, e.g. `Calendar(2026-07-05)`. One extra sentence of context every day. Older than 14 days → eligible for a category-3 "revert?" question.
- `secondary_channels` — channels de-prioritized (category 1's "lower priority" action, as opposed to full removal): still fetched, but folded into one compressed line instead of full detail.
- `digest_format` — `full` (default) or `top5` (category 3).
- `tone_style` — `default` or `concise` (category 3).
- `signal_trail` — structural signals accumulated over the last 14 days, e.g. `topic-scattered:AI:2026-06-25|2026-06-29|2026-07-03`. Prune any date older than 14 days every run. Looks past yesterday.
- `question_history` — last-asked date per category, e.g. `1:2026-06-20;3:2026-07-01`. Drives rotation and cooldown in Stage B.
- `highlighted_keywords` — confirmed important names/projects/companies, stored with the date last seen, e.g. `Andrew(2026-07-05), PluginLaunch(2026-07-08)`. Every occurrence across any tile today gets wrapped in `**bold**`. Update the date whenever a keyword actually appears today. Unseen for 10+ days → drop it silently. **Hard cap 10:** if confirming a new keyword would exceed 10, silently drop the entry with the oldest last-seen date to make room.
- `keyword_cooldown` — declined keywords with the date declined, e.g. `PluginLaunch(2026-07-01)`. Prune entries older than 14 days every run.
- `tuning_log` — a rolling 7-day log of the concrete changes applied each day, e.g. `2026-07-14:added #product|retired Crypto;2026-07-16:switched to top5`. Append today's applied changes with today's date every run; prune entries older than 7 days. Feeds the weekly recap. Looks past yesterday.
- `last_recap` — ISO week of the last weekly recap written, e.g. `2026-W29`. Prevents the recap from firing more than once per calendar week.

---

## Stage A — Read & apply yesterday's feedback

**When the caller runs this:** near the start of its flow, **before fetching
today's data** (it needs yesterday's page). On the very first digest ever
(`is_first_ever` = true) there's no prior page — **skip the read-and-apply below,
but everything else in this workflow still runs in full** (Stage B analyses
today's own data instead of yesterday's, and the `### 🎛️ Tune your digest` tile
is still written). "First digest ever" changes *what data Stage B looks at* — it
never means "skip the feedback tile."

Call `mcp__xtiles__xtiles_get_planner_content` once for **yesterday's** date
(`period: "day"`) and split the result:

- **Yesterday's content tiles** — every tile except the meta tiles (Emails, Slack, Workload, pinned-topic tiles, etc.). Read-only input for Stage B's structural analysis; never an answer source.
- **Yesterday's `### 🎛️ Tune your digest` tile**, if present — a checkbox list (`- [ ]` / `- [x]`). Each item the user **ticked** (`- [x]`) is a "yes"; each left unticked (`- [ ]`) is "not now". **Ignore the italic *Applied from yesterday* line at the top** — that's context this workflow wrote, never a question and never an answer.
- **Yesterday's `### 🔑 Important keywords` tile**, if present — same checkbox convention.

**Apply the feedback tile answers immediately:**
- Each item **ticked** (`- [x]`) → patch the config field its category maps to (see the 4 categories in Stage B).
- Each item **left unticked** (`- [ ]`) → don't apply it. Update `question_history` for that category to today's date with a short **7-day** cooldown — an untouched box is not a hard "no", so don't penalize it harshly.
- **Every box left unticked** (the tile wasn't touched at all, e.g. a scheduled run nobody watched) → treat every item as unticked with the short 7-day cooldown, never the harsh one.
- The tile is a pure checkbox list now — there is no free-text reply to parse. If the user *also* typed something free-form, read it as extra intent but never require it.
- No feedback tile found (first-ever digest, or the user removed it) → nothing to apply.

**Track what you actually applied.** As you patch config, keep a short
human-readable list of the concrete changes made — in the user's language,
verb-first, one clause each (e.g. "added #product", "retired Crypto", "switched
to top-5 format"). This list is used twice: (1) returned to the caller and shown
as the italic *Applied from yesterday* line at the top of today's feedback tile;
and (2) appended to `tuning_log` under today's date for the weekly recap. If
nothing was applied (no ticks, or first-ever digest) the list is empty — write no
applied-line and append nothing to `tuning_log`; never invent or pad it.

**Apply the keyword tile answers** (from the same page read):
- Each candidate **ticked** → add to `highlighted_keywords`, stamped with today's date (respecting the hard cap of 10).
- Each candidate **left unticked** → add to `keyword_cooldown` with today's date — don't re-propose that exact keyword for 14 days.
- No keyword tile found → nothing to apply.

**Never suggest a no-op** (used again in Stage B): never propose adding something
already in config, removing something not in config, or "more detail" for a
section already in `detail_sections` or one the user never selected.

**Config-bloat guardrail — keep the digest from growing forever:**
- `highlighted_keywords` self-limits (hard cap 10 + 10-day decay) — no question needed.
- `pinned_topics` has a soft cap of 6. At 6+, the single strongest **category-2 "retire one?"** question (targeting the quietest pinned topic by stored date) is forced to the top of Stage B's ranking and is exempt from cooldown — it keeps reappearing until the user retires something or the list drops below 6. Never auto-retire to enforce the cap; only propose. Only one such crowding question per digest.
- `trial_topics` hold at most 2 pending — if 2 already await a tick, don't propose a third new topic until one resolves.

**Stage A returns** the patched config and the applied-changes list.

---

## Stage B — Propose today's questions & compose the meta tiles

**When the caller runs this:** after it has composed today's content tiles, and
**before** its single write call — the meta-tile markdown gets appended to that
same write.

### B1 — Structural analysis (feeds the questions)

Read **yesterday's** content tiles (from Stage A) for structural signals. **On
the first-ever digest, run this same analysis against TODAY's own content** —
categories 1, 2, and 3 only need one day of data (a dominant source, a big topic
or new theme, a thin section), so day one is not signal-free. Only genuinely
cross-day checks (near-empty *repeatedly*, pinned-topic-quiet, detail-stale,
scattered-topic-over-14-days) are unavailable on day one. Look for:
- **Biggest tile** — notably longer than the others → category 2 (split it up).
- **Duplicate tiles** — substantial overlap → category 2 (merge).
- **Dominant source** — one channel/sender across multiple tiles → category 1 (promote).
- **Scattered topic** — mentioned across 2+ tiles without its own tile → log into `signal_trail` (`topic-scattered:` key, today's date). Becomes a strong category-2 candidate only once `signal_trail` shows 3+ distinct days within 14.
- **Near-empty tile** — 1 item or "No updates today," repeatedly → category 1/2.
- **Pinned topic gone quiet** (3-day rule) → category 2 — retire from `pinned_topics`.
- **`detail_sections` entry older than 14 days** → category 3 — keep or revert?
- **`pinned_topics` at or above its soft cap of 6** → highest-priority category-2 — propose retiring the quietest.

### B2 — The 4 question categories

Each is a family of related config actions; pick the one concrete action the
signal points to and vary the verb (add / remove / merge / split / retire / boost
priority / switch format / shorten) so repeats don't read like a template with
one variable swapped. Every question is yes/no only — a concrete proposal to tick
or leave, never "describe what's wrong":

1. **Sources** → patches the channel/newsletter list or `secondary_channels`. Add a source, remove one, or lower its priority — exactly one action per question. *"Source X came up several times today around topic Y — add it as its own channel?"* / *"Channel Z hasn't surfaced anything useful in the last 5 days — lower its priority?"*
2. **Structure** → patches `trial_topics` or `pinned_topics`. Split an overflowing topic into its own tile, track a genuinely new theme, merge two overlapping tiles, or retire a quiet pinned topic. *"Topic 'AI regulation' took up half of today's digest — split it into its own tile?"* / *"A new direction ('space') showed up across 3 sources today — track it as its own topic?"* / *"Tiles 'Crypto' and 'Fintech' keep overlapping — merge them?"* / *"Topic '[X]' hasn't come up in days — retire it from the pinned tiles?"*
3. **Detail & format** → patches `detail_sections`, `digest_format`, or `tone_style`. Add or revert extra detail on a section, switch to a shorter top-5 digest, or make the tone shorter and simpler. *"Topic X only gets headlines — add a short context sentence going forward?"* / *"The digest has been running long — switch to a 'top-5 + details on request' format?"* / *"Headlines have been long and formal lately — make them shorter and simpler?"*
4. **Schedule** → sets a `weekday_only` flag on the relevant `pinned_topics` entry. *"Activity on topic X is minimal on weekends — skip that topic on Saturday/Sunday?"*

Keyword highlighting is **not** one of these 4 — it has its own tile (B4).

### B3 — Select 3–5 questions

A hard range — never fewer than 3, never more than 5. **This tile is never
silently skipped, on any digest, including the first ever.**
1. Rank categories with a real B1 signal highest; break ties by oldest `question_history` date. **If the config-bloat guardrail is active (`pinned_topics` at 6+), its category-2 "retire one?" question outranks everything and always claims a slot.**
2. Skip categories currently in cooldown — unless that's the only way to reach the 3-question minimum, then break cooldown for the single oldest-cooldown category. The crowding question from step 1 is exempt from cooldown.
3. Still fewer than 3? **Day-one fallback — one per category, use verbatim (translated/adapted to the user's language), filling only as many as needed to reach 3:**
   - *"Want me to always add a specific channel or newsletter you check daily, even if it doesn't come up much yet?"* (category 1)
   - *"Want a particular topic always split into its own tile going forward?"* (category 2)
   - *"Should any section get more detail by default, or would you prefer a shorter 'top-5' digest?"* (category 3)
   - *"Want to skip low-activity topics on weekends?"* (category 4)

   Generic on day one **only** — every day after, real signals should crowd them out. Never use this fallback once `question_history` shows any category has real prior signal data.
4. Prefer one question per category. A second from the same category is allowed only when it's a genuinely different action (e.g. split one topic *and* merge two others) — never two near-identical asks.
5. Update `question_history` for every category actually asked today, stamped with today's date.

Personalize with the real name/topic/channel that triggered each question —
never generic filler. **Exception: the day-one fallback is deliberately generic.**

### B4 — Keyword candidates

Watch for a person, project, or company name that comes up **2+ times across at
least 2 different tiles today** — a single mention isn't enough. Skip any that's
already in `highlighted_keywords` or currently in `keyword_cooldown`. Cap at 5
candidates. **If zero candidates → omit the keyword tile entirely** — it has no
forced minimum, unlike the feedback tile.

### B5 — Weekly recap

Fires on the **first digest of each ISO week** — when today's ISO week differs
from `last_recap`. Normally Monday; if the schedule skipped Monday, the first run
of the week fires it. Set `last_recap` to today's ISO week the moment the recap
tile is composed. Read `tuning_log` and render each day's changes as a dated
line, plus a closing footprint line. **Omit the recap tile entirely** when
`tuning_log` is empty, on the first-ever digest, or when `last_recap` already
equals this week. Never fabricate changes to fill it.

---

## Meta-tile markdown formats

All meta tiles carry the standard annotations right after the heading (no blank
line between), `@colorSize: LIGHTER` and a `@color` picked from the caller's color
list without repeating the previous tile's color.

### 📈 Digest tuning this week (optional — B5)

Read-only prose, no checkboxes, never read back.

```
### 📈 Digest tuning this week
@colorSize: LIGHTER
@color: [pick from the color list]

**Mon 14 Jul** — added #product, retired Crypto

**Wed 16 Jul** — switched to top-5 format

---

Now tracking 4 pinned topics · 7 highlighted keywords
```
- One bold dated line per day that had changes, in the user's language/date format; blank line between them; skip days with no changes.
- Closing line reports the current footprint (`pinned_topics` and `highlighted_keywords` counts). Omit it if both are empty.
- Omit the whole tile per B5's skip conditions.

### 🔑 Important keywords (optional — B4)

A **checkbox list** — the user ticks the names that matter.

```
### 🔑 Important keywords
@colorSize: LIGHTER
@color: [pick from the color list]

- [ ] [Name/Project 1]

- [ ] [Name/Project 2]

Tick the names that genuinely matter to you
```
- Second-to-last tile, right before the feedback tile.
- Cap 5 candidates. Blank line between checkbox items. The hint line comes last.
- Read back in Stage A by checked state: `- [x]` = confirm → `highlighted_keywords`; `- [ ]` = decline → `keyword_cooldown`.
- **Applying confirmed keywords:** wrap every occurrence of a name/project/company from `highlighted_keywords` in `**bold**`, in every tile it appears in today. Don't double-wrap inside an existing bold span. If a tile is genuinely dominated by a highlighted keyword, prefer `BERMUDA` as its `@color` when the rotation allows — but never break the "no repeat color two tiles in a row" rule.

### 🎛️ Tune your digest (always — B3)

A **checkbox list** — the user ticks the changes they want applied to tomorrow's
digest. **Always the last tile of the whole digest, every run, no exceptions.**
Before finalizing, verify the write includes this tile; if not, go back to B3
(it must yield 3–5 items — use the day-one fallback if nothing else qualifies).

```
### 🎛️ Tune your digest
@colorSize: LIGHTER
@color: [pick from the color list]

*Applied from yesterday: added #product, retired Crypto*

- [ ] [Question 1 — concrete proposal, ends in "?"]

- [ ] [Question 2]

- [ ] [Question 3]

Tick the ones you want applied to tomorrow's digest
```
- **The italic *Applied from yesterday* line comes first**, listing the changes tracked in Stage A. **Omit it entirely** when nothing was applied (no ticks, an untouched tile, or the first-ever digest) — never write "Applied from yesterday: nothing" and never invent a change. Stage A's read-back ignores this line, so it never interferes with parsing ticks.
- 3–5 checkbox items, never fewer than 3, never more than 5. Blank line between every item; the hint line comes last.
- Read back in Stage A by checked state: `- [x]` = apply; `- [ ]` = not now.
- Never a separate chat widget, never a follow-up message — it's a tile in the same write as every other tile, and participates in the caller's `tile-layout` pass like any other tile.

---

## Return to the caller

- After **Stage A**: return the patched config + applied-changes list; the caller continues fetching today's data.
- After **Stage B**: return the ordered meta-tile markdown (📈 → 🔑 → 🎛️) and the remaining config patches. The caller appends the tiles after its content tiles, writes everything in one `xtiles_create_tiles_*` call, runs the `tile-layout` pass over the full write, and persists the updated config in its scheduled-task prompt.

Never message the user directly from this workflow.
