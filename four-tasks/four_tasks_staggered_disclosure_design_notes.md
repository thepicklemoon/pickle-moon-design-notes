# Staggered Feature Disclosure — Design Principle

**Status:** LOCKED as a design principle (session 2). Applies to every
v1.0+ feature, retroactively where useful.

**Scope:** this is a *meta-principle* about how features are introduced
to the user over time. It doesn't dictate which features exist; it
governs *when* the user discovers them.

---

## The principle

Four Tasks reveals its surface area to the user over **days, weeks, and
months of use**, not all at once at onboarding.

Onboarding teaches the bare minimum: name, tasks, slot, emoji, your
partner exists, here is the calendar. Everything else gets surfaced at
the moment it's most useful to learn, which is rarely "right now, while
the user is overwhelmed by a brand new app."

## Why

Habit trackers are dense. The web build has: tasks, MOTD (diary),
stamps, coins, streaks, milestones, rest days, partner reactions
(planned), themes, leaderboard, bug catcher, help menu, subscription,
account recovery. Dumping all of that into onboarding kills the app for
new users.

Staggering accomplishes two things:
1. **Reduces cognitive load early** — user only learns what they need
   to do *today*, day 1.
2. **Creates rewarding discovery** — features unlock or appear over
   time, which feels good in itself ("oh I can do *that* now?").

## Example schedule (illustrative, not final)

These are *examples* of when features could appear. Real cadence to be
tuned in Phase 4 / Phase 5.

| Day  | Feature surfaced                                          |
|------|-----------------------------------------------------------|
| 0    | Core onboarding: tasks, MOTD, calendar, partner, stamp    |
| 1    | Stamp animation + coins drop in morning sequence          |
| 2    | Coins tutorial popup (what they're for, how they spend)   |
| 3    | Rest day intro ("did you know — long-press a future cell") |
| 4    | Partner reactions intro (you can react to each others sealed days) |
| 5    | Streak milestone popup (first 5-day streak)               |
| 7    | Theme picker hint (long-press your sticker, pick leader or palette) |
| 14   | Coin name reroll unlocks (first time enough coins exist)  |
| 21   | Subscription nudge (trial about to end / value summary)   |
| 30   | Leaderboard offer ("want to compete?")                    |

Some features are *triggered* not *scheduled* — e.g. partner reaction
trigger-gated REAFFIRMATION can fire if a partner reaction lands
before day 4 in the user's life. But the PRIMARY teaching moment for
partner reactions is the day-4 scheduled reveal, so users have
vocabulary to notice the indicator before reactions start appearing.
This was a session-5 refinement (Q7); the earlier framing of partner
reactions as purely trigger-gated has been superseded.

The schedule above is for time-gated reveals; pure trigger-gated
reveals (e.g. first-coin-payout, first-top-10-on-leaderboard) fire
when the relevant event happens regardless of day count.

## Architectural implications

### 1. Each feature needs a "first-shown-at" flag, or equivalent

For features that should appear once on a schedule:
- Either store `<feature>_shown_at` timestamps on `users`
- Or store an aggregate `tutorial_progress JSON` blob

I lean toward the **second** — one column on `users` holding a JSON
object like:
```json
{
  "coins_intro": 1715472000,
  "rest_day_intro": 1715645200,
  "theme_picker_hint": null
}
```

Single column, easily extensible, no migration cost when a new feature
is added to the list. Null = not yet shown.

This is a SCHEMA RESERVATION that should land alongside coin names —
worth doing once, then never again.

### 2. Logic lives in a coordinator, not scattered

A `TutorialCoordinator` autoload (Godot side) checks on app boot:
- What day of use is this for the user?
- Which scheduled features are due?
- Which feature first-shown-at flags are null but should be set?
- Show the next eligible reveal, set the flag.

One reveal per session boot, not a queue dump. The user gets one new
thing at a time.

### 3. Triggered reveals are signal-driven

For trigger-gated features (partner reaction, leaderboard top-10
landing, first-coin-payout, etc.), the relevant code emits a signal
that TutorialCoordinator picks up and renders. Decouples feature
logic from its reveal logic.

### 4. The reveal copy itself is content, not code

Each reveal has a popup with copy. Copy lives in a Resource file
(e.g. `res://data/tutorial_reveals.tres`) for iteration without code
changes. Glitch-typewriter applies if appropriate.

## Cross-cutting hooks for current and planned tiles

Tiles that should be designed with staggered disclosure in mind:

- **Tile 3.1-3.8 (Onboarding)** — be ruthless about what stays. Anything
  that can wait until day 2+ should wait.
- **Tile 4.6 (Morning sequence)** — natural surfacing window for
  scheduled reveals. "Yesterday wrapped + here's the new thing."
- **Tile 4.5 (MOTD reroll)** — Reroll is a power feature. Could be
  introduced day 5+ rather than day 1. (Cost model superseded session
  5 — see four_tasks_morning_sequence_design_notes.md Q8 section.)
- **Tile 4.11 (Help menu)** — exists for users who want everything at
  once. Always-available escape hatch from the staggered approach.
- **Tile 4.14a (Basic sticker picker — pre-fork)** — pool toggle is
  available immediately, no reveal needed.
- **Tile 4.14b (Sticker picker context menu — post-fork, Four Tasks
  only)** — the long-press context menu (calendar leader + palette
  source) is a scheduled reveal at day 7+. APPtrioc never gets this
  surface; APPtrioc users see the picker without the context menu.
- **Tile 4.16 (Partner reactions)** — day-4 scheduled reveal (primary
  teaching) + trigger-gated reaffirmation (if partner reacts earlier).
  Session 5 refinement; see Q7 section in morning sequence design doc.
- **Tile 4.18 (Coin name reroll)** — example time-gated reveal,
  unlocks first time user has enough coins.

## Open questions

- **Schedule data structure:** time-based ("day N of use") vs
  event-based ("after first 5-day streak"). I've sketched both above;
  real implementation might want a unified abstraction.
- **Holdouts for re-engagement:** are some features surfaced *only*
  if the user comes back after a lapse, as a "welcome back, did you
  know..." nudge? Probably yes, post-v1.0.
- **Skip-everything mode:** for power users (Morgan testing on his
  own device), a devkit toggle to dismiss all tutorial reveals
  immediately. Implementation hook in tile 0.4's devkit pattern.

## Cross-references

- `four_tasks_coin_name_design_notes.md` — feature that *requires*
  this principle to land well (reroll unlocking later).
- `four_tasks_partner_reactions_design_notes.md` — day-4 scheduled
  reveal (per Q7 session 5 refinement).
- `four_tasks_morning_sequence_design_notes.md` — Q7 partner
  reaction surfacing model + the day-4 vs day-6 timing rationale.
- `four_tasks_theme_design_notes.md` — sticker picker context menu
  is a post-fork (Four Tasks only) reveal at day 7.
- Tile 4.11 (help menu) — the "show me everything" escape valve.
- Future tile: TutorialCoordinator autoload + tutorial_progress
  schema reservation on `users`.
