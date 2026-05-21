# Four Tasks — Week Mode Design Notes

Last edit: 2026-05-21 AWST

Status: DESIGN LOCKED (session 8 mobile, session 12 cascade flow added).
        v1.0 scope (promoted from v1.x at session 9).
        Implementation tile not yet numbered; lands in Phase 4.

Schema implications: one new optional table (user_weekday_overrides).
Reveal: day-2 cascade from long-press philosophy reveal (per staggered
disclosure design notes). See "DAY-2 CASCADE FLOW" below.

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
| Week-mode, today   | Opens standard-four editor     | Edits that task for the |
|                    | with inline warning banner     | weekday template;       |
|                    | (red, persistent for duration  | applies to today (if    |
|                    | of editor session) explaining  | not sealed) and future  |
|                    | today is templated and edits   | instances of weekday    |
|                    | here won't apply to today      |                         |
| Week-mode, future  | (Same as today's behaviour     | INERT — gesture is not  |
|                    | when navigated to that day)    | available on navigated  |
|                    |                                | future days. See        |
|                    |                                | "Today-only edit rule"  |
|                    |                                | below.                  |
| Sealed past day    | Read-only — past is immutable | Inert                  |

WARNING BANNER ON CAL-ICON (week-mode days):
  Not a separate popup. The standard-four editor opens as
  normal; a high-contrast inline banner (likely red) is
  superimposed at the top of the editor modal explaining that
  today is on a [weekday] template, so edits here apply to
  non-templated days only.
  The banner is persistent for the duration of the editor
  session — it doesn't dismiss until the user closes the
  editor. No "don't show again" toggle; the warning fires
  every time because the state it warns about is per-session
  ambiguous (user might forget which weekdays are templated).
  The banner doubles as a pointer: it tells the user the
  correct path for editing today (long-press a task label
  below).

TODAY-ONLY EDIT RULE:
  Long-press-task-label is live ONLY on the day the user is
  currently viewing as today. Navigating to a future Tuesday and
  long-pressing a task label does nothing — even if week mode is
  on for Tuesdays. The gesture remains inert on past sealed
  Tuesdays for obvious immutability reasons; it ALSO remains
  inert on future Tuesdays despite them being editable in
  principle.

  Reasoning:
    - There is one moment, one place, where template edits
      happen: today, on the day the user is currently living
      through. This keeps the edit point unambiguous.
    - If the gesture were live on a navigated future Tuesday,
      the user could edit it. The template propagation rule
      would push that change forward to all subsequent Tuesdays
      but NOT back to today's Tuesday (today is "past" relative
      to the edited day). Result: today renders the OLD template,
      next Tuesday renders the NEW template, and the user has no
      clear sense of where the cutover happened. Bad UX.
    - Allowing the gesture on past Tuesdays would suggest
      historical edits are possible, which is a wrong affordance
      under the immutable-past principle.
    - There is no legitimate use case for editing a non-today
      Tuesday. Future Tuesdays render the template; they don't
      author it. Past Tuesdays are sealed. The edit always wants
      to happen today.

  Implementation note: the long-press handler on task labels
  must check (a) week mode is on for the current weekday AND
  (b) the day currently being rendered is today's day. Both
  must be true for the gesture to fire. "Today" here means
  "the day matching system clock in user's IANA timezone,"
  NOT "the most recently selected day-cell that happens to be
  today's date."

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
DAY-2 CASCADE FLOW (session 12)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The staggered disclosure design notes specify a day-2 long-press
philosophy reveal that cascades into the week mode tutorial. This
section locks how that cascade interacts with the toggle gesture
without breaking either feature's design.

THE COLLISION:
  The day-2 philosophy reveal dismisses when the user performs
  the taught gesture (long-press on a day name). The week mode
  feature uses the same gesture to toggle week mode on for that
  weekday. The dismissal IS the toggle. Side-effect of a
  tutorial gesture is normally a smell, but in this case the
  cascade is designed to make it the lesson — the user finds
  out what they just did by being told.

THE FLOW:
  1. Day 2 fires. Long-press philosophy reveal appears
     (per staggered disclosure doc).
  2. User long-presses a day name. The philosophy reveal
     dismisses. week_mode_weekdays bit for that weekday is
     set to ON.
  3. The week mode tutorial reveal fires immediately with copy
     along these lines:

     > You've turned week mode on for [weekday]. From now on,
     > your [weekday]s can have their own four. Long-press a
     > task label to give it a go, or long-press [weekday]
     > again to toggle it off — if you're happy with your
     > tasks being the same every day.

  4. EITHER user action clears the tutorial and cedes control
     back to the app:
       a. Long-press a task label → opens the task label editor
          for the user to author divergence (this is the
          "give it a go" path).
       b. Long-press the day name again → toggles week mode
          OFF for that weekday and dismisses the tutorial
          (this is the "happy with tasks being the same"
          path).
  5. Both paths set tutorial_progress.week_mode_intro to a
     timestamp. The reveal never fires again, per the
     server-as-source-of-truth merge semantics in the
     staggered disclosure doc.

