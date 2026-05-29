# Four Tasks — Timezone & Sealing Design Notes

Last edit: 2026-05-29 AWST

Status: LOCKED (session 6, mobile drafting). Implementation landed
at tile 1.3, session 12 — lazy seal absorbed into the claim endpoint
transaction in `server/src/index.ts`. Tile 1.4 closed as absorbed.
Required reading before: tile 4.6 (morning sequence coordinator).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PROBLEM
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Four Tasks ships to a global audience. A user in Perth and a user in
Melbourne are two hours apart; a user in Perth and a user in London are
seven or eight hours apart depending on DST. The product's notion of
"a day" must be each user's local day, not UTC and not the server's
day. Otherwise a Perth user's morning payout could fire at 8am local
(midnight UTC) which is fine, but a London user's morning payout would
fire at 1am or 12am local — wrong feeling.

The prior tile 1.4 framing ("Worker scheduled cron — nightly sweep")
assumed a single UTC midnight tick walking all pairs. That doesn't
hold up under localisation.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DECISION SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Each user has a stored IANA timezone (e.g. "Australia/Perth").
   Mutable. Not part of the pair-key hash.

2. Sealing is LAZY, not scheduled. The Worker does not run a cron job.
   A day seals when the user it belongs to opens the app on a later
   local date and the claim endpoint fires.

3. Each user's seal is their own concern only. Opening the app never
   triggers a write to the partner's row. The partner panel renders
   from whatever is in the DB; if their local day has rolled and they
   haven't opened the app, the panel truthfully shows their incomplete
   day frozen at whatever state they left it.

4. Tile 1.4 (cron sweep) is REFRAMED, not deleted. See the tile 1.4
   section below.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
V1.0 SCOPE — WHAT ACTUALLY SHIPS (vs the target below)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The sections below describe the full IANA-based target architecture.
v1.0 ships a deliberately smaller subset; the rest is the v1.x target,
retained here as the plan.

What v1.0 actually does:

1. Client computes local_date from the OS *offset*, not an IANA name.
   Godot cannot read the device's IANA zone name without a native
   plugin, so v1.0 uses `Time.get_time_zone_from_system().bias` (the
   current UTC offset, already DST-correct for "now"). No date library
   on the client. This is safe because the client only ever computes
   the *current* local_date for the write in hand — it never recomputes
   a historical date — so it never needs DST history. (Replaces the
   shipped `State.gd` hardcoded +8h. Verify the bias sign on a real
   Android device.)

2. IANA-name capture is DEFERRED to v1.x — that's what needs the native
   plugin. Until then `users.timezone` is written best-effort / left at
   its 'UTC' default and is NOT relied on.

3. `users.timezone` therefore has NO live v1.0 consumer. The three
   things that would read it are all unbuilt or sidestepped:
   - Server plausibility check: SPECCED, NOT BUILT. The server trusts
     the client's local_date in v1.0. Anti-scrubbing is a non-goal (see
     that section), so this costs nothing.
   - Partner-panel "are they in a new local day" math: NOT BUILT. The
     partner panel reads the partner's rows in the VIEWER's date frame
     (viewer's "yesterday" for the coin bonus, viewer's "today" for
     stickiness), with no partner-timezone conversion. Cross-tz boundary
     fuzz — a partner a few hours ahead/behind sitting on a date
     boundary — is ACCEPTED, not corrected.
   - tz-aware coin multiplier: unbuilt; when it lands it reads the
     sealed row's own date, not a tz conversion.

The column stays in the schema as forward-compat for v1.x, when the
plugin lands and the IANA features above get built for real.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SCHEMA
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

In the v1.0 schema (`server/schema.sql`, session 12 lock):

  users.timezone TEXT NOT NULL DEFAULT 'UTC'

Notes:
- IANA name only. "Australia/Perth", "Europe/London", "America/New_York".
- DEFAULT 'UTC' is a safety net; onboarding always writes the real
  device timezone, so new users never hit the default.
- Not part of the pair-key. Changing timezone (travel, move, DST shift
  if name changes — e.g. "Europe/Kiev" → "Europe/Kyiv") does not
  trigger a pair-key rotation. This is a render-and-date-math field.
- Mutable via `PUT /users/:user_id` (session 12 endpoint).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CLIENT BEHAVIOUR
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

v1.0 NOTE: v1.0 computes local_date from the OS offset, not the IANA
name (Godot has no native IANA read without a plugin). This section is
the v1.x IANA target — see V1.0 SCOPE above.

Boot-only sync. The client checks timezone exactly once per app boot,
not on a polling loop. If the user changes timezone mid-session
(stepped off a plane and opened the app, then crossed time zones
without quitting), the change gets caught on next boot. The cost of
missing a mid-session timezone change for a few hours is zero in
this product; the cost of polling is real (battery, write churn).

On every app boot:
  1. Read the device's current IANA timezone.
  2. Compute local_date as the user's current calendar date in that tz.
  3. If the device timezone differs from the stored server-side
     timezone, fire `PUT /users/:user_id` with the new timezone
     before any other work. Single write, fire-and-forget; failure
     means we'll catch it next boot.
  4. Proceed to load user state, run morning sequence if the claim
     endpoint indicates a payout is owed, etc.

On every write that depends on "what day is it":
  Client sends local_date in the payload. Server validates the date
  is plausible against the stored timezone (within a few hours of
  the user's current local now) and uses the client-reported date
  as canonical for that write.

Why trust the client's local_date? Because the alternative is
re-deriving it server-side from the stored timezone and the server's
current UTC, which introduces a race: the user just changed timezone,
the server hasn't received the update yet, every other write in
between gets dated wrong. Letting the client tell the server "I
believe it is 2026-05-14 here" is the single source of truth that
matches the user's lived reality.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SERVER BEHAVIOUR
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

v1.0 NOTE: the local_date plausibility check described here is SPECCED,
NOT BUILT. In v1.0 the server trusts the client's local_date (anti-
scrubbing is a non-goal, so nothing turns on it). See V1.0 SCOPE above.

Server stores `users.timezone` but rarely needs to compute dates from
it. Its job on writes is to validate that the client-reported
local_date is plausible — i.e. matches the user's stored timezone to
within a small window (say ±4 hours of the user's current local
midnight boundary, generous to allow for clock skew). Reject as 400
"invalid_local_date" if it's not.

Server uses a date library (recommendation: Luxon, since it handles
IANA names natively and Cloudflare Workers run modern V8). Do not
roll your own DST math. Do not store offsets like "+08:00" — they
break across DST transitions. IANA names only.

The claim endpoint (`POST /users/:user_id/claim`, session 12) uses
the stored timezone and the client-reported local_date together:

  - Server walks back from local_date - 1 looking for the most
    recent unsealed past day belonging to this user that has data.
    By app-open semantics there is AT MOST ONE such row.
  - If found, the seal-and-payout transaction runs against that
    row: stamp tier resolved, coins computed, streak updated,
    theme snapshot written, `sealed_at` set. Atomic.
  - If not found (user has already claimed everything, or has no
    past days with data), claim returns 200 with empty payout.

Note on the "intervening days" framing in earlier drafts: the
session 12 implementation does NOT walk a multi-day gap and seal
each one. The user has at most one unsealed day at any time
because every app open triggers the same walkback. If the user
opens the app three days late, day-1's seal happened at day-2's
open, day-2's at day-3's, day-3's seals today. The single-row
walkback is correct by the same reasoning the
`morning_payout_due_for` column was redundant (session 12).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TILE 1.4 — ABSORBED INTO TILE 1.3 (session 12)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Tile 1.4 was originally framed as "Worker scheduled cron — nightly
sweep." That framing was superseded at session 6 to "lazy seal-on-
open inside the claim endpoint transaction" (this doc). The
implementation then landed inside tile 1.3 at session 12, since the
seal logic is the claim endpoint's core responsibility — splitting
it across two tiles would have been ceremonial.

There is no cron job. There is no Worker scheduled trigger. The
Worker only runs on incoming HTTP requests. Sealing happens when
the user shows up, only for that user's row.

Tile 1.4 is closed in the todo as "absorbed into tile 1.3."

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CROSS-USER INTERACTION (revisited)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

