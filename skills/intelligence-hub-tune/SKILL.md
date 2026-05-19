---
name: intelligence-hub-tune
description: >
  Adjust Intelligence Hub settings after initial setup — change digest schedule, add or remove sources,
  update Slack watchlists, Gmail filters, or any other preference.

  Trigger when the user says: "remove the evening digest", "add #design to Slack", "stop showing GitHub",
  "change my digest to weekly only", "too much noise from X", "I want more Y", "update my Intelligence Hub settings".

  Makes targeted changes and outputs an updated prompt for the user to paste into Cowork.
  Does not restart the full setup flow — only touches what changed.
allowed-tools: mcp__xtiles__xtiles_get_planner_content, mcp__xtiles__xtiles_create_tiles_from_markdown_in_my_planner
---

# Intelligence Hub — Tune

Handles post-setup adjustments. Never restarts the full setup — only updates what changed, then outputs a ready-to-paste prompt.

---

## Starting point

If invoked without a specific request — don't show a form or list of options. Just ask what they want to change. One sentence.

If the request is ambiguous — ask one clarifying question before doing anything.

---

## Step 1 — Read current config

Call `xtiles_get_planner_content` for today and find the config block. Parse all fields:
`sources`, `schedule`, `slack_channels`, `metrics`, `timezone`, `language`.

If no config found — tell the user to run `intelligence-hub-setup` first.

---

## Step 2 — Apply the change

Map the request to the relevant config field:

| User says               | What to update                         |
|-------------------------|----------------------------------------|
| "add #design to Slack"  | `slack.watchlist` += `#design`         |
| "remove Slack"          | `slack.enabled` = false                |
| "bring back Slack"      | `slack.enabled` = true                 |
| "ignore newsletters"    | `gmail.blacklist` += pattern           |
| "focus on PRs only"     | `github.watchlist` = PRs only          |
| "remove evening digest" | `schedule` — remove evening entry      |
| "switch to weekly only" | `schedule` = weekly-pulse Friday 18:00 |
| "add morning digest"    | `schedule` += morning-brief 08:00      |

When disabling a source — set `enabled: false`, don't delete the config. Makes it easy to re-enable later.

---

## Step 3 — Show diff

Before generating the prompt, show what changed:

```
I'll update your config:
  slack.watchlist: #general, #product → #general, #product, #design

Anything else to change?
```

Accept more changes before moving on. When the user confirms — go to Step 4.

---

## Step 4 — Generate updated prompt

Output the full updated config as a ready-to-paste prompt:

```
/intelligence-hub-digest

CONFIG:
- timezone: [value]
- language: [value]
- sources:
    gmail:    enabled: [true/false] | watchlist: [...] | blacklist: [...]
    slack:    enabled: [true/false] | watchlist: [...] | blacklist: [...]
    calendar: enabled: [true/false] | upcoming_days: 1
    notion:   enabled: [true/false] | watchlist: [...]
    github:   enabled: [true/false] | watchlist: [...]
    figma:    enabled: [true/false] | watchlist: [...]
    analytics:enabled: [true/false] | watchlist: [...]
```

Then tell the user: go to Cowork → Scheduled → find your digest task → replace the prompt with this one.

If the schedule changed — specify which tasks to update or create.

---

## Schedule reference

| Rhythm            | Tasks                                                  |
|-------------------|--------------------------------------------------------|
| Morning only      | daily-morning-brief at 08:00                           |
| Morning + evening | daily-morning-brief 08:00 + daily-evening-digest 17:00 |
| Weekly only       | weekly-pulse Friday 18:00                              |

Weekly-pulse can be added alongside any rhythm — it reads from Planner, not from source APIs.

---

## How to behave

- Confirm every change in one sentence
- Never ask "are you sure?" for small changes
- Never restart setup flow
- After 7 days if a source produces zero 🔴/🟡 items consistently — surface once: "I notice [source] hasn't had anything important this week. Want to keep it or remove it?" Don't ask again if the user says keep.