EITHER ACTION IS A VALID DISMISSAL:
  The tutorial does not have a "Got it" button. The dismissal
  pattern follows the staggered disclosure doc's gesture-
  teaching language — the act of using the feature (or
  rejecting it) is the dismissal. This matches the day-2
  philosophy reveal pattern that immediately preceded it.

NO BACK-OUT:
  Once the user long-presses a day name and the toggle fires,
  the week_mode_weekdays bit stays set unless the user
  explicitly toggles off via path (b). If the user closes the
  app mid-tutorial, the bit stays set and the tutorial reveal
  re-fires on next app boot until one of the two paths is
  taken. This is fine — week mode is on but no template
  exists, so rendering falls through to standard four (rule 3
  in the render logic). No visible behaviour change until the
  user engages with the gesture properly.

USERS WHO NEVER ENGAGE AGAIN:
  A user who long-presses a day name to dismiss the philosophy
  reveal, then ignores the week mode tutorial and never
  returns to it, leaves the app with one weekday toggled on
  and no template. Render rules handle this fine — no visible
  difference from non-templated rendering. The subtle day-
  heading indicator (see VISUAL INDICATORS section) flags the
  state for anyone who looks at it later.

  This was considered as a potential issue worth fixing (auto-
  toggle-off if the user disengages from the tutorial). Decided
  not to fix: invisible-state cost is zero, and the latent
  toggle means a user who returns to the feature months later
  already has a starting point. Cost is harmless; benefit is
  real for re-discovery.

WHICH WEEKDAY GETS TOGGLED:
  Whichever day the user happens to long-press during the
  philosophy reveal. There is no guidance in the philosophy
  reveal copy steering the user toward any specific day. The
  user picks whatever day name is most visible to them in the
  moment. If they pick poorly (today is Sunday, they long-
  press Wednesday for no particular reason), the path (b)
  off-ramp is the recovery.

  Acceptable randomness in feature onboarding. Forcing the
  user to pick a "correct" day before the reveal completes
  would over-engineer the tutorial and break the gesture-
  dismissal language.

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
  Columns: user identity (matching project convention —
  likely (pair_key, name) compound key), weekday (0-6 or
  "Mon"-"Sun"), task_1, task_2, task_3, task_4, created_at,
  updated_at. Rows exist only for weekdays where the user has
  authored divergence. Absence of a row = no template = render
  standard four.

WEEK MODE TOGGLE STATE:
  Whether the long-press-task-label gesture is enabled for a
  given user × weekday. Independent of whether a template
  exists.

  Schema: stored on users table. Two options to decide at
  implementation:
    - Seven boolean columns (week_mode_sun ... week_mode_sat).
      Pros: legible everywhere it's read, matches
      architectural preference for clarity over cleverness.
      Cons: seven columns for one logical concept.
    - One INTEGER bitmask column (week_mode_weekdays).
      Pros: one column, one migration line.
      Cons: every read site needs bitshift math
      (`(week_mode_weekdays >> 2) & 1`), reduced legibility.
  Architectural preference doc leans toward booleans for this
  app's throughput profile (~6 writes/user/day). Lock the
  choice when this migration lands.

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

  MID-DAY TOGGLE BEHAVIOUR:
  If the user toggles week mode ON for today's weekday mid-day
  AND a template already exists for that weekday, today's
  display switches from standard four to template immediately
  per rule 2. Already-ticked tasks remain ticked under their
  original labels (immutable per the partial-tick rule), but
  unticked task slots re-render with template labels. This is
  consistent but may surprise users; the day-2 cascade flow
  largely sidesteps the issue because the toggle's first
  encounter is gesture-paired with the tutorial.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EDITING PROPAGATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CAL-ICON EDITS (standard four):
  Apply to all not-yet-completed days that AREN'T currently
  rendering a template. A week-mode day with a divergent
  template ignores cal-icon edits for its task display, but
  the standard four is still updated for non-templated days.

  On a week-mode day, the cal-icon menu opens with an inline
  warning banner (red, persistent for the duration of the
  editor session) at the top of the editor modal explaining
  that today's task display is templated and that edits to
  the standard four won't affect today. The banner is the
  redirect — it points the user at the task-label long-press
  path for editing the template.

  The banner copy should communicate clearly without scolding.
  Something like: "This is your standard four. Today's Tuesday
  is on its own template — edits here won't change today's
  tasks. To edit today's Tuesday template, long-press the
  task labels below." Refine at implementation.

LONG-PRESS TASK LABEL EDITS (template):
  Available ONLY on today's day view, when week mode is on for
  today's weekday (see "Today-only edit rule" in the interaction
  table). Inert on all other day views.

  An edit applies to today's day (if unsealed) and all future
  instances of the same weekday. Never back-propagates to sealed
  past days.

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
REVEAL TIMING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

LOCKED (session 12): Day 2, cascaded from the long-press
philosophy reveal. See DAY-2 CASCADE FLOW above for the full
interaction sequence and the staggered disclosure design
notes for the philosophy reveal that precedes it.

