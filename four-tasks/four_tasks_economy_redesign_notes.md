# Four Tasks — Economy Redesign Notes

**Status:** LOCKED for v1.0 (session 27). AMENDED 2026-06-11 (Morgan, mobile):
streak rule softened (≥1 task advances; only zero-task days break — sealed grey
or never-opened alike), gap days now break streaks at the next seal (closing the
previously-unnoticed no-show hole), and the STREAK RESCUE sink added (the
morning-sequence escape hatch). AMENDED AGAIN 2026-06-11 (s32, doc-locked ahead
of build — tile 4.25): WALKBACK SPAN-SEALING — the morning claim adjudicates
every day between the last seal and the claim date, oldest first; never-opened
days seal grey at first contact; the rescue offer anchors to the zero-task
day's own seal and fires only when exactly one such day stands between the
streak and survival. This doc is the AUTHORITY for the coin
economy: earn rates, multipliers, subscription tiers, sinks, and the per-user
ownership model. It SUPERSEDES the conflicting parts of
`four_tasks_monetisation_position.md` (the COIN ECONOMY TUNING section, the
pair-shared-coins claims, and the pair-pricing pack model) — those described the
prototype, not the shipped v1.0 schema. Where this doc and the monetisation doc
disagree, this doc wins.

**Implementation:** tile 4.3 (CoinFX, the visual payout) + tile 4.6 beat 6 (the
payout beat) consume the per-task numbers. The server claim handler
(`POST /users/:user_id/claim`, shipped tile 1.3) is where the multiplier, sub
bonus, and streak bonus get applied — see "Shipped vs target" below. Pack
pricing lands with the picker/store work (4.14a+ and Phase 5 subscription
plumbing).

Cross-references:
- `four_tasks_monetisation_position.md` — brand position, subscription product
  framing, price structure, founders model (all still authoritative); coin
  tuning + pair-pricing superseded here.
- `four_tasks_morning_sequence_design_notes.md` — the claim is where payout is
  computed; beat 6 is where it's animated. The walk mechanics (span-sealing,
  rescue gate two-phase shape) are specified there; this doc carries the RULES.
- `four_tasks_theme_design_notes.md` — pack/sticker feature-catalogue model;
  library access mechanic.
- `four_tasks_pair_key_design_notes.md` — per-user identity; subscription is
  per-user.

---

## The one-line model

Everything is **per-user**. Each person earns their own coins, holds their own
balance, spends on whatever they want, owns their own library. The ONLY thing
that crosses the pair boundary is **library access** (the first-subscription
unlock): you may *use* a sticker your partner owns. There is no shared wallet,
no shared purchase history, no co-owned packs, no pair-wide progress of any
kind.

This matches the shipped schema: `coins`, `lifetime_coins`, `streak`,
`longest_streak` all live on the `users` table, per-user. The monetisation
doc's "coins remain pair-shared per the prototype" line is STALE — it describes
the prototype, not what shipped in the session-12 wholesale rewrite.

---

## Earn

### Per-task base
- **900–1400 coins per completed task**, rolled independently per task
  (obfuscated range, carried from the prototype). A perfect 4-task day earns
  ~3600–5600 base (midpoint ~4600) before any multiplier.
- Rest day earns **0 at claim** (the rest day was paid for at designation, not
  earned at claim).
- 0-task accounted day (the grey "disappointment" tier): 0 coins.
- Never-opened day sealed grey by the span walk (s32): 0 coins, no payout
  of any kind — there is nothing to pay for.
- **Shadowed accounted task day (s32 edge):** if two consecutive claim
  failures strand an accounted task day mid-span (rare; needs the daily claim
  to fail across two separate days while task writes succeeded), span-sealing
  seals it at its natural tier WITH its natural payout, silently batched into
  the same claim — the user did the work and keeps the coins. The ceremony
  still presents only the most recent day; the balance simply arrives a little
  larger than the presented `coins_earned`. Honest, rare, and better than the
  s30 behaviour (the day was shadowed forever and its payout silently lost).

### Partner-completion multiplier
Reads the **partner's** task completion for that same day (their state, not
their coins). Applied to your payout:

| Partner did | Multiplier |
|---|---|
| 1 task | 1.2× |
| 2 tasks | 1.3× |
| 3 tasks | 1.4× |
| 4 tasks | 1.5× |

Flat across free and one-sub pairs — the first subscription does NOT scale this.
(Supersedes the old flat "base 1.5×" in the monetisation doc with the graduated
1.2→1.5 ladder.)

