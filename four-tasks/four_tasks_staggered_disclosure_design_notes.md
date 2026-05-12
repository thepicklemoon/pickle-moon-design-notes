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
| 5    | Streak milestone popup (first 5-day streak)               |
| 7    | Theme picker hint (tap your emoji to swap)                |
| 14   | Coin name reroll unlocks (first time enough coins exist)  |
| 21   | Subscription nudge (trial about to end / value summary)   |
| 30   | Leaderboard offer ("want to compete?")                    |

Some features are *triggered* not *scheduled* — e.g. partner reaction
appears the first time partner actually reacts to a sealed day. The
schedule above is for time-gated reveals; trigger-gated reveals fire
when the relevant event happens.

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
  introduced day 5+ rather than day 1.
- **Tile 4.11 (Help menu)** — exists for users who want everything at
  once. Always-available escape hatch from the staggered approach.
- **Tile 4.16 (Partner reactions)** — trigger-gated reveal (first time
  partner reacts).
- **Future coin name reroll tile** — example time-gated reveal.

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
- `four_tasks_partner_reactions_design_notes.md` — trigger-gated
  reveal example.
- Tile 4.11 (help menu) — the "show me everything" escape valve.
- Future tile: TutorialCoordinator autoload + tutorial_progress
  schema reservation on `users`.
