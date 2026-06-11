# Four Tasks — Timezone & Sealing Design Notes

Last edit: 2026-06-11 AWST (s32)

Status: LOCKED (session 6, mobile drafting). Implementation landed
at tile 1.3, session 12 — lazy seal absorbed into the claim endpoint
transaction in `server/src/index.ts`. Tile 1.4 closed as absorbed.
AMENDED s32 (recording s31 ships): the SEALED-BEATS-CLOCK client rule
and the ±2-day coin-source bound on claim/resolve — see CLOCK BOUNDS.
The single-row walkback described under SERVER BEHAVIOUR is superseded
ON BUILD by tile 4.25 span-sealing (doc-locked s32; forward note inline).
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

2. Sealing is LAZY, not scheduled. The Worker does not run a cron job
   for sealing. A day seals when the user it belongs to opens the app
   on a later local date and the claim endpoint fires. (The Thursday
   content-drop cron added s29 is unrelated to sealing — release
   scheduling only.)

3. Each user's seal is their own concern only. Opening the app never
   triggers a write to the partner's row. The partner panel renders
   from whatever is in the DB; if their local day has rolled and they
   haven't opened the app, the panel truthfully shows their incomplete
   day frozen at whatever state they left it.

4. Tile 1.4 (cron sweep) is REFRAMED, not deleted. See the tile 1.4
   section below.

5. (s31) The client owns the clock — BOUNDED. The server accepts the
   client's local_date for the coin source (claim/resolve) only within
   ±2 days of server UTC, and on the client a SEALED day beats the
   clock in every render/editability decision. See CLOCK BOUNDS.

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
   - Server plausibility check: SPECCED, NOT BUILT (the fine-grained
     ±hours IANA version). v1.0 instead ships the coarse ±2-DAY bound
     on claim/resolve only (s31, CLOCK BOUNDS below) — it needs no
     timezone data at all, which is exactly why it could ship now.
   - Partner-panel "are they in a new local day" math: NOT BUILT. The
     partner panel reads the partner's rows in the VIEWER's date frame
     (viewer's "yesterday" for the coin bonus, viewer's "today" for
     stickiness), with no partner-timezone conversion. Cross-tz boundary
     fuzz — a partner a few hours ahead/behind sitting on a date
     boundary — is ACCEPTED, not corrected.
   - tz-aware coin multiplier: built (s28) WITHOUT tz awareness — it
     reads the sealed row's own date, not a tz conversion, exactly as
     planned here.

The column stays in the schema as forward-compat for v1.x, when the
plugin lands and the IANA features above get built for real.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CLOCK BOUNDS — WHAT SHIPPED s31
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Two rules, one server-side and one client-side, both found via
backwards-clock testing on device (session 31). Together they are the
operational meaning of "the client owns the clock": ownership within
bounds, and never over sealed history.

1. COIN-SOURCE BOUND (server). `POST /users/:id/claim` and
   `POST /users/:id/claim/resolve` reject a `local_date` outside
   ±2 days of server UTC — 400 `bad_date`
   (`LOCAL_DATE_BOUND_DAYS` in index.ts; wire test W18).

   Scope is deliberate and narrow:
   - The claim is the ONLY coin source. Unbounded trust let a rolled
     clock mint days — advance the clock, claim, advance, claim —
     fabricating payouts rather than merely skipping days. The bound
     closes the mint.
   - Sinks and day writes stay UNBOUNDED. They create no coins; a
     scrubbed rest purchase or task tick is self-spoiling only.
     Rationale recorded at the constant.
   - ±2 days never false-positives a legitimate user: the widest real
     spread between a device's honest local date and server UTC is
     ~±1 day across the date line. No timezone data needed — which is
     why this could ship in v1.0 while the fine-grained IANA
     plausibility check stays v1.x.
   - Time travel inside the bound remains legal and self-spoiling
     (you run real seals against real rows; you can spoil your own
     ceremony, not print money). The DevKit mock clock deliberately
     flows through the same path, so mocked dates exercise the real
     seal pipeline — and the bound caps how far a mock can travel.

