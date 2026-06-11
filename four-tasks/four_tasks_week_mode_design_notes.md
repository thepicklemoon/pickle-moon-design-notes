# Four Tasks — Week Mode Design Notes

Status: DESIGN LOCKED (session 8 mobile, session 12 cascade flow,
        2026-06-11 INTERACTION FORK RESOLVED → MODEL B, the popup
        editor; Model A retired with reasoning recorded below).
        v1.0 SCOPE (promoted from v1.x at session 9; reconfirmed
        2026-06-10 against forecast-review pressure to defer —
        Morgan's call, stays in).
        Schema: v1.0 shipped shape needs ONE pre-build change —
        see SCHEMA SUMMARY (override table reshaped per-slot; no
        user data exists, drop-and-recreate is safe).
        Implementation tile 4.D2 (Phase 4).
        Streak question RESOLVED 2026-06-10: moot — week mode is
        label-overrides only; every day still has exactly four
        tasks, so streaks, sealing, payout, and stamps are
        untouched (consistent with the "single clock" reasoning
        below). The old PARKED "streak semantics under week mode"
        item is closed.

Schema implications: user_weekday_overrides (reshaped — per-slot
text + active bitmask); users.week_mode_weekdays RETIRED-UNUSED.
Reveal: day-2 cascade from long-press philosophy reveal (per staggered
disclosure design notes). See "DAY-2 CASCADE FLOW" below — reshaped
2026-06-11 for the popup model; one-line sync owed to the staggered
disclosure doc.

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

Week mode lets the user maintain per-weekday overrides of their
standard four — any slot of any weekday can carry its own task
label, diverging from the universal default.

Off by default for everyone. No override exists for any slot of
any weekday of any user until deliberately authored.

Granularity is per-SLOT per-weekday: a user might override only
task 2 on Tuesdays and tasks 1+4 on Sundays. Non-overridden slots
continue to track the standard four live. The granularity matches
how people actually live — most users have a few anchored
commitments and the rest is free.

THE SHAPE (Model B, locked 2026-06-11):
  - User long-presses the day name in the task tray heading
    (e.g. "TUESDAY") of any open day's tray. A popup opens: the
    weekday template editor for that weekday.
  - The popup shows four input fields, one per slot. Each field
    is greyed and uninteractable by default, showing the slot's
    CURRENT EFFECTIVE label as a placeholder (the standard four,
    or the dormant override text's slot-effective render — see
    render rules). A radio selector sits beside each field.
  - Selecting a radio makes that field writable. The field
    pre-fills with the dormant override text if one exists for
    that slot, else with the current standard label. Whatever
    the field holds while the radio is ON is that slot's
    override.
  - An override applies to all not-yet-sealed instances of that
    weekday — today included if today matches and is unsealed.
  - Deselecting a radio reverts that slot to the standard four
    on all not-yet-sealed instances. The authored text is
    PRESERVED dormant (the session-8 "paused, not deleted"
    answer, now per-slot). Reselecting restores it.
  - Sealed days are immutable. Template changes never retcon
    sealed past days. (Server-enforced regardless.)
  - Closing the popup with no radios selected changes nothing —
    zero state, no rows written.

PERSONAL, NOT SHARED. Each user's week mode state is their own.
Partners do not see or sync each other's overrides.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
INTERACTION FORK — RESOLVED 2026-06-11 → MODEL B
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Two candidate interaction models existed. Both shared the data
model (per-weekday label overrides, propagation to all
not-yet-sealed matching weekdays, personal-not-shared); they
differed in surface and gesture grammar.

MODEL A (retired) — the two-gesture model: long-press the day
name in the tray heading to silently toggle the weekday; then
long-press an individual task label to edit it. No popup.

MODEL B (locked) — the popup editor: long-press the day name to
open a popup with four fields + radio selectors, described in
THE SHAPE above.

WHY B WON (Model A's epitaph, per the decision rule — recorded
so this isn't re-litigated):

  1. The popup rejection didn't apply. The discovery arc
     rejected a CONFIRMATION prompt ("propagate Tuesdays?
     yes/no") as two-beat. Model B's popup is the EDITOR
     itself, not a confirmation — one long-press lands on a
     working surface. Same beat count as A's toggle, but the
     user arrives somewhere visible instead of flipping
     invisible permission state.

  2. A's today-only edit rule was an artefact of its surface,
     and it was the model's worst constraint. Editing in-place
     on the live tray forced "you can only author Wednesday's
     template ON a Wednesday" — a FIFO user building a roster
     week needed five separate days of setup, and the doc's own
     first-iteration reasoning cites FIFO rosters as the
     motivating population. B's popup edits the TEMPLATE, not
     the day, so the gesture is legitimately live on any open
     day's heading — sealed past days included, because the
     popup never touches that day's frozen data. After one week
     of use, every weekday is reachable from calendar history.
     (Future cells stay dead per 4.14c; nothing changes there.)

  3. A's divergence-on-edit left it ambiguous whether un-edited
     slots track later standard-four changes once a day
     "diverges." B's per-slot radios make it explicit: radio
     OFF = live standard-four tracking, radio ON = frozen
     override. The pause semantics map per-slot for free.

  4. Discoverability. A was two invisible gestures whose
     entire teaching burden fell on the day-2 cascade; its
     toggle "does nothing visible at first." B is one gesture
     opening a self-explaining surface.

  What A had that B keeps: the cheap-toggle insight survives
  as cheap-radios (no template row until a radio is selected);
  the pause-not-delete answer survives per-slot; the cal-icon
  warning banner and the bypass bonus survive unchanged (see
  below).

  What dies with A: the weekday-level toggle (and with it the
  users.week_mode_weekdays bitmask — retired-unused), the
  today-only edit rule, the task-label long-press gesture, the
  day-2 cascade's toggle-as-side-effect awkwardness ("no
  back-out," "which weekday gets randomly toggled").

DECISION DETAIL (2026-06-11): selecting a radio stores the
override immediately, pre-filled text and all, regardless of
whether the user edits it. Radio state means what it shows;
storing only-if-different-from-standard would be cleverness.
A selected-but-unedited slot is a frozen copy of the standard
label at selection time — later standard-four edits won't
propagate to it, and the popup's grey-vs-active rendering is
what makes that frozen-ness legible.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE DISCOVERY ARC — HOW THIS MODEL EMERGED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Captured for future-Morgan because the back-and-forth itself
shows why the final model is what it is. Iterations 1-6
produced Model A; the seventh resolved the fork to Model B.

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
  interaction where one would do. (NOTE, 2026-06-11: this
  rejection was of a CONFIRMATION prompt. It was later
  mis-remembered as a rejection of popups generally, which
  nearly killed the better model — an editor surface is not a
  confirmation. Recorded so the distinction survives.)

THIRD ITERATION: long-press task label edits "this task forever."
  Once week mode is on for Tuesday, long-press a task label
  edits that task across all future Tuesdays. (Superseded by
  the popup's radio-fields, but the propagation rule it
  established survived.) The version of the model at this
  stage had the cal-icon (top-right calendar menu) greyed out
  on week-mode days, with redirect copy.

  Wrong. The cal-icon menu is the universal editor for the
  user's STANDARD FOUR tasks, separate from any weekday
  templating. Greying it out on templated days would prevent
  the user from editing their standard four while ON a
  templated day, which is overreach. The cal-icon must stay
  live, but with a warning communicating that edits there
  won't affect today's templated slots.

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
  Open question: when the user turns an override off, does it
  get deleted, paused, or prompted ("keep or discard?").

  Settled: paused. The authored text is preserved silently;
  turning it back on restores it. No popup, no theatre, no
  state confusion — the friendliest behaviour is also the
  simplest. The user doesn't lose work for experimenting.
  (Under Model B this operates per-slot via the radios.)

SIXTH ITERATION: divergence-on-edit, not on-toggle.
  Under Model A: toggling a weekday on created no template;
  templates existed as a byproduct of editing. The surviving
  principle: state creation is cheap and lazy. Under Model B
  the same principle holds — opening the popup creates
  nothing; selecting a radio is the creation event.

SEVENTH ITERATION (2026-06-11): the fork resolution.
  Morgan's prototype-era model resurfaced (popup + per-slot
  radios) and won against the locked two-gesture model. Full
  reasoning in the fork section above. The short version: the
  popup is an editor not a confirmation, it dissolves the
  today-only constraint by decoupling editor from day, the
  radios disambiguate slot-tracking, and it's self-explaining
  where A was invisible.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE FINAL MODEL — INTERACTION TABLE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| Surface / day state       | Behaviour                                |
|---------------------------|------------------------------------------|
| Long-press tray-heading   | Opens the weekday template popup for     |
| day name — ANY open       | that day's weekday. Live on today AND    |
| day's tray                | past days (sealed included): the popup   |
|                           | edits the template, never the day.       |
|                           | Propagation is forward-only regardless   |
|                           | of which day's tray hosted the gesture.  |
| Future cells              | No tray opens at all (4.14c, DEAD_       |
|                           | FUTURE non-interactive) — moot.          |
| Popup: radio OFF slot     | Greyed, uninteractable field showing the |
|                           | slot's current effective label as        |
|                           | placeholder. Slot tracks the standard    |
|                           | four live.                               |
| Popup: radio ON slot      | Writable field; override stored on       |
|                           | selection (pre-filled with dormant text  |
|                           | if any, else current standard label).    |
|                           | Propagates to all not-yet-sealed         |
|                           | instances of the weekday, today included |
|                           | if matching + unsealed.                  |
| Popup: deselect a radio   | Slot reverts to standard four on all     |
|                           | not-yet-sealed instances; authored text  |
|                           | preserved dormant.                       |
| Long-press cal-icon,      | Opens standard-four editor as normal.    |
| non-templated day         |                                          |
| Long-press cal-icon, day  | Opens standard-four editor with the      |
| whose weekday has active  | inline warning banner (below).           |
| override slots            |                                          |
| Long-press task label     | INERT — the gesture does not exist in    |
|                           | Model B. (Retired with Model A.)         |
| Sealed past day's TASKS   | Read-only — past is immutable. Only the  |
|                           | heading long-press (template entry) is   |
|                           | live on a sealed day's tray.             |

WARNING BANNER ON CAL-ICON (days with active override slots):
  Not a separate popup. The standard-four editor opens as
  normal; a high-contrast inline banner (likely red) is
  superimposed at the top of the editor modal explaining that
  today's [weekday] carries template overrides, so edits here
  apply to the overridden slots on non-templated days only.
  The banner is persistent for the duration of the editor
  session — it doesn't dismiss until the user closes the
  editor. No "don't show again" toggle; the warning fires
  every time because the state it warns about is per-session
  ambiguous (user might forget which weekdays are templated).
  The banner doubles as a pointer: it tells the user the
  correct path for editing today's template (long-press the
  day name in the tray heading).

  Precision note: only ACTIVE override slots ignore standard-
  four edits. A day with overrides on slots 2+3 still reflects
  standard-four edits on slots 1+4 immediately. Banner copy
  should not overclaim ("today won't change at all" is wrong
  for partially-templated days). Refine copy at implementation.

CAL-ICON BYPASS BONUS (carried from Model A — still holds):
  The cal-icon menu has a "completed tasks become uneditable"
  rule, and when all four are ticked the menu greys out
  entirely. Template editing bypasses this: the heading
  long-press works regardless of tick state, so a user can
  revise a weekday template mid-day with all four complete.
  The cal-icon's locking rules and the template-edit path are
  deliberately decoupled. (Ticked slots still don't re-render
  today — see propagation rules — but the TEMPLATE is always
  editable.)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DAY-2 CASCADE FLOW (session 12; reshaped 2026-06-11 for Model B)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The staggered disclosure design notes specify a day-2 long-press
philosophy reveal that cascades into the week mode tutorial.
Under Model B the cascade gets SIMPLER: the taught gesture opens
a self-explaining surface instead of silently toggling state.

THE FLOW:
  1. Day 2 fires. Long-press philosophy reveal appears
     (per staggered disclosure doc).
  2. User long-presses the day name in the tray heading. The
     philosophy reveal dismisses. The weekday template popup
     opens for that weekday. NOTHING is toggled — opening the
     popup creates zero state.
  3. On the popup's FIRST-EVER open (any weekday), a one-time
     intro line renders inside the popup, above the fields.
     Copy along these lines:

     > This is your [weekday] template. Switch a slot on to
     > give your [weekday]s their own task — it'll carry to
     > every [weekday] from here on. Leave everything off and
     > your days stay the same.

  4. Any exit clears the tutorial: select a radio and author
     (the "give it a go" path), or close the popup untouched
     (the "happy as-is" path). Both set
     tutorial_progress.week_mode_intro to a timestamp; the
     intro line never renders again, per the server-as-source-
     of-truth merge semantics in the staggered disclosure doc.

WHAT THE RESHAPE DELETES (Model A artefacts, recorded):
  - "Dismissal IS the toggle" — there is no toggle. Dismissal
    opens an editor that commits nothing by itself.
  - "No back-out" handling — closing the popup untouched leaves
    zero state. Nothing latent, nothing to recover from.
  - "Which weekday gets toggled" randomness — whichever day's
    tray the user long-pressed simply determines which
    template the popup SHOWS; picking "wrong" costs nothing.
  - "Users who never engage again" — no latent toggled bit
    exists to be forgotten.

REACH ON DAY 2:
  The gesture is hosted on open trays, so a day-2 user can
  reach only today's and yesterday's weekdays. Acceptable:
  reach grows to all seven weekdays within the first week of
  use, and the popup is re-discoverable from any tray heading
  forever after. Help-menu copy (4.11) covers re-discovery.

SYNC OWED: one line in the staggered disclosure doc updating
the cascade's destination from "toggle + tutorial reveal" to
"popup with one-time intro line."

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE STANDARD FOUR — CLARIFYING WHAT IT IS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

THE STANDARD FOUR:
  The user's universal default — the four task names they use
  across all non-overridden slots. Maintained via the cal-icon
  menu (top-right of the main screen). When edited, propagates
  to all not-yet-completed days' slots that aren't carrying an
  active override.

  The cal-icon's existing tick-locking rule still applies:
  tasks completed today are uneditable from the cal-icon for
  the rest of the day, to prevent retroactive ambiguity.

  Schema: the standard four live in `users.task_labels` (a JSON
  array of four strings, shipped v1.0). Unchanged.

WEEKDAY TEMPLATES:
  Optional per-user per-weekday per-SLOT overrides of the
  standard four. A template row exists for a weekday once the
  user has ever selected a radio for any of its slots; each
  slot independently carries authored text (persistent) and an
  active flag (the radio).

WHAT RENDERS ON A GIVEN DAY:
  Logic (in order, per SLOT):
    1. If the day is sealed, render the frozen seal state.
       Stop. (Whole-day rule, not per-slot.)
    2. If an ACTIVE override exists for this user × weekday ×
       slot, render the override text for that slot.
    3. Otherwise, render the standard four label for that slot.

  A dormant override (text preserved, radio off) renders
  nothing — rule 3 applies. Rule 2 requires the active flag.

  MID-DAY RADIO CHANGES (today's weekday):
  Selecting or deselecting a radio for today's weekday
  re-renders today's affected slots immediately per the rules
  above — EXCEPT already-ticked slots, which remain ticked
  under their original labels (immutable per the partial-tick
  rule). Unticked slots re-render. Consistent, and the popup
  makes the cause visible in a way Model A's silent toggle
  never did.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EDITING PROPAGATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CAL-ICON EDITS (standard four):
  Apply to all not-yet-completed days on every slot NOT
  carrying an active override. Active-override slots ignore
  standard-four edits entirely (frozen copies). Dormant-
  override slots track the standard four like any other (the
  dormant text is untouched and unrelated).

  On a day whose weekday has active overrides, the cal-icon
  editor opens with the inline warning banner (see interaction
  table). The banner is the redirect — it points at the
  heading long-press for template editing.

TEMPLATE EDITS (popup):
  Available from any open day's tray heading. An active
  override applies to today (if matching weekday + unsealed)
  and all future instances of the weekday. Never back-
  propagates to sealed days — server-enforced; the seal is
  immutable regardless of client behaviour.

  If today's tasks are partially ticked when an override lands,
  only unticked slots update. Ticked slots are immutable
  history and stay as they were.

WRITE PATTERN:
  Overrides are server state, self-write only, following the
  standard optimistic-local-with-rollback convention. One
  write endpoint per weekday row (shape at 4.D2 build; the
  uniform envelope + status-code rules apply as everywhere).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
VISUAL INDICATORS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

DAY-HEADING WHEN THE WEEKDAY CARRIES ACTIVE OVERRIDES:
  Subtle visual change to the day name (e.g. "TUESDAY") in the
  tray heading, signalling this day renders templated slots.
  Highlight, underline, small typographic shift — something at
  the threshold of noticeability.

  Specifically NOT a loud indicator. No badge, no "WEEKLY"
  label. The user authored the template, they know it exists;
  the indicator's job is to remind, not announce. It also
  quietly advertises the long-press surface.

  Defer the exact visual treatment to implementation (4.D2
  with paint in 4b). Constraint: subtle enough that non-users
  don't notice or wonder, consistent enough that a template
  user can read any open day's state at a glance.

POPUP FIELD STATES:
  The grey-vs-active field rendering inside the popup IS the
  primary state display: greyed placeholder = tracking the
  standard four; active dark text = frozen override. No
  additional indicator needed inside the popup.

(RETIRED with Model A: the per-task-label affordance indicator —
no label gesture exists to advertise.)

DAY-CELL CALENDAR INDICATOR (on the month grid):
  Deferred and possibly unnecessary, unchanged reasoning: the
  divergent tasks are visible whenever the user opens a
  matching day; the cal-icon banner fires on edit attempts.
  Don't ship in v1 of the feature; add later only if user
  testing reveals confusion.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
REVEAL TIMING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

LOCKED (session 12): Day 2, cascaded from the long-press
philosophy reveal. See DAY-2 CASCADE FLOW above for the
(reshaped) interaction sequence and the staggered disclosure
design notes for the philosophy reveal that precedes it.

The gesture itself is live from day 1 — there is no gate, only
a teach. A day-1 user who long-presses a heading unprompted
gets the popup with the one-time intro line; the day-2 reveal
then has nothing to cascade into and follows the staggered
disclosure doc's already-consumed semantics.

Help menu coverage: the help menu (tile 4.11) must include
reference copy for week mode, per the staggered disclosure
doc's session 12 scope obligation. Reveals never re-fire;
users who want to revisit the explanation go to help.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SCHEMA SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

TARGET SHAPE (2026-06-11, Model B — supersedes the session-12
shipped shape; apply at 4.D2 build):

  -- Per-user, per-weekday template row.
  -- Row exists only once the user has ever selected a radio
  -- for any slot of that weekday.
  CREATE TABLE user_weekday_overrides (
    user_id      TEXT NOT NULL REFERENCES users(user_id),
    weekday      INTEGER NOT NULL,    -- ISO weekday 1-7
    task_labels  TEXT NOT NULL,       -- JSON array of 4 entries:
                                      -- string (authored text,
                                      -- persists while dormant)
                                      -- or null (never authored)
    active_slots INTEGER NOT NULL DEFAULT 0,
                                      -- bitmask, bit N = slot N
                                      -- radio is ON (N = 0-3)
    PRIMARY KEY (user_id, weekday)
  );

  -- On users: week_mode_weekdays INTEGER — RETIRED-UNUSED.
  -- Model B has no weekday-level toggle; the per-slot radios
  -- are the whole state. Column remains in the shipped schema
  -- (default 0, harmless, never read or written). Drop at the
  -- next deliberate schema tidy, not before.

MIGRATION NOTE: the shipped session-12 table (task_labels only,
four strings) has no data anywhere — the feature is unbuilt.
Drop-and-recreate in dev and prod at 4.D2 time is safe and is
the plan. CAUTION: schema.sql is IF NOT EXISTS throughout, so
editing the CREATE alone will NOT reshape an existing table —
the drop must be explicit (and `--remote`, per the wrangler
gotcha).

Notes:
  - Keyed on `user_id` — personal data attaches to the person,
    not the pair.
  - `task_labels` + `active_slots` as two simple columns rather
    than JSON objects-in-array: the bitmask matches the house
    precedent and keeps the JSON shape flat. Render logic:
    slot N active AND labels[N] non-null → override; else
    standard four. (A set bit with a null label is an invalid
    state the write endpoint must reject — selecting a radio
    always stores text, per the decision detail above.)
  - Dormancy = bit clear, text retained. Deleting text only
    happens if the user clears the field while the radio is on
    and the write stores the cleared... no — empty-string
    overrides are rejected at the write endpoint (a task needs
    a name); the user empties a slot by deselecting its radio.
  - Self-write only. Partners cannot read or write each other's
    week mode state. Write rules treat these as private user
    fields.
  - Neither column participates in pair-key hashing — freely
    changeable without rotation (see pair-key doc reference).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EDGE CASES AND OPEN QUESTIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

These are flagged for revisit at implementation time, NOT now.

  - WHAT HAPPENS WHEN THE STANDARD FOUR HAVE FEWER THAN FOUR
    NAMES? Not currently possible (four always set). If a
    future feature ever permits fewer, the template logic
    needs to align. Probably stays four (the app is called
    Four Tasks); flag for confirmation then.

  - WHAT IF THE USER EDITS THE STANDARD FOUR WHILE A TEMPLATED
    DAY IS DISPLAYED? Banner fires, user confirms, standard
    four updates. Today's ACTIVE-override slots remain
    unchanged; today's non-overridden slots update live (if
    unticked). Correct but potentially surprising — clear
    banner copy is the mitigation.

  - WHAT IF AN OVERRIDE'S TEXT EQUALS THE STANDARD LABEL?
    Functionally identical render, structurally still a frozen
    override (won't track future standard-four edits). Fine;
    the popup's active-field rendering shows the slot is
    overridden. No auto-cleanup — deleting the row/bit behind
    the user's back would surprise them on next popup open.

  - DAYLIGHT SAVING / TIMEZONE CHANGES NEAR MIDNIGHT? The
    weekday-of-a-date calculation is pure calendar arithmetic
    on the day's ISO date (per the timezone+sealing doc).
    Templates are user-private, so no partner-timezone
    interaction.

  - PARTNER ANALYTICS LEAKAGE? Partner sees their partner's
    tasks each day; identical Tuesdays let them infer a
    template. Fine — the system doesn't promise secrecy of
    templates, just that they're not synchronised.

  - REST DAY ON A TEMPLATED WEEKDAY? Rest days override the
    daily task tray entirely. A rest-day Tuesday doesn't show
    the Tuesday template; the template is preserved and
    returns next Tuesday. The heading long-press remains live
    on a rest day's tray (it edits the template, not the day).

  - POPUP ON A SEALED DAY WHOSE LABELS PREDATE THE TEMPLATE?
    The popup always shows the TEMPLATE's current state, not
    the sealed day's frozen labels — the sealed tray below it
    keeps showing history. Two surfaces, two truths, both
    correct. Worth one line of help copy if testing shows
    confusion.

  - MULTIPLE USERS ON SHARED DEVICE? Not a current scenario.
    Per-user state switches with the active user if that ever
    ships. Not a v1.0 concern.

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
  is their own. Partners don't see or sync templates, don't
  even know whether their partner is using the feature.

  NOT default-on for anyone. No override exists for any user
  until deliberately authored. Has to be discovered or
  revealed.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED DOCS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  - four_tasks_staggered_disclosure_design_notes.md
    Day-2 long-press philosophy reveal cascades into the week
    mode popup per this doc's DAY-2 CASCADE FLOW section.
    SYNC OWED: cascade destination changed from toggle+tutorial
    to popup+intro-line (2026-06-11). Help menu obligation:
    reference copy for week mode lives in tile 4.11.

  - four_tasks_timezone_and_sealing_design_notes.md
    Weekday-of-date calculation; sealed past days are immutable,
    which constrains template propagation rules (forward-only).

  - four_tasks_system_map.md
    Live schema reference. SYNC OWED at 4.D2 build: the
    override-table reshape (per-slot active_slots bitmask) and
    week_mode_weekdays retirement (§4.2, §4.3). Self-write only.

  - four_tasks_pair_key_design_notes.md
    Neither week-mode column participates in pair-key hashing.
    Private user state, freely changeable without rotation.

  - four_tasks_architectural_preference.md
    Clarity over cleverness shaped two calls here: store-on-
    radio-select (vs store-only-if-different — cleverness), and
    two flat columns (text array + bitmask) over nested JSON
    objects.

  - four_tasks_tracking_design_notes.md
    Not directly related, but the same conversation produced
    both. Week mode is a counter-example to scope creep — a
    feature request was reshaped into something that fits the
    app's positioning rather than rejected outright OR added
    wholesale.

  - four_tasks_godot_devlog.txt (+ v1/v2 archives)
    Session 8 captures the discovery conversation. Session 12
    adds the day-2 cascade. Session 29 reopens the interaction
    fork; 2026-06-11 resolves it to Model B.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CHANGE HISTORY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

2026-06-11 — FORK RESOLVED → MODEL B (popup editor):
- Model A (two-gesture: heading toggle + label long-press)
  retired; epitaph recorded in the fork section. Key reasons:
  the original popup rejection targeted confirmations not
  editors; A's today-only rule was a surface artefact that
  punished the FIFO use case; B's per-slot radios disambiguate
  slot tracking; B is self-explaining where A was invisible.
- Per-weekday toggle removed; users.week_mode_weekdays
  RETIRED-UNUSED (column kept, default 0).
- user_weekday_overrides reshaped: per-slot authored text
  (nullable, persists dormant) + active_slots bitmask. Apply
  via explicit drop-and-recreate at 4.D2 (no data exists;
  IF NOT EXISTS won't reshape on its own).
- Pause-not-delete semantics moved per-slot (radio off =
  dormant, text preserved).
- Decision: override stored on radio selection regardless of
  edit; placeholder always shows the slot's current effective
  label; pre-fill restores dormant text.
- Day-2 cascade reshaped: dismissal opens the popup (creates
  zero state); one-time intro line inside the popup; "no
  back-out" / random-toggle artefacts deleted. One-line sync
  owed to the staggered disclosure doc.
- Today-only edit rule deleted; template entry is any open
  day's tray heading (sealed past days included — the popup
  edits the template, never the day).
- Task-label gesture + its affordance indicator retired.
- Cal-icon banner retargeted (points at heading long-press);
  precision note added for partially-templated days.

SESSION 12:
- DAY-2 CASCADE FLOW added (original toggle-shaped version).
- Reveal timing locked at day 2.
- Cal-icon warning clarified as inline banner.
- Today-only edit rule implementation note clarified.
- Mid-day toggle behaviour documented.
- Schema corrected to the v1.0 lock (bitmask + JSON-array
  override table, user_id-keyed).
