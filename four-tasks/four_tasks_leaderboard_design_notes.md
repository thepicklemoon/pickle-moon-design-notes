# Leaderboard — Design Notes

**Status:** LEANING TOWARD RESTRAINT (session 6). Design conversation
closed in principle; final ship decision deferred until tile 4.13 is
actually adjacent. Captures the full discussion including alternatives
that were considered and rejected, so the rejection reasoning is
recoverable when the tile lands.

**Implementation timing:** tile 4.13 (UI) + tile 4.13a (endpoint).
Both deferred until Phase 4 UI work is well underway and the coin
name censorship layer (tile 4.17) has shape.

**References:**
- `four_tasks_architectural_preference.md` — clarity-over-cleverness,
  applied here as "fewer surfaces over more surfaces."
- `four_tasks_staggered_disclosure_design_notes.md` — informs the
  long-press discoverability gradient and v1.x layering.
- `four_tasks_coin_name_design_notes.md` — coin names are what
  appears on the leaderboard, not raw usernames; censorship layer
  applies before display.
- `four_tasks_monetisation_position.md` — the calm, low-pressure
  tone the app holds; the leaderboard must not violate this.

---

## The problem the leaderboard is trying to solve

Users want some sense of where they sit in the broader community.
"How am I doing compared to other people doing this?" is a real
question with a real motivational lift when answered well.

Four Tasks already has the strongest social-comparison mechanic
baked in at the most local layer — the partner panel. Every user
sees their partner's progress in real time, all the time. That's
where most of the comparison energy goes, and it's healthy because
the comparison is between two people in a known relationship doing
the same thing.

The leaderboard is the *next* layer out. The question is: what
shape of broader-community comparison is healthy enough to be worth
shipping, given the partner panel already covers the local
comparison need?

---

## The recommendation — one leaderboard, lifetime coins, with current streak as a secondary number

Ship a single leaderboard. Ranked by `lifetime_coins`. Each row shows
the user's coin name (the censored, server-generated display name)
plus their current streak as a secondary number to the right of the
coin total.

Access: long-press the user's own coin display in the util bar.
The popup opens with the user's row centred, ten rows above and
below visible. Top-of-leaderboard is reachable by scroll.

That's the entire leaderboard surface. One metric, one popup, one
gesture to access.

### Why lifetime coins

Lifetime coins is the only metric that survives the four problems
that kill most leaderboards:

**Strictly monotonic.** Every coin ever earned, never decremented
by spending. A user who spends aggressively on rerolls is identical
on the leaderboard to a user who hoards. The leaderboard does not
push users to avoid spending — the reroll economy stays intact.

**Tenure-aware in a healthy way.** A 400-day user with a 12-current-
streak has earned more lifetime coins than a 30-day user with a
28-current-streak. That ordering feels right — the long-haul user
has objectively done more total work. The new user climbs through
consistency over time. No artificial gating.

**Engagement-sensitive.** Lifetime coins captures consistency of
*task completion* (4-task days earn more than 1-task days, streak
multipliers compound) not just consistency of *day completion*. Two
users with identical current streaks can have meaningfully different
lifetime totals based on how thoroughly they engaged each day.

**Doesn't punish single off-days catastrophically.** A user who
misses one day still has all their accumulated lifetime coins
tomorrow. The leaderboard position barely moves. This is the
opposite of streak-ranked leaderboards, where one missed day tanks
a user from #5 to #500.

### Why current streak as a secondary number

The rank is lifetime coins. The streak number sits next to it as
a "what is this person doing right now" signal that humanises the
row without changing rank.

A user with massive lifetime coins but a broken streak reads as
"this person played hard for a long time and is currently taking
a break." Not a loser. Just a person with a story.

A user with modest lifetime coins but a long current streak reads
as "this person is newer and crushing it right now." Visibly
catching up.

Both stories are legible on the same row. Neither alone determines
rank.

### Why not show coins (current spendable balance) anywhere

Current coins on a leaderboard would create a perverse incentive:
spending coins drops you publicly, therefore don't spend coins.
That directly guts the reroll economy and conflicts with the whole
point of having a spendable resource.

Current coins are private and stay private. They appear on the
user's own util bar but not on any social surface.

---

## What was considered and rejected

This section preserves the full conversation that led to the
recommendation, so future re-evaluation can compare like-for-like.

### Alternative 1 — install-date clustering