2. SEALED-BEATS-CLOCK (client). A day row with `sealed_at` set renders
   SETTLED and is never toggleable, REGARDLESS of what the local clock
   says today is. Backwards clock travel — manual date change, or a
   westward timezone move — can make "today" a sealed day. The server
   held throughout (every write 409'd `day_sealed`); the client gap was
   that `task_list` gated toggleability on the clock alone, so a sealed
   TODAY rendered editable and every tap silently bounced. Fixed s31:
   `is_sealed` threads through `_render` / `_resolve_row_state`;
   App's rest long-press bails on sealed days.

   The rule generalises: ANY client surface deciding editability or
   settled-vs-live rendering checks `sealed_at` FIRST and the date
   second. Pre-payout mode (the ceremony's own staging of the
   already-sealed day) is the single deliberate exception. The 4.5
   MOTD-edit client half MUST carry the same gate (noted on the tile).

   Related, not yet built: tile 4.23 (App local-date-change watcher) —
   on focus-in, if today_iso moved in EITHER direction, repaint both
   calendars and close any open tray whose date is now future.
   Backwards moves currently leave stale paint; forward moves are
   already claim-handled.

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
the v1.x IANA target — see V1.0 SCOPE above. The s31 client rule that
DID ship — sealed-beats-clock — is under CLOCK BOUNDS.

Boot-only sync. The client checks timezone exactly once per app boot,
not on a polling loop. If the user changes timezone mid-session
(stepped off a plane and opened the app, then crossed time zones
without quitting), the change gets caught on next boot. The cost of
missing a mid-session timezone change for a few hours is zero in
this product; the cost of polling is real (battery, write churn).
(Note the boot-only model is for the TIMEZONE FIELD sync only — the
local DATE is re-checked on every focus-in per the morning trigger's
per-local-day semantics, s29, and tile 4.23 extends that to repaint.)

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
matches the user's lived reality. (s31 qualifier: trusted within
±2 days on the coin source — see CLOCK BOUNDS. The trust model
stands; the mint is fenced.)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SERVER BEHAVIOUR
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

v1.0 NOTE: the fine-grained local_date plausibility check described
here (±hours against the stored IANA timezone) is SPECCED, NOT BUILT —
it lands with the v1.x IANA work. The coarse ±2-day bound on the coin
source DID ship (s31, CLOCK BOUNDS). See V1.0 SCOPE above.

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
the client-reported local_date:

  - Server walks back from local_date - 1 looking for the most
    recent unsealed past day belonging to this user that has data.
    By app-open semantics there is AT MOST ONE such row in normal
    use (soft invariant — see below).
  - If found, the seal-and-payout transaction runs against that
    row: stamp tier resolved, coins computed, streak updated,
    theme snapshot written, `sealed_at` set. Atomic.
  - If not found (user has already claimed everything, or has no
    past days with data), claim returns 200 with empty payout.

Note on the "intervening days" framing in earlier drafts: the
session 12 implementation does NOT walk a multi-day gap and seal
each one — the single-row walkback was judged correct by the same
reasoning the `morning_payout_due_for` column was redundant.

  SUPERSEDED ON BUILD — tile 4.25 (doc-locked s32). The earlier
  drafts were closer to right than session 12 gave them credit for:
  span-sealing reinstates the multi-day walk. The claim will
  adjudicate EVERY day between the last seal and local_date, oldest
  first — never-opened days seal grey (no payout, silent), accounted
  days seal at tier, the rescue gate pre-scans the span, and history
  becomes contiguous. Two faults of the single-row model forced it:
  a hole only broke the streak at the NEXT seal (retroactive
  collapse, one day late), and a stranded accounted day behind a
  newer one was shadowed forever (the "at most one row" invariant is
  soft, and the most-recent-first LIMIT 1 never drains a backlog).
  Design: morning-sequence doc Q5 + economy doc. This doc's lazy
  seal-on-open model is unchanged — the walk just finishes the job.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TILE 1.4 — ABSORBED INTO TILE 1.3 (session 12)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Tile 1.4 was originally framed as "Worker scheduled cron — nightly
sweep." That framing was superseded at session 6 to "lazy seal-on-
open inside the claim endpoint transaction" (this doc). The
implementation then landed inside tile 1.3 at session 12, since the
seal logic is the claim endpoint's core responsibility — splitting
it across two tiles would have been ceremonial.

