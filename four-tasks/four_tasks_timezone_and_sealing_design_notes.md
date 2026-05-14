# Four Tasks — Timezone & Sealing Design Notes

Status: LOCKED (session 6, mobile drafting)
Supersedes: tile 1.4 prior framing as "Worker scheduled cron — nightly sweep"
Required reading before: tile 1.3 implementation, tile 1.4 (now reframed),
                         tile 4.6 (morning sequence coordinator)

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
SCHEMA
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

migration_004_user_timezone.sql:

  ALTER TABLE users ADD COLUMN timezone TEXT NOT NULL DEFAULT 'UTC';

Notes:
- IANA name only. "Australia/Perth", "Europe/London", "America/New_York".
- DEFAULT 'UTC' is a safety net for pre-existing rows. Onboarding
  always writes the real value, so new users never hit the default.
- Not part of the pair-key. Changing timezone (travel, move, DST shift
  if name changes — e.g. "Europe/Kiev" → "Europe/Kyiv") MUST NOT
  trigger a pair-key migration. This is a render-and-date-math field.
- Mutable via PUT /pair/:key/users/:target (already a write rule
  endpoint per tile 1.3 design).

Apply alongside or after migration_003. Order doesn't matter — they
touch different columns.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CLIENT BEHAVIOUR
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

On every app boot:
  1. Read the device's current IANA timezone.
  2. Compute local_date as the user's current calendar date in that tz.
  3. If the device timezone differs from the stored server-side
     timezone, fire PUT /pair/:key/users/:self with the new timezone
     before any other work. This is the "I just landed in Sydney"
     case. Single write, fire-and-forget; failure means we'll catch
     it next boot.
  4. Proceed to load pair state, run morning sequence if
     morning_payout_due_for indicates a payout is owed, etc.

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

The claim endpoint (see morning sequence doc) uses the stored
timezone and the client-reported local_date together:
  - Server reads users.morning_payout_due_for.
  - If morning_payout_due_for is non-null AND <= the reported
    local_date, fire the payout, null the field, and seal any
    intervening days.
  - If morning_payout_due_for is null or > the reported local_date,
    no payout owed, return 200 with empty payload.

Sealing intervening days: if the user was offline for three days,
their morning_payout_due_for might be 2026-05-11 and they're opening
on 2026-05-14. Days 11, 12, 13 all need to be sealed (locked,
multiplier resolved against partner's state, written immutable).
Server walks the gap inside the claim transaction. This is bounded
work — even a year offline is 365 row updates, tiny.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TILE 1.4 REFRAMED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

OLD: "Worker scheduled cron — nightly sweep. Walks all pairs at
      midnight (UTC or per-pair timezone, decision pending), seals
      yesterday's diary entries."

NEW: "Lazy seal-on-open. Sealing logic lives in the claim endpoint
      (tile 1.3 write rules). When a user opens the app and the
      claim endpoint fires, any unsealed days <= the user's
      current local_date - 1 get sealed in the same transaction."

There is no cron job. There is no Worker scheduled trigger. The
Worker only runs on incoming HTTP requests. This is simpler, removes
a moving part, and removes the "user offline at midnight" edge case
entirely — sealing happens when the user shows up, and only for
that user's row.

Tile 1.4 now becomes part of tile 1.3 implementation in practice
(the seal logic lives inside the claim endpoint transaction). The
tile is kept on the list as a separate line item only because the
seal logic is conceptually distinct from the payout logic and is
worth its own commit + curl-verification pass.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CROSS-USER INTERACTION (revisited)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Confirmed minimal:

  1. Partner reactions: writes to partner's row, sealed days only.
     Already locked in partner reactions design doc. Sealed-ness is
     per-user; partner reacting to your sealed day-N is fine even
     if their day-N is still open on their side.

  2. 1.5x multiplier check: when computing your payout, server
     reads your partner's most recent day completion. No write to
     their row, no dependency on their seal state — past days are
     immutable by write rules regardless of explicit seal flag.

  3. Partner panel rendering: read-only. Client uses the partner's
     stored timezone to compute "are they in a new local day" purely
     for display (e.g. "their today is empty because their day just
     rolled" vs "their today is empty because they haven't done
     anything yet"). No DB writes triggered.

That's the entire cross-user surface. No bridging code needed beyond
what these three already imply.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NON-GOAL: ANTI-DATE-SCRUBBING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

We are not defending against users changing their device clock to
farm coins. The existing protections already handle the realistic
abuse surface:

  - morning_payout_due_for is a single-use voucher. Server issues
    one per user-day; once claimed, nulled. Rolling the clock
    forward skips days, doesn't farm them.

  - Rolling the clock backward is rejected by the local_date
    plausibility check (server compares against stored timezone +
    a generous window).

  - No leaderboard is planned, removing the primary motivation for
    coin farming. (If a leaderboard ever ships, this section gets
    revisited.)

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

- Tile 1.3 (defensive write rules): the local_date plausibility
  check and seal-on-claim logic land here. Update tile 1.3 scope
  to reference this doc.
- Tile 1.4 (cron sweep): reframed to seal-on-open. Tile description
  updated to reflect new architecture. Implementation merges with
  tile 1.3 transaction.
- Tile 3.2 (Welcome step) or wherever onboarding captures device
  timezone: add a line "silently read and store IANA timezone."
- Tile 4.6 (morning sequence coordinator): the claim endpoint
  contract documented in the morning sequence doc gains a
  local_date parameter. Update the morning sequence doc's claim
  endpoint section with a cross-reference here.
- Future settings screen tile (no number yet): exposes timezone
  as read-only display with manual override.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OPEN QUESTIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

None at the time of writing. Surface here as they appear.
