# Staggered Feature Disclosure — Design Principle

**Status:** LOCKED as a design principle (session 2, refined session 5, substantial revisions session 8 to reflect onboarding scope expansion and week mode v1.0 inclusion, tutorial_progress merge semantics added session 12, week-mode cascade destination updated to Model B session 32 — recording the s30 fork resolution).

**Scope:** this is a *meta-principle* about how features are introduced to the user over time. It doesn't dictate which features exist; it governs *when* the user discovers them.

---

## The principle

Four Tasks reveals its surface area to the user over **days and weeks of use**, not all at once at onboarding.

Onboarding teaches the bare minimum required to use the app today, plus the features whose UI is permanently visible from day zero (because hiding them would mean showing the user UI elements with no explanation).

Everything else gets surfaced at the moment it's most useful to learn, which is rarely "right now, while the user is overwhelmed by a brand new app."

## Why

Habit trackers are dense. Four Tasks has: tasks, MOTD, stamps, coins, streaks, milestones, rest days, partner reactions, themes, picker context menu (post-fork), week mode, bug catcher, help menu, subscription, account recovery. Dumping all of that into onboarding kills the app for new users.

Staggering accomplishes two things:

1. **Reduces cognitive load early** — the user only learns what they need to do *today*.
2. **Creates rewarding discovery** — features unlock or appear over time, which feels good in itself ("oh I can do *that* now?").

## The onboarding floor

Onboarding teaches:

- Your name, your username, your starting avatar (the leader sticker).
- Your themed companion pool and the per-day rotation mechanic.
- Your message of the day (MOTD) — spin mechanic, save mechanic, coin cost of rerolls.
- Your partner's calendar (where it lives, that pairing is post-onboarding via a persistent button, that subscription enables bilateral library sharing).
- Your four tasks.
- That coins are real, earned each day you finish your tasks, and spent on stickers or MOTD rerolls.

Onboarding does NOT teach:

- Rest days.
- Long-press as the app's gesture for deliberate decisions.
- Week mode (per-weekday task templating).
- Picker context menu (per-slot UI element assembly).
- Partner reactions.
- Streak milestones.
- Coin name personalisation.
- Subscription specifics (only the library-sharing concept is planted; the actual disclosure lives at day 21).
- Bug catcher / help menu (always accessible, not surfaced).

The change vs prior versions of this doc: theme depth and MOTD moved from scheduled reveals into onboarding (session 8) because both are permanent UI elements on the calendar. Hiding them from day zero meant the user encountered visible UI elements with no framing, which violated the staggered principle's own logic.

## The schedule

Day counts are absolute from install date, not pair-formation date. Some reveals fire as scheduled regardless of pair state (e.g. partner reactions was designed to fire on day 4 whether or not the user is paired — but partner reactions is deferred to v1.x, so that slot is dormant at launch; for solo users, the framing would become a soft recruitment nudge for when they pair).

| Day | Feature surfaced                                          |
|-----|-----------------------------------------------------------|
| 0   | Core onboarding (see above)                               |
| 1   | First morning sequence — stamp animation, coin payout     |
| 2   | Long-press philosophy reveal — relays into week mode      |
| 3   | Rest day intro                                            |
| 4   | Partner reactions intro — DEFERRED to v1.x with the feature |
| 5   | First streak milestone (trigger-gated, fires on 5-day)    |
| 14  | Coin name personalisation (trigger-gated by affordability)|
| 21  | Subscription disclosure                                   |
| 30  | Leaderboard offer (if leaderboard ships at launch — see leaderboard design notes; currently deferred) |

### Day 1 — first morning sequence

Not a tutorial reveal in the conventional sense. The morning sequence itself is the daily ritual, designed to be its own teaching moment through ceremony. Day 1 fires the full sequence for the first time against an empty "yesterday" state — implementation detail handled in the morning sequence design doc.

### Day 2 — long-press philosophy reveal

The most distinctive reveal in the schedule. A meta-philosophy moment, not a feature introduction. The popup teaches the app's design ethos around deliberate interaction:

> Most apps are over-responsive. They reward accidental taps and treat your attention as something to capture, by any means.
>
> We don't work like that. Long-press is our gesture for any decision that matters — it keeps things calm in here and honours your attention.
>
> Try it now: long-press a day of the week on your calendar.

**Dismissal pattern:** no Got it button. The popup clears when the user performs the long-press on a day name. The act of dismissing IS the learning. This establishes a new pattern in the app's tutorial language: gesture-teaching reveals dismiss via the taught gesture; content reveals dismiss via Continue.

**Cascading effect:** the long-press on a day name opens week mode itself (see below). The day-2 reveal stays focused on philosophy; the gesture serves as a relay into the feature.

**Implementation note:** the popup needs to be positioned so day names are visible underneath it, or the gesture-dismissal magic breaks. Layout discipline required.

### Day 2 (cascading) — week mode intro

**Resolved s32 (recording the s30 Model-B fork resolution; supersedes "tutorial flow TBD"):** there is no separate week-mode tutorial. The cascade destination is the week-mode feature surface itself — the day-2 long-press opens the **week-mode popup** (Model B: a popup off the weekday header with the four task slots and per-slot override controls), with a **one-time intro line** inside the popup on first open. The feature teaches itself; the intro line is one sentence of framing, not a flow. `tutorial_progress.week_mode_intro` marks the line as shown. Note: week mode is in v1.0 scope (session 8 decision — features touching schema, calendar UI, ritual loop, or write rules ship at launch rather than retrofitted against frozen code); interaction-model detail lives in the week mode design notes (4.D2).

### Day 3 — rest day intro

Per existing design. Long-press on a future cell. The day-2 reveal has already established long-press as the app's gesture, so day 3 reuses the gesture on a different surface. Sequential teaching that compounds across consecutive days.

### Day 4 — partner reactions intro — DEFERRED (v1.x)

> Partner reactions are deferred to v1.x (tile 4.16; doc in `deferred/`). This schedule slot is held for when the feature lands — it does NOT fire in v1.0. The design below is retained so the slot is ready.

Per existing design. Primary teaching moment for the partner reaction mechanic. Trigger-gated reaffirmation still fires if a partner reaction lands earlier than day 4, but day 4 is the scheduled primary reveal.

Solo-user framing: for users without a partner on day 4, the reveal frames the feature prospectively ("when you pair up, you'll be able to react to each other's days"). Acts as a soft recruitment nudge. No conditional cadence — day 4 fires regardless.

### Day 5 — first streak milestone

Trigger-gated, not strictly time-gated. Fires when the user hits a 5-day streak. Typically lands around day 5 for engaged users.

### Day 14 — coin name personalisation

Trigger-gated by affordability. Fires the first time the user has enough coins to reroll their coin name (see coin name design notes for cost). The day-14 entry is the typical landing point under expected economy pacing, not a hard schedule.

### Day 21 — subscription disclosure

Per monetisation v2.0. Value summary against the user's actual play history at this point — coins earned, stickers owned vs catalogue size, days played, longest streak. Lands as "subscribe to access partner's library and grind faster" framing, not as a trial-end nag.

The library-sharing concept was planted on day zero (onboarding screen 7, partner calendar reveal). The day-21 disclosure builds on that mental hook rather than introducing the concept cold.

### Day 30 — leaderboard offer

If leaderboard ships at launch (currently deferred per the leaderboard design notes — possibly not shipping at v1.0). If deferred, this entry is removed and the schedule terminates at day 21.

## Trigger-gated reveals (not on the schedule)

Some reveals fire on action, not on day count:

### Picker context menu hint

Fires the first time the user opens the sticker picker. An inline overlay or tooltip points at a sticker and suggests "long-press for more options." Surfaces the per-slot context menu feature at the moment the user is actually browsing stickers, which is the moment the hint becomes actionable.

This reveal was previously day-7 scheduled. Moved to trigger-gated (session 8) because:

- The picker isn't opened on day 0-6 by all users; a scheduled day-7 reveal could fire before the user has ever seen the picker.
- The day-2 long-press philosophy reveal now covers the gesture as a general principle, so the picker context menu doesn't need to introduce the gesture from scratch.
- Trigger-gating respects the user's discovery rhythm — the hint appears when the action becomes possible.

State requirement: the picker needs to track whether it has been opened before by this user. Single boolean on the user's local state or on `tutorial_progress` JSON column.

### Coin name reroll affordability

The first time the user has enough coins to afford a coin name reroll, the day-14 entry fires (if not already shown). Trigger overrides schedule if affordability comes earlier.

### Partner reaction reaffirmation

If a partner reaction lands on the user's calendar before day 4 (i.e. their partner reacted to one of their sealed days), the reveal fires immediately as a "this is what just happened" explanation. The day-4 scheduled reveal still fires for users who haven't received a reaction by then.

## Architectural implications

### 1. `tutorial_progress` JSON column on `users`

Single column holding a JSON object of feature-name → timestamp (or null for not-yet-shown):

```json
{
  "long_press_philosophy": 1715472000,
  "week_mode_intro": 1715472020,
  "rest_day_intro": 1715645200,
  "partner_reactions_intro": null,
  "picker_context_menu_hint": null,
  "coin_name_personalisation": null,
  "subscription_disclosure": null
}
```

Forward-compatible with future feature additions without schema migrations. Schema reservation lands alongside the coin names schema (per the coin name design notes' reservation pattern).

**Merge semantics:** trivial by design. Once a reveal has fired for a user, the server sets `tutorial_progress[reveal_id]` to the timestamp and it is immutable for the life of that user record. Reveals never re-fire — not on reinstall, not on a new device, not on any subsequent session. The server is the source of truth; the client treats any non-null entry as "already shown, skip." The client never writes a null back over a non-null entry. Timestamps are informational only, not load-bearing for any logic. Users who want to revisit a reveal's content go to the help menu, which surfaces all reveal content as plain reference docs.

### 2. TutorialCoordinator autoload

Godot-side autoload checks on app boot and at key state transitions:

- What day of use is this for the user?
- Which scheduled reveals are due?
- Which trigger-gated reveals have fired but not yet shown?
- Show the next eligible reveal, set the timestamp on `tutorial_progress`, write through to server.

One reveal per session boot (or per qualifying state transition). Not a queue dump. The user gets one new thing at a time.

### 3. Trigger emission

For trigger-gated reveals, the relevant code emits a signal that TutorialCoordinator picks up:

- Picker opens for first time → `picker_opened_first_time` signal.
- Affordability threshold crossed → `coin_name_reroll_affordable` signal.
- Incoming partner reaction → `partner_reaction_received` signal.

Decouples feature logic from its reveal logic. Each feature's code doesn't need to know what tutorial state it's tied to.

### 4. Reveal content as Resource

Each reveal's copy and layout config lives in a Resource file (`res://data/tutorial_reveals.tres`) for iteration without code changes.

### 5. Gesture-dismissal popups

The day-2 long-press philosophy reveal introduces a new dismissal pattern: popup clears when the user performs a specific gesture on the underlying UI. This pattern should be reused for any future gesture-teaching reveal. The implementation hook:

- Popup overlay accepts a "dismiss-on-event" parameter.
- Underlying UI emits the relevant gesture signal.
- Coordinator listens for the signal while the popup is showing and dismisses on match.

## Cross-cutting hooks for current and planned tiles

- **Onboarding (tiles 3.1-3.x)** — locked at the eight-screen flow documented in onboarding design notes. Includes theme, MOTD, partner panel intro, coin grant, library sharing plant. Anything beyond that floor waits for the staggered schedule.
- **Morning sequence (tile 4.6)** — natural surfacing window for scheduled reveals on subsequent days. Day-2 long-press philosophy fires here. Day-3 rest day intro fires here. Day-4 partner reactions would fire here when that feature ships (deferred to v1.x).
- **Week mode (tile 4.D2)** — the day-2 long-press relays straight into the week-mode popup; a one-time intro line on first open is the entire "tutorial" (Model B, s30/s32). No separate tutorial flow exists or is planned.
- **MOTD reroll** — fully introduced in onboarding screen 6. No separate reveal needed.
- **Help menu (tile 4.11)** — always-available escape hatch from the staggered approach. Scope obligation: must include reference copy for every staggered reveal that ships (day-2 philosophy, day-3 rest day, picker context menu, coin name personalisation, subscription disclosure, streak milestones; partner reactions copy lands with the feature in v1.x). The "reveals never re-fire" rule depends on the help menu being a complete reference for users who dismissed too fast or want to revisit.
- **Basic sticker picker (tile 4.14a, pre-fork)** — pool toggle is available immediately. No reveal needed. APPtrioc inherits this picker.
- **Sticker picker context menu (tile 4.14b, post-fork)** — trigger-gated on first picker open, not day-7 scheduled. APPtrioc never gets this surface (it's the conversion mechanic).
- **Partner reactions (tile 4.16)** — DEFERRED to v1.x (doc in `deferred/`). When it ships: day-4 scheduled reveal + trigger-gated reaffirmation if a reaction lands earlier.
- **Day-21 subscription disclosure** — value summary against actual play history per monetisation v2.0. Builds on the day-0 library-sharing plant. Copy authoring job, lands Phase 5 alongside paywall UI.
- **Coin name personalisation** — trigger-gated by affordability, not strict day count.

## Open questions

- **Schedule data structure:** time-based ("day N of use") vs trigger-based ("after first 5-day streak"). The current schedule uses both. Real implementation might want a unified abstraction with trigger overrides.
- **Holdouts for re-engagement:** are some features surfaced *only* if the user comes back after a lapse, as a "welcome back, did you know..." nudge? Probably yes, post-v1.0.
- **Skip-everything mode:** devkit scenario to dismiss all tutorial reveals immediately. Hooks into the tile 2.9 DevKit scenario menu (the old tile-0.4 devkit skeleton was killed at session 17).
- **Order conflicts:** if a trigger-gated reveal fires on the same day as a scheduled reveal, which goes first? Likely the trigger-gated one (because it's responding to a user action), but worth spec'ing.

## Cross-references

- `four_tasks_coin_name_design_notes.md` — feature surfaced day 14 trigger-gated. Provides the schema reservation pattern that `tutorial_progress` follows.
- `four_tasks_partner_reactions_design_notes.md` (in `deferred/`) — day-4 scheduled reveal + trigger-gated reaffirmation, when the feature ships in v1.x.
- `four_tasks_morning_sequence_design_notes.md` — natural surfacing window for daily reveals. The day-1 morning sequence is the user's first encounter with the ritual.
- `four_tasks_theme_design_notes.md` — sticker picker context menu is a trigger-gated reveal on first picker open (session 8 change from day-7 scheduled).
- `four_tasks_onboarding_design_notes.md` — onboarding floor. Substantially expanded session 8 to include theme, MOTD, partner panel, coin grant. Anything beyond the floor lives in this doc's schedule.
- `four_tasks_monetisation_position.md` — day-21 subscription disclosure builds on the day-0 library-sharing plant.
- `four_tasks_week_mode_design_notes.md` — week mode is v1.0 scope (session 8 decision). The day-2 long-press relays into the week-mode popup (Model B); the one-time intro line lives there.
- `four_tasks_achievements_brainstorm.md` — counter-example: hidden Easter-egg achievements explicitly DO NOT use staggered disclosure. Pure discovery, no scheduled reveal.
- Tile 4.11 (help menu) — the "show me everything" escape valve. Scope obligation noted in cross-cutting hooks above.
- Future tile: TutorialCoordinator autoload + `tutorial_progress` schema reservation on `users`.

## Session 8 changes summary

- Onboarding scope expanded: theme, MOTD, partner panel, coin grant, library sharing plant all now day-zero.
- Day-2 coins tutorial removed (covered by onboarding).
- Day-2 long-press philosophy reveal added, with gesture-dismissal pattern and week mode tutorial cascade.
- Day-7 theme picker context menu hint changed from scheduled to trigger-gated on first picker open.
- Partner reactions framing clarified for solo users (still day-4 scheduled, recruitment nudge for solo).
- Gesture-dismissal popup pattern introduced as new tutorial language primitive.
- Week mode pulled into v1.0 scope, tutorial chained off day-2 long-press reveal.
- Cross-reference to onboarding design notes updated to reflect expanded floor.

## Session 12 changes summary

- `tutorial_progress` merge semantics paragraph added to architectural implications section 1. Server-as-source-of-truth, reveals immutable once fired, never re-show across reinstalls or devices. Closes the parked item from session 11.
- Help menu (tile 4.11) cross-cutting hook upgraded with scope obligation: must carry reference copy for every staggered reveal. The immutability of reveals depends on the help menu being a complete fallback.

## Session 32 changes summary

- Week-mode cascade destination resolved (recording the s30 Model-B fork resolution): the day-2 long-press opens the week-mode popup itself; a one-time intro line on first open replaces the separate tutorial flow. "Week mode tutorial" renamed "week mode intro" accordingly; schedule table, cascading-effect paragraph, cross-cutting hook, and week-mode cross-reference aligned.
