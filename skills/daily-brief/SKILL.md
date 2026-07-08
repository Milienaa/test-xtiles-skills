---
name: daily-brief
description: >
  Use when the user wants to set up OR run their xTiles Daily planner —
  a Daily page that serves as a live morning brief from connected tools
  (Slack, Gmail) plus signals that need attention.
  Only the Daily period is supported.

  Setup triggers: "set up my planner", "personalize my workspace",
  "connect my planner to my tools", "create daily",
  "onboard a new xTiles user into the Planner".

  Digest triggers: "show me my morning brief", "what do I need to know today",
  "run my digest". Also runs automatically via scheduled tasks.

  Config is read from the scheduled task prompt — no separate file needed.
  It also carries pinned_topics/trial_topics/detail_sections/secondary_channels/
  digest_format/tone_style/signal_trail/question_history/highlighted_keywords/
  keyword_cooldown, built up over time from answers to the "Tune your digest"
  and "Important keywords" tiles written at the end of each digest (see step
  4.0 and the tile formats in step 7).
  For manual runs: look for config in today's Planner; if there's none, start
  from the survey flow below.
allowed-tools: >
  mcp__xtiles__xtiles_get_planner_content,
  mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner,
  mcp__xtiles__xtiles_list_tasks,
  mcp__xtiles__xtiles_get_user_timezone,
  mcp__xtiles__xtiles_get_page_layout,
  mcp__xtiles__xtiles_set_page_layout,
  mcp__claude_ai_Slack__slack_search_channels,
  mcp__claude_ai_Slack__slack_search_public_and_private,
  mcp__claude_ai_Slack__slack_read_channel,
  mcp__claude_ai_Slack__slack_read_user_profile,
  mcp__claude_ai_Gmail__search_threads,
  mcp__claude_ai_Gmail__list_labels,
  mcp__claude_ai_Gmail__get_thread,
  mcp__claude_ai_Gmail__unlabel_message,
  mcp__claude_ai_Google_Calendar__list_events,
  mcp__claude_ai_Granola__list_meetings,
  mcp__claude_ai_Google_Drive__list_recent_files,
  mcp__claude_ai_Linear__list_issues,
  mcp__mcp-registry__suggest_connectors,
  anthropic-skills:schedule,
  mcp__scheduled-tasks__create-scheduled-tasks
---

# xTiles Daily Planner — Setup & Daily Digest

## Three principles

1. **Survey first, write to xTiles last.** Nothing gets created until the user has seen the preview and said "yes".
2. **Real data, not placeholders.** Pull from connectors before preview so the user sees live content.
3. **Match the user's language** throughout the entire flow — match the language of the user's first message and adapt if they switch.
4. **Every write to the planner is immediately followed by a layout pass — automatically, no exceptions.** The instant tiles are added in step 7, re-lay-out only those new tiles into a justified grid (step 7.3) *before* anything else — before the CTA button, before the schedule widget, before any message to the user. This never waits for the user to ask, is never skipped as "not needed this time," and is never left for a later run.

---

## Algorithm

**Period is always Daily.** At the start of the flow, tell the user: "I'll set up your **Daily** planner page — a live morning brief from your connected tools." Never ask which period to set up.

**Run mode — detect before step 1:**
- **Scheduled run**: the incoming message contains `role:`, `tools:`, and `daily_content:` (config injected by the `schedule` skill). Do not show the survey. Extract the config from the message. If a connector from the config is not detected — offer to walk the user through connecting it before continuing. **Skip steps 5 and 6 (preview and approval) — after the fetch, write directly to xTiles. Also skip the schedule widget in step 7 — the task is already scheduled.** Then jump to **step 4 (Silent data fetch)**.
- **Fast-track or fresh manual run**: proceed to step 1.

### 1. Fast-track

If the user is specific ("give me daily for today", "I want to see Slack in the morning") — skip the full survey. Minimum needed: which connectors to pull from — infer from the message, then check the detection table (step 2) to confirm which are actually available. If a required connector is not detected — offer to walk the user through connecting it (see **How to connect connectors**); wait for confirmation before proceeding. Pull only from connectors that are both mentioned and confirmed available. Jump to **step 4**.

If the request is general — run the full flow.
 
---

### 2. Survey — who are you and what's connected

**Before calling `show_widget`**: Make a lightweight test call to each connector's identifying MCP tool (e.g. `list_events` with `maxResults:1` for Calendar, `slack_search_channels` with query `general` for Slack — this is an auth check only, not channel discovery). For any connector that responds without an auth error, pre-select its card in the widget HTML by setting `class="card sel"` **and matching `data-tool` attribute** (see Survey widget HTML below — the card's pre-selected visual state and its underlying `tools` value must both be set, or a user who deselects it will have the click silently do nothing and it'll leak back in). Generate the widget with those pre-selections applied, then call `show_widget`.

**The submitted `tools` list is authoritative from this point on.** If the user deselects a pre-selected/technically-connected tool (e.g. Calendar responds fine to the probe but the user unchecks it), it must not reappear anywhere downstream — not as a content option in step 3, not in the fetch in step 4, not as a tile in step 7 — regardless of what the detection probe found. "Detected" only controls whether a tool is *offered*; the user's actual selection controls whether it's *used*.

**Show the survey widget** (HTML form) in Cowork. In Claude Code (no Cowork environment), ask the same questions inline as plain text — role, tools, content preferences, schedule.

**Connected tools** (multi select, show all regardless of what's actually detected — this matches the actual Survey widget cards below, one-to-one):
- Slack
- Gmail
- Calendar
- Granola
- Linear
- Google Drive
- Other


After receiving answers — detect which MCP tools are actually available:

| Connector | Identifying MCP tools                                                                                           |
|-----------|-----------------------------------------------------------------------------------------------------------------|
| Slack     | `mcp__claude_ai_Slack__slack_search_channels`, `mcp__claude_ai_Slack__slack_read_channel`                      |
| Gmail     | `mcp__claude_ai_Gmail__search_threads`, `mcp__claude_ai_Gmail__list_labels`, `mcp__claude_ai_Gmail__get_thread`|
| xTiles    | `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`                                                  |
| Calendar  | `mcp__claude_ai_Google_Calendar__list_events`                                                                   |
| Granola   | `mcp__claude_ai_Granola__list_meetings`                                                                         |
| Google Drive | `mcp__claude_ai_Google_Drive__list_recent_files`                                                             |
| Linear    | `mcp__claude_ai_Linear__list_issues`                                                                            |

These connectors are external and optional — they are not shipped with this plugin. The user must connect them separately.

**For "Other" connectors named by the user** — treat them identically to the known connectors above: attempt detection via available MCP tools (use `ToolSearch` with the connector's name to find its MCP tools, since it won't be in the table above); if not detected, walk through connecting via `mcp__mcp-registry__suggest_connectors`. **Before starting the connection flow, say the connector name explicitly** (e.g. "I'll now connect Plaud for you"). After the connection flow completes, explicitly resume: "Plaud connected. Continuing with [full list of tools]…". Carry the full list of selected tools — including every custom connector — through every subsequent step. Never drop a custom connector that the user named, even during multi-step connection flows.

**Ask what to pull from a custom connector — don't guess.** Right after naming/connecting a custom connector, ask what content it should contribute to the Daily. Generate 2–3 concrete options from the connector's likely purpose plus an "Other" free-text option, e.g. for Plaud: "1) List of meetings · 2) Meetings + action points · 3) Other (describe)". Use `show_widget` with a small options list (same `.pill`/`.card` style as the rest of the survey) in Cowork, or ask inline in Claude Code. Store the answer as part of that connector's config entry (e.g. `Plaud:meetings+action_points`) — this is what step 4's generic custom-connector fetch (below) uses to know what to pull.

**A custom connector is only "connected" once it can actually produce a tile — connecting auth is not enough on its own.** See step 4 for the generic fetch/write instructions that make this concrete; a custom connector that has no fetch path by the time step 4 runs must be surfaced as an error (per step 4's rule below), never silently dropped.

**If xTiles is not connected** — do not continue. Immediately walk the user through connecting xTiles (see **How to connect connectors** below). Wait for confirmation that xTiles is connected before proceeding.

**If a connector the user selected isn't connected** (Gmail, Slack, etc.) — immediately walk them through connecting it step by step. Do not move to the next step until they confirm it's connected or explicitly choose to skip that connector.

---

### 3. Daily content clarification

Question: "What do you want to see on your Daily each morning?"

Options — include only those relevant to connected tools:
- Unread emails that need a reply *(only if Gmail connected)*
- Newsletters — curated summaries from your subscriptions *(only if Gmail connected)*
- Slack messages from key channels *(only if Slack connected)*
- Workload — calendar analysis: day shape, conflicts, focus windows *(only if Calendar connected)*
- Other (describe in next message)

Do NOT suggest tasks — they're already in xTiles by default.

**If "Unread emails that need a reply" is selected — ask one follow-up:** "Should I mark ⚪ Noise emails (notifications, automated alerts — nothing to act on) as read automatically, so your inbox count reflects what actually needs attention?" Yes/No. Store the answer as `mark_noise_read: yes` or `mark_noise_read: no` in the config (default `no` if unanswered — never mark emails read without explicit opt-in). See step 4 for how this is applied.

**If Slack is selected and the user has not already named their channels:**

The goal is to surface *this specific user's* channels — the ones relevant to their profession and the ones they actually use — not a generic company list. Two independent signals drive this: **relevance-by-topic** (does the channel match their role?) and **activity-by-usage** (does the user actually post/get mentioned there?). The final list is their union; channels in both are the strongest picks and are labelled **core**.

**Step A — establish the role anchor.** Call `mcp__claude_ai_Slack__slack_read_user_profile` with **no `user_id`** — it returns the current user's own profile. Take the `title` field (and `department` if present) as the primary anchor. Combine it with the role captured in the survey form: if they agree, you have a strong anchor; if the form role is more specific, prefer it; if the profile is empty, fall back to the form role alone.

**Step B — expand the role into keywords.** A profession rarely matches a channel name word-for-word, so expand the title/role into 5–8 related terms before searching: the role name itself, common abbreviations, adjacent technical terms, and the typical tools/processes of that profession. Generate these from the anchor (do not rely on a fixed table). Examples of the mapping to produce:

- **AI Engineer** → ai, ml, llm, agent, automation, mcp, bot, gpt
- **Product Manager** → product, roadmap, feature, launch, growth, roundup
- **Designer** → design, ui, ux, figma, brand
- **Backend / DevOps** → infra, backend, deploy, ci, incidents, oncall
- **Marketing** → marketing, growth, campaigns, seo, content

If the user explicitly named topics or interests, add those to the keyword set first.

**Step C — universal channels (every role).** Call `mcp__claude_ai_Slack__slack_search_channels` for each of these names: `general`, `all`, `team`, `company`, `announcements`, `product`. Collect every channel that actually exists — these are candidates for the shared/general slot, never for the specialized slot.

**Step D — keyword search (relevance-by-topic).** For each keyword from Step B, call `mcp__claude_ai_Slack__slack_search_channels` with `query=<keyword>` and `channel_types="public_channel,private_channel"`. Private channels in results are already ones the user belongs to (the API filters by token access), so no extra membership check is needed. `slack_search_channels` returns at most 20 per call — paginate via `cursor` when a keyword yields more. Collect all results and dedupe by channel ID. Then run one broader pass for interest/affinity-group channels that exist regardless of role — e.g. `women`, `parents`, `wellness`, `book club`, `volunteering`, `pride`, `remote`, `pets`, or other terms the role context suggests. Score each candidate by boolean signals (no weighting needed): channel **name** contains a keyword (strong); **purpose/topic** mentions a keyword (strong); channel was **created by the user** (strong — it's "their" topic); type is `private_channel` (moderate — profile-specific work channels are more often private than open ones).

**Step E — activity signal (activity-by-usage, the strongest relevance signal).** Separately from keyword search, measure where the user actually writes. Call `mcp__claude_ai_Slack__slack_search_public_and_private` with query `from:<@USER_ID>` (the user's own ID from Step A's profile), `sort="timestamp"`, `limit=20`, paginating through `cursor` for 3–5 pages (≈60–100 recent messages); narrow with `after:YYYY-MM-DD` if a specific period is wanted. Also run `to:me` to catch channels where the user is @mentioned or replied to. Per channel, track **recency** (timestamp of the user's most recent post/mention) and **frequency** (total hit count), and flag DMs / group DMs (personal) separately from named channels (team activity). Recency is the primary signal — a channel the user posted in yesterday outranks one with more total hits but nothing in weeks; frequency only breaks ties between similarly-recent channels.

**Step F — merge, rank, and label.**
1. Merge every channel from Steps C–E, removing duplicates (a channel found in more than one step counts once, keeping its highest score).
2. Drop low-signal: name contains `random`, `fun`, `off-topic`, `bots`, `test`, `hiring`, `onboarding`. Exclude DMs / group DMs from the channel list.
3. Rank, in this order: **Step E activity first** (by recency, then frequency), **then** relevance-by-topic from Step D (keyword/created-by-user matches), **then** bare universal presence from Step C alone.
4. Label each channel: **core** if it appears in *both* Step D (topic-relevant) and Step E (actually used) — these are the strongest picks; **relevant-by-topic** if only Step D; **active-by-usage** if only Step E, even when it never matched a keyword (high activity is itself a signal of importance).
5. Show every remaining discovered channel as a selectable card — cast a wide net across role, interests, and affinity groups so the user has real options, not a token list.
6. **Pre-select (mark active) up to 5 total, no more:** all **core** channels first, then the highest-scoring **active-by-usage**, then **relevant-by-topic**, capped so at most 2 come from the universal slot (highest-scoring first, e.g. general → all → team → announcements → company → product). Every other discovered channel stays visible but unchecked — the user decides what else matters each morning.

**Fallback:** if Steps D and E both return zero results — call `mcp__claude_ai_Slack__slack_search_public_and_private` with query `team update` and extract channels from those results. If keyword search returns too few channels, widen the Step B keyword set with more synonyms for the role.

Generate an HTML multi-select widget with the discovered channels as selectable cards — mark the up-to-5 chosen in Step F.6 with the pre-selected `sel` state — and call `show_widget`. Include a free-text input for unlisted channels. Use `sendPrompt()` to submit. Template (inject one card per discovered channel; add `sel` to the class and keep `data-v` in sync for the pre-selected ones):

```html
<style>
:root{--c-surface:#fff;--c-text:#1a1a1a;--c-border:#e0e0e0;--c-border2:#aaa;--c-btn-p:#1a1a1a;--c-btn-p-text:#fff}
@media(prefers-color-scheme:dark){:root{--c-surface:#2c2c2c;--c-text:#f0f0f0;--c-border:#3d3d3d;--c-border2:#666;--c-btn-p:#f0f0f0;--c-btn-p-text:#1a1a1a}}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:transparent}
.wrap{max-width:480px;margin:0 auto;background:var(--c-surface);border-radius:16px;padding:24px;box-shadow:0 2px 12px rgba(0,0,0,.12)}
h2{font-size:15px;font-weight:700;margin-bottom:14px;color:var(--c-text)}
.cards{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:12px}
.card{padding:6px 14px;border-radius:20px;border:1.5px solid var(--c-border);font-size:13px;cursor:pointer;background:var(--c-surface);color:var(--c-text);user-select:none;transition:all .15s}
.card:hover{border-color:var(--c-border2)}
.card.sel{background:var(--c-btn-p);color:var(--c-btn-p-text);border-color:var(--c-btn-p)}
.grp{flex-basis:100%;font-size:11px;font-weight:600;text-transform:uppercase;letter-spacing:.04em;color:var(--c-border2);margin:6px 0 -2px}
input{width:100%;padding:8px 12px;border:1.5px solid var(--c-border);border-radius:8px;font-size:13px;margin-bottom:14px;outline:none;background:var(--c-surface);color:var(--c-text)}
input:focus{border-color:var(--c-border2)}
.btn{width:100%;padding:11px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;background:var(--c-btn-p);color:var(--c-btn-p-text)}
</style>
<div class="wrap">
  <h2>Which channels do you open first each morning?</h2>
  <p style="font-size:12px;color:#888;margin:-8px 0 12px">Pre-selected: your most active and relevant channels. Add any others you want to see.</p>
  <div class="cards" id="ch">
    <!-- Group the discovered channels under the three Step F.4 labels, each preceded by a group heading. Omit any group that has no channels.
         <div class="grp">Core — relevant &amp; active</div>
         <div class="grp">Active — where you post most</div>
         <div class="grp">Relevant to your role</div>
         Inject one card per channel under its group: <div class="card[ sel]" data-v="#channelname" onclick="tog(this,'#channelname')">#channelname</div>
         Add " sel" to the class for the up-to-5 pre-selected channels from Step F.6 (all core first). -->
  </div>
  <input type="text" id="other-ch" placeholder="Other channel…">
  <button class="btn" id="sub-ch" onclick="submit()">Confirm</button>
</div>
<script>
var sel=new Set();
document.querySelectorAll('#ch .card.sel').forEach(function(el){sel.add(el.dataset.v)});
function tog(el,v){el.classList.toggle('sel');el.classList.contains('sel')?sel.add(v):sel.delete(v)}
function submit(){var b=document.getElementById('sub-ch');b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';b.textContent='⏳…';var o=document.getElementById('other-ch').value.trim();if(o)sel.add(o);sendPrompt('Selected channels: '+Array.from(sel).join(', '))}
</script>
```

**If Newsletters is selected:**
**Important:** if the user selected "Newsletters" in the survey widget (step 2), this discovery flow must still run — do not skip it because newsletters was pre-selected there. The survey captures the preference; this step discovers the actual sources.

First, silently call `mcp__claude_ai_Gmail__search_threads` with query `from:(@substack.com OR @beehiiv.com OR @convertkit.com OR @mailchimp.com) newer_than:30d` to discover newsletters already in the inbox. Extract unique sender/publication names from results.

If publications found — call `show_widget` with an HTML multi-select listing the discovered newsletters as selectable cards. Template (inject one card per found publication):

```html
<style>
:root{--c-surface:#fff;--c-text:#1a1a1a;--c-border:#e0e0e0;--c-border2:#aaa;--c-btn-p:#1a1a1a;--c-btn-p-text:#fff}
@media(prefers-color-scheme:dark){:root{--c-surface:#2c2c2c;--c-text:#f0f0f0;--c-border:#3d3d3d;--c-border2:#666;--c-btn-p:#f0f0f0;--c-btn-p-text:#1a1a1a}}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:transparent}
.wrap{max-width:480px;margin:0 auto;background:var(--c-surface);border-radius:16px;padding:24px;box-shadow:0 2px 12px rgba(0,0,0,.12)}
h2{font-size:15px;font-weight:700;margin-bottom:14px;color:var(--c-text)}
.cards{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:12px}
.card{padding:6px 14px;border-radius:20px;border:1.5px solid var(--c-border);font-size:13px;cursor:pointer;background:var(--c-surface);color:var(--c-text);user-select:none;transition:all .15s}
.card:hover{border-color:var(--c-border2)}
.card.sel{background:var(--c-btn-p);color:var(--c-btn-p-text);border-color:var(--c-btn-p)}
.grp{flex-basis:100%;font-size:11px;font-weight:600;text-transform:uppercase;letter-spacing:.04em;color:var(--c-border2);margin:6px 0 -2px}
input{width:100%;padding:8px 12px;border:1.5px solid var(--c-border);border-radius:8px;font-size:13px;margin-bottom:14px;outline:none;background:var(--c-surface);color:var(--c-text)}
input:focus{border-color:var(--c-border2)}
.btn{width:100%;padding:11px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;background:var(--c-btn-p);color:var(--c-btn-p-text)}
</style>
<div class="wrap">
  <h2>Which newsletters do you want in your Daily?</h2>
  <div class="cards" id="nl">
    <!-- inject: <div class="card" onclick="tog(this,'Publication Name')">Publication Name</div> -->
  </div>
  <input type="text" id="other-nl" placeholder="Other newsletter…">
  <button class="btn" id="sub-nl" onclick="submit()">Confirm</button>
</div>
<script>
var sel=new Set();
function tog(el,v){el.classList.toggle('sel');el.classList.contains('sel')?sel.add(v):sel.delete(v)}
function submit(){var b=document.getElementById('sub-nl');b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';b.textContent='⏳…';var o=document.getElementById('other-nl').value.trim();if(o)sel.add(o);sendPrompt('Selected newsletters: '+Array.from(sel).join(', '))}
</script>
```

If nothing found — call `show_widget` with a simple text input widget asking the user to name the newsletters they want to track.

Add all selected/typed senders to the config. Tip: newsletters typically come from `@substack.com`, `@beehiiv.com`, `@convertkit.com`, `@mailchimp.com`.

**General rule:** if the user writes something custom — add it as-is. Don't reshape it into a predefined option.

---

### 4. Silent data fetch

**4.0 Read yesterday's digest and apply its feedback tile** (skip the read/apply parts only on the very first digest ever — nothing to read yet; still do 4.0d by pure rotation).

Call `mcp__xtiles__xtiles_get_planner_content` once for yesterday's date (`period: "day"`) and split the result into two things that are used very differently:
- **Yesterday's content tiles** — every tile except the feedback tiles (Emails, Slack tiles, Workload, Linear, Google Drive, custom connectors, etc.). This is read-only input for 4.0b's structural analysis; never treated as an answer source.
- **Yesterday's feedback tile** — the single `### 🎛️ Tune your digest` tile, if present. It's a numbered list of 3–5 questions followed by a prompt line asking the user to write which numbers they want. The reply is **whatever text exists in the tile beyond the original numbered items and prompt line** — the user may type it directly below the prompt, but they might also edit inline or add it elsewhere on the same tile. Diff against what was originally written; don't assume a fixed position. Read only for that reply, never re-analysed for new signals.

**Apply yesterday's feedback tile answers immediately, before anything else in this step:**
- Parse the user's free-text reply (in whatever language they wrote it) for referenced item numbers — digits ("1, 3"), ordinal words ("the first and third"), "all" (treat every item as mentioned), or a clear paraphrase of a specific question's content. Be liberal about matching intent, but when a mention is genuinely ambiguous, don't apply it.
- Each numbered item **mentioned** in the reply → patch the config field its category maps to (see the 9 categories in 4.0c).
- Each numbered item **not mentioned** → don't apply it. Update `question_history` for that category to today's date with a short 7-day cooldown — silence isn't a hard "no," so don't penalize it as harshly as an explicit decline.
- If the reply explicitly declines everything (e.g. "none", "not now") → same as above for every item, but use the longer 14–28-day cooldown (21 default) for all of them — this is an explicit signal, not silence.
- No reply at all (blank, or the tile wasn't edited since it was written) → treat exactly like "not mentioned" for every item — the short 7-day cooldown, not the harsh one. A scheduled run with nobody watching shouldn't be penalized as a rejection.
- No feedback tile found (first-ever digest, or the user removed it) — nothing to apply; proceed straight to today's fetch.

**Config fields carried in the scheduled task prompt** (alongside role/tools/daily_content), all optional/empty until first used:
- `pinned_topics` — topics confirmed to always get their own tile, stored with the date they last had real content, optionally followed by a `weekday_only` flag (category 8), e.g. `TopicA(2026-07-05), TopicB(2026-07-08,weekday_only)`. Keep splitting each of these out into its own `###` tile today automatically — unless it's flagged `weekday_only` and today is Saturday/Sunday, in which case skip it for today only. A pinned topic whose stored date is more than 3 days old is gone quiet — skip its tile today and let that feed a category-4 question in 4.0c.
- `trial_topics` — topics proposed via category 3 or 6 (4.0c), split into a one-off tile once, awaiting the yes/no that promotes them into `pinned_topics` (or drops them if declined).
- `detail_sections` — sections given extra detail, stored with the date added, e.g. `Calendar(2026-07-05)`. One extra sentence of context every day. Older than 14 days → eligible for a category-5 "revert?" question.
- `secondary_channels` — channels de-prioritized (category 2's "lower priority" action, as opposed to full removal): still fetched, but folded into one compressed line instead of full detail.
- `digest_format` — `full` (default) or `top5` (category 7).
- `tone_style` — `default` or `concise` (category 9).
- `signal_trail` — structural signals accumulated over the last 14 days, e.g. `topic-scattered:AI:2026-06-25|2026-06-29|2026-07-03`. Prune any date older than 14 days every run. **This is the only field that looks past yesterday** — everything else in this step reads yesterday alone.
- `question_history` — last-asked date per category, e.g. `1:2026-06-20;3:2026-07-01`. Drives rotation and cooldown in 4.0d.
- `highlighted_keywords` / `keyword_cooldown` — driven by the separate **keyword tile** cycle, not the 9 categories above. See 4.0e.

**Matching topics across days:** topic names are free text, so the same topic may be phrased slightly differently day to day (e.g. "API migration" vs "migrating to the new API"). Match by meaning, not exact string.

**Always check current config membership before proposing anything — never suggest a no-op:**
- Never suggest adding a channel, newsletter, topic, or keyword that's already in the relevant config field (`pinned_topics`/`trial_topics`/`highlighted_keywords`/channel list).
- Never suggest dropping/removing a channel, newsletter, topic, or keyword that isn't currently part of the config — nothing to remove.
- Never suggest "give more detail" for a section already in `detail_sections`, or for a section the user didn't select at all.

**4.0b Analyse yesterday's content tiles for structural signals** (skip entirely on the first-ever digest). Read the actual tiles yesterday's digest wrote — the already-composed output, not raw Slack/Gmail data — and look for:
- **Biggest tile** — notably more items/length than the others → category 3 candidate (split it up).
- **Duplicate tiles** — two tiles whose content substantially overlaps → category 4 candidate (merge).
- **Dominant source** — one channel/sender appearing across multiple tiles/sections → category 1 candidate (promote it).
- **Scattered topic** — mentioned in passing across 2+ tiles without its own tile → log it into `signal_trail` under a `topic-scattered:` key with today's date. Only becomes a strong category-3/6 candidate once `signal_trail` shows it on 3+ distinct days within the last 14 days — one day's mention is too weak to act on alone.
- **Near-empty tile** — only 1 item, or "No updates today," repeatedly → category 2/4 candidate.
- **Pinned topic gone quiet** (per 4.0's 3-day rule) → category 4 candidate — retire it from `pinned_topics`.
- **A `detail_sections` entry older than 14 days** (per 4.0) → category 5 candidate — ask whether to keep the extra detail or revert to normal length.

**4.0c The 9 question categories.** Each maps to a config action; vary the verb inside a category (add / remove / merge / split / mute / boost priority / switch format) so repeats don't read like a template with one variable swapped. All nine are yes/no only — a concrete proposal to confirm or decline, never "describe what's wrong":

1. **Channels — adding** → patches the channel/newsletter list. *"Source X came up several times today in connection with topic Y — add it as its own channel?"*
2. **Channels — priority / noise** → patches `secondary_channels` (lower priority) OR the channel list (remove) — pick exactly one action per question, never both in the same yes/no. *"Channel Z hasn't surfaced anything useful in the last 5 days — lower its priority?"*
3. **Structure — new tile** → patches `trial_topics`. *"Topic 'AI regulation' took up half of today's digest — split it into its own tile?"*
4. **Structure — merge / simplify / retire inactive** → either merges two topics/tiles into one `pinned_topics` label, or retires a `pinned_topics` entry that's gone quiet. *"Tiles 'Crypto' and 'Fintech' have been partially overlapping this past week — merge them?"* / *"Topic '[X]' hasn't come up in several days — retire it from the pinned tiles?"*
5. **Depth of information** → patches `detail_sections`, in either direction. *"Topic X only gets headlines right now — add a short context sentence going forward?"* / *"Topic X has had expanded context for [N] days now — keep it, or revert to the terse format?"*
6. **New direction / theme** → patches `trial_topics`, same mechanism as category 3 but triggered by a genuinely new theme with zero prior config presence rather than an existing one overflowing. *"A new direction ('space') has shown up — it's never been in the config, but 3 sources covered it today. Add it as a tracked topic?"*
7. **Presentation format** → patches `digest_format`. *"The digest has been running long this past week — switch to a 'top-5 + details on request' format?"*
8. **Frequency / schedule** → sets a `weekday_only` flag on the relevant `pinned_topics` entry. *"Activity on topic X is minimal on weekends — skip that topic on Saturday/Sunday?"*
9. **Tone / delivery style** → patches `tone_style`. *"Headlines have been running long and formal lately — make them shorter and simpler?"*

Keyword highlighting is **not** one of these 9 — it has its own dedicated tile with its own read/propose/apply cycle. See "Keyword tile" below.

**4.0d Select 3–5 questions for today's feedback tile** (a hard range — never fewer than 3, never more than 5):
1. Rank categories with a real 4.0b signal highest; break ties by oldest `question_history` date.
2. Skip categories currently in cooldown (per 4.0) — unless that's the only way to reach the 3-question minimum, in which case break cooldown for the single oldest-cooldown category rather than shipping fewer than 3.
3. Fewer than 3 signal-backed categories? Fill the rest by rotation — least-recently-asked in `question_history` first. On the first-ever digest every category is unasked, so just take 3–5 in any stable order.
4. Never two questions from the same category in one tile.
5. Update `question_history` for every category actually asked today, stamped with today's date.

Never phrase a question as generic filler — personalize with the real name/topic/channel that triggered it.

**4.0e Keyword tile — read yesterday's proposals, then propose today's** (its own separate cycle, independent of 4.0a–4.0d; skip the read/apply half only on the first-ever digest).

*Read yesterday's answer* (from the same `xtiles_get_planner_content` call in step 4.0): find the `### 🔑 Important keywords` tile, if present — a numbered list of candidates followed by a prompt asking the user to write which numbers matter, plus whatever free text they typed below it.
- Parse the reply for referenced numbers (digits, ordinal words, or a clear paraphrase of the keyword itself).
- Each candidate **mentioned** → add it to `highlighted_keywords`, stamped with today's date.
- Each candidate **not mentioned**, or no reply at all → add it to `keyword_cooldown` with today's date — don't re-propose that exact keyword for 14 days. A fresh recurrence after the cooldown passes is fair game again.
- No keyword tile found → nothing to apply.

*Propose today's candidates* (while composing today's content in the main fetch below — not from yesterday's data): watch for a person, project, or company name that comes up **2+ times across at least 2 different tiles today** — a single mention in one place isn't enough. For each candidate, skip it if it's already in `highlighted_keywords` or currently in `keyword_cooldown`. Cap at 5 candidates.

**If zero candidates are found — omit the `### 🔑 Important keywords` tile entirely for today.** Unlike the "Tune your digest" tile, this one has no forced minimum — an organic signal that isn't there shouldn't be manufactured.

`highlighted_keywords` — confirmed important names/projects/companies, stored with the date last seen, e.g. `Andrew(2026-07-05), PluginLaunch(2026-07-08)`. Every occurrence across any tile today gets wrapped in `**bold**` (see step 7). Update the date whenever a keyword actually appears today. Unseen for 10+ days → drop it silently from `highlighted_keywords` (no question needed — this one decays quietly rather than asking, since re-adding it later costs nothing).

`keyword_cooldown` — declined keywords with the date declined, e.g. `PluginLaunch(2026-07-01)`. Prune entries older than 14 days every run.

**Silently, without messaging the user**, pull fresh data from connectors based on selected sections and content choices:

- **Gmail — unread emails**: `mcp__claude_ai_Gmail__search_threads` — query `is:important in:inbox newer_than:1d`. For each thread call `mcp__claude_ai_Gmail__get_thread` to get sender, subject, and threadId for the direct link (`https://mail.google.com/mail/u/0/#inbox/{threadId}`). `get_thread` returns the thread's individual messages — also capture each message's own `messageId` (not just the threadId), needed later for `mark_noise_read` (see below); a thread can contain several messages, each with a distinct `messageId`.
- **Gmail — newsletters**: `mcp__claude_ai_Gmail__search_threads` — query `from:({sender1} OR {sender2} ... OR @substack.com OR @beehiiv.com OR @convertkit.com) is:unread newer_than:1d` — combine user-named senders with common newsletter domains. Fetch each thread with `get_thread` for a one-line summary and `threadId` for the link.
- **Slack**: two parallel reads:
  1. `mcp__claude_ai_Slack__slack_read_channel` for each chosen channel (top 50 messages). Filter to last 24 hours (timestamp ≥ now − 24 h). Discard older messages. Skip channels with no messages silently.
  2. `mcp__claude_ai_Slack__slack_search_public_and_private` with query `to:me` to find messages where the user was @mentioned or DM'd. Filter results to last 24 hours. This covers both public and private channels, including ones not in the chosen list.

  After collecting, analyse all messages together and group semantically. For every item include a **direct permalink to the specific message** — extract `permalink` from the message object (or build `https://slack.com/archives/{channel_id}/p{ts_without_dot}`). Never link to the channel homepage — always to the individual message.

  - **Mentions** *(highest priority)* — all messages from the `to:me` search. For each: who mentioned the user, in which channel, what was asked or said — one line per mention, message permalink. If the mention requires a response — flag it as ⚡.
  - **Потребує дії** — every ⚡-flagged mention also becomes an item here: a short poke-style line (what's being asked, second person, same tone as email's 🔴 bucket) plus the message permalink. For each, derive one verb-first action item (e.g. "Відповісти Марії в #product"). These flow into the same task-dedup step as email action items below — collect them together, don't dedup separately.
  - **Topics** — what was discussed in channels; group by theme, one topic = one line, permalink to the most relevant message, channel attribution `[#channel](permalink)`
  - **Decisions** — where something was agreed, committed to, or confirmed — include message permalink
  - **Open questions** — where a question was raised but no clear answer came yet — include message permalink, mark as ⏳

- **Calendar**: `mcp__claude_ai_Google_Calendar__list_events` — today's events. For each event extract: start/end time, title, participant names (first name + last name or company), and meeting link (Google Meet, Zoom, or other video URL from event data). Compute:
  - **Summary line**: event count, total hours occupied, longest free focus window (HH:MM–HH:MM, duration in hours)
  - **Per-event row**: time range · title — participants list · [meeting link label](url) if present
  - **🧠 context** (only if Granola or Gmail connected): for each meeting, find the last Granola note involving the same participants and/or the most recent open Gmail thread with the organiser — write one sentence summarising what the meeting is about or what was discussed last time. Only include if relevant context is found; skip silently otherwise.
  - **⚠️ anomalies** — collect all, show at the bottom of the tile (not inline): overlapping events, back-to-back with no gap, events after 20:00, events without description/agenda, potential duplicate titles close together

- **Linear**: `mcp__claude_ai_Linear__list_issues` — issues assigned to or recently updated by the user, filtered to the last 24 hours (or since the last digest). For each: title, status, and the issue URL. Group into two lines: **New** (created in the last 24h) and **Updated** (status/comment changed in the last 24h). Skip silently if empty on a given day, but the tile itself follows the same "always create, write 'No updates today' if empty" rule as Slack Topics — its total absence would look like a connector failure.
- **Google Drive**: `mcp__claude_ai_Google_Drive__list_recent_files` — files modified in the last 24 hours. For each: file name, who last modified it, and the file's URL. One line per file, newest first. Omit the tile entirely if nothing changed (unlike Linear, a quiet Drive day is unremarkable and not worth calling out).
- **Custom ("Other") connectors** — this skill can't hardcode every possible tool, so every custom connector needs this generic path (this is the step that was previously missing and caused custom connectors to silently vanish from the digest):
  1. Use the MCP tools discovered for this connector during step 2's detection/connection (search with `ToolSearch` if needed to find the right "list recent items" style tool).
  2. Pull data matching the content preference collected in step 2 (e.g. `Plaud:meetings+action_points` → fetch recent meetings and their action points), scoped to roughly the last 24 hours unless the connector only supports a longer window.
  3. If the fetch succeeds — this connector gets its own `###` tile later (step 7), titled with the connector's name and a fitting emoji.
  4. **If the fetch fails, or no working MCP tool can be found for this connector at all** — do not silently drop it. Surface it explicitly in step 5's preview as "Could not fetch [Connector] data — connector error" (same rule as the built-in connectors below), and still write that line into the digest in step 7 rather than omitting the connector's tile entirely. The user must see that something they asked for didn't work, never see it just disappear.

Classify emails into three buckets. **Newsletters are fetched separately — exclude them here entirely and do not count them in any bucket.**

- 🔴 **Потребує дії** — emails where the user must take a concrete next step (reply, decide, act, log in)
- 🟡 **До уваги** — FYI only: confirmed meetings, signed documents, payments, status updates — past/present tense, nothing to do
- ⚪ **Шум** — notifications, automated alerts, service emails — do not describe individually; count only

**If `mark_noise_read: yes` is set in the config:** for every **thread** placed in the ⚪ Шум bucket, call `mcp__claude_ai_Gmail__unlabel_message` with `labelIds: ["UNREAD"]` once per **individual message** inside that thread (using each message's own `messageId` captured above) — `unlabel_message` acts on one message at a time, and a thread with several unread messages needs one call per message, or only some (or none) of it will actually be marked read. Do this silently — don't ask again each run. Mention the count briefly in the preview/tile line, e.g. "⚪ Шум — N сповіщень (sources) — позначено прочитаними." If `mark_noise_read` is `no` or unset, leave messages untouched (current behavior — count only, don't act on them).

**Tone for 🔴 and 🟡 — Poke-style, capitalized:**
- Retell the email, do not copy the subject line. Subject → action → consequence in second person: not "Your account closed" but "Google закрив твій рекламний акаунт учора"
- For 🔴: weave the next step into the sentence: "Залогінься і віднови — вікно на апеляцію обмежене"
- Use people's names, not email addresses. Context in parentheses if needed: "Стефан (influencers.club)"
- Telegraphic, conversational. First letter capitalized, no bureaucratic language.
- 🟡 items are one-liners — no link needed.

For every 🔴 email, derive one verb-first action item (e.g. "Відновити рекламний акаунт Google"). Collect as a flat list, **combined with the verb-first action items derived from Slack's ⚡-flagged mentions** (see the Slack section's "Потребує дії" bullet above) — one shared list across both sources.

Then call `mcp__xtiles__xtiles_list_tasks` **once** with `completed: false` to fetch all open tasks. For each action item in the combined list, check if an open task with the same or very similar meaning already exists. **Keep only items that have no match** — email-derived ones go into the Emails tile's Action items block as `- [ ] Task`, Slack-derived ones go into the `### 🔴 Потребує дії` tile's Задачі block as `- [ ] Task` (see step 7). Silently drop items that already exist as open tasks, regardless of which source they came from.

Use only real data from connectors. Do not invent names, events, or messages.
All names and message content must come directly from API responses — never from examples in this skill file.

If a connector call fails (error, timeout, 401) — record the failure. Do not write "No data" for a failed call — surface the error explicitly in step 5 so the user knows the connector did not respond.

---

### 5. Preview — show content in chat

Show real content with real data. Not structure, not headings with "(TBD)" — actual text.

Format (adapt to selected pages):

```
Here's what I've prepared:

---
📅 DAILY — [actual date]

### Emails
🔴 Потребує дії (N)
- [Poke-style description — 1–2 sentences, second person, action + consequence]
  → [Відкрити лист](https://mail.google.com/mail/u/0/#inbox/{threadId})

- [Next 🔴 email, same format]
  → [Відкрити лист](https://mail.google.com/mail/u/0/#inbox/{threadId})

🟡 До уваги (N)
- [One-line item — no link]
- [One-line item]

⚪ Шум
- N сповіщень (sources) — нічого термінового

**Action items:**
- [ ] [verb-first task from 🔴 email 1]
- [ ] [verb-first task from 🔴 email 2]

*(omit Action items entirely if no 🔴 emails)*

### 📧 Newsletters

**[Newsletter Name](https://mail.google.com/mail/u/0/#inbox/{threadId})**
One-line summary.

**[Another Newsletter](https://mail.google.com/mail/u/0/#inbox/{threadId})**
One-line summary.

### 🔴 Потребує дії
- [Poke-style one-liner of what's being asked] — [#channel](url)

**Задачі**
- [ ] [verb-first task]

### ⚡ Mentions
- **@Name** in [#channel](url) — what they asked/said ⚡

### 💬 Topics
**Channels:** #channel1 (N) · #channel2 (N)
- **[Topic name]** — [one-sentence summary] — [#channel](url)

### ✅ Decisions
- [Decision] — [#channel](url)

### ❓ Open
- [Question] — [#channel](url) ⏳

### 📅 Workload
**N events · ~X h occupied · longest focus window HH:MM–HH:MM (X h)**

**HH:MM–HH:MM · Meeting name** — Participant1, Participant2 · [Google Meet](url)

🧠 [one sentence: meeting point / what will be discussed]

**HH:MM–HH:MM · Meeting name**

**HH:MM–HH:MM · Meeting name** — Participant from Company · [Google Meet](url)

🧠 [meeting point]

⚠️ [anomaly — e.g. two external calls back-to-back in the evening, 30 min gap between them]

### 🔑 Important keywords
*(only if at least one real candidate was found — see 4.0e)*
1. [Name/Project 1]
2. [Name/Project 2]

Write the numbers of the words that genuinely matter to you

### 🎛️ Tune your digest
1. [Question 1]
2. [Question 2]
3. [Question 3]

Write the numbers of the items you'd like applied to the next digest

---
```

Each 🔴 email uses the real `threadId` from `get_thread` for the [Відкрити лист] link. 🟡 items are one-liners with no link.
Each newsletter is shown as its own named section in the preview — never mixed into the Emails section.
Separate each item with a blank line for readability.

**Rules:**
- Show only selected sections the user asked for
- If a connector returned no data — write exactly that ("No unread emails", "No newsletters today", "No Slack updates today") — never skip the section silently; its absence looks like a bug
- If a connector call failed — write "Could not fetch [connector] data — connector error" (not "No data")
- No placeholder names, example events, or invented data — ever
- `### 🎛️ Tune your digest` is always the last section, regardless of which other sections are present — never reorder it earlier
- `### 🔑 Important keywords` sits right before it (second-to-last) whenever it's present — omitted entirely on days with no real candidate
- After the preview, **stop and wait**. Do not write anything to xTiles yet.
---

### 6. Approval

**Mandatory. Never skip this step.** After showing the preview, call `show_widget` with the **Approval widget HTML** (see below).

Do not call `xtiles_create_tiles_from_markdown_in_my_planner` until the user explicitly clicks **"Looks good — create it"**.

If the user asks for a change — clarify exactly what, update only that section, re-show preview, ask again.

---

### 7. Write to xTiles

**Only after explicit approval.**

Tool: `mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner`
- `period`: "day"
- `date`: current date in ISO 8601

**Write all sections in a single call.** Combine all selected connectors (Gmail, Slack, etc.) into one markdown and call the tool once — never split into separate calls per connector.

**Do NOT create a date/header tile.** Never write `### [date]` or any title-only tile as the first item — start directly with content tiles.

**Tile formatting** — each `###` section must include color and style annotations immediately after the heading (no blank line between):

```
### [emoji] [Title]
@colorSize: LIGHTER
@color: [COLOR]

[content]
```

- `@colorSize` is always `LIGHTER`
- `@color` — pick randomly for each section from this list **exactly as written**:
  `GHOST, CUMULUS, GOSSIP, COLDTURKEY, BLUE_CHALK, MILK_PUNCH, HAWKES_BLUE, PATTENS_BLUE, SAIL, ATHENS_GRAY, BERMUDA, PERFUME, SELAGO, RICE_FLOWER, WHITE_LINEN, POLAR`
  **CRITICAL: never use semantic color names (RED, BLUE, GREY, ORANGE, YELLOW, GREEN, etc.) — they will not render. Only the exact names from the list above.**
- Each section gets a different color — do not repeat the same color twice in a row

**Content formatting inside each tile:**
- **All links must be Markdown hyperlinks** — always `[text](url)`, never a bare URL. If you include a link, it must have a label.
- Separate each item with a blank line — never write items as a continuous block
- **Emails**: structure the tile in three labeled blocks followed by action items:
  ```
  🔴 **Потребує дії (N)**

  - [Poke-style description — 1–2 sentences, second person, action + consequence]
    → [Відкрити лист](https://mail.google.com/mail/u/0/#inbox/{threadId})

  - [Next 🔴 item, same format]
    → [Відкрити лист](url)

  🟡 **До уваги (N)**

  - [One-line item — no link, never]

  - [One-line item]

  ⚪ **Шум**

  - N сповіщень (sources) — нічого термінового

  ---

  **Action items**

  - [ ] [Verb-first task from 🔴 email 1]
  - [ ] [Verb-first task from 🔴 email 2]
  ```
  Omit `Action items` section entirely if no 🔴 emails. Newsletters are in the separate `### 📧 Newsletters` tile — never include them here.
- **Newsletters**: ALL newsletters go in a **single `### 📧 Newsletters` tile** — never create a separate tile per newsletter. Structure:
  - Each newsletter as a bold hyperlink title followed by a one-line summary on the next line:
    ```
    **[Newsletter Name](https://mail.google.com/mail/u/0/#inbox/{threadId})**
    One-line summary.
    ```
  - Blank line between entries.
  - The link IS the title — no separate "Open" button or link at the bottom of each entry.
  - Omit the entire tile only if there are no unread newsletters at all.
- **Slack**: split into **separate tiles per category** — never one big tile. Each tile uses `###` as its header. All Slack links must point to the specific message permalink, never to the channel homepage.
  - `### 🔴 Потребує дії` (Slack) — the actionable subset: one line per ⚡-flagged mention: `- [Poke-style one-liner of what's being asked] — [#channel](message_permalink)`. Below that, a `**Задачі**` block with one verb-first checkbox per item: `- [ ] [verb-first task]` (e.g. "Відповісти Марії в #product"). These tasks go through the **same open-task dedup step as email action items** (see step 4 — call `xtiles_list_tasks` once, covering both sources, keep only items with no existing match). **Omit tile entirely if no ⚡ mentions today.** This tile is a rollup, not a replacement — the same messages still appear in `### ⚡ Mentions` below for full context.
  - `### ⚡ Mentions` — one line per mention: `- **@Name** in [#channel](message_permalink) — what they asked/said`. Add ` ⚡` if a response is needed. **Omit tile entirely if no mentions.**
  - `### 💬 Topics` — first line: `**Channels:** #channel1 (N) · #channel2 (N)`. Then one line per topic: `- **Topic name** — one-sentence summary — [#channel](message_permalink)`. **Always create this tile** — if no messages today, write a single line: `No updates today.` Its absence looks like a connector failure.
  - `### ✅ Decisions` — one line per decision: `- Decision made — [#channel](message_permalink)`. **Omit tile entirely if no decisions.**
  - `### ❓ Open` — one line per unanswered question: `- Question — [#channel](message_permalink) ⏳`. **Omit tile entirely if no open questions.**
- **Calendar**: tile titled `### 📅 Workload`. Use this exact structure:
  ```
  ### 📅 Workload
  @colorSize: LIGHTER
  @color: [pick randomly from the color list]

  **N events · ~X h occupied · longest focus window HH:MM–HH:MM (X h)**

  **HH:MM–HH:MM · Meeting name** — Participant1, Participant2 · [Google Meet](url)

  🧠 [one sentence: meeting point / what will be discussed]

  **HH:MM–HH:MM · Meeting name**

  **HH:MM–HH:MM · Meeting name** — Participant from Company · [Zoom](url)

  🧠 [meeting point]

  ⚠️ [anomaly]
  ```
  Rules:
  - Summary line is bold, always first
  - Each event on its own bold line: `**HH:MM–HH:MM · Title**` — append ` — Participants · [Link label](url)` if participants or meeting link exist
  - 🧠 goes on the next paragraph directly under its event — only if Granola or Gmail context was found; omit otherwise. Use Markdown hyperlinks for Granola note and Gmail thread: `[Note title](url)` + `[Thread subject](gmail-url)`
  - All ⚠️ anomalies collected at the bottom, one per line
  - Blank line between every item (event, 🧠, ⚠️) for readability
  - Omit tile entirely if Calendar returned no events
- This ensures the tile is scannable, not a wall of text
- **Linear**: tile titled `### 📌 Linear`. Two labeled lines: `**New**` followed by one item per newly created issue (`- [Title](url) — status`), then `**Updated**` the same way for issues that changed status/got comments. **Always create this tile** if Linear is selected — if nothing happened, write a single line: `No updates today.` (same rule as Slack Topics — its absence would look like a connector failure).
- **Google Drive**: tile titled `### 📁 Google Drive`. One line per changed file: `- [File name](url) — edited by Name`. Blank line between entries. Omit the tile entirely if nothing changed in the last 24 hours.
- **Custom ("Other") connectors**: one `###` tile per connector, titled with its name and a fitting emoji (e.g. `### 🎙️ Plaud`). Structure the content to match the preference collected in step 2 (meetings list, meetings + action points, etc.) — one item per blank-line-separated entry, Markdown hyperlinks where a URL exists, same `@colorSize`/`@color` annotation as every other tile. **If the connector's fetch failed or had no working MCP tool** (see step 4), still create this tile with a single line: "Could not fetch [Connector] data — connector error." Never omit the tile silently — its disappearance is indistinguishable from the connector never having been asked for in the first place.
- **Keyword tile — second-to-last, only when 4.0e found at least one real candidate.** Titled `### 🔑 Important keywords`, as a **numbered list plus a free-text prompt** (never checkboxes):
  ```
  ### 🔑 Important keywords
  @colorSize: LIGHTER
  @color: [pick randomly from the color list]

  1. [Name/Project 1]

  2. [Name/Project 2]

  Write the numbers of the words that genuinely matter to you
  ```
  - Omit this tile entirely on a day with zero candidates — no forced minimum, unlike the feedback tile below.
  - Cap 5 candidates. Blank line between numbered items. The prompt line always comes last, after a blank line.
  - Next digest reads this exact tile back in step 4.0e — whatever the user types on the page below the prompt line is the reply; don't reword the structure into something the read-back can't parse.
  - **Applying confirmed keywords**: wrap every occurrence of a name/project/company from `highlighted_keywords` in `**bold**`, in every tile it appears in today (Emails, Slack, Workload participant names, Linear, Google Drive, custom connectors — anywhere the exact keyword shows up in running text). Don't double-wrap a keyword that's already inside an existing bold span. If a tile's content is genuinely dominated by a highlighted keyword (not just a passing mention), prefer `BERMUDA` as that tile's `@color` when the rotation allows it, so the same "this matters" color becomes recognizable over time — but never break the "no repeat color two tiles in a row" rule to force it.
- **Feedback tile — always the last tile in the write, every digest from the first one on.** Titled `### 🎛️ Tune your digest`, as a **numbered list plus a free-text prompt** (never checkboxes) containing exactly the 3–5 questions selected in step 4.0d:
  ```
  ### 🎛️ Tune your digest
  @colorSize: LIGHTER
  @color: [pick randomly from the color list]

  1. [Question 1 — concrete proposal, ends in "?"]

  2. [Question 2]

  3. [Question 3]

  Write the numbers of the items you'd like applied to the next digest
  ```
  - This is a **tile in the digest itself**, in the same `xtiles_create_tiles_from_markdown_in_my_planner` call as every other tile — never a separate chat widget, never a follow-up message. It participates in step 7's layout pass exactly like every other tile (via `tile_ids`) — no special handling needed there.
  - Always present, on every digest including the very first one (first-ever picks its 3–5 by pure rotation per 4.0d — there's simply no history to weight them by yet).
  - Never fewer than 3 items, never more than 5.
  - Blank line between every numbered item. The prompt line always comes last, after a blank line.
  - The next digest reads this exact tile back in step 4.0 — whatever the user types on the page below the prompt line is the reply, parsed for referenced item numbers. Don't reword or restructure it in a way that would make that read-back ambiguous.

**If xTiles is not connected** — do not output the digest as plain text in chat. Walk the user through connecting xTiles (see **How to connect connectors**), wait for confirmation, then write.

**If the page already exists:**
1. Call `mcp__xtiles__xtiles_get_planner_content`
2. Compare existing H3 headers (`###`) with what you're about to add
3. Append only sections whose headers don't exist yet
4. If everything already exists — ask: replace all, append anyway, or cancel?

**After each successful write — run these steps in order, no exceptions. Step 3 (the layout pass) is not optional and is never deferred, asked about, or judgment-called away — it runs automatically, immediately after every single write, before step 4's CTA button is even composed:**

1. Write `✅ Daily created.`
2. Read `view_id` and `tile_ids` **directly from the response of the write call in this step** (`xtiles_create_tiles_from_markdown_in_my_planner` returns both). `tile_ids` is ordered to match the `###` sections in the markdown you just wrote — `tile_ids[i]` is the tile created for the *i*-th section. **Do not** re-derive `view_id` via `get_planner_content`. This list is only an identity key for step 3 — it tells you *which* tiles are new; it is not a substitute for reading the page's layout.
3. **Justified-grid layout pass — mandatory, silent, automatic, every single run (scheduled runs included, fast-track included, any tile count included).** Newly written tiles land at default sizes that rarely fit their content. Immediately re-lay-out **only the tiles in `tile_ids` from step 2** so each row of them ends flush at one bottom line and every tile's width/height is proportional to its content. Do not message the user about this pass, do not ask for confirmation, and do not decide to skip it because the content looks "simple enough" or the math looks tedious — always run steps 3.1–3.9. The **only** permitted skip is the last-resort case at the end of step 3.9, after both the primary call and its fallback retry have been attempted and rejected.

   1. Call `mcp__xtiles__xtiles_get_page_layout` with the `view_id` — this is mandatory and gives you the full picture of the current page: the grid bounds (`max_width`, `max_height`, `min_tile_width`, `min_tile_height`) and **every tile on the page** with its `tile_id`/`x`/`y`/`w`/`h`, new and pre-existing alike. Coordinates are grid cells: `(x, y)` = top-left corner (0-based), `(w, h)` = size. Now split that full tile list by `tile_id`: those matching the `tile_ids` set from step 2 are the **added** tiles — the only ones whose position/size you will change. Every other tile returned is **existing** — read its `x/y/w/h` only to treat it as a fixed obstacle; never modify it or include it in step 9's call.
   2. **Reuse the markdown you already composed** for the write in step 7 to measure each added tile's content — no need to re-fetch it. Parse each added tile's content into blocks, each one of: **header** (a `####` / bold subsection label — 1 line, width-independent); **short-line** (a one-line item that never wraps even at `min_tile_width` — checkbox, `**Channels:**` summary, short bullet — 1 line); **paragraph** (a longer block that wraps; its line count depends on width). If unsure, compare the item's character length to the line capacity at `min_tile_width`: fits in one line even there → short-line, else → paragraph.
   3. **Line capacity by width:** `chars_per_line(w) = (w * col_px − padding_px) / avg_char_px`, with heuristic constants `col_px ≈ 25`, `padding_px ≈ 48`, `avg_char_px ≈ 7.5` (mixed Cyrillic/Latin ~14px; calibrate against a real tile if you can). For a paragraph at width `w`: `lines = ceil(char_count / chars_per_line(w))`. Header/short-line = 1 line regardless of `w`.
   4. **Tile height at a width** (grid rows ≈ text lines): `H_tile(w) = Σ lines(block, w) + (#header blocks) + (#paragraph→paragraph gaps) + 2` — the `+2` is tile chrome (title + top/bottom padding), the header/gap terms are the blank lines that don't shrink with width. Round up to whole rows, clamp to `≥ min_tile_height`. Naive "height ∝ 1/width" fails because only paragraphs shrink with width.
   5. **Compose rows** from the added tiles in their existing top-to-bottom order (do not reshuffle content, do not pull in any existing tile). Place them in free grid space from the full-page layout (step 1) that doesn't overlap any existing tile — typically the first empty `y` below everything already on the page. Default to **2 tiles per row**; give a very heavy tile its **own full-width row** (`w = max_width`); avoid a lonely narrow tile when it can pair with a neighbor.
   6. **Distribute width in each row** across the full `max_width` (fill the band — no leftover gap), proportional to content weight (≈ paragraph char count): light tiles get just above `min_tile_width`; the rest of the columns go to the heavy tile(s) so they wrap less. Every tile stays `≥ min_tile_width`; widths sum to exactly `max_width`.
   7. **Equalize height (flush-bottom rule):** compute `H_tile(w)` at each tile's assigned width, then set **every tile in the row to `h = max(H_tile)`** so the row ends at one bottom line (empty space under a lighter tile is fine — dashboard look). Set `x` left→right (first tile at the row's left edge, next `x` = previous `x + w`), same `y` for the row, next row `y` = this row's `y + H`. If the max is dominated by one heavy tile, first try widening it (redo step 6) to pull `H` down. If a tile still doesn't fit: narrow its neighbor toward `min_tile_width` and give it the columns; if the neighbor is already at `min_tile_width`, raise `H` by exactly the shortfall.
   8. **Self-check before calling:** widths in each row sum to exactly `max_width`; every `w`/`h` is `≥ min_tile_width`/`min_tile_height`; no added-tile rectangle overlaps another added tile or any existing tile from step 1's full layout. Fix any violation yourself — don't rely on the server to catch it.
   9. Call `mcp__xtiles__xtiles_set_page_layout` once, listing **only the added tiles** (`tile_id` + new `x/y/w/h`) — the same set identified in step 1 by matching against `tile_ids`. Never include an existing tile; tiles you omit keep their current position automatically. **If the call is rejected**, retry once with a simpler fallback: rows of 2 added tiles at equal width (`max_width / 2` each, or the full `max_width` for a lone leftover tile) and, per row, a shared height equal to the tallest `H_tile` computed in step 4 for that row. Only if the retry also fails, skip silently and continue — a slightly-off layout beats a broken flow.
4. Call `show_widget` with the **CTA widget HTML** (see below), replacing `{VIEW_URL}` with `https://xtiles.app/{view_id}`. Translate the button label into the user's language. **Never output a markdown link instead of the widget.**
5. **For non-scheduled runs only**: Immediately call `show_widget` with the **Schedule widget HTML** (see below). Do not skip this step, do not ask first — just show it.

**If an error occurs:** briefly say what went wrong, offer to retry or skip that page.

---

### 8. Schedule (optional)

The schedule widget is shown in step 7 above. This step handles the user's response.

In Claude Code (no Cowork): after writing, ask inline: "Want me to run this every morning automatically? What time? (default: 9:00 AM)"

- If the user selects **"Yes, schedule it"** — first invoke `anthropic-skills:schedule`, then call `mcp__scheduled-tasks__create-scheduled-tasks`. Pass to both:
  - **`prompt`**: the full config string assembled from values collected during setup —
    ```
    Run daily digest — role: {role} · tools: {tools} · daily_content: {content} · schedule: daily-{HH:MM} days:{days}
    ```
    Replace `{role}`, `{tools}`, `{content}`, `{HH:MM}`, and `{days}` with the actual values parsed from the widget response. Do not leave placeholders. If `mark_noise_read` was answered in step 3, append `· mark_noise_read: yes` (only when "yes" — omit the field entirely when "no", since "no" is the default). Once the feedback tile starts producing answers (step 4.0), append whichever of `pinned_topics`, `trial_topics`, `detail_sections`, `secondary_channels`, `digest_format`, `tone_style`, `signal_trail`, `question_history`, `highlighted_keywords`, `keyword_cooldown` are non-empty to this same string when re-creating the scheduled task — this is how those fields persist across runs. Use `;` to separate multiple entries within a single field (topic/channel names may contain commas).
  - **`schedule`**: cron expression derived from the widget. The widget sends `cron: HH:MM days:1-5` or `cron: HH:MM days:*` — parse both values: time gives H and M, days gives the weekday field. Build: `M H * * {days}`. Examples: `cron: 08:30 days:1-5` → `30 8 * * 1-5` · `cron: 08:30 days:*` → `30 8 * * *`. Default if missing: `0 9 * * 1-5`.
  - **`timezone`**: the user's local timezone — call `mcp__xtiles__xtiles_get_user_timezone` to get it before scheduling if it hasn't been fetched yet.

  This prompt fires each morning and triggers `daily-brief` in scheduled-run mode — the full config must be embedded so the survey is skipped automatically.

  After scheduling succeeds, confirm: "Done — your Daily will be ready in xTiles every morning at [chosen time]." Then call `show_widget` with the **CTA widget HTML**, replacing `{VIEW_URL}` with `https://xtiles.app/{view_id}` (the same `view_id` from step 7). Never output a markdown link here — always the button widget.
- If the user selects **"No, thanks"** — acknowledge briefly.

Either way, continue to **step 9 (Related workflows)** — do not stop here.

---

### 9. Related workflows

**After every manual run, once step 8 is resolved** (scheduled or declined) —
offer related workflows. Skip this on scheduled runs, which end silently
after step 7.

Ask via `AskUserQuestion` (single select): "Want to set up anything else on
xTiles?"
- 🌙 Evening Reflection — an end-of-day synthesis seeded for tomorrow
- 📰 Today News — a daily news digest on topics you care about
- 📊 Weekly Review — a weekly summary of what moved forward this week
- Nothing else, thanks

**Never list these as plain text requiring the user to retype a choice —
always use the interactive question.**

On selection, send the exact matching phrase to hand off to that skill (do
not attempt to run it yourself):
- Evening Reflection → `Set workflow of Evening Reflection (evening-reflection) on xTiles MCP`
- Today News → `Set workflow of Today News (today-news) on xTiles MCP`
- Weekly Review → `Set workflow of Weekly Review (weekly-review) on xTiles MCP`
- "Nothing else" — acknowledge briefly and stop.

---

## How to connect connectors

Do not send the user to settings manually and do not give a URL to follow.
Call `mcp__mcp-registry__suggest_connectors` — it renders interactive connect buttons directly in the Cowork UI.

**Flow:**
1. Call `mcp__mcp-registry__suggest_connectors` passing the names of the missing connectors.
2. Show the **Done widget** (see **Done widget HTML** below) directly under the connector form.
3. The user clicks the connect buttons in the UI — the auth flow runs natively. When finished, they click **"Done"**.
4. **Re-probe before declaring success — never trust the "Done" click alone.** Re-run the same lightweight detection call from step 2 for each connector that was just connected. Only for the ones that now respond without an auth error, confirm: "Connected. Continuing…" and resume the flow.
   - **If a connector still fails detection after "Done"** — say so plainly (e.g. "Outlook still isn't responding — want to try connecting it again, or skip it for now?"). Never silently continue as if it worked; a connector that reports "connected" but then has no working fetch path in step 4 is exactly how a custom connector gets lost from the digest without anyone noticing.
---

## Approval widget HTML

Show this via `show_widget` after the preview in step 6. If the user clicks "Change something" — ask what to change in plain text, update that section, re-show the preview, then show this widget again.

```html
<style>
:root{--c-btn-p:#1a1a1a;--c-btn-p-text:#fff;--c-btn-s:#f0f0f0;--c-btn-s-text:#1a1a1a;--c-muted:#aaa}
@media(prefers-color-scheme:dark){:root{--c-btn-p:#f0f0f0;--c-btn-p-text:#1a1a1a;--c-btn-s:#333;--c-btn-s-text:#f0f0f0;--c-muted:#666}}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:16px;background:transparent}
.btns{display:flex;flex-direction:column;gap:8px}
.btn{width:100%;padding:11px 20px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:background .15s}
.btn-yes{background:var(--c-btn-p);color:var(--c-btn-p-text)}
.btn-edit{background:var(--c-btn-s);color:var(--c-btn-s-text)}
.btn-cancel{background:transparent;color:var(--c-muted);font-weight:400}
</style>
<div class="btns">
  <button class="btn btn-yes" id="btn-yes" onclick="approve()">✓ Looks good — create it</button>
  <button class="btn btn-edit" id="btn-edit" onclick="edit()">Edit</button>
  <button class="btn btn-cancel" id="btn-cancel" onclick="cancel()">Cancel</button>
</div>
<script>
function collapse(msg){document.querySelector('.btns').innerHTML='<p style="font-size:13px;color:var(--c-muted);text-align:center;padding:4px 0">'+msg+'</p>';}
function approve(){collapse('⏳ Creating…');sendPrompt('Looks good — create it');}
function edit(){collapse('✓ Got it');sendPrompt('Change something');}
function cancel(){collapse('✓ Cancelled');sendPrompt('Cancel');}
</script>
```

---

## Done widget HTML

Show this widget via `show_widget` immediately after calling `mcp__mcp-registry__suggest_connectors`.
The user clicks **Done** when they have finished connecting — this sends a message to chat and resumes the flow.

```html
<style>
:root{--c-btn-p:#1a1a1a;--c-btn-p-text:#fff}
@media(prefers-color-scheme:dark){:root{--c-btn-p:#f0f0f0;--c-btn-p-text:#1a1a1a}}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:16px;background:transparent}
.btn{width:100%;padding:11px 20px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;background:var(--c-btn-p);color:var(--c-btn-p-text);transition:background .15s}
</style>
<button class="btn" id="btn-done" onclick="doneIt()">✓ Done</button>
<script>
function doneIt(){var b=document.getElementById('btn-done');b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';b.textContent='⏳…';sendPrompt('Done — connectors connected, continue the flow');}
</script>
```

---

## Survey widget HTML

Show this form via `show_widget` at the start of setup in Cowork.
After Submit, the user sends a string of answers to chat — process it and continue the flow.

```html
<style>
    :root{--color-background-primary:#fff;--color-background-secondary:#f5f5f5;--color-background-tertiary:#f8f8f8;--color-text-primary:#1a1a1a;--color-text-secondary:#888;--color-border-secondary:#aaa;--color-border-tertiary:#e0e0e0}
    @media(prefers-color-scheme:dark){:root{--color-background-primary:#2c2c2c;--color-background-secondary:#383838;--color-background-tertiary:#222;--color-text-primary:#f0f0f0;--color-text-secondary:#999;--color-border-secondary:#666;--color-border-tertiary:#3d3d3d}}
    *{box-sizing:border-box;margin:0;padding:0}
    body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:transparent;color:var(--color-text-primary)}
    .wrap{max-width:560px;margin:0 auto;background:var(--color-background-primary);border-radius:16px;padding:28px;box-shadow:0 2px 12px rgba(0,0,0,.08)}
    h2{font-size:18px;font-weight:700;margin-bottom:4px;color:var(--color-text-primary)}
    .step-label{font-size:12px;color:var(--color-text-secondary);margin-bottom:20px}
    .sec{margin-bottom:22px}
    .sec-title{font-size:13px;font-weight:600;color:var(--color-text-primary);margin-bottom:8px}
    .hint{font-size:12px;color:var(--color-text-secondary);margin-bottom:8px}
    .pills{display:flex;flex-wrap:wrap;gap:7px}
    .pill{padding:6px 14px;border-radius:20px;border:1.5px solid var(--color-border-tertiary);font-size:13px;cursor:pointer;background:var(--color-background-primary);color:var(--color-text-primary);user-select:none;transition:all .15s}
    .pill:hover{border-color:var(--color-border-secondary)}
    .pill.sel{background:var(--color-text-primary);color:var(--color-background-primary);border-color:var(--color-text-primary)}
    .cards{display:flex;flex-wrap:wrap;gap:8px}
    .card{display:flex;align-items:center;gap:7px;padding:8px 13px;border-radius:10px;border:1.5px solid var(--color-border-tertiary);font-size:13px;cursor:pointer;background:var(--color-background-primary);color:var(--color-text-primary);user-select:none;transition:all .15s}
    .card:hover{border-color:var(--color-border-secondary)}
    .card.sel{background:var(--color-text-primary);color:var(--color-background-primary);border-color:var(--color-text-primary)}
    .chk{width:15px;height:15px;border-radius:4px;border:1.5px solid var(--color-border-secondary);display:flex;align-items:center;justify-content:center;font-size:9px;flex-shrink:0}
    .card.sel .chk{background:var(--color-background-primary);border-color:var(--color-background-primary);color:var(--color-text-primary)}
    .custom-in{margin-top:9px}
    .custom-in input{width:100%;padding:7px 11px;border:1.5px solid var(--color-border-tertiary);border-radius:8px;font-size:13px;outline:none;background:var(--color-background-primary);color:var(--color-text-primary)}
    .custom-in input:focus{border-color:var(--color-border-secondary)}
    .checks{display:flex;flex-direction:column;gap:5px;margin-top:6px}
    .ci{display:flex;align-items:center;gap:9px;padding:7px 11px;border-radius:8px;border:1.5px solid var(--color-border-tertiary);font-size:13px;cursor:pointer;background:var(--color-background-primary);color:var(--color-text-primary);user-select:none;transition:all .15s}
    .ci:hover{border-color:var(--color-border-secondary)}
    .ci.sel{border-color:var(--color-text-primary);background:var(--color-background-secondary)}
    .ci .chk{flex-shrink:0}
    .ci.sel .chk{background:var(--color-text-primary);border-color:var(--color-text-primary);color:var(--color-background-primary)}
    .divider{height:1px;background:var(--color-border-tertiary);margin:18px 0}
    .btn-row{display:flex;gap:10px;margin-top:22px}
    .btn{padding:9px 18px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:all .15s}
    .btn-p{background:var(--color-text-primary);color:var(--color-background-primary);flex:1}
    .btn-p:hover:not(:disabled){opacity:0.9}
    .btn-p:disabled{opacity:0.5;cursor:not-allowed}
    .btn-s{background:var(--color-background-secondary);color:var(--color-text-primary)}
    .btn-s:hover{background:var(--color-border-secondary)}
</style>


<div class="wrap" id="app">

  <!-- STEP 1 -->
  <div id="s1">
    <div class="step-label">Step 1 of 2</div>
    <h2>About you</h2>

    <div class="sec" style="margin-top:18px">
      <div class="sec-title">What's your role?</div>
      <div class="pills" id="role-pills">
        <div class="pill" onclick="pickRole(this,'Product Manager')">Product Manager</div>
        <div class="pill" onclick="pickRole(this,'Designer')">Designer</div>
        <div class="pill" onclick="pickRole(this,'Engineer')">Engineer</div>
        <div class="pill" onclick="pickRole(this,'Growth & Marketing')">Growth & Marketing</div>
        <div class="pill" onclick="pickRole(this,'Founder / CEO')">Founder / CEO</div>
        <div class="pill" onclick="pickRole(this,'Support & Success')">Support & Success</div>
        <div class="pill" onclick="pickRole(this,'__other__')">Other role…</div>
      </div>
      <div class="custom-in" id="role-other-wrap" style="display:none">
        <input id="role-other-in" type="text" placeholder="Your role…" oninput="chkNext()">
      </div>
    </div>

    <div class="sec">
      <div class="sec-title">Which tools do you use?</div>
      <div class="hint">Select all that apply — I'll pull live data from them</div>
      <div class="cards" id="tool-cards">
        <div class="card" data-tool="Slack" onclick="togTool(this,'Slack')"><div class="chk">✓</div>Slack</div>
        <div class="card" data-tool="Gmail" onclick="togTool(this,'Gmail')"><div class="chk">✓</div>Gmail</div>
        <div class="card" data-tool="Calendar" onclick="togTool(this,'Calendar')"><div class="chk">✓</div>Calendar</div>
        <div class="card" data-tool="Granola" onclick="togTool(this,'Granola')"><div class="chk">✓</div>Granola</div>
        <div class="card" data-tool="Linear" onclick="togTool(this,'Linear')"><div class="chk">✓</div>Linear</div>
        <div class="card" data-tool="GoogleDrive" onclick="togTool(this,'GoogleDrive')"><div class="chk">✓</div>Google Drive</div>
      </div>
      <div class="custom-in" style="margin-top:8px">
        <input type="text" id="other-tool" placeholder="Other connector (e.g. Plaud, Notion)…">
      </div>
    </div>

    <div class="btn-row">
      <button class="btn btn-p" id="next-btn" onclick="go2()" disabled>Next →</button>
    </div>
  </div>

  <!-- STEP 2 -->
  <div id="s2" style="display:none">
    <div class="step-label">Step 2 of 2</div>
    <h2>Your Daily</h2>

    <div class="sec" style="margin-top:18px">
      <div class="sec-title">What do you want to see each morning?</div>
      <div class="checks" id="daily-content"></div>
    </div>

    <div class="btn-row">
      <button class="btn btn-s" onclick="go1()">← Back</button>
      <button class="btn btn-p" onclick="submit()">Set up Daily</button>
    </div>
  </div>
</div>

<script>
var role=null, tools=new Set(), content=new Set();

var TM={
  'Slack':       {daily:['Slack messages — work chat signals']},
  'Gmail':       {daily:['Important emails — unread inbox','Newsletters — curated summaries']},
  'Calendar':    {daily:['Workload — calendar analysis']},
  'Granola':     {daily:['Granola — meeting notes & summaries']},
  'Linear':      {daily:['Linear issues — new & updated']},
  'GoogleDrive': {daily:['Google Drive — shared files updated']}
};
var AM=[];

var ROLE_DEFAULTS={
  'Product Manager':   ['Slack messages — work chat signals','Important emails — unread inbox','Workload — calendar analysis','Linear issues — new & updated','Granola — meeting notes & summaries'],
  'Designer':          ['Slack messages — work chat signals','Important emails — unread inbox','Workload — calendar analysis'],
  'Engineer':          ['Slack messages — work chat signals','Important emails — unread inbox','Linear issues — new & updated','Workload — calendar analysis'],
  'Growth & Marketing':['Important emails — unread inbox','Newsletters — curated summaries','Slack messages — work chat signals','Workload — calendar analysis'],
  'Founder / CEO':     ['Slack messages — work chat signals','Important emails — unread inbox','Newsletters — curated summaries','Granola — meeting notes & summaries','Workload — calendar analysis'],
  'Support & Success': ['Important emails — unread inbox','Slack messages — work chat signals']
};

document.querySelectorAll('#tool-cards .card.sel').forEach(function(el){tools.add(el.dataset.tool)});
function pickRole(el,v){
  document.querySelectorAll('#role-pills .pill').forEach(function(p){p.classList.remove('sel')});
  el.classList.add('sel'); role=v;
  document.getElementById('role-other-wrap').style.display=v==='__other__'?'block':'none';
  chkNext();
}
function togTool(el,v){
  el.classList.toggle('sel');
  el.classList.contains('sel')?tools.add(v):tools.delete(v);
  chkNext();
}
function chkNext(){
  var ok=role&&(role!=='__other__'||document.getElementById('role-other-in').value.trim());
  document.getElementById('next-btn').disabled=!ok;
}
function go2(){
  if(!content.size){
    var r=role==='__other__'?null:role;
    (ROLE_DEFAULTS[r]||[]).forEach(function(v){content.add(v);});
  }
  // Always pre-select content for every tool the user explicitly picked
  tools.forEach(function(t){if(TM[t]&&TM[t].daily)TM[t].daily.forEach(function(v){content.add(v);});});
  renderContent();
  document.getElementById('s1').style.display='none';
  document.getElementById('s2').style.display='block';
}
function go1(){document.getElementById('s2').style.display='none';document.getElementById('s1').style.display='block';}

function renderContent(){
  var items=[];
  tools.forEach(function(t){if(TM[t]&&TM[t].daily)TM[t].daily.forEach(function(i){if(items.indexOf(i)<0)items.push(i)})});
  AM.forEach(function(i){if(items.indexOf(i)<0)items.push(i)});
  var html='';
  items.forEach(function(o){
    var s=content.has(o)?' sel':'';
    html+='<div class="ci'+s+'" onclick="togCI(this,\''+o.replace(/'/g,"\\'")+'\')" ><div class="chk">✓</div>'+o+'</div>';
  });
  document.getElementById('daily-content').innerHTML=html;
}
function togCI(el,v){el.classList.toggle('sel');el.classList.contains('sel')?content.add(v):content.delete(v);}
function submit(){
  document.querySelectorAll('.btn').forEach(function(b){b.disabled=true;b.style.opacity='0.5';b.style.cursor='default';});
  var r=role==='__other__'?document.getElementById('role-other-in').value.trim():role;
  var oth=document.getElementById('other-tool').value.trim();
  if(oth)tools.add(oth);
  var tArr=Array.from(tools);
  var valid=[];
  tArr.forEach(function(t){if(TM[t]&&TM[t].daily)TM[t].daily.forEach(function(i){if(valid.indexOf(i)<0)valid.push(i)});});
  AM.forEach(function(i){if(valid.indexOf(i)<0)valid.push(i)});
  var items=Array.from(content).filter(function(i){return valid.indexOf(i)>=0;});
  var parts=['Daily planner setup — role: '+r+' · tools: '+(tArr.join(', ')||'none')+' · daily_content: '+(items.join(', ')||'none')];
  sendPrompt(parts.join(' · '));
}
</script>
```

---

## CTA widget HTML

Show this immediately after a successful write. Replace `{VIEW_URL}` with the real xTiles page URL before calling `show_widget`.

```html
<style>
:root{--c-btn-p:#1a1a1a;--c-btn-p-text:#fff}
@media(prefers-color-scheme:dark){:root{--c-btn-p:#f0f0f0;--c-btn-p-text:#1a1a1a}}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:12px;background:transparent}
.btn{display:block;width:100%;padding:12px 20px;border-radius:10px;font-size:15px;font-weight:700;color:var(--c-btn-p-text);background:var(--c-btn-p);text-align:center;text-decoration:none;transition:background .15s}
</style>
<a class="btn" href="{VIEW_URL}" target="_blank">Open in xTiles →</a>
```

---

## Schedule widget HTML

Show this widget via `show_widget` after a successful write in Cowork.
After the user clicks a button, the widget calls `sendPrompt()` and the response lands in chat.

```html
<style>
:root{--c-surface:#fff;--c-surface2:#f3f3f3;--c-text:#1a1a1a;--c-text2:#888;--c-text3:#444;--c-btn-p:#1a1a1a;--c-btn-p-text:#fff;--c-btn-s:#f0f0f0;--c-btn-s-text:#555}
@media(prefers-color-scheme:dark){:root{--c-surface:#2c2c2c;--c-surface2:#383838;--c-text:#f0f0f0;--c-text2:#999;--c-text3:#ccc;--c-btn-p:#f0f0f0;--c-btn-p-text:#1a1a1a;--c-btn-s:#383838;--c-btn-s-text:#bbb}}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;padding:20px;background:transparent;color:var(--c-text)}
.wrap{max-width:480px;margin:0 auto;background:var(--c-surface);border-radius:16px;padding:28px;box-shadow:0 2px 12px rgba(0,0,0,.12);text-align:center}
.icon{font-size:36px;margin-bottom:12px}
h2{font-size:17px;font-weight:700;margin-bottom:6px;color:var(--c-text)}
.sub{font-size:13px;color:var(--c-text2);margin-bottom:20px;line-height:1.5}
.time-row{display:inline-flex;align-items:center;gap:8px;background:var(--c-surface2);border-radius:10px;padding:8px 16px;font-size:13px;font-weight:600;color:var(--c-text3);margin-bottom:24px}
.time-row select,.time-row input[type=time]{border:none;background:transparent;font-size:15px;font-weight:700;color:var(--c-text);outline:none;cursor:pointer}
.btns{display:flex;flex-direction:column;gap:10px}
.btn{padding:11px 20px;border-radius:10px;border:none;font-size:14px;font-weight:600;cursor:pointer;transition:all .15s}
.btn-yes{background:var(--c-btn-p);color:var(--c-btn-p-text)}
.btn-no{background:var(--c-btn-s);color:var(--c-btn-s-text)}
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
  <div class="btns">
    <button class="btn btn-yes" id="btn-yes" onclick="scheduleIt()">Yes, schedule it</button>
    <button class="btn btn-no" id="btn-no" onclick="noThanks()">No, thanks</button>
  </div>
</div>
<script>
function collapse(msg){document.querySelector('.btns').innerHTML='<p style="font-size:13px;color:#aaa;text-align:center;padding:4px 0">'+msg+'</p>';}
function scheduleIt(){
  var days=document.getElementById('sched-days').value;
  var t=document.getElementById('sched-time').value||'09:00';
  var parts=t.split(':'),h=parseInt(parts[0],10),m=parts[1];
  var label=(h%12||12)+':'+m+' '+(h>=12?'PM':'AM');
  var dLabel=days==='1-5'?'weekdays':'every day';
  collapse('⏳ Scheduling…');
  sendPrompt('Yes, schedule my daily digest at '+label+' '+dLabel+' (cron: '+t+' days:'+days+')');
}
function noThanks(){collapse('✓ Got it');sendPrompt('No schedule needed');}
</script>
```

---

## How to behave

- Use the survey widget for setup; ask inline for approval and any follow-up clarifications
- **Never output the digest as plain text in chat** and ask the user to copy it manually — always write to xTiles directly, or walk through connecting xTiles first
- **Never skip a connector** the user selected — if it's not connected, walk through the connection before continuing, don't silently drop it
- Never create anything without preview and explicit approval
- Never put example names, example events, or example messages into the preview — only real data from connectors
- **All clarifying questions and approvals after the main survey form** (channel selection, newsletter names, approval, change requests) — use `show_widget` with HTML, never `AskUserQuestion` or plain text
- If context is missing — ask, don't guess
- If the user gives new information along the way — pick it up, don't wait for the "right step"
- Real data from connectors always beats placeholders
- Daily is the only period. If the user asks for Weekly or Monthly, tell them only the Daily planner is currently supported and offer to create a Daily page instead — never silently downscope.
- Match the user's language, adapt if they switch
- Show the survey widget in Cowork only — in Claude Code, ask the same questions inline
- **The "Tune your digest" and "Important keywords" tiles are written to xTiles as part of the digest itself — never a chat widget, never a follow-up message.** "Tune your digest" is always present (3–5 questions, last tile in the write, every digest including the first); "Important keywords" is second-to-last and only appears when there's a real candidate.
 