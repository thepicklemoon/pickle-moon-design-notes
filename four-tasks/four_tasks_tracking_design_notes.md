# Four Tasks — Tracking & Stats Design Discussion

Status: DEFERRED — NOT COMMITTED. Captured for future-Morgan.

This document is a discussion-as-document. It exists to record a
design conversation about per-user stat tracking (lifetime coins,
longest streak, total task counts, source-tagged coin earnings,
etc.) so that the thinking is preserved when future-Morgan or a
future Claude session returns to it. Nothing in this doc is
locked. Most of what's in it is explicitly recommended against
implementing.

Created: session 8 (15 May 2026).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHY THIS DOC EXISTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Morgan raised the question of tracking a broad set of per-user
counters — lifetime coins, longest streak, total tasks completed,
1/2/3/4-task day counts, coins earned from partner bonus, coins
earned from promotions, coins earned from subscription multiplier,
task rename count, and so on.

Cost-wise it's nothing. Storage and write performance are not
constraints at any reasonable scale (math in "Cost analysis"
below).

The honest question turned out NOT to be "can the database handle
it" but "does this kind of tracking fit Four Tasks." After
discussion, Morgan's instinct was that it cuts against the grain
of the experience the app is positioning toward. This doc captures
both sides of that discussion plus the small subset that IS
already committed via other locked design docs.

If implemented now, the schema additions are trivial and forward-
compatible. If never implemented, the absence costs nothing
beyond the inability to retroactively populate later. The decision
is therefore reversible in one direction but not the other —
which is the actual interesting property.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
COST ANALYSIS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The "can the database handle this at 1M users" question, settled:

STORAGE:
  Each counter is one INTEGER column on `users`. SQLite/D1
  stores integers in 1-8 bytes depending on value size. Eleven
  counters per user = ~80 bytes worst case per user. One
  million users × 80 bytes = 80MB. D1 free tier is 5GB, paid
  is 10GB+. Not even a rounding error.

READ COST:
  Authenticated requests already SELECT the user's row. Adding
  ten columns to a row that's already being SELECTed is free —
  not a new query, just more bytes in a query that already
  exists.

WRITE COST:
  Each tracked event INCREMENTs the relevant counter inside
  the existing transaction for whatever triggered it. UPDATE
  users SET lifetime_coins = lifetime_coins + N WHERE user_id
  = ?. Single-row UPDATE, sub-millisecond. Write rate is
  approximately 6 writes per user per day. At 1M users that's
  ~70 writes/sec average, peaking maybe 500-1000/sec at user-
  cluster timezone boundaries. D1 handles that without
  breathing hard.

DERIVATION vs RUNNING COUNTER:
  Some counters are derivable from existing tables. Total
  tasks completed = SUM across all `days` rows. 1/2/3/4-task
  day counts = COUNT WHERE task_count = N. For derivable
  counters there's a choice:
    (a) Store the running counter — fast read, must maintain
        increment sites carefully.
    (b) Compute on demand — no maintenance, slower query
        when accessed.
  For trackers shown rarely (a yearly wrap-up surface, an
  achievements eligibility check) compute-on-demand is fine
  and avoids the maintenance burden. For trackers shown on
  high-traffic surfaces (home screen, leaderboard) the
  running counter is worth it.

CAPACITY IS NOT THE CONSTRAINT.
  This is established. The rest of the doc treats capacity as
  a non-issue and focuses on design + UX implications.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE CANDIDATE COUNTER LIST
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Brainstormed during the conversation. Each annotated with current
status, derivability, and Goodhart vulnerability.

LIFETIME_COINS
  COMMITTED: yes — leaderboard doc locked this as the primary
  sort key.
  Derivable: NO (running tally only — you cannot reconstruct
  total earnings from past `days` rows because per-day payout
  varies and intermediate spends shrink the visible total).
  Goodhart vulnerability: LOW. Coins are the in-app currency.
  Growing them by playing is the system working. The leaderboard
  surface is the locked use case.

LONGEST_STREAK
  COMMITTED: yes — leaderboard doc locked this as secondary
  display number.
  Derivable: technically yes, by scanning all `days` and
  computing run lengths, but every leaderboard view would
  trigger that scan for every user displayed. Running counter
  is the right call.
  Goodhart vulnerability: MEDIUM-HIGH. The Duolingo case study —
  users will play through illness, dilute task ambition, mark
  fake completions to preserve streaks. Buddy-ware positioning
  mitigates this somewhat (your partner sees you're playing
  rest days, social check on degeneracy) but the risk is real.
  Worth surfacing carefully — perhaps not as a literal counter
  on the home screen, perhaps only as a secondary leaderboard
  number where competition is the explicit framing.

TOTAL_TASKS_COMPLETED
  COMMITTED: no.
  Derivable: yes, from `days` (SUM of task_count across all
  rows for the user).
  Goodhart vulnerability: LOW. The number can only go up by
  showing up. Hard to game without literally doing the thing.
  CANDIDATE FOR INCLUSION if any future "your journey" surface
  ships. Compute-on-demand is fine — wrap-up surfaces aren't
  high-frequency.

DAYS_1_TASK / DAYS_2_TASKS / DAYS_3_TASKS / DAYS_4_TASKS
  COMMITTED: no.
  Derivable: yes, from `days` (COUNT WHERE task_count = N).
  Goodhart vulnerability: MEDIUM. Visibility of the
  "perfect day" count (DAYS_4_TASKS) re-creates the streak-
  preservation pathology in a different shape — users would
  inflate task completion to keep the perfect-day count
  growing. The 1/2/3-task day counts are mostly neutral; the
  4-task count is the dangerous one.
  Not worth storing as columns. If ever needed, compute on
  demand from `days`.

COINS_FROM_PARTNER_BONUS
  COMMITTED: no.
  Derivable: NO (no per-payout audit trail in current schema).
  Goodhart vulnerability: HIGH and BAD. Surfacing this to the
  user makes them aware of how much of their economy depends
  on their partner. Resentment vector when the partner is
  inactive; smugness vector when the partner is active. Neither
  is the relationship dynamic Four Tasks is trying to encourage.
  STRONGLY DO NOT SURFACE. If tracked at all, track silently
  for internal analytics only — and even then, ask whether the
  analytics question being answered justifies the data
  collection.

COINS_FROM_PROMOTIONS
  COMMITTED: no.
  Derivable: NO.
  Goodhart vulnerability: HIGH and BAD in a different direction.
  Surfacing makes the user feel their rewards aren't "real" —
  the in-game currency develops a tier system in the user's
  head ("earned coins" vs "freebie coins") that the app's
  design specifically avoids. The currency is fungible and that
  fungibility is part of the kindness.
  STRONGLY DO NOT SURFACE. If ever wanted for backend purposes
  (audit trail of promo code redemptions), that's a separate
  table (`coin_grants` or similar), not a column on users.

COINS_FROM_SUBSCRIPTION_MULTIPLIER
  COMMITTED: no.
  Derivable: NO (no audit trail in current schema).
  Goodhart vulnerability: MEDIUM. Less corrosive than the above
  two — subscribers know they're subscribed, the multiplier
  is their reward, surfacing the number is just transparency
  about a benefit they paid for. Could appear in a "subscription
  benefits summary" surface as honest accounting.
  Probably also not worth a column. If a benefits-summary
  surface is ever built, compute the value as a function of
  subscribed-days × average payout × multiplier rate, or
  derive from a coin_grants ledger table.

TASK_RENAMES
  COMMITTED: no.
  Derivable: NO (no rename audit in current schema).
  Goodhart vulnerability: LOW (no one's competing on rename
  count).
  Value of tracking: LOW. There's no obvious use case. Maybe
  a future analytic question — "do users who rename tasks
  retain better than users who don't" — but that question can
  be answered by a one-off log mining if event logs are kept,
  rather than a permanent column.
  DO NOT IMPLEMENT.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE GOODHART AUDIT — METHOD
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Goodhart's Law: when a measure becomes a target, it ceases to be
a good measure. For habit-tracking specifically: the moment a
proxy metric is shown to the user as a number to grow, the user
optimises for the proxy rather than the underlying habit.

Duolingo is the textbook case study. Streak preservation became
its own behaviour, decoupled from language learning. Users set
11:58pm alarms, completed token lessons, paid for streak freezes.
The streak became the target; whether language was being learned
became secondary.

The audit method for any candidate tracker:

  1. Identify the underlying behaviour the app wants to
     encourage.
  2. Identify the proxy the tracker measures.
  3. Ask: "If a user could see this number and was incentivised
     to grow it, would the resulting behaviour be the thing I
     actually want, or a degenerate version of it?"
  4. If degenerate: either don't surface the tracker, or surface
     it in a frame that mitigates the degeneracy.

For Four Tasks, the underlying behaviour is "build a habit that
improves the user's life." The proxies — task completion, streak
length, day counts — can all be gamed by lowering task ambition,
faking completion, or playing through circumstances where playing
isn't healthy.

Four Tasks has SOME structural protection against the worst
Goodhart pathologies:
  - No streak freezes for sale. Removes the predatory monetisation
    of streak anxiety.
  - Rest days as first-class state. The system normalises not
    playing in a way Duolingo doesn't.
  - Partner visibility. Your partner sees what you mark complete,
    which is a soft social check on faking.
  - Immutable past. Past days can't be retroactively fixed up,
    which removes the incentive to "correct" a missed day later.

But surfacing many tracked numbers undermines these protections.
The more dashboard-y the UX, the more the user starts playing for
the dashboard.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE POSITIONING ARGUMENT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

This is the argument that ended the discussion in favour of
deferring tracking implementation.

Four Tasks' positioning, as expressed across the locked design
docs:
  - Buddy-ware, not solo grind.
  - Partnered, not competitive.
  - Non-predatory, not retention-engineered.
  - Immutable past, not gamification with retroactive fixes.
  - No streak protection sold as IAP.
  - No shame on missed days.

The current surface area is small. The home screen is four tasks,
a stamp, a sticker, a partner. That smallness is the brand.

Adding a stats screen with eleven counters dilutes that. It
puts Four Tasks into the same visual category as MyFitnessPal,
Habitica, every dashboard-heavy habit app — which is the category
the positioning is specifically trying to escape.

The two trackers that DO have committed UX (lifetime_coins,
longest_streak) earn their place by being on the leaderboard,
which is itself an opt-in surface (long-press the coin display).
The user has to seek it out. The default home experience stays
small.

Adding more trackers would either:
  (a) Live in the schema but have no UX home, in which case
      they're inert engineering overhead.
  (b) Get surfaced somewhere, in which case they expand the
      app's visual surface and dilute the positioning.

Neither is a good outcome.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE REVERSIBILITY ARGUMENT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Worth examining honestly: the decision to NOT track is partially
reversible in one direction, fully irreversible in the other.

If counters are added now and never surfaced:
  - Low cost (storage, write transaction overhead).
  - Future-Morgan can decide to surface them, OR can delete the
    columns, OR can leave them dormant indefinitely.
  - Fully reversible.

If counters are NOT added now:
  - Zero cost.
  - Future-Morgan can add them at any time but only collects from
    that point forward. Pre-existing user history is lost.
  - Partially reversible (forward-only).

The asymmetry argues weakly in favour of storing the counters
silently as a hedge. Not surfacing them, not letting them shape
UX, just having them sit there for if-and-when a future feature
wants them.

The counter-argument is the discipline one: a column in the
schema is a temptation. Future-Morgan or a future contributor
sees `coins_from_partner_bonus` sitting there and thinks "well,
the data exists, might as well surface it..." The discipline of
not-storing is also the discipline of not-surfacing.

The doc declines to pick a winner on this. Morgan's instinct
during the conversation was to keep the app small, which biases
toward not-storing. That instinct is the more important signal
than the schema math. Trust it.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHAT IS ACTUALLY COMMITTED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The intersection of "tracked counters" and "locked design" is
small. Locked across other design docs:

LIFETIME_COINS (column on users):
  Surfaced by: leaderboard (primary sort key).
  Increment site: claim_morning endpoint, inside the existing
  transaction that pays out morning coins. Also incremented by
  any other coin-earning event (promo redemption, future
  achievement unlocks if those ship).
  Decrement site: NONE. Lifetime is monotonic — spending coins
  does not decrement lifetime_coins. The user's current spendable
  coin balance is a separate column.
  Migration: lands in whichever migration introduces the
  leaderboard support (TBD — not part of migration_003-005
  bundle currently sketched for tile 1.3).

LONGEST_STREAK (column on users):
  Surfaced by: leaderboard (secondary display number).
  Increment site: claim_morning endpoint, computed from the
  current streak state. If today's claim continues a streak and
  the resulting streak length exceeds longest_streak, update
  longest_streak.
  Migration: same as lifetime_coins.

Everything else discussed in this doc is NOT committed and
should be treated as inert design conversation until and unless
future-Morgan revisits.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHEN TO RE-VISIT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Plausible triggers to re-open this conversation:

  - A "year in review" / wrap-up feature gets proposed (Spotify-
    Wrapped-style annual summary). Total tasks completed and
    day-distribution counts become directly useful then. Adding
    columns at that point means losing pre-launch history.
    Whether that's acceptable depends on how the wrap-up is
    framed — "your year with Four Tasks" can launch year 1 and
    grow richer year 2 onward, no pre-history needed.

  - An achievements framework actually ships (currently brainstorm
    only — four_tasks_achievements_brainstorm.md). Some
    achievements may need running counters for atomic adjudication.
    Re-visit the candidate counter list at that point, scoped to
    what achievements actually need.

  - User research surfaces a specific surface where a counter
    would land well. E.g. user testing reveals people want to
    know "how many partner-supported coins did I earn this
    month" in a relationship-positive frame. Re-visit then with
    that specific surface in mind.

  - Backend analytics need (very different from user-facing
    trackers). E.g. determining churn correlation with specific
    behaviours. Probably better handled by event logging than
    column-on-users counters anyway.

In none of these triggers does the schema cost matter. The
question every time is "does surfacing this counter serve the
user." Default: no.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EVENT LOGGING — A DIFFERENT QUESTION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Briefly flagged because it kept getting confused with tracking
during the discussion:

A `coin_grants` (or similar) ledger table is a different concept
from per-user counter columns. The ledger records EVERY grant
event with user, amount, source (claim_morning / promo /
subscription_multiplier / achievement / etc.), timestamp. Reading
the ledger answers retrospective questions ("how much has this
user earned from promos?") without needing dedicated counter
columns.

The ledger approach has tradeoffs:
  Pro: source-tagged history is queryable forever without
       committing to a fixed counter set.
  Pro: doesn't pollute the users table with surface-bound
       columns.
  Pro: source of truth for audit / dispute / refund cases.
  Con: storage scales with grant events, not users (probably
       fine — 6 grants/day/user × 1M users × 365 days × ~30
       bytes/row = ~65GB/year. Manageable with partitioning
       but a real number, vs the trivial counter cost).
  Con: every read of "earned from X" is a SUM aggregation
       rather than a single column read.

The ledger approach is the right answer IF the analytics need
ever materialises. It's NOT the right answer for surface-bound
trackers like "show me my streak on the home screen" — those
want denormalised columns for read speed.

Not committed. Captured as the parallel design path that exists
alongside the column-tracking option, in case future-Morgan
reaches for one when actually the other is the better fit.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED DOCS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  - four_tasks_leaderboard_design_notes.md
    Locked use case for lifetime_coins and longest_streak.
    Defines the only currently-committed surfacing of any
    tracked counter.

  - four_tasks_achievements_brainstorm.md
    Speculative. If the achievement framework ever ships, some
    achievements may need running counters for atomic
    adjudication. Re-visit this doc's candidate list at that
    point.

  - four_tasks_monetisation_position.md
    The subscription multiplier and promo redemption are the
    source events for the source-tagged coin earnings discussion.
    If a "what subscription got me" surface ever ships, it
    pulls from here.

  - four_tasks_staggered_disclosure_design_notes.md
    Any tracker that gets surfaced needs a disclosure schedule —
    when does the user first see this number, in what frame.
    Re-visit alongside this doc if any tracker is approved for
    surfacing.

  - four_tasks_architectural_preference.md
    Clarity over cleverness — informed the discussion's bias
    toward NOT adding columns that have no clear use case. A
    column with no surface is engineering debt with no
    corresponding feature.

  - four_tasks_godot_devlog.txt
    Session 8 captures the discussion that produced this doc.