For v1.0 the cross-user surface is read-only:

  v1.0 NOTE: the partner panel reads the partner's rows in the VIEWER's
  date frame — no partner-timezone conversion. Item 1 below is the v1.x
  target; in v1.0, cross-tz boundary fuzz is accepted. See V1.0 SCOPE.

  1. Partner panel rendering: read-only. Client uses the partner's
     stored timezone to compute "are they in a new local day" purely
     for display (e.g. "their today is empty because their day just
     rolled" vs "their today is empty because they haven't done
     anything yet"). No DB writes triggered.

  2. (Coin multiplier check, if/when added: server reads partner's
     most recent day completion. No write to their row, no
     dependency on their seal state. Not in session 12 claim
     implementation; placeholder coin formula is per-task random
     900-1400, no partner-dependent multiplier yet.)

Partner reactions (deferred at session 12, originally the third
cross-user case) would have introduced a partner-writes-to-other-
user's-row endpoint. Eliminated from v1.0; schema columns retained
for v1.x.

That's the entire cross-user surface for v1.0. No bridging code
needed beyond reads.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NON-GOAL: ANTI-DATE-SCRUBBING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

We are not defending against users changing their device clock to
farm coins. The existing protections already handle the realistic
abuse surface:

  - The claim endpoint walkback finds at most one unsealed past day
    per call. Rolling the clock forward skips days, doesn't farm
    them — the skipped days remain unsealed and can be claimed if
    the clock goes back to where it should be, but each day still
    yields one payout.

  - Rolling the clock backward is rejected by the local_date
    plausibility check (server compares against stored timezone +
    a generous window). (v1.0: this check is not yet built — the server trusts the client's local_date; it lands with the v1.x IANA work. Anti-scrubbing is a non-goal regardless, so nothing turns on it.)

  - No leaderboard is planned for v1.0 (deferred), removing the
    primary motivation for coin farming. (If a leaderboard ever
    ships, this section gets revisited.)

  - The theoretical "scrub-to-unlock-everything-then-sell-account"
    attack is real but vanishingly rare and already economically
    defused: account portability between devices requires the full
    pair-key, which requires the partner's matching values. You
    can't sell what isn't transferable in isolation.

Future-Morgan: do not re-litigate this. The cost of pretending to
care about date-scrubbing is meaningful UX friction (rejecting
legitimate timezone changes, false-positive 400s on travellers, etc).
The cost of ignoring it is zero in the product as designed.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ONBOARDING IMPLICATIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The Welcome step (tile 3.2) or one of the early onboarding steps
needs to capture timezone. Two ways:

  (a) Silently read device timezone, store it. No UI.
  (b) Show user a "Your timezone: Australia/Perth — correct?" line
      with an edit option.

Recommend (a). The device knows. If the user travels, the boot-time
check (see "Client behaviour" above) catches the change. Surfacing
this in onboarding adds a question the user shouldn't have to think
about. Staggered disclosure logic applies: don't reveal a setting
the user doesn't need to touch.

Settings screen (whenever that lands — currently no tile assigned)
should expose the timezone field as a read-only display with a
manual override option. Manual override exists for users whose
device timezone is wrong and they can't fix it (work-issued device,
weird Android variant, etc).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DST EDGE CASES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Twice a year in DST-observing regions, a calendar day is 23 or 25
hours long. IANA names handle this correctly via Luxon. The product-
visible consequence is: nothing. A day is still a day. The morning
payout fires on the first app open of the new local date, whatever
that local date's wall-clock duration was.

WA does not observe DST (good for testing on Morgan's device). Make
sure the test pair includes one DST-observing tz (e.g. Melbourne,
London) before shipping, so the date library is exercised on a
transition boundary.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED TILES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- Tile 1.3 (defensive write endpoints): CLOSED session 12. The
  local_date plausibility check and seal-on-claim logic landed
  here.
- Tile 1.4 (cron sweep): CLOSED as absorbed into tile 1.3 claim
  endpoint transaction.
- Tile 3.x (Welcome / onboarding screens): the onboarding flow
  silently reads device timezone and stores it via the user
  creation endpoint. No UI surface for timezone in onboarding.
- Tile 4.6 (morning sequence coordinator): claim endpoint contract
  documented in the morning sequence doc takes a local_date
  parameter.
- Future settings screen tile (no number yet): exposes timezone
  as read-only display with manual override.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OPEN QUESTIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

None at the time of writing. Surface here as they appear.