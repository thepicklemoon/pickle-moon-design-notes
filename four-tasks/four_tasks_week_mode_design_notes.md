# Four Tasks — Week Mode Design Notes

Status: DESIGN LOCKED (session 8 mobile)
        Implementation deferred — not part of any current tile.
        Lands as a v1.x feature, post-launch.

Schema implications: one new optional table (user_weekday_overrides).
Reveal: deferred to a future staggered disclosure pass — flagged
as a feature deserving its own reveal moment, not bundled.

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
THE FEATURE — WEEK MODE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Week mode is a per-weekday opt-in toggle that lets the user
maintain a separate task template for any specific weekday,
diverging from their standard four.

Off by default for everyone. Off by default for every weekday
of every user.

Toggled per-weekday: a user might have week mode on for Tuesday
and Sunday, off for the rest. Each weekday is independent. The
granularity matches how people actually live — most users have
a few anchored days and the rest is free.

THE SHAPE:
  - User long-presses the day name in the task tray heading
    (e.g. "Tuesday") to toggle week mode for that weekday.
  - Toggle ON does nothing visible at first. The day still
    renders the standard four. The toggle just enables long-press
    on individual task labels.
  - To diverge from the standard four, the user long-presses a
    specific task label and edits it. THAT'S when a Tuesday-
    specific template begins existing.
  - Once any task is changed, that change applies to all not-yet-
    completed Tuesdays (today included if not sealed).
  - Toggle OFF makes future Tuesdays render the standard four
    again. The diverged template is preserved in the background.
  - Toggle ON again restores the diverged template silently.

PERSONAL, NOT SHARED. Each user's week mode state is their own.
Partners do not see or sync each other's week mode toggles.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE DISCOVERY ARC — HOW THIS MODEL EMERGED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Captured for future-Morgan because the back-and-forth itself
shows why the final model is what it is.

INITIAL PROPOSAL: same-weekday inheritance default.
  Original idea: change the default so that opening Monday
  inherits from last Monday instead of yesterday. Rejected as
  too rigid — encodes an assumption about weekly rhythms that
  many users don't have. FIFO-roster users especially have
  "Mondays" that swing between work and rest depending on the
  roster. A universal same-weekday default would be wrong more
  often than right for some user populations.

SECOND PROPOSAL: per-weekday opt-in toggle with prompts.
  Move from default-for-everyone to opt-in-per-weekday. Long-
  press the day heading → prompt asking "propagate Tuesdays?"
  → user confirms or cancels.

  Rejected the prompt. Long-press-then-confirm is a two-beat
  interaction where one would do. Toggles should be direct —
  long-press toggles, visual indicator changes, done. If the
  user toggled by accident, they long-press again to undo. The
  first time a weekday gets toggled on, a one-time tooltip
  could explain the behaviour, but that's a teaching moment
  not a confirmation.

THIRD ITERATION: long-press task label edits "this task forever."
  Once week mode is on for Tuesday, long-press a task label
  edits that task across all future Tuesdays. This part
  survived. But the version of the model at this stage had the
  cal-icon (top-right calendar menu) greyed out on week-mode
  days, with redirect copy saying "edit from task labels for
  today."

  Wrong. The cal-icon menu is the universal editor for the
  user's STANDARD FOUR tasks, separate from any weekday
  templating. Greying it out on week-mode days would prevent
  the user from editing their standard four while ON a templated
  day, which is overreach. The cal-icon must stay live, but
  with a warning communicating that edits there won't affect
  today (because today is on a template).

FOURTH ITERATION: override-just-today UX.
  Spent design time on "user is sick on a templated Tuesday,
  wants today different without changing the template."
  Proposed indicator + tap-to-detach mechanic.

  Wrong scope. Rest days already exist as the system-level
  "ignore today's tasks" mechanism. If a sick Tuesday is bad
  enough to want different tasks, it's bad enough to be a rest
  day. If it's not bad enough to be a rest day, just tick what
  you can of the template and accept the slight friction. The
  override-today problem doesn't need its own UX because rest
  days are already the answer.

FIFTH ITERATION: toggle-off-deletes-template?
  Open question: when the user toggles week mode off, does the
  template get deleted, paused, or prompted ("keep or discard?").

  Settled: paused. Toggling off makes future Tuesdays render
  the standard four. The diverged template is preserved
  silently. Toggling on restores it. No popup, no theatre, no
  state confusion — the friendliest behaviour is also the
  simplest. The user doesn't lose work for experimenting with
  the toggle.

SIXTH ITERATION: divergence-on-edit, not on-toggle.
  The key clarification. Toggling week mode ON for Tuesday does
  NOT create a template. It just enables the long-press-task-
  label gesture for that weekday. The day continues to render
  the standard four until the user actually changes a task.

  The first edit creates the divergence. From that point on,
  Tuesdays render the diverged template instead of the standard
  four.

  This means the toggle is incredibly cheap. No template to
  create on toggle-on, none to destroy on toggle-off. Toggle is
  pure permission-state: "can I long-press task labels on this
  weekday or not." Templates exist as a byproduct of editing,
  not toggling.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE FINAL MODEL — INTERACTION TABLE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| Day state          | Long-press cal-icon            | Long-press task label   |
|--------------------|--------------------------------|-------------------------|
| Non-week-mode      | Opens standard-four editor     | Inert (no behaviour)    |
| Week-mode, today   | Opens standard-four editor +   | Edits that task for the |
|                    | shows highly visible warning   | weekday template;       |
|                    | that today is templated and    | applies to today (if    |
|                    | edits here won't apply to      | not sealed) and future  |
|                    | today                          | instances of weekday    |
| Week-mode, future  | (Same as today's behaviour     | Same as today's        |
|                    | when navigated to that day)    | behaviour              |
| Sealed past day    | Read-only — past is immutable | Inert                  |

CAL-ICON BYPASS BONUS:
  In the prototype, the cal-icon menu has a "completed tasks
  become uneditable" rule, and when all four are ticked the
  menu greys out entirely. That rule was a small frustration
  worth flagging — sometimes users want to revise their
  standard four even when today's done.

  Week mode unintentionally solves this. On a week-mode day,
  the user has access to the long-press-task-label gesture
  regardless of whether tasks are ticked. They can edit their
  template even mid-day with all four tasks complete. The
  template-edit path bypasses the cal-icon's tick-locks
  entirely.

  This wasn't designed in. It's a happy emergent property of
  the model. Recorded here so future-Morgan recognises that
  the cal-icon's locking rules and the task-label edit path
  are deliberately decoupled.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE STANDARD FOUR — CLARIFYING WHAT IT IS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

This was implicit in the prototype but worth documenting
explicitly now that there's a second concept (per-weekday
templates) sitting alongside it.

THE STANDARD FOUR:
  The user's universal default — the four task names they use
  across all non-templated days. Maintained via the cal-icon
  menu (top-right of the main screen). When edited, propagates
  to all not-yet-completed days that aren't on a weekday
  template.

  The cal-icon's existing tick-locking rule still applies for
  non-templated days: tasks completed today are uneditable from
  the cal-icon for the rest of the day, to prevent retroactive
  ambiguity.

  Schema: stored on `users` table. Currently `users.task_1,
  users.task_2, users.task_3, users.task_4` (or whatever the
  prototype's existing convention is — confirm at implementation
  time). No new columns needed for this concept; it already
  exists.

WEEKDAY TEMPLATES (NEW):
  Optional per-user per-weekday overrides of the standard four.
  Each template has four task names. Exists only when the user
  has changed at least one task while on a week-mode day for
  that weekday.

  Schema: new table `user_weekday_overrides` (proposed name).
  Columns: user_id, weekday (0-6 or "Mon"-"Sun"), task_1,
  task_2, task_3, task_4, created_at, updated_at. Rows exist
  only for weekdays where the user has authored divergence.
  Absence of a row = no template = render standard four.

WEEK MODE TOGGLE STATE:
  Whether the long-press-task-label gesture is enabled for a
  given user × weekday. Independent of whether a template
  exists.

  Schema: probably a small bitmask column on users
  (`users.week_mode_weekdays INTEGER`, 7 bits, one per weekday).
  Or a JSON array. Either way, lightweight — seven booleans per
  user.

  Possible (forward-compatible) cleanup at implementation time:
  consider whether to fold this into the user_weekday_overrides
  table by treating "row exists" as "week mode on." But that
  conflates two things — toggle state (permission to edit) and
  divergence (actual template). Keep them separate.

WHAT RENDERS ON A GIVEN DAY:
  Logic (in order):
    1. If the day is sealed, render the frozen seal state.
       Stop.
    2. If a template exists for this user × weekday AND week
       mode is on for this weekday, render the template.
    3. Otherwise, render the standard four.

  Note: a template can exist while week mode is off (the user
  authored it then toggled off). In that case, rule 3 applies
  — standard four. Rule 2 requires BOTH conditions.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EDITING PROPAGATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CAL-ICON EDITS (standard four):
  Apply to all not-yet-completed days that AREN'T currently
  rendering a template. A week-mode day with a divergent
  template ignores cal-icon edits for its task display, but
  the standard four is still updated for non-templated days.

  On a week-mode day, the cal-icon menu opens with a highly
  visible warning explaining that today's task display is
  templated and that edits to the standard four won't affect
  today. The warning is the redirect — it points the user at
  the task-label long-press path for editing the template.

  The warning copy should communicate clearly without scolding.
  Something like: "This is your standard four. Today's Tuesday
  is on its own template — edits here won't change today's
  tasks. To edit today's Tuesday template, long-press the
  task labels below." Refine at implementation.

LONG-PRESS TASK LABEL EDITS (template):
  Apply to today's day (if unsealed) and all future instances
  of the same weekday. Never back-propagate to sealed past
  days.

  If today's tasks are partially ticked when the edit happens,
  only unticked tasks update. Ticked tasks are immutable
  history and stay as they were.

THE TOGGLE ITSELF:
  Doesn't propagate to anything. It's permission-state for the
  long-press gesture. Toggling on enables the gesture for that
  weekday; toggling off disables it.

  No prompt, no confirmation, no propagation warning. Just an
  immediate visual state change on the day heading.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
VISUAL INDICATORS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

DAY-HEADING WHEN WEEK MODE IS ON:
  Subtle visual change to the day name (e.g. "Tuesday") to
  signal that long-press is now meaningful. Highlight, underline,
  small typographic shift — something at the threshold of
  noticeability.

  Specifically NOT a loud indicator. No badge, no "WEEKLY"
  label, no slashed-across-the-word treatment. The user
  toggled it on, they know it's on; the indicator's job is
  to remind, not announce.

  Defer the exact visual treatment to implementation. Constraint:
  must be subtle enough that non-week-mode users don't notice
  or wonder about it, but consistent enough that a week-mode
  user can glance at any day heading and read its state at a
  glance.

TASK LABELS WHEN WEEK MODE IS ON FOR TODAY:
  Subtle persistent visual indicator on each task label,
  hinting that long-press is available. Even more subtle than
  the day-heading indicator — task labels are the densest text
  area in the app and any indicator competes with the task
  content itself.

  Defer exact treatment to implementation. Note for the
  designer: this indicator may not be needed at all if the
  introductory copy / help-menu explanation is sufficient.
  Listed here because it might prove useful, but flagged as
  optional rather than required.

  Constraint for the UI designer: this area is high-density
  and prone to collision with other future features. Be
  defensive when designing what sits next to / around task
  labels.

DAY-CELL CALENDAR INDICATOR (on the month grid):
  Deferred and possibly unnecessary. The argument: a user
  might toggle week mode on for Tuesday months ago, forget,
  and not realise their Tuesdays are diverging.

  Counter-argument: the divergent tasks are visible whenever
  they look at a Tuesday. The cal-icon warning fires whenever
  they try to edit standard-four from a templated day. The
  user has feedback channels. A calendar-grid indicator would
  be over-engineering for a problem the existing surfaces
  already handle.

  Conclusion: don't ship this in v1 of the feature. If user
  testing reveals confusion in practice, add a calendar-level
  indicator later.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
REVEAL — DEFERRED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Reveal timing is deferred to a future staggered-disclosure
design pass.

Some constraints worth recording here for that future pass:

  - This is NOT a day-1 feature. Day-1 users are still
    figuring out the basic four-tasks loop. Adding a feature
    reveal here competes with fragile early habit formation.
  - This feature deserves its own dedicated reveal moment, NOT
    bundled with another disclosure. The picker long-press
    reveal (existing in the staggered disclosure doc) is
    about the theme system; day-heading long-press is about
    task structure. Conceptually different. Bundling them
    might confuse users about what long-press IS in this app.
  - The reveal should land when the user has experienced the
    basic rhythm long enough to recognise the friction this
    feature solves. Likely day 10-14+ range, but the right
    answer depends on overall disclosure cadence which is
    being designed elsewhere.
  - The reveal copy should not patronise. It should
    acknowledge: "Four Tasks defaults to simple and uninvasive.
    We know some users want more control over how their week's
    tasks look. Here's a tool for that." (Refined version of
    Morgan's draft framing from session 8 conversation.)
  - The feature should ALSO be discoverable via the in-app
    help menu for users who didn't see the reveal or want to
    explore.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SCHEMA SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

New schema additions for this feature (lands in whichever
migration introduces it — NOT bundled with current Phase 1
migrations 003/004/005):

  ALTER TABLE users ADD COLUMN week_mode_weekdays INTEGER
    DEFAULT 0;
    -- Bitmask of weekdays where week mode is on for this user.
    -- Bit 0 = Sunday, bit 1 = Monday, ..., bit 6 = Saturday.
    -- (Or whichever weekday-indexing convention the rest of the
    -- codebase uses; align at implementation.)

  CREATE TABLE user_weekday_overrides (
    user_id TEXT NOT NULL,
    weekday INTEGER NOT NULL,  -- 0-6, same convention as above
    task_1 TEXT NOT NULL,
    task_2 TEXT NOT NULL,
    task_3 TEXT NOT NULL,
    task_4 TEXT NOT NULL,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL,
    PRIMARY KEY (user_id, weekday),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
  );

Notes:
  - A row in user_weekday_overrides exists only when the user
    has authored divergence (changed at least one task while on
    a week-mode day for that weekday). Absence = no template.
  - The week_mode_weekdays bitmask is independent. A template
    can exist while week mode is off (user authored then
    toggled off). The render logic combines both.
  - This is a self-write only. Partners cannot read or write
    each other's week_mode_weekdays or user_weekday_overrides.
    Write rules at implementation time treat these as private
    user fields.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EDGE CASES AND OPEN QUESTIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

These are flagged for revisit at implementation time, NOT now.

  - WHAT HAPPENS WHEN THE STANDARD FOUR HAVE FEWER THAN FOUR
    NAMES? The prototype currently always has four tasks set.
    If a future feature ever permits fewer than four standard
    tasks, the template logic needs to align — does a templated
    day still need four tasks? Probably yes (the app is called
    Four Tasks), but flag for confirmation.

  - WHAT IF THE USER EDITS THE STANDARD FOUR WHILE A WEEK-MODE
    DAY IS DISPLAYED? The cal-icon warning fires. The user
    confirms the edit. Standard four updates. Today's templated
    day display remains unchanged (it's on its template).
    Tomorrow (if non-templated) reflects the new standard four.
    This is correct behaviour but might surprise users on first
    encounter — flagged for clear warning copy.

  - WHAT IF A TEMPLATE'S TASKS BECOME THE SAME AS THE STANDARD
    FOUR AGAIN? The user edits the template back to match the
    standard four. The template row still exists in the
    database. The render logic still pulls from it (rule 2 wins
    over rule 3 when week mode is on). Functionally identical
    output to non-templated rendering. Probably fine. Doesn't
    auto-delete the row because that would surprise the user
    next time they toggled week mode off and back on (they'd
    expect the template to come back).

  - DAYLIGHT SAVING / TIMEZONE CHANGES NEAR MIDNIGHT? The
    weekday-of-today calculation uses the user's IANA timezone
    (per the timezone+sealing doc). Standard rules apply. Week
    mode templates are user-private state, so they don't
    interact with partner timezone differences.

  - PARTNER ANALYTICS LEAKAGE? Could partner observe that user
    is on week mode (e.g. via inferring from task patterns)?
    Yes — partner sees their partner's tasks each day. If
    Tuesday tasks are always identical, the partner can infer
    a template. This is fine; the system doesn't promise
    secrecy of templates, just that they're not synchronised.

  - REST DAY ON A TEMPLATED WEEKDAY? Rest days override the
    daily task tray entirely (per existing prototype behaviour).
    A rest-day Tuesday doesn't show the Tuesday template; it
    shows the rest-day UI. The template is preserved and
    returns next Tuesday.

  - MULTIPLE USERS ON SHARED DEVICE? Not a current scenario —
    one user per device per identity. If multi-user-per-device
    ever ships, week mode state is per-user and switches with
    the active user. Not a concern for v1.0.

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
  is their own. Partners don't see toggles, don't sync
  templates, don't even know whether their partner is using
  the feature.

  NOT default-on for anyone. Off by default for every user,
  every weekday. Has to be deliberately discovered or revealed.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED DOCS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  - four_tasks_staggered_disclosure_design_notes.md
    Reveal timing for week mode lives here when the disclosure
    pass updates. This doc flags the constraints to honour.

  - four_tasks_timezone_and_sealing_design_notes.md
    Weekday-of-today calculation uses the user's IANA timezone.
    Sealed past days are immutable, which constrains template
    propagation rules.

  - four_tasks_write_rules_design_notes.md
    user_weekday_overrides and week_mode_weekdays are self-write
    only. Partner cannot read or write either. Write rules at
    implementation time enforce this.

  - four_tasks_pair_key_design_notes.md (v2)
    Neither week_mode_weekdays nor user_weekday_overrides
    participates in pair-key hashing. They're private user
    state, freely changeable without identity migration.

  - four_tasks_architectural_preference.md
    Clarity over cleverness informed the divergence-on-edit
    model. The cheaper alternatives (toggle creates a template
    immediately, or toggle and edit happen as separate actions)
    were rejected for being slightly more complex without being
    any clearer.

  - four_tasks_tracking_design_notes.md
    Not directly related, but the same conversation produced
    both. Week mode is a counter-example to scope creep — a
    feature request was reshaped into something that fits the
    app's positioning rather than rejected outright OR added
    wholesale.

  - four_tasks_godot_devlog.txt
    Session 8 captures the discovery conversation that produced
    this doc.