**Pitch:** cluster users with others who started Four Tasks around
the same time, so a 30-day user isn't comparing against a 400-day
veteran's coin total. The new user sees a leaderboard of their
contemporaries; the long-haul user sees one of theirs.

**Rejected because:**

1. **Hides growth context arbitrarily.** Users who joined with
   friends a few weeks apart could land in different clusters if
   the boundary fell between their join dates. The cluster
   boundaries become arbitrary social slicing — friends who'd
   actually want to compare can't.

2. **Thins the leaderboard.** Early-stage Four Tasks with a few
   hundred users would have clusters of 30-50. The leaderboard
   works better as broad social proof ("look how many people are
   doing this") than as a tight competition.

3. **Rewards lurkers.** A user who installed in January but didn't
   engage until June is clustered with January-installers whose
   engagement curves are way ahead. *Install date doesn't represent
   what the framing assumes it represents* — engagement age is the
   honest variable, and it's harder to measure cleanly.

4. **The problem it solves is solved better by lifetime coins.**
   A new user looking at a global lifetime-coins leaderboard can
   see the climb ahead of them as concrete numbers earned by real
   behaviour, not as an arbitrary gulf created by install timing.
   They're not competing against accumulated lead; they're seeing
   how they'd track up if they kept playing.

**However:** the install-date framing surfaces a real concern — new
users feeling dispirited by tenure-driven leaderboards. That concern
is valid. Lifetime coins addresses it through monotonicity + the
"climb is visible" property rather than through clustering.

### Alternative 2 — swipeable multi-metric leaderboard popup

**Pitch:** long-press the coin display to bring up the coin
leaderboards; swipe horizontally between lifetime coins, current
coins, install-date-clustered coins. Long-press the streak display
for a similar 2-3 page leaderboard with current streak, longest
streak, and other streak breakdowns. Cheap to engineer; gives every
user the comparison view that suits them.

**Engineering cost:** genuinely small. One endpoint shape with a
metric query parameter (`GET /leaderboard?metric=lifetime_coins`),
different ORDER BY clauses, same handler. Cheap.

**Rejected because engineering cost wasn't the right thing to
optimise for:**

1. **Each leaderboard is a thing that has to feel good to be on.**
   A leaderboard with 12 users on it (because install-date
   clustering thinned it) is sad. A leaderboard where #1 holds
   47,000 lifetime coins makes the #500 user feel like garbage.
   Every variant added is a new surface where users can compare
   themselves unfavourably. Multiplying surfaces multiplies ways
   to feel bad.

2. **Decision fatigue on a small popup.** Swiping through 5-6
   views to find "the one that makes me look best" is casino
   logic — pull the lever until you find the rank that validates.
   That's a behaviour pattern to avoid encouraging, not enable.

3. **Each metric displayed is a metric users are implicitly told
   to optimise.** Showing current coins says "save coins, don't
   spend." That kills the reroll economy. Showing current streak
   as a *rank* (not a secondary number) creates brutal pressure
   not to miss a day. Each metric carries its own behavioural
   incentive; shipping all of them broadcasts a confused set of
   "what should I be doing?" signals.

4. **App tone doesn't support a busy comparison UI.** Four Tasks
   is calm and low-pressure. The monetisation position doc and
   staggered disclosure doc both lean on *not* overwhelming users
   with surfaces. A swipeable multi-metric popup is the busiest
   surface in the app, sitting inside an already-comparison-heavy
   feature.

5. **The instinct behind it ("different users want different
   things") is real, but the right response is "pick the
   healthiest metric for the most users," not "show every metric
   so each user picks their own." Health > preference because the
   leaderboard is recurring; an unhealthy leaderboard sours users
   on it over time.

**However:** the swipeable-popup framing surfaces a real desire —
giving users a sense of agency over which comparison they engage
with. v1.x revisit should consider whether that agency is better
served by opt-in friend leaderboards (see below) rather than
multiple global metrics.

### Alternative 3 — streak as primary rank

**Pitch:** rank by current streak. Naturally normalises against
tenure — a new user with a 30-day streak ranks above a long-haul
user with a 12-day current streak. Captures "who's doing this well
right now" rather than "who's accumulated the most."

**Initially compelling because:** present engagement is more
motivating than accumulated work.

**Rejected because:** streak ranking is brutal on streak breaks.
One missed day tanks a user from #5 to #500 overnight. That creates
unhealthy pressure to never miss a day, which is the opposite of
the calm tone the app holds. Lifetime coins captures most of the
same "present engagement matters" property via streak multipliers
on coin payout, without the cliff.

Streak remains visible as a secondary number on every row of the
lifetime-coins leaderboard, preserving its role as a "what's this
person doing now" signal without making it rank-determining.

### Alternative 4 — no leaderboard at all

**Pitch:** the partner panel covers the social comparison need.
The leaderboard adds a global comparison surface that doesn't fit
the app's calm tone and creates emotional risk for users outside
the top cohort. Ship without one.

**Considered seriously.** This is the cleanest answer in terms of
"health of the comparison surface" because the only way a
leaderboard can't make a user feel bad is to not have one.

**Not rejected; held in reserve.** If at implementation time
(tile 4.13) the design still feels uncertain, ship without a
leaderboard for v1.0 and revisit in v1.x. There's no architectural
cost to deferring — no schema, no endpoint, no popup. The
`lifetime_coins` column should still be added (it has uses beyond
the leaderboard — yearly keepsake, future achievements) but
nothing has to surface to the user.

**Why we lean toward shipping one anyway:** the partner panel
covers the *local* comparison need but not the "I'm one of a real
community doing this" need. A single, restrained, monotonic
leaderboard fills that gap without creating the multi-surface
problems Alternative 2 has. It's the minimum useful version of
the social proof Four Tasks would benefit from.

---

## Schema implications

### New column — users.lifetime_coins

```sql
ALTER TABLE users ADD COLUMN lifetime_coins INTEGER NOT NULL DEFAULT 0;
```

Monotonic accumulator. Set to 0 at user creation. Incremented by
the same delta as `users.coins` whenever coins are paid out.
Never decremented.

Likely lands as `migration_004_lifetime_coins.sql`.

The morning-payout claim endpoint (designed in the morning sequence
doc, deferred for implementation alongside tile 1.3) currently
writes to `users.coins`. It will additionally write to
`users.lifetime_coins` with the same delta. Trivial change to the
claim endpoint when both implementations land.

### No new column for streak ranking

The leaderboard reads existing `users.streak` for the secondary
display number. No schema change.

### No new column for opt-out

Captured below as a deferred decision — opt-out flow needs its
own design pass on whether the column lives on `users` or a
separate `users_privacy` table.

---

## Endpoint shape — GET /leaderboard

Pattern:
```
GET /leaderboard?around=<pair_key>&self=<name>
```

The `around` parameter tells the server to centre the response on
the requesting user's position. Response shape:

```json
{
  "ok": true,
  "data": {
    "rows": [
      { "rank": 1234, "coin_name": "Frosty Pickle", "lifetime_coins": 8432, "streak": 47 },
      ...
    ],
    "self_rank": 1247,
    "total_users": 8421
  }
}
```

Returns ~21 rows: self centred, 10 above, 10 below. If the user is
in the top 10 globally, the top 21 rows are returned. If near the
bottom, the bottom 21.

The "view top of leaderboard" interaction is a separate request:
```
GET /leaderboard?top=20
```

Returns the top 20 rows, no self-context.

### Censorship layer

Coin names go through the censorship pass (tile 4.17 / coin name
design doc) before being returned. Any name failing the censor is
replaced with a fallback (TBD — see deferred items).

### Opt-out behaviour

Users opted out of the leaderboard:
- Their row is excluded from response entirely.
- Their `self_rank` is `null`.
- The leaderboard popup, when opened, shows a different state for
  opted-out users (TBD — see deferred items).

### Rate limit tier

`/leaderboard` is a paired endpoint (caller's pair_key required
for the `around` parameter to make sense). Inherits the paired-tier
rate limit budget from `four_tasks_rate_limiting_design_notes.md`.
Likely shares the 40/min budget with GET /pair/:key as a "reads"
group, or gets its own modest budget (10-15/min, since it's a
deliberate user action, not polled).

---

## UX surfacing

**Access gesture:** long-press the coin display in the util bar.
Same pattern as the sticker picker's long-press-context-menu —
familiar gesture from elsewhere in the app.

**Reveal timing:** staggered disclosure (see staggered disclosure
design notes). The leaderboard isn't shown to onboarding-stage
users. It reveals on day N (TBD, probably 7-14 days) once the user
has enough lifetime coins to occupy a meaningful position. Showing
it on day 1 puts the user at the bottom of a population that's
been playing for months; that's the dispiriting state to avoid.

**Popup style:** matches the partner-reaction confirmation popup
and the bug catcher popup. Same visual language across the app's
overlay surfaces.

**Empty / opted-out state:** if the user is opted out, the popup
explains the opt-out and offers to opt back in. If the user just
hasn't earned enough to appear (early days), the popup shows a
gentle "your rank appears once you've earned X lifetime coins"
state instead of "rank: 8,421 of 8,421."

---

## Deferred / open items

### Opt-out flow design

Needs its own focused pass. Open questions:
- Where does opt-out live in settings UI?
- Is opt-out global ("don't show me on any leaderboard") or
  granular (current implementation has only one leaderboard so
  this is moot for v1.0, but matters if Alternative 2 is ever
  reconsidered)?
- Is opted-out state stored on `users` (simple boolean column) or
  a separate `users_privacy` table (cleaner separation, but adds
  schema complexity for a single field)?
- Default: opted in or opted out?

Lean: opt-out lives on `users.leaderboard_opt_out BOOLEAN DEFAULT 0`
(opted-in default), settings UI has a single "show me on the
leaderboard" toggle. Simplest version that works.

### Censorship fallback for coin names

When a user's coin name fails the censor, what shows on the
leaderboard? Options:
- A fixed placeholder like "(name hidden)" — clear but bland.
- A re-rolled coin name with a different seed — possibly cleaner.
- The censored coin name (asterisks etc.) — ugly.

Tied to tile 4.17 (coin name censorship). Resolve at that tile's
implementation time.

### Reveal threshold

What's the lifetime-coins or days-active threshold that gates
leaderboard visibility for new users? Picked from staggered
disclosure principles. Probably 7-14 days OR a coin threshold,
whichever lands first. Resolve when staggered disclosure's reveal
schedule is finalised.

### Opt-in friend leaderboards (v1.x revisit)

The "different users want different things" instinct from
Alternative 2 is better served by *opt-in friend leaderboards*
than by multiple global metric views. Concept:

- A user can see a leaderboard among the partners' partners they
  have encountered (a small, local social graph).
- Both halves of any pair opt-in to make their pair visible in
  this surface.
- Ranks within this surface use the same metric (lifetime coins)
  as the global leaderboard for consistency.
- Surfaces "where do I rank among people I'm connected to" rather
  than "where do I rank globally."

Architecturally substantial — requires tracking the partners-of-
partners graph, opt-in state on both sides, a separate query
path. Not v1.0 scope. Worth revisiting in v1.x once usage data
indicates whether the global leaderboard is doing its job alone.

### Multi-metric swipeable popup (Alternative 2 — explicit deferral)

Captured here so the rejection reasoning is on record but the
*possibility* of revisiting is also on record. If post-v1.0 usage
data shows that the single lifetime-coins leaderboard creates real
problems (users feel locked out, comparison feels stale, etc.),
revisit whether a second metric view is worth the surface-area
cost.

The threshold to revisit should be specific: real user feedback
indicating the single leaderboard isn't doing its job, not
"engineering is cheap so why not."

### Install-date clustering (Alternative 1 — explicit deferral)

Same logic. The rejection reasoning is recorded. If real-world
usage shows new users are bouncing off the leaderboard because of
the tenure gap, the *response* should be to revisit the reveal
threshold (don't show the leaderboard until users are competitive)
or the empty-state design (show climb-ahead framing) before
reaching for clustering.

---

## What this doc does not cover

- **Coin name censorship rules** (tile 4.17, separate design doc
  pending).
- **The morning-payout coin economy** (already covered in
  morning sequence design notes — this doc just consumes that
  output via the `lifetime_coins` increment).
- **Achievements / badges / other comparison mechanics**
  (DEFERRED v1.x or later).
- **Leaderboard analytics / observability** — not v1.0 scope.

---

## Closing note

The principle this doc holds: **engineering cost is cheap; user
attention and emotional energy are not.** Four Tasks is built
around calm, low-pressure engagement. The leaderboard is the
single feature most likely to violate that principle if shipped
expansively. Restraint here is not under-shipping — it's
matching the surface to the app's tone.

The doc leans toward "one leaderboard, lifetime coins, with
current streak as a secondary number." It explicitly preserves
the option of shipping no leaderboard at all if tile 4.13 lands
and the design still feels off. It captures the rejected
alternatives in full so future review can compare against
recorded reasoning rather than reconstructing from memory.