### Solo users
No partner means no partner multiplier — **unless the solo user is subscribed**,
in which case they get the **phantom 1.5×**: their payout is treated as if a
partner had completed all four tasks. Subscribed-solo only; an unsubscribed solo
user earns flat base with no multiplier.

(This supersedes the monetisation doc's "solo user: base rate, no partner
multiplier" — the multiplier DOES fire solo, but only when subscribed. Logged
clearance #1.)

### Streak bonus
- **+1% per streak day, perpetual, stacking.** A 50-day streak = +50% earn; a
  100-day streak = +100% (double).
- **TWO-SUB-ONLY.** This bonus is inert unless BOTH partners in the pair are
  subscribed. Free pairs and one-sub pairs never receive it.
- **Streak rule (AMENDED 2026-06-11 — supersedes the green-only rule below):**
  the streak **advances (+1) on any day with ≥1 completed task**, **holds** on
  purple (rest days), and **breaks only on zero-task days** — whether that day
  sealed grey (opened, did nothing) or was never opened at all (a no-show gap).
  Rationale: no paternalising in-app; the bare minimum keeps the chain alive.
- **Gap handling (AMENDED s32 — span-sealing, tile 4.25):** the morning claim
  adjudicates EVERY day between the most recent sealed day and the claim date,
  oldest first, each ending terminal in walk order — never-opened days seal
  grey (`accounted_for` stays 0, no payout), accounted days seal at their
  natural tier. Streak math runs per-day in that same order: a grey seal
  zeroes the streak AT the day it happened, and later days in the walk build
  from the post-break value. The streak BONUS on the payout day reads the
  streak as the walk delivers it — you are never paid a bonus across a break.
  History becomes **contiguous**: every day between install and the last seal
  ends terminal, so the streak the user sees is always already adjudicated.
  - SUPERSEDED s30/s31 mechanism (recorded for history): single-target
    walkback + date-adjacency check against the most recent previously-sealed
    day. Correct about WHETHER a gap broke, wrong about WHEN the user learned
    it — a streak could advance past a hole on day N and retroactively
    collapse on day N+1, because holes were only checked behind the payout
    target. Span-sealing kills the wart.
  - SUPERSEDED original rule (session 27, recorded for history): advance on
    green (4-task) only, hold on purple, reset on anything else. Also note: as
    originally IMPLEMENTED, no-show gaps never broke streaks at all (unsealed
    days never ran the streak math) — a hole found and closed with the first
    2026-06-11 amendment.
- Reads the **user's own** streak (per-user), not the partner's.

### Streak rescue (the morning escape hatch — ADDED 2026-06-11, AMENDED s32)
When the streak (≥1) is about to break and **exactly one purchasable day would
save it**, the morning claim pauses before sealing anything and offers ONE
chance to buy that day as a rest day. Locked decisions (Morgan, 2026-06-11):
- **Eligible breakers:** zero-task days only — sealed-grey or never-opened
  alike. Partial days never need rescue under the amended rule (they maintain
  the streak themselves, which is the point).
- **First-contact anchoring (s32):** the offer fires DURING the span walk, AT
  the zero-task day, anchored to that day's own seal — decline seals it grey,
  accept converts/creates it as rest (purple). It no longer matters whether a
  payout target follows: skip Tuesday, open Wednesday, and Tuesday is
  adjudicated — offer included — Wednesday morning, possibly on a claim that
  pays nothing. (Under the superseded s30 mechanism the offer waited for the
  next payout target, which is exactly what produced the retroactive
  collapse.)
- **Exactly one, by pre-scan (s32 — generalises "multi-day gaps are dead"):**
  before adjudicating, the walk counts the zero-task days in the whole span.
  Exactly one AND streak ≥1 → offer fires at that day. Two or more —
  consecutive gap days, or a gap plus a grey day anywhere in the span — and
  the whole span seals silently with the break: no offer, no chained offers,
  no two-purchase saves. One purchase saves the streak or nothing is for
  sale. False hope stays dead.
- **Price: RESCUE_COST = REST_COST = 50,000.** Same as designation — "not
  gamifying that hard." Shipped as its own server constant inside the economy
  payload (`rescue_cost`) so a future premium is a one-constant change.
- **No frequency cap.** Price is the only guard; users may rescue as often as
  they can afford. Accepted with eyes open — see OPEN TENSION below.
- **One-shot per breaker, enforced by the seal itself:** declining seals the
  zero-task day grey; accepting converts/creates it as rest and seals purple.
  While the offer is pending, NOTHING in the span seals — resolve re-walks
  from current state and seals everything in one atomic batch. Both paths end
  with the breaker sealed, so there is nothing to re-offer and no state to
  track. An unresolved offer (app died mid-dialog) leaves the span unsealed
  and the next open re-offers — self-healing, not a loophole.
Mechanism (two-phase claim + `POST /users/:id/claim/resolve`, span walk shape)
is specified in `four_tasks_morning_sequence_design_notes.md` (STREAK RESCUE
GATE + WALKBACK SPAN-SEALING).

---

## Subscription tiers (economy view)

The subscription PRODUCT framing (what it is, why subscription, price,
founders, trial) is unchanged and lives in the monetisation doc. What each tier
delivers to the ECONOMY:

**Tier 1 — one subscription in the pair (either partner):**
- **Library cross-pollination only.** Each user may use stickers the other
  owns. The nudge is "use your partner's favourite, then buy your own copy."
- **Zero economy effect.** No multiplier change, no streak bonus. A one-sub
  pair earns *identically* to a free pair. The first sub is purely relational.

**Tier 2 — both partners subscribed:**
- **Unlocks all economy advantage.** Two things switch on:
  1. **Streak bonus** (+1%/day perpetual, per the rule above).
  2. **+50% on the partner bonus** — the partner multiplier is multiplied by
     1.5, so the 1.5× max becomes **2.25×**. (e.g. partner-did-4: 1.5 × 1.5 =
     2.25×. Partner-did-2: 1.3 × 1.5 = 1.95×.)

**Solo subscribed:** can only ever reach "one sub" (no partner for a second), so
the tier model can't deliver the economy unlock the normal way. Instead, a
subscribed solo user gets the **phantom 1.5×** (above) as their economy benefit.
No streak bonus (can't reach two-sub). Library cross-pollination is nominal solo
(no partner to share with). We do NOT facilitate a solo user reaching two-sub —
if they want the full economy, they pair up.

### Why this split
Concentrating the earn-rate advantage at two-sub keeps the *entry* subscription
(tier 1) purely social and non-competitive — sharing libraries with a partner,
nothing mechanical. The accelerator is an explicit opt-in for committed pairs.

**Stated brand shift (not hidden):** this is a move away from the monetisation
doc's "subscription is a tip, not a class divide" framing. A perpetual stacking
earn multiplier gated behind the top payment tier IS a mechanical class divide.
The defence: it is earn-RATE, not content or ownership. Free players keep
permanent ownership of everything they buy; the library is shared at tier 1; no
one is locked out of the actual app or its catalogue. The advantage is speed of
grind for the most-committed paying cohort. This is a deliberate, documented
shift, weighed against the conviction line and accepted.

---

## Sinks

- **Rest day: 50,000 coins.** The anchor sink.
- **Streak rescue: 50,000 coins** (`rescue_cost`, == rest cost for v1.0). The
  coin sink that lives INSIDE the open-app flow — welcomed not resented because
  it protects the growing streak. See the Streak rescue section above.
- **Stickers/packs: 20,000–100,000 coins**, by the volume of UI/theme work that
  ships alongside each (a sticker that ships a full theme + chrome costs near
  100k; a minimal one near 20k). Single ramp, taste-driven, not tier-bucketed.
  (Supersedes the monetisation doc's 300–600 curve — that was an early sketch,
  off by two orders of magnitude. Logged clearance #3.)
- **First pack FREE:** the onboarding sticker is unlocked in full as the
  starter. Unchanged from the monetisation doc.
- Packs are bought and owned **individually** (per the per-user model).
  Spending your coins buys *you* the pack; your partner using it is the library
  access mechanic, not co-ownership. (Supersedes the monetisation doc's
  pair-wide pack list / shared purchase history / "one user spending is the pair
  spending." Logged clearance #4.)

---

## Cadence check (the locked anchor)

The monetisation doc anchors the economy on rest-day cadence. Checking the
locked numbers:

- **Free pair, perfect play:** ~4600 base × 1.5 (partner did 4) = ~6900/day.
  50,000 / 6900 ≈ **7.2 days per rest day.** Holds the doc's ~7-day anchor.
- **One-sub pair:** identical earn (tier 1 is economy-neutral) → **same ~7.2
  days.**
- **Two-sub pair:** ~4600 × 2.25 = ~10,350/day at base, climbing with streak.
  Materially faster, by design.

**The monetisation doc's cadence TABLE is DEAD** (free 7 / one-sub 6 / two-sub 5
days). Under this model one-sub == free; only two-sub accelerates. Superseded.

---

## Shipped vs target (claim handler)

HISTORICAL NOTE: the naive-handler gap this section described closed in session
28 — the claim handler applies the full model below (partner multiplier,
two-sub boost, streak bonus, immutable bake-in), device-verified. The
2026-06-11 amendment (streak rule + rescue gate) ships on top of it. The order
of application at claim:

```
base      = sum of per-task 900–1400 (rest = 0)
mult      = partner-completion multiplier (1.2–1.5), ×1.5 if two-sub
            (solo: 1.0, or 1.5 if subscribed-solo)
streakAmt = (two-sub only) base*mult * (streak_days * 0.01)
            — streak_days is the PRE-payout streak as the span walk delivers
            it (any break earlier in the walk has already zeroed it; no bonus
            across a broken streak)
payout    = round(base * mult) + streakAmt
```

Subscription state is read per-user at claim time (both users' `subscription_
active`), per the monetisation doc's pair-pricing resolution. The handler reads
the partner row already (for pairing); it gains a partner-completion read and a
two-subscription check.

**SPAN-SEALING STATUS (s32):** the walkback span-sealing amendment
(first-contact adjudication, pre-scan rescue, contiguous history) is
DOC-LOCKED, NOT YET SHIPPED. The live handler still runs the s30 single-target
walkback + adjacency check. Tile 4.25 brings the handler to this doc; until it
lands, this doc describes the target and the wire behaves per the superseded
s30 mechanism noted in the streak section.

NOTE: a sealed day's recorded payout is immutable — it never recomputes if a
subscription later lapses. Coins already paid stay paid. (Monetisation doc,
immutable-past principle.)

---

## OPEN TENSION (flagged, not resolved)

**Perpetual streak bonus vs fixed-price sink.** +1%/day never levels off, while
the rest-day sink is fixed at 50k. For a long enough unbroken streak, a two-sub
pair's earn outpaces the sink without bound, and coins eventually stop being
scarce. This is the idle-clicker compounding the monetisation doc explicitly
names as a NON-GOAL (line ~304).

It is deliberately LEFT IN. The blast-radius picture, updated 2026-06-11:
1. **Two-sub-gated** — only the most-committed paying cohort, a small fraction
   of the population, ever sees it. It does not define the currency's meaning
   for free/one-sub users.
2. **Self-limiting via the streak rule — MATERIALLY WEAKENED 2026-06-11.** The
   original containment relied on harsh resets (any non-perfect day zeroed the
   bonus). Under the amended rule (≥1 task maintains) plus the uncapped streak
   rescue, an engaged two-sub user's streak is near-unbreakable: only an
   unrescued zero-task day ends it, and zero-task days can be bought back at
   50k indefinitely. Consequence, stated honestly: somewhere past streak ~300
   the +1%/day compounding makes every current sink (rescue, rest, stickers at
   20k–100k) trivial for that cohort, and coins stop being scarce for them.
   ACCEPTED (Morgan, 2026-06-11, twice): "even at a 400-day streak it's still
   not something you'd want to happen to you, and if someone stays subbed for a
   year they've paid for the app." The advantage remains earn-RATE only; no
   content or ownership is gated. (Span-sealing does not move this tension
   either way — the pre-scan makes multi-day absences strictly unrescuable,
   same as before; single misses remain purchasable without cap.)
3. **Marketing-positive** — free and long-streak players are loud about their
   progress; that visible progression is word-of-mouth advertising.

If it bites at scale, the cap option remains the dormant lever: +1%/day up to a
ceiling (e.g. +30% at 30 days, then flat). Not applied now. Revisit with real
usage data — and note the rescue's purchase telemetry will show exactly which
streak lengths are buying their way past resets.

---

## OPEN / LIVE TUNING (not locked)

- **Sub-bonus exact rate.** The +50%-partner-bonus (2.25× cap) and the streak
  unlock are the tier-2 deliverables as locked. An earlier floated alternative
  was +25%/sub on a flat ladder; rejected in favour of the current split, since
  the second sub's real value is now the streak unlock, not a multiplier bump.
  Leaving as locked unless balance testing says otherwise.
- **Per-task range obfuscation.** 900–1400 carries from the prototype. Whether
  to keep the exact range or retune is a balance-pass question; the structure
  (random per-task, summed) is locked.

---

## Contradiction-clearances logged (vs monetisation doc)

These were the don't-apply-blind items; cleared this session:
1. **Solo phantom 1.5×** — supersedes "solo user: no partner multiplier." Fires
   solo only when subscribed.
2. **Economy gated behind two-sub** (streak + 50% partner bonus) — reframes
   "tip, not class divide"; documented as an accepted brand shift.
3. **Sticker pricing 20k–100k** — supersedes the 300–600 curve.
4. **Per-user coins / individual ownership** — supersedes pair-shared coins,
   pair-wide pack list, and co-owned packs. (Already true in the shipped schema;
   the monetisation doc's prose was stale.)