There is no sealing cron job. The Worker's only scheduled trigger is
the Thursday content-drop release promotion (s29) — release
scheduling, no user-row writes, nothing to do with sealing. Sealing
happens when the user shows up, only for that user's rows.

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

  2. Coin multiplier check — BUILT (s28) per the plan: the server
     reads the partner's completion FOR THE SEALED ROW'S OWN DATE.
     No write to their row, no dependency on their seal state, no
     timezone conversion (the sealed row's date is the frame).

Partner reactions (deferred at session 12, originally the third
cross-user case) would have introduced a partner-writes-to-other-
user's-row endpoint. Eliminated from v1.0; schema columns retained
for v1.x.

That's the entire cross-user surface for v1.0. No bridging code
needed beyond reads.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NON-GOAL: ANTI-DATE-SCRUBBING (qualified s31)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

We are not defending against users changing their device clock as a
general posture. ONE narrow exception was taken s31 — the ±2-day
bound on the coin source (CLOCK BOUNDS) — and the reason it was taken
is that the original analysis below under-counted:

  s31 CORRECTION: the first bullet's claim that "rolling the clock
  forward skips days, doesn't farm them" was wrong under fully-
  trusted local_date. Each forward roll makes yesterday's row
  claimable; roll, claim, roll, claim mints a payout per roll —
  fabricating days, not skipping them. The claim is the only coin
  source, so this was the one place scrubbing converted to currency.
  The ±2d bound closes the mint with zero traveller friction (the
  honest worst case is ~±1 day across the date line). Everything
  else below stands.

  - Rolling the clock backward (to re-edit history) is dead on two
    fronts: sealed days reject every write server-side (409
    day_sealed), and the client renders them settled regardless of
    the clock (sealed-beats-clock, s31). The fine-grained ±hours
    plausibility check remains v1.x.

  - Within the ±2d bound, time travel is legal and SELF-SPOILING:
    the traveller runs real seals against their real rows, spoils
    their own ceremony, and earns nothing they wouldn't have earned
    by waiting. The DevKit mock clock uses this corridor on purpose.

  - No leaderboard is planned for v1.0 (KILLED 2026-06-10), removing
    the primary motivation for coin farming.

  - The theoretical "scrub-to-unlock-everything-then-sell-account"
    attack is real but vanishingly rare and already economically
    defused: account portability between devices requires the full
    pair-key, which requires the partner's matching values. You
    can't sell what isn't transferable in isolation.

Future-Morgan: the standing instruction not to re-litigate this
holds, with the s31 exception now on the record. The test for any
further exception is the same one the bound passed: does it close a
coin mint, at zero cost to a legitimate traveller, without needing
timezone data we don't have? Anything that risks false-positive 400s
on real users is still a worse trade than the abuse it prevents.

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
the user doesn't need to touch. (As shipped, Phase 3 went with (a):
onboarding captures silently via create_user; no UI surface.)

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

Known v1.0 edge (s29, parked): `State.yesterday_iso` derives as
epoch − 86400s, which is wrong by ~1 hour around a DST transition
for users in DST-observing zones (a 23/25-hour day is not 86400s).
WA is unaffected. Same family and same fix window as the IANA-name
capture — lands with the v1.x timezone work.

WA does not observe DST (good for testing on Morgan's device). Make
sure the test pair includes one DST-observing tz (e.g. Melbourne,
London) before shipping, so the date library is exercised on a
transition boundary.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED TILES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- Tile 1.3 (defensive write endpoints): CLOSED session 12. The
  seal-on-claim logic landed here.
- Tile 1.4 (cron sweep): CLOSED as absorbed into tile 1.3 claim
  endpoint transaction.
- Tile 3.x (Welcome / onboarding screens): CLOSED Phase 3 — the
  onboarding flow silently reads device timezone and stores it via
  the user creation endpoint. No UI surface for timezone.
- Tile 4.6 (morning sequence coordinator): SHIPPED s26-28 — the
  claim contract takes local_date per the morning sequence doc.
- Tile 4.23 (local-date-change watcher): designed s31, open — the
  client-side repaint/close-tray reaction to date moves.
- Tile 4.25 (walkback span-sealing): doc-locked s32, open —
  supersedes the single-row walkback in SERVER BEHAVIOUR.
- Wire test W18: the ±2d bound (CLOCK BOUNDS).
- Future settings screen tile (no number yet): exposes timezone
  as read-only display with manual override.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OPEN QUESTIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

None at the time of writing. Surface here as they appear.
