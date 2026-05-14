# Four Tasks — Morning Sequence Design Notes

Status: LOCKED for v1.0 implementation (session 5, terminology refreshed session 7)
Implementation tile: 4.6 (full coordinator)
Related tiles: 4.4 (slot machine), 4.5 (MOTD reroll cost),
4.7 (stamp slap animation), 4.8 (rest day system), 4.16 (partner
reactions surfacing), the claim_morning endpoint (lands with tile 1.3).

The morning sequence is the centrepiece of the daily experience.
Four sessions of design conversation circled it without ever
fully documenting it in one place. This doc fixes that. Eight
discrete design questions (Q1-Q8 in the session-5 conversation)
landed answers here; each is captured under its own subheading.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHAT THE MORNING SEQUENCE IS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

A ritual that fires once per day, on the user's first app-open
of that day, when there is a pending morning payout to claim. It
is not a *mode* the app is in; it is an *overlay-on-top* of the
normal calendar view. The base state is always "calendar visible,
no trays open, full user control." The morning sequence is what
happens *to* the user when they open the app at the right moment.

User loses control deliberately for the duration of the sequence
(~15-30 seconds). The app performs a small ceremony that:

  1. Pays out yesterday's coin earnings, visually.
  2. Stamps yesterday with a tier-coloured judgement and message.
  3. Announces today's MOTD by spinning it up and typewriter-
     glitching it into its permanent header position.

Then control returns and the day begins.

The user-locked-out part is structurally load-bearing. Most
productivity apps maximise user agency at every moment. Four
Tasks does the opposite *at this specific moment* because the
moment is about acknowledgement, not action. The user is being
*given* a moment, not driving one.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE BEAT-BY-BEAT (Q1)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Reference: the prototype implements approximately this sequence.
The Godot port replicates it with cleaner state machine, atomic
server claim, and one fix (sticker pool randomisation — see Q1
notes on alternating bug below).

The beats, in order:

  1. User opens app. Calendar visible. No trays open. (Q3 lock —
     calendar always opens flat, sequence is overlay-on-top.)

  2. App checks `users.morning_payout_due_for`. If NULL, no
     sequence — user retains control immediately. If set to a
     date, sequence begins.

  3. The specified yesterday's day cell is tapped automatically.
     Task tray slides out for that day, using the normal tray
     animation timing.

  4. Once the tray reaches its usual position, it unlocks from
     normal docking and wobbles briefly. Wobble draws the user's
     focus.

  5. Tray flies up, detached from the calendar, to the centre of
     the screen. Brief pause.

  6. Each completed task jiggles in sequence. Coins fly out
     visually, blitz up onto the coin counter in the UI. Each
     checkbox is green-ticked. Tasks that were *not* completed
     get a strike-through and a red cross in the unfilled
     checkbox. Rest-day variant differs — see Q6 below.

  7. Once all four tasks have been treated, a stamp comes down
     and seals the day. Stamp colour matches the tier reached:
       - 1 task done  → red stamp, message from "woops" pool
       - 2 tasks done → orange stamp, message from "solid effort" pool
       - 3 tasks done → yellow stamp, message from "not bad" pool
       - 4 tasks done → green stamp, message from "hell yeah" pool
       - Rest day     → purple stamp, message from rest pool
     Message text typewriter-glitches into the task tray as part
     of the stamping moment.

  8. The tray slides back to its docked position with the new
     stamp + message visible.

  9. The MOTD picker appears (centred on the screen). It auto-
     spins a new MOTD. Settles after the spin. Brief pause.

  10. The day's MOTD typewriter-glitches into its header position
      (centred above the streak bar, see screenshot reference in
      session 5 conversation — the "eerily ignored the mind 👽"
      slot). This is the day's *fingerprint* — visible all day,
      until tomorrow's sequence replaces it.

  11. Yesterday's task tray slides down. Today's task tray slides
      out, signalling the day has begun.

  12. User regains control.

Total duration: approximately 15-30 seconds, target. Specific
timing values to tune at implementation time against feel. Must
be long enough to feel ceremonial but short enough not to annoy.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE MOTD STRUCTURE (Q1)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The MOTD is *four spinners*, not three:

  adverb + verb + noun + emoji

Example from the prototype screenshot: "eerily" + "ignored" +
"the mind" + "👽".

The emoji slot reads from the *same pool* as the user's day-cell
completion stickers — `users.active_stickers` (the pool the user
manages via tap-to-toggle in the picker, per theme design notes).
The user's sticker selections in the picker affect both the
visual identity of the MOTD and what stickers slap onto completed
days. Single source of truth.

This means: change your active pool, and your MOTDs change emoji
character along with your completed-day stickers. They move
together.

MOTD FONT: locked global per theme design notes session 7
rewrite. The MOTD typography is the same across all themes — only
the emoji slot changes per pack. Brand-voice consistency over
per-theme expressive typography.

PROTOTYPE BUG TO FIX IN PORT:
  The prototype's spinners *alternate* visibly when spun (visible
  in screenshot: alien emoji and pixel-art space invader sticker
  alternating across the completed days). The intended behaviour
  for the Godot port is *true randomisation* — each slot rolls
  independently with no alternation pattern. Fix at tile 4.4
  implementation (the slot machine).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE CRASH-RESISTANCE MODEL (Q2)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The morning sequence is *purely cosmetic*. The server's coin
balance is updated as a number (not derived from the animation
playing). The animation is the user's *experience* of the payout,
not the payout itself.

This makes the sequence robustly crash-resistant by design:

  - If the app crashes BEFORE animation begins:
    Next launch sees `morning_payout_due_for = <date>`, replays
    the full sequence. No data loss.

  - If the app crashes DURING the animation:
    Server has not yet been told the user "watched" the sequence
    (the claim endpoint hasn't fired). Server flag is still set.
    Next launch replays the full sequence from the start. User
    sees their coins already accounted for in the counter (because
    the server's coin balance was updated by the seal-on-open
    transaction, not by the claim endpoint visually). MOTD will
    be a new random one. Cosmetic-only loss is "I didn't see the
    moment happen." Acceptable degradation.

  - If the app crashes AFTER the claim endpoint fires but before
    the sequence visually completes:
    Server flag is cleared. Next launch — no sequence. Calendar
    just shows yesterday already stamped, today already has its
    MOTD. User skips the ceremony. This is the least-good case
    but still data-correct.

The "infinite coins" failure mode (user crashes at the right
moment, gets paid multiple times) is structurally impossible
because the claim endpoint is atomic and idempotent on the
server side — see CLAIM ENDPOINT ARCHITECTURE below.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE BASE STATE IS ALWAYS CALENDAR-FLAT (Q3)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

When the user opens the app, the FIRST frame they see is the
calendar in its normal state. No trays open. No animations
already in progress. Standard interactive UI.

THEN the morning sequence fires (if a payout is pending). It is
an overlay-on-top, layered on the calendar base state. When the
sequence ends, the user is back at the same base state, plus
yesterday now has its stamp and today has its MOTD.

This is structurally cleaner than the alternative "app boots
into morning-sequence mode if payout pending, else boots into
normal mode." Same code path always; the sequence is just an
interaction layered on the calendar.

Implications:
  - Loader scene (tile 2.1) handles identity check and routes to
    App scene. Loader does NOT check for pending payout.
  - App scene boots, calendar paints, then a separate _ready
    check asks "is there a pending morning payout?" If yes, fire
    the sequence. If no, do nothing.
  - Mid-sequence app backgrounding: when foregrounded, sequence
    state machine decides whether to resume, restart, or skip
    (per Q2 model).
  - Late-in-the-day opens: no sequence (already claimed earlier).
    Standard calendar view.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE PARTNER PANEL DURING THE SEQUENCE (Q4)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Your morning sequence is private. Partner does NOT see it.

Your sequence plays on YOUR panel. If you swipe to the partner
panel mid-sequence... actually the user is locked out during the
sequence so swipe is disabled. The partner panel is invisible
to you during your own ceremony.

More importantly: the partner panel is also "dead" with respect
to your data until you've claimed.

This means: if partner opens their app at 6am, plays their
sequence, claims their payout — and you open YOUR app at 7am —
on YOUR partner panel you immediately see *their* sealed
yesterday with stamp visible (because their claim endpoint has
fired). They have seen their own ceremony. Server-side, their
sealed state is visible.

Conversely: until YOU claim, your partner sees your previous
day as unstamped on their partner panel. They learn you've
claimed via polling (tile 2.8, 30s cadence). Within 30s of your
claim, their partner panel updates live.

THE LOAD-BEARING INSIGHT:
  The morning sequence's stamp/MOTD/payout is only visible to
  the partner *after the user claims*. The server flag
  `morning_payout_due_for` gates this. Until the claim endpoint
  fires:
    - days.stamp is NULL for that date (or unset to partner's view)
    - days.diary still shows yesterday's MOTD (no change yet)
  After the claim:
    - days.stamp is populated with tier colour
    - days.diary may have been updated with the new MOTD's text
    - partner's polling will pick this up

CRITICAL BUG TO AVOID:
  The prototype had a "prestamped on the proto" bug where days
  appeared stamped to the partner before the user had claimed.
  Morgan correctly identified and fixed this — partners should
  NOT see your stamp before you've experienced your own ceremony.
  The Godot port must enforce this through the claim endpoint
  being the moment of state-becomes-visible.

WHY THIS IS GOOD DESIGN (not just a limitation):
  The "dead until they've claimed" property makes the connection
  feel alive. State moves when the partner moves it, not when
  the server seals it. The morning ceremony is a private moment
  in a shared system. Seeing your partner's stamp appear live on
  your partner panel because they just opened their app is the
  textural moment of connection — like watching a "delivered →
  read" indicator change on iMessage. Small but real.

POLLING IMPLICATION:
  Tile 2.8 (partner panel polling, 30s cadence) needs to refresh
  day-cell state, not just metadata. The morning-reveal pattern
  depends on day-cell state being something polling catches.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE MULTI-DAY WALKBACK (Q5)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

"What if the user is away for a week and opens the app?"

Answer: there is no backlog of payouts.

The data model enforces this for free: a payout can only be
claimed for a day where tasks were ticked. The app only allows
task-ticking on a day the user has the app open. If the user
hasn't opened the app for a week, the days between are EMPTY
days — never had tasks ticked, never earned a payout.

So "user away for a week" produces at most ONE pending payout:
the most recent day they actually ticked anything on. Everything
since then is empty and gets no ceremony, no stamp, no message.

THE WALKBACK ALGORITHM:
  On app open, the seal-on-open transaction walks back from
  the user's current local date looking for the most recent
  date where:
    - days.tasks_done > 0 (or days.rest_day = 1), AND
    - days.stamp IS NULL (not yet ceremoniously sealed)
  That's the date to set as morning_payout_due_for. The morning
  sequence plays for that date.
  If no such date exists (e.g. fresh install, or all prior days
  fully processed), no sequence fires.

STREAK CONSEQUENCES:
  Days between the last-ticked day and today are empty. Streak
  breaks during the absence (no four-of-four days). Rest days
  cover the gap only if user pre-placed them. Otherwise streak
  resets. Morning sequence on return is a ceremony for the
  *last actually-played day*; user is dropped into today with
  streak in whatever state the gap left it. No special "you
  broke your streak" announcement during the sequence — that
  fact reveals itself naturally as the user notices their
  counter has dropped.

PUSH NOTIFICATIONS:
  Daily nudge to tick tasks would partially solve the absence
  problem. Push remains DEFERRED to v1.x. v1.0 ships with "no
  open, no play" as a feature, not a bug.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE REST DAY VARIANT (Q6)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Rest days play the SAME morning sequence with content swaps.
Not a separate sequence — same beat structure, same timing,
same lockout. Content slots populate from different pools.

DIFFERENCES VS A WORKED DAY:

  Beat 6 (tasks treated):
    - Each task jiggles. Instead of green-ticking the user's
      original task labels, the labels typewriter-glitch into
      rest-vocabulary phrases ("relaxed", "had a cheat meal",
      "unworried", "called your mum", etc.).
    - Checkboxes get purple ticks (or hearts, or hyphen marks —
      art decision for tile 4.7) instead of green/red.
    - No coins fly out. Rest cost was paid in advance when the
      user long-pressed to designate the rest day (tile 4.8).

  Beat 7 (stamp lands):
    - Stamp is PURPLE (sacred semantic — the rest-day colour, not
      themed against active palette source).
    - Message comes from the "rest" pool, separate from the four
      tier pools. Tone is affirmation of the choice to rest, not
      celebration of effort. ("good on you for the rest",
      "earned", "well-deserved", "soft day, well chosen".)

REST-DAY-SPECIFIC CONTENT POOLS NEEDED:
  - Rest-day task labels: ~15-30 phrases, randomly drawn four
    per rest day. Different pool from EXAMPLE_TASKS (tile 3.5).
    NEW CONTENT to author at tile 4.8 implementation time.
  - Rest-day stamp messages: small pool, same structure as the
    four tier pools but with affirmation tone. NEW CONTENT to
    author at tile 4.7 / 4.6 implementation time.

ARCHITECTURAL IMPLICATION:
  Tile 4.6 implementation uses ONE sequence with parameterised
  content (pools, colours, task-label-source). Rest-day handling
  is a content swap, not a code branch. The state machine
  doesn't know it's rendering a rest day; it just knows the
  content pools to load are different. Cleaner than two
  sequences with parallel maintenance.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PARTNER REACTION SURFACING (Q7)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

This is technically a partner reactions concern (tile 4.16) but
the answer locked in session 5 affects how partner data
surfaces in general — including via the morning sequence's
side effects on partner-visible state.

LOCKED PRINCIPLE:
  Partner reactions are TIME-AGNOSTIC, not event-driven.

A reaction is not a "fresh news" moment that needs immediate
surfacing. It's a backlog feature — partner accumulates sealed
days you might react to; you accumulate sealed days they might
react to; either side can engage with the backlog whenever,
in any quantity, or not at all.

This means:
  - NO push notifications when partner reacts.
  - NO morning sequence beat for "your partner left this for
    you yesterday."
  - NO urgent visual badge on the partner panel demanding
    attention.

INSTEAD: a benign visual indicator on past-day cells, in both
directions.

Direction 1 (your view of partner's panel):
  - Partner's sealed days that you have NOT YET reacted to get
    a small ambient marker. Soft, not urgent. (Specific visual
    style TBD at tile 4.16 / art-tile time — a small lit corner,
    subtle glow, tiny dot in soft colour. NOT a red dot, NOT a
    notification-spam "1" badge.)
  - Marker means "you can do something here if you want."
  - Marker clears when you react (or explicitly skip — though
    skip may simply mean "tap and close picker without
    selecting").
  - The reaction itself (heart, fire) remains visually on the
    day after the marker clears.

Direction 2 (partner's view of your panel — when they've
reacted to one of your days):
  - Your past-day cells that have RECEIVED a reaction get a
    marker drawing your eye to them.
  - Marker clears after you've viewed it (tapped into the tray
    or otherwise acknowledged).
  - The reaction itself stays visually on the day forever.

SYMMETRY:
  Both directions use the same indicator pattern. Same visual
  vocabulary. Neither direction gets ceremonial moment
  treatment. Both get the quiet "there's something to look at
  if you want."

  Asymmetric handling would feel weird — if reactions felt
  urgent when received but casual when available-to-give, the
  rhythm would be off. Symmetry resolves that.

POSSIBLE DUAL-INDICATOR STATE:
  A cell could simultaneously have "you haven't reacted yet"
  AND "partner reacted to this." UI needs to read cleanly when
  both are present. Likely solution: different sides of the
  cell, or different shapes, or one is a border treatment and
  the other a small mark. Detail for tile 4.16 implementation.

DISCOVERY VIA STAGGERED DISCLOSURE:
  Partner reactions tutorial reveal fires at DAY 4 (preferred)
  or DAY 6 (backup). Day 4 because by then user has several
  sealed days accumulated in partner's panel to actually try
  the feature on. Day 5 would crowd the streak milestone moment
  (tile 4.9). Day 6 is backup giving breathing room post-streak.

  This is a SCHEDULED reveal in the staggered disclosure model,
  not purely TRIGGER-GATED. The earlier framing in
  four_tasks_partner_reactions_design_notes.md described it as
  trigger-gated ("first time partner reacts"). The session-5
  refinement: schedule the reveal at day 4 so the user has
  the vocabulary to *notice* the indicator when a reaction
  does arrive, rather than being surprised by an unexplained
  marker.

  Trigger-gated reaffirmation can still fire if partner has
  already reacted before the scheduled reveal lands — but the
  scheduled reveal is the primary teaching moment.

SUPERSEDES:
  four_tasks_partner_reactions_design_notes.md "Surfacing
  reactions to the receiver" section, which described the
  surfacing as push-based. Push is no longer the model.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MOTD REROLL COST — SUPERSEDES DOUBLING (Q8)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

This is the substantive change from the prototype's mechanic.

PROTOTYPE MODEL:
  First reroll free (during morning sequence). Subsequent rerolls
  cost coins, starting at 1 coin and DOUBLING each press. Day-1
  free, Day-2+ doubling per the original tile 4.5 description.

PROBLEM IDENTIFIED IN SESSION 5:
  Doubling is the wrong mechanic for casual aesthetic rerolls.
  In low-stakes incremental games, doubling makes sense because
  the friction is the feature — the player must decide whether
  *this* press is worth it. But Four Tasks's MOTD reroll exists
  to let users express a preference about a piece of text that
  matters only for today. It's not supposed to be a major
  economic event.

  Uncapped doubling means a user not paying close attention
  could blitz through hard-saved coins in ~12 presses (e.g.
  1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096 →
  ~8000 coins burned). That's an entire month of multiplied
  task payouts gone on one annoyed evening of rerolling.

LOCKED MODEL (session 5):
  FLAT RANDOM RANGE per press: 90-110 coins, regardless of how
  many rerolls have happened.

  Approximately 1/10 of an unmultiplied daily task payout
  (assumes ~1000 coins per task at base × 4 tasks = ~4000 per
  day max base). Tune the exact range when the coin economy is
  tuned more broadly.

  The 90-110 range (vs flat 100) creates a tiny moment of
  uncertainty per press — "what's it gonna be this time" — that
  keeps the act of pressing the button slightly textural. Costs
  nothing to implement.

  No escalation. Press 100 times, press costs 100x base. The
  user can't accidentally bankrupt themselves; the cost stays
  predictable.

WHY THIS IS THE RIGHT MECHANIC FOR THIS FEATURE:
  MOTD reroll is a *vent* for the coin economy, not a *sink*.
  The user spends a small amount to express preference about
  something transient (today's MOTD, gone tomorrow). It should
  feel casual and low-stakes.

  Real sinks (where significant coins should accumulate to):
    - Sticker pack purchases (single ramp 300-600 coin per pack
      per monetisation v2.0 session 7 update)
    - Coin name rerolls (post-v1.0 — keeps doubling-cost
      mechanic because identity rerolls should escalate)
    - Rest days (tile 4.8 — pre-paid cost when designated)

  These are the architectural targets the coin economy is
  balanced against. MOTD reroll is just edge-nibble at the
  counter to make expression feel meaningful without making
  it punitive.

DOUBLING-COST STAYS WHERE IT FITS:
  Coin name reroll (post-v1.0) keeps the doubling-cost mechanic.
  Why: coin name is a persistent identity element. Escalating
  cost is the right friction because each reroll changes
  something the user lives with for a while. Forces deliberation,
  which is appropriate.

  General architectural principle that fell out of this
  conversation: **doubling cost belongs on features where the
  user output should ESCALATE in significance; flat cost
  belongs on features that should stay casual.**

OPEN: PRICE SCALING WITH MULTIPLIERS:
  Possibility A — the 90-110 range scales gently with the user's
  current task multiplier. A 2x-streak player pays 180-220 per
  press. Keeps proportional pain consistent across player
  progression.

  Possibility B — the range stays at 90-110 forever. High-streak
  players genuinely feel rerolls as cheap (they've earned the
  casualness); new players feel each press as real.

  Either is defensible. Decide at tile 4.5 implementation when
  the broader coin economy is being tuned.

TILE 4.5 DESCRIPTION CHANGE:
  Old: "MOTD reroll cost system. Day-1 free, Day-2+ doubling.
       Particle, blitz, flash on press. Lock button +
       confirmation popup."
  New: "MOTD reroll cost system. First reroll during morning
       sequence is free (the auto-roll). Subsequent rerolls cost
       a flat random 90-110 coins each, no escalation. Particle,
       blitz, flash on press. Lock button + confirmation popup.
       Reroll mechanic is a coin economy VENT, not a sink — see
       four_tasks_morning_sequence_design_notes.md MOTD section."

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CLAIM ENDPOINT ARCHITECTURE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Specced in four_tasks_write_rules_design_notes.md "Deferred /
parked items" section. This doc consolidates the requirements.

ENDPOINT:
  POST /pair/:key/users/:target/claim_morning
  Empty request body. Server knows what's pending from the
  morning_payout_due_for column.

INTEGRATED WITH SEAL-ON-OPEN:
  Per the timezone doc, sealing of yesterday happens during the
  claim endpoint transaction itself (NOT via a nightly cron).
  The claim endpoint:
    1. Reads user's local_date from the request envelope (sent
       on every write per the timezone doc).
    2. If local_date has advanced past the most recent sealed
       day, performs seal-on-open: marks intervening days as
       sealed, computes streak, sets morning_payout_due_for
       for the latest ticked day if any.
    3. Then proceeds with claim logic against the pending
       payout.
  Tile 1.4 (formerly "nightly cron") is REFRAMED as part of the
  claim endpoint's transaction surface.

ATOMICITY:
  Single D1 transaction. Server:
    1. Seal-on-open prelude (per timezone doc).
    2. Reads users.morning_payout_due_for for the target user.
       If NULL, return 404 (nothing to claim — defensive against
       double-claim races).
    3. Reads days.tasks_done for (pair, target, that date).
    4. Computes stamp tier from tasks_done count (or detects
       rest day from days.rest_day field).
    5. Picks random message from the appropriate tier pool
       (server-held content; not on client).
    6. Writes days.stamp = tier_colour.
    7. Captures days.day_theme_state JSON snapshot from the
       user's active_leader + active_theme + active_stickers
       at this moment. Immutable past — this snapshot drives
       the historical theme renderer per monetisation v2.0.
    8. Writes days.diary if MOTD has been rerolled (the new
       MOTD text from the spinner). Note: this may interact
       with how diary writes are tracked — the MOTD that lands
       at end of morning sequence becomes the *new* day's
       MOTD, not yesterday's. May need a separate "today's
       MOTD" field on users or on days for today's row. To be
       designed during tile 4.6 implementation.
    9. Pays out coins: tasks_done count × per-task coin amount,
       multiplied by current streak multiplier if any. Updates
       users.coins.
   10. Updates users.streak (increment if four-of-four, hold if
       rest day, reset if less than four).
   11. Clears users.morning_payout_due_for (sets to NULL).
   12. Commits transaction.

RESPONSE BODY:
  Returns the full state the client needs to play the sequence
  with real data:
    {
      ok: true,
      data: {
        date: "2026-05-11",
        tasks_done: 4,
        was_rest_day: false,
        stamp_tier: "green",
        stamp_message: "PHWOAR",
        motd_text: "eerily ignored the mind 👽",
        coins_earned: 4000,
        coins_total: 36946,
        streak: 11,
        streak_changed: true
      }
    }

  Client uses this to drive the animation. The sequence is
  cosmetic — the data is the truth.

IDEMPOTENCY:
  Second call after first succeeds: 404 (morning_payout_due_for
  is NULL). No double-payout. Safe.

CLIENT CALL TIMING:
  Open question — when in the sequence does the client fire the
  claim?
    Option A: at sequence start (when user opens app, before
              animations begin). Server response drives the
              animation. Crash mid-animation means no replay
              (server already cleared the flag).
    Option B: after stamp lands (mid-sequence). Animation up to
              that point runs against client-cached
              expectations; from stamp onward uses server data.
              Crash before stamp means replay; crash after stamp
              means skip.
    Option C: at sequence end (right before user regains
              control). Maximum replay-coverage but means the
              UI plays animation BEFORE the server commits.
              Server payout could differ from animation (e.g.
              streak reset detected at server only).

  Option A is the cleanest. Server response is authoritative
  before any animation plays. Animation renders what the server
  says happened, not what the client guesses. The client never
  plays an animation that the server might disagree with.

  Locked recommendation: Option A. Server claim fires at the
  start of beat 3 (just as yesterday's tile is tapped
  automatically). Response data populates the rest of the
  sequence. Decide finally at tile 4.6 implementation.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
INTERRUPTION MODEL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

What happens if the user backgrounds the app mid-sequence?

GIVEN the claim endpoint architecture above (Option A — claim
at sequence start), the answer collapses cleanly:

  - If claim has fired before backgrounding: server flag is
    cleared. When user foregrounds, no sequence replays. They
    see the calendar with yesterday already stamped and today's
    MOTD already in place. They missed the moment but the data
    is correct.

  - If claim has NOT yet fired before backgrounding (extreme
    edge case — they opened the app and immediately
    backgrounded within ~200ms): server flag is still set.
    Foregrounding plays sequence from start.

In practice, the claim fires very early in the sequence (~1
second after open). The "background before claim" window is
small. Most backgroundings happen mid-animation, after claim.

DEGRADATION CASES:
  - Network failure on claim: client retries on next open.
    Server flag still set. Sequence replays. Robust.
  - Server 500 on claim: same — flag still set, replay on next
    open.
  - User backgrounds during the partial animation that runs
    before claim returns: animation queued, finishes on
    foreground OR cancels and replays from start. Implementation
    decision at tile 4.6.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MIGRATION WINDOW PROPERTY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The morning sequence's user-locked-out window has been called
"the natural migration window" in earlier design docs (notably
four_tasks_pair_key_design_notes.md). This doc constrains
that claim with real numbers.

PROPERTY: during the morning sequence, the user has no
interactive controls. The server has predictable uninterruptible
time to perform writes.

BUDGET: roughly 15-30 seconds of uninterruptible animation,
during which any server-side work tied to the user's identity
(pair-key migration in particular) can be performed safely.

WHAT FITS COMFORTABLY:
  - The claim endpoint's atomic transaction (likely <100ms).
  - One pair-key migration completing on the server side
    (current estimate: a few D1 transactions, <1s total).
  - Polling cycle catch-up on the partner side.

WHAT DOESN'T FIT:
  - Bulk data operations (e.g. processing accumulated
    notifications across weeks of absence). Don't try to bundle
    these into the morning sequence window.
  - User-facing migration UX moments. If migration needs to
    explain itself to the user, that's outside the sequence.

HARD UPPER BOUND:
  If any migration or operation cannot complete in the morning
  sequence window, design needs to either (a) split it into
  pre-claim and post-claim halves, or (b) accept the migration
  may need a follow-up sync after control returns.

The point: don't treat "the morning sequence is the migration
window" as a magic bucket. It's a real time budget. Measure and
respect it.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED TILES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Implementation:
  - 4.4 (slot machine) — MOTD spinner mechanism. Fix the
    "alternating bug" from prototype; randomise truly.
  - 4.5 (MOTD reroll cost) — rewrite per Q8 (flat random
    90-110 range, supersedes doubling).
  - 4.6 (morning sequence — full coordinator) — this doc is
    the design reference.
  - 4.7 (stamp slap animation) — the stamp-down beat. Tier
    colours and message pools authored here.
  - 4.8 (rest day system) — sets users up for Q6 variant.
    Rest-day task label pool authored here.
  - 4.9 (streak popup with confetti) — fires AFTER morning
    sequence if a milestone was hit. Not part of the sequence
    itself; layered after.

Data layer:
  - 1.3 (Worker defensive write rules) — already-locked design
    covers the claim endpoint's write rules.
  - 1.4 (lazy seal-on-open, REFRAMED) — sealing happens during
    the claim endpoint transaction, not via a nightly cron.
    See timezone doc for full architecture.
  - Future: claim endpoint implementation tile (post-1.3).

Partner-side:
  - 2.8 (partner panel polling) — picks up the day-cell state
    change after the user claims.
  - 4.16 (partner reactions) — surfacing model from Q7 lives
    in that tile's implementation.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHAT THIS DOC LEAVES OPEN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Decisions deferred to tile 4.6 implementation:

  - Specific timing values for each beat (wobble duration,
    pause durations, total sequence length, etc.).
  - Final stamp message pools (red/orange/yellow/green/purple
    pools, ~10-20 messages each). Content authoring.
  - Rest-day task label pool (15-30 phrases). Content
    authoring.
  - Rest-day stamp message pool. Content authoring.
  - Exact MOTD reroll price range (90-110 is a starting point;
    tune with broader economy).
  - Whether MOTD reroll price scales with task multipliers
    (Q8 Possibility A vs B).
  - Visual style of the partner reaction indicators (Q7) —
    art-tile decision at tile 4.16 / 4.E time.
  - Whether the partner reactions tutorial reveal fires day 4
    or day 6 — tune at tile 4.19 (TutorialCoordinator) content
    authoring.
  - Background-mid-sequence interruption animation handling
    (cancel + restart vs queue + finish).
  - Final shape of how today's new MOTD is stored — diary
    field on today's row, or a separate users-level "current
    MOTD" field. Affects schema slightly. Resolve at tile 4.6
    implementation.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED DOCS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  - four_tasks_pair_key_design_notes.md
    Identity model. The "morning sequence is the natural
    migration window" property is constrained here.

  - four_tasks_write_rules_design_notes.md
    Claim endpoint defensive write rules. Schema deltas for
    migrations 003 + 004 + 005 bundled here.

  - four_tasks_timezone_and_sealing_design_notes.md
    Seal-on-open architecture. The claim endpoint and seal logic
    share the same transaction surface.

  - four_tasks_partner_reactions_design_notes.md
    Q7's surfacing model supersedes "Surfacing reactions to
    the receiver" section in that doc.

  - four_tasks_staggered_disclosure_design_notes.md
    Partner reactions reveal at day 4 lives in that schedule.

  - four_tasks_theme_design_notes.md
    Q1: the MOTD emoji slot reads from the user's
    active_stickers pool. MOTD font is locked global per the
    session 7 rewrite.

  - four_tasks_coin_name_design_notes.md
    Q8's contrast: coin name reroll keeps doubling cost
    mechanic; MOTD reroll does not.

  - four_tasks_architectural_preference.md
    Clarity over cleverness — the "one sequence parameterised
    for rest day vs worked day" choice (Q6) follows directly.
