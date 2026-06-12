# Four Tasks — Week Mode Design Notes

Status: DESIGN LOCKED. Feature shape s8, day-2 cascade s12,
        interaction model RESOLVED to MODEL B 2026-06-10, popup
        detail design LOCKED session 34 (Morgan decisions 1-10).
        v1.0 SCOPE (promoted from v1.x at session 9; reconfirmed
        2026-06-10 against forecast-review pressure to defer —
        Morgan's call, stays in).
        Implementation tile 4.D2 (Phase 4).
        Streak question RESOLVED 2026-06-10: moot — week mode is
        label-overrides only; every day still has exactly four
        tasks, so streaks, sealing, payout, and stamps are
        untouched (consistent with the "single clock" reasoning
        below).

Schema: the s12-shipped shape needs the 4.D2 RESHAPE (per-slot
text + active bitmask; explicit drop-and-recreate — IF NOT EXISTS
won't restructure) + a write endpoint. `users.week_mode_weekdays`
is RETIRED-UNUSED under Model B. See SCHEMA SUMMARY.

Reveal: day-2 long-press reveal relays into the popup with a
one-time intro line (staggered-disclosure doc, synced s32). See
REVEAL TIMING.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ORIGIN — A REQUEST AND A REJECTION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

A user asked whether Four Tasks would support weekly tasks. The
ask was rejected.

Reasoning:
  - The whole premise of the app is the name. Four. Tasks. Today.
    The user picks four things and does them today. Tomorrow they
    get four again. The daily rhythm is the thesis, not an
    implementation choice.
  - Weekly tasks introduce a second clock. Sealing, claiming,
    MOTD, stamps, streaks, coin payout, partner reactions — all
    hinge on day-boundaries. A parallel weekly cycle would
    fracture every one of those.
  - Weekly tasks break the partner cadence. Two people on the
    same daily rhythm is buddy-ware's whole social mechanic. Add
    a weekly clock and asynchrony enters.
  - "Weekly tasks" tends to mean "permission to defer," which is
    the opposite of what a habit app should give.
  - The slope from weekly tasks to monthly tasks to yearly
    goals is short and well-trodden. That's the dashboard-style
    habit-tracker category Four Tasks is specifically not
    competing in.
  - A user with a weekly thing can already put it in today's
    four whenever they decide today is the day for it. The
    "weekly" cadence lives in the user's head, not the app's
    data model.

But the request pointed at a real underlying friction: users with
stable weekly rhythms have to retype Monday's tasks every Monday
morning. The "inherit from yesterday" default fights the texture
of how some people actually live. That friction shaped a different
feature.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE FEATURE — WEEK MODE (MODEL B SHAPE)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Week mode lets the user override individual task slots for any
specific weekday, diverging per-slot from their standard four.

Off by default for everyone. No overrides exist until authored.

THE SHAPE:
  - Long-press a weekday label in the WEEKDAY STRIP (MON TUE
    WED … above the calendar grid) to open that weekday's
    override POPUP.
  - The popup shows four slots. Each slot has an independent
    activation control + a text field. Inactive slots show the
    current standard four ghosted as placeholders — the user
    sees exactly what they'd be overriding.
  - Activating a slot and writing a task overrides THAT SLOT
    for that weekday. Slots are independent: a Tuesday can
    render two override slots and two standard slots.
  - One save on popup confirm/close writes the whole weekday
    row. Overrides apply to every not-yet-sealed matching
    weekday (today included if unsealed, unticked slots only).
  - Deactivating a slot reverts it to the standard four. The
    override text is PRESERVED in the row (the "paused"
    behaviour, now per-slot) — reactivating restores it.

PERSONAL, NOT SHARED. Each user's overrides are their own.
Partners do not see or sync each other's week mode state.

GRANULARITY: per-slot per-weekday matches how people actually
live — most users have a few anchored commitments on a few
anchored days, not whole alternate days.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
INTERACTION MODEL — RESOLVED (2026-06-10; detail locked s34)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Two candidate models existed. MODEL B WON.

MODEL B (the shape above, and the prototype's behaviour):
  Long-press the weekday strip label → a popup with four slots,
  per-slot activation + text. One surface, explicit UI,
  everything visible at once, per-SLOT overrides.

MODEL A (superseded — the loser's reasoning, recorded per this
doc's own decision rule so it isn't re-litigated):
  Two silent gestures — long-press the day name in the tray
  heading to toggle a weekday-level permission state, then
  long-press individual task labels to author a whole-day
  template. Gesture-minimal, zero chrome. It lost because it
  was INVISIBLE (toggle-on "does nothing visible at first";
  discoverability rested entirely on the day-2 reveal), and
  because its popup-rejection rationale didn't actually apply:
  the discovery arc rejected a CONFIRMATION prompt (a two-beat
  interaction where one would do), not an EDITING surface.
  Model B's popup IS the editor — one gesture opens the whole
  feature, self-explaining, nothing latent.

  Model A machinery that dies with it: the weekday-level toggle
  (and `users.week_mode_weekdays`), toggle-as-permission-state,
  divergence-on-edit, whole-template pause/restore (replaced by
  per-slot pause/restore), the task-label long-press gesture,
  the today-only edit rule (the popup is WEEKDAY-scoped, not
  day-scoped — there is no "edit this Tuesday vs next Tuesday"
  ambiguity to defend against), and the cal-icon bypass bonus
  (the popup provides the equivalent: template edits are
  available regardless of tick state — see greying rules).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE DISCOVERY ARC — HOW THE DATA MODEL EMERGED (HISTORY)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Captured for future-Morgan. NOTE (s34): the arc produced MODEL A's
gesture grammar, which is superseded — but its data-model
conclusions SURVIVE into Model B: per-weekday opt-in (not a
universal default), overrides paused-not-deleted on deactivation,
personal-not-shared, rest-days-answer-the-sick-day case, and
edits-never-touch-sealed-days. Read it for the WHY of those.

INITIAL PROPOSAL: same-weekday inheritance default.
  Original idea: change the default so that opening Monday
  inherits from last Monday instead of yesterday. Rejected as
  too rigid — encodes an assumption about weekly rhythms that
  many users don't have. FIFO-roster users especially have
  "Mondays" that swing between work and rest depending on the
  roster. A universal same-weekday default would be wrong more
  often than right for some user populations.

SECOND PROPOSAL: per-weekday opt-in toggle with prompts.
  Move from default-for-everyone to opt-in-per-weekday, gated
  by a confirmation prompt. The PROMPT was rejected: long-press-
  then-confirm is a two-beat interaction where one would do.
  (s34 note: this is the rejection Model A over-read as
  "no popups ever" — it forbids confirmation theatre, not an
  editing surface.)

THIRD ITERATION: long-press task label edits "this task forever."
  The per-task edit gesture (superseded by the popup's slots).
  The surviving piece: the cal-icon (top-right calendar menu)
  must NOT be greyed out on override days — it's the universal
  editor for the STANDARD FOUR, separate from any weekday
  templating. It stays live, with a warning that edits there
  won't affect overridden slots today. (See cal-icon banner.)

FOURTH ITERATION: override-just-today UX. Rejected — wrong
  scope. Rest days already exist as the system-level "ignore
  today's tasks" mechanism. If a sick Tuesday is bad enough to
  want different tasks, it's bad enough to be a rest day.

FIFTH ITERATION: deactivation-deletes-template? Settled: PAUSED.
  Deactivating preserves the authored text silently; reactivating
  restores it. No popup, no theatre, no lost work for
  experimenting. (Now per-slot under Model B.)

SIXTH ITERATION: divergence-on-edit, not on-toggle. Model A's
  cheap-toggle insight. Superseded as a mechanism (there is no
  toggle), but the spirit survives: opening the popup writes
  NOTHING — no row exists until the user saves an activation.
  Looking is free.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE POPUP — DETAIL DESIGN (LOCKED s34)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

TRIGGER:
  Long-press a weekday strip label (MON/TUE/…). The strip is
  currently inert, so there is no gesture collision. Tap stays
  inert. Works from any displayed month — the strip is
  month-agnostic, the popup is weekday-scoped.

LAYOUT:
  Modal popup titled for the weekday ("TUESDAYS" or similar —
  copy at implementation). Four slot rows, each:
    [activation control] [text field]
  Save/confirm + close. v1.0 ships stock controls; a richer
  control treatment is deferred to the UI-chrome art pass /
  v1.x (Morgan, s34 — "normie ones in there for now").

SLOT SEMANTICS:
  - Controls are INDEPENDENT per-slot toggles (checkbox
    semantics, not radio one-of-four), despite the prototype's
    "radio selector" framing.
  - Inactive slot: control off, text field shows the current
    standard-four task for that slot as a GREYED PLACEHOLDER.
  - Activating: control on, field becomes editable (placeholder
    as starting context, user writes the override).
  - Deactivating: control off, field returns to the greyed
    standard-four placeholder. The previously authored text is
    preserved in the row (pause, not delete) and returns on
    reactivation.
  - EMPTY-ON-BLUR REVERT: clearing an active slot's field and
    unfocusing (tap out, etc.) reverts the slot to inactive +
    greyed placeholder. Save additionally normalises any
    empty-text active slot to inactive — blank task labels
    never render, ever.

TICK-STATE GREYING (only when the popup's weekday IS today's
weekday and today is unsealed):
  - PARTIAL TICKS: today's ticked slots render with their
    activation control DISABLED (greyed). Rationale: within one
    popup-opening, edits should mean ONE thing — unticked-slot
    edits apply today + future; a ticked-slot edit could only
    apply future. Mixing both semantics in one surface is
    confusing, so the future-only ones lock.
  - ALL FOUR TICKED (or today sealed / rest day): EVERY slot
    un-greys, and a single copy line states that changes apply
    from the next [weekday]. Uniform future-only semantics =
    one banner, no per-slot locks. Yes, the fourth tick
    UNLOCKS the other three — that is correct, not a bug: a
    finished day has nothing mid-flight to corrupt. (This also
    inherits Model A's "edit your template even when today's
    done" bonus.)

COMMIT MODEL:
  One save on confirm/close — whole-row write, optimistic with
  per-operation rollback snapshot (house pattern). No
  per-keystroke writes. Opening and closing without saving
  writes nothing.

COPY:
  No tutorial text lives in the popup beyond (a) the one-time
  day-2 intro line (staggered-disclosure relay) and (b) the
  all-ticked future-only line when applicable. Reference copy
  lives in the help menu (tile 4.11 obligation).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
INTERACTION TABLE (MODEL B)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| Surface                  | Gesture     | Behaviour                          |
|--------------------------|-------------|------------------------------------|
| Weekday strip label      | Long-press  | Opens that weekday's override popup|
| Weekday strip label      | Tap         | Inert                              |
| Cal-icon (standard four) | Open        | Standard-four editor as normal; if |
|                          |             | today's weekday has any ACTIVE     |
|                          |             | override, an inline warning banner |
|                          |             | (persistent for the session) notes |
|                          |             | edits here change only today's     |
|                          |             | non-overridden slots, and points   |
|                          |             | at the weekday popup for the rest  |
| Task labels (tray)       | Long-press  | Inert (Model A gesture removed)    |
| Sealed past day          | —           | Read-only; overrides never         |
|                          |             | back-propagate                     |

CAL-ICON BANNER NOTES:
  Inline banner inside the editor modal, not a separate popup.
  Persistent for the editor session; no "don't show again" —
  the state it warns about is per-session ambiguous. Copy
  reworked per-slot, without scolding; refine at implementation.
  The cal-icon's existing tick-locking rules for the standard
  four are unchanged and deliberately decoupled from the
  override popup's greying rules.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE STANDARD FOUR vs WEEKDAY OVERRIDES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

THE STANDARD FOUR:
  The user's universal default — the four task names used on
  every slot that isn't overridden. Maintained via the cal-icon
  menu. Edits propagate to all not-yet-completed days' slots
  that aren't overridden. Schema: `users.task_labels` (JSON
  array of four strings, shipped v1.0). Unchanged.

WEEKDAY OVERRIDES (per-slot):
  Optional per-user, per-weekday, PER-SLOT overrides. A row per
  weekday the user has ever saved an activation for, holding
  four slot texts + a 4-bit active mask. Absence of a row = the
  user never authored that weekday.

WHAT RENDERS, per slot i (0-3) of a given day:
  1. Day sealed → frozen seal state. Stop.
  2. Override row exists for (user, weekday) AND active bit i
     set AND slot text non-empty → the override text.
  3. Otherwise → standard four [i].

  A row with mask 0 (everything deactivated) renders identically
  to no row — the texts are just paused, waiting.

MID-DAY ACTIVATION:
  Saving an activation when today is that weekday: today's
  UNTICKED matching slots re-render immediately; ticked slots
  keep their original labels (immutable history) and pick up
  the override from the next instance. Consistent with the
  popup's greying rules — the surface the user just used told
  them exactly this.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EDITING PROPAGATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CAL-ICON EDITS (standard four):
  Apply to all not-yet-completed days, on slots that aren't
  actively overridden. An overridden slot ignores standard-four
  edits while its bit is on, and reflects the latest standard
  four whenever it's deactivated.

POPUP SAVES (overrides):
  Apply to today (if that weekday, unsealed, unticked slots
  only) and all future instances of the weekday. Never
  back-propagate to sealed days.

There is no toggle and nothing else propagates.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
VISUAL INDICATORS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

WEEKDAY STRIP LABEL with any ACTIVE override:
  Subtle treatment — highlight, bold, or a small underline
  (Morgan s34: "doesn't have to be huge, just a reminder, and
  reassurance it's in effect"). Specifically NOT a badge or a
  "WEEKLY" label. Must be subtle enough that non-users never
  wonder about it, consistent enough to read at a glance.
  Exact treatment at implementation.

DAY-CELL CALENDAR INDICATOR: still not shipped (s8 reasoning
  stands — the divergent tasks are visible whenever the day is
  opened, and the cal-icon banner covers the editing confusion
  case). Revisit only if real usage shows confusion.

(Model A's task-label indicator is dead with the gesture.)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
REVEAL TIMING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

LOCKED: Day 2, relayed from the long-press philosophy reveal —
authority is the staggered-disclosure doc (SYNCED s32 to Model B:
the taught long-press lands on a weekday strip label, the reveal
dismisses, the popup opens carrying a ONE-TIME intro line; the
old multi-step tutorial flow and its toggle side-effect are
gone — opening the popup writes nothing, so there is no latent
state for a user who looks and leaves).

`users.tutorial_progress.week_mode_intro` semantics per the
staggered-disclosure doc. Reveals never re-fire; reference copy
lives in the help menu (tile 4.11).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SCHEMA SUMMARY (4.D2 RESHAPE — supersedes the s12 shape)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

TARGET SHAPE (build at 4.D2):

  -- Per-user, per-weekday, per-slot divergent task labels.
  -- Row exists only once the user has SAVED an activation for
  -- that weekday; mask 0 = all paused (texts preserved).
  CREATE TABLE user_weekday_overrides (
    user_id      TEXT NOT NULL REFERENCES users(user_id),
    weekday      INTEGER NOT NULL,   -- ISO 8601, 1=Mon … 7=Sun
    task_labels  TEXT NOT NULL,      -- JSON array of 4 strings
    active_mask  INTEGER NOT NULL DEFAULT 0,  -- bit i = slot i active
    PRIMARY KEY (user_id, weekday)
  );

MIGRATION NOTE: the s12-shipped table lacks `active_mask` —
explicit DROP TABLE + CREATE (IF NOT EXISTS won't restructure).
No production rows exist; nothing to migrate.

RETIRED: `users.week_mode_weekdays` (the Model A weekday-level
toggle bitmask). Left in place as a DEAD column — dropping a
users-table column is a migration for zero benefit; nothing
reads or writes it. Note RETIRED in the system map at the 4.D2
schema pass.

WRITE ENDPOINT (4.D2 server half):
  PUT /users/:user_id/weekday_overrides/:weekday
  Body: { task_labels: [4 strings], active_mask: 0-15 }
  Standard envelope; self-write only; validation: weekday 1-7,
  exactly four labels, mask 0-15, label length caps + the `|`
  ban per the identity-field policy; empty-text active slots
  normalised inactive server-side too (belt and braces with the
  client rule). Whole-row write, idempotent.
  Overrides ship in the GET /users/:user_id payload so render
  logic has them on load — and across the pair boundary they do
  NOT ship: partner payloads exclude them (personal data).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EDGE CASES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  - OVERRIDE TEXT IDENTICAL TO THE STANDARD FOUR: allowed; the
    row keeps it and rule 2 still wins. Functionally identical
    output. No auto-deactivation — that would surprise the user
    when the standard four later changes out from under them.

  - DST / TIMEZONE NEAR MIDNIGHT: weekday-of-today uses the
    user's local clock per the timezone+sealing doc. Overrides
    are private state; no partner-timezone interaction.

  - PARTNER INFERENCE: a partner can infer a template from
    identical Tuesdays. Fine — the system promises overrides
    aren't synchronised, not that they're secret.

  - REST DAY ON AN OVERRIDDEN WEEKDAY: rest UI overrides the
    task tray entirely; the overrides are untouched and return
    next instance. The popup remains reachable from the strip
    (future-only banner case).

  - FEWER THAN FOUR STANDARD TASKS: not currently possible. If
    a future feature permits it, slots stay four (the app is
    called Four Tasks); flag for confirmation then.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHAT THIS FEATURE IS NOT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Worth stating explicitly to prevent feature creep:

  NOT weekly tasks. Tasks remain daily. The day is the unit
  of completion, streak, payout, partner reaction, everything.
  Week mode is a templating convenience for the daily display,
  not a new temporal cadence.

  NOT a calendar of activities. The user is not scheduling
  events. They're maintaining a set of habit-shaped task lists,
  one per weekday for users who want that granularity.

  NOT a project manager. No multi-day tasks, no dependencies,
  no due dates, no priorities. Tasks are just labels the user
  ticks once a day if and when they do them.

  NOT a partner-shared concept. Each user's week mode state
  is their own. Partners don't see overrides, don't sync them,
  don't even know whether their partner is using the feature.

  NOT default-on for anyone. No overrides exist for any user,
  any weekday, until deliberately authored.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED DOCS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  - four_tasks_staggered_disclosure_design_notes.md
    AUTHORITY for the day-2 reveal → popup relay + one-time
    intro line (synced to Model B, s32). Help menu obligation:
    reference copy in tile 4.11.

  - four_tasks_timezone_and_sealing_design_notes.md
    Weekday-of-today uses the user's local clock. Sealed past
    days are immutable, which constrains propagation.

  - four_tasks_system_map.md
    Live schema. Update at the 4.D2 pass: override table
    reshape, week_mode_weekdays RETIRED, new write endpoint.
    Overrides are self-write only, excluded from partner reads.

  - four_tasks_pair_key_design_notes.md
    Overrides don't participate in pair-key hashing — private
    user state, freely changeable without rotation.

  - four_tasks_architectural_preference.md
    Clarity over cleverness: the popup is the legible surface;
    whole-row writes over per-field patching; pause-not-delete
    over state cleverness.

  - four_tasks_tracking_design_notes.md
    Same originating conversation; week mode as the
    counter-example to scope creep.

  - four_tasks_godot_devlog.txt
    s8 discovery conversation, s12 cascade flow, 2026-06-10
    fork resolution, s34 popup detail decisions.