Help menu coverage: the help menu (tile 4.11) must include
reference copy for week mode, per the staggered disclosure
doc's session 12 scope obligation on the help menu. Reveals
never re-fire; users who want to revisit the explanation go
to help.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SCHEMA SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

New schema additions for this feature (lands in whichever
migration introduces it — NOT bundled with current Phase 1
migrations 003/004/005):

  -- Option A (preferred per architectural preference doc):
  ALTER TABLE users ADD COLUMN week_mode_sun BOOLEAN DEFAULT 0;
  ALTER TABLE users ADD COLUMN week_mode_mon BOOLEAN DEFAULT 0;
  ALTER TABLE users ADD COLUMN week_mode_tue BOOLEAN DEFAULT 0;
  ALTER TABLE users ADD COLUMN week_mode_wed BOOLEAN DEFAULT 0;
  ALTER TABLE users ADD COLUMN week_mode_thu BOOLEAN DEFAULT 0;
  ALTER TABLE users ADD COLUMN week_mode_fri BOOLEAN DEFAULT 0;
  ALTER TABLE users ADD COLUMN week_mode_sat BOOLEAN DEFAULT 0;

  -- Option B (denser, if column count is a concern):
  -- ALTER TABLE users ADD COLUMN week_mode_weekdays INTEGER
  --   DEFAULT 0;
  --   -- Bitmask of weekdays where week mode is on for this user.
  --   -- Bit 0 = Sunday, bit 1 = Monday, ..., bit 6 = Saturday.

  -- Template table (compound primary key matching project convention):
  CREATE TABLE user_weekday_overrides (
    pair_key TEXT NOT NULL,
    name TEXT NOT NULL,
    weekday INTEGER NOT NULL,  -- 0-6, same convention as above
    task_1 TEXT NOT NULL,
    task_2 TEXT NOT NULL,
    task_3 TEXT NOT NULL,
    task_4 TEXT NOT NULL,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL,
    PRIMARY KEY (pair_key, name, weekday),
    FOREIGN KEY (pair_key, name) REFERENCES users(pair_key, name)
  );

Notes:
  - A row in user_weekday_overrides exists only when the user
    has authored divergence (changed at least one task while on
    a week-mode day for that weekday). Absence = no template.
  - The week mode toggle state is independent. A template
    can exist while week mode is off (user authored then
    toggled off). The render logic combines both.
  - This is a self-write only. Partners cannot read or write
    each other's week mode state or user_weekday_overrides.
    Write rules at implementation time treat these as private
    user fields.
  - Foreign key shape assumes pair_key + name as the user
    identity convention. Confirm against the canonical users
    table at implementation time.

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
    DAY IS DISPLAYED? The cal-icon banner fires. The user
    confirms the edit. Standard four updates. Today's templated
    day display remains unchanged (it's on its template).
    Tomorrow (if non-templated) reflects the new standard four.
    This is correct behaviour but might surprise users on first
    encounter — flagged for clear banner copy.

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
    Day-2 long-press philosophy reveal cascades into the week
    mode tutorial per this doc's DAY-2 CASCADE FLOW section.
    Help menu obligation: reference copy for week mode lives
    in tile 4.11.

  - four_tasks_timezone_and_sealing_design_notes.md
    Weekday-of-today calculation uses the user's IANA timezone.
    Sealed past days are immutable, which constrains template
    propagation rules.

  - four_tasks_pair_key_design_notes.md (v2)
    Neither week mode columns nor user_weekday_overrides
    participates in pair-key hashing. They're private user
    state, freely changeable without identity migration.

  - four_tasks_architectural_preference.md
    Clarity over cleverness informed the divergence-on-edit
    model AND the seven-boolean schema preference over the
    bitmask. The cheaper alternatives (toggle creates a template
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
    this doc. Session 12 adds the day-2 cascade flow.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SESSION 12 CHANGES SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- DAY-2 CASCADE FLOW section added. Resolves the design
  collision between the staggered disclosure day-2 long-press
  philosophy reveal and the week mode toggle gesture. Locks
  the flow: dismissal IS the toggle, the tutorial reveal
  explains what just happened, either user action (long-press
  task label OR long-press day name again to off) clears the
  tutorial and cedes control.
- Reveal timing section updated from "deferred" to locked at
  day 2.
- Cal-icon warning clarified as inline banner inside the
  editor modal, not a separate popup. Interaction table
  updated accordingly.
- Today-only edit rule implementation note clarified: "today"
  means system-clock-today in user timezone, NOT
  most-recently-tapped-day-cell-that-happens-to-be-today.
- Mid-day toggle behaviour explicitly documented in render
  logic section.
- Schema: bitmask column replaced as primary option with
  seven boolean columns, per architectural preference doc.
  Bitmask retained as commented Option B fallback.
- Schema: user_weekday_overrides FK shape aligned to
  (pair_key, name) compound key matching project convention.
- Cross-references updated to point to the staggered
  disclosure doc's day-2 cascade and help menu obligations.