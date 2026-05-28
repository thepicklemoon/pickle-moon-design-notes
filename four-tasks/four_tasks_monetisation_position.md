# Four Tasks — Monetisation Position

**Last updated:** 14 May 2026 (session 7, mobile — fortnightly cadence locked, schema updated to JSON column, tier references cleaned up to align with feature-catalogue theme model)
**Status:** v2.0 — subscription mechanic fundamentally reframed.
Brand position, pricing, founders model, store mechanics carry
forward from v1.1 (session 1). The "what does subscription buy"
answer is superseded. Session 7 updates: fortnightly alternating
drop cadence locked, schema reframed to a single JSON column for
per-day theme state, tier language dropped to align with theme
doc's feature-catalogue model. (That schema shipped in the v1.0
wholesale rewrite, session 12 — see Schema implication below.)

**Source:**
  - v1.1 — session 1 decision conversation (subscription product
    positioning, $0.99/wk pricing, founders model).
  - v2.0 — session 6 mobile design session (bilateral library
    sharing mechanic, immutable past, founders pricing structure).

Cross-references:
  - four_tasks_theme_design_notes.md (feature-catalogue slot model,
    pack architecture)
  - four_tasks_staggered_disclosure_design_notes.md (day-21 subscription beat)
  - four_tasks_pair_key_design_notes.md (pair model)
  - four_tasks_morning_sequence_design_notes.md (coin payout mechanics)
  - four_tasks_onboarding_design_notes.md (solo mode, partner panel)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHAT CHANGED FROM v1.1
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

v1.1 (session 1) assumed: subscription unlocks ongoing content
drops. Free users keep launch content forever, subscribers get new
packs as they're released. Content is the paywall.

v2.0 (session 6) supersedes: subscription unlocks BILATERAL LIBRARY
SHARING within a pair. ALL packs are bought with coins by all
users — free, solo, paired. Coins are universal currency, not the
paywall. Subscription gives you (a) access to your partner's
library while subbed, and (b) a small stacking coin bonus. Past
days are immutable — calendar history preserves the themes that
were active on each day, even if those themes came from a partner's
library during a now-lapsed subscription period.

The brand position, anti-patterns list, founders model, and store
mechanics survive intact. The mechanic of WHAT subscription delivers
is what changed.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE SHAPE (carries from v1.1)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Four Tasks is a subscription product. The subscription pays for
continued investment in a living space — ongoing hand-painted asset
additions (packs, themes, stamp variants) that keep the app
expanding for as long as the studio exists. This isn't "feature
access fee." It's "the maker is still actively making, and your
subscription keeps that real."

This positions Four Tasks alongside products like Finch, Animal
Crossing, or Cookie Clicker — products where the value is the
ongoing breath of the world, not a fixed feature set. Closest
peer-set is NOT productivity habit trackers like Streaks or Habitica.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHY SUBSCRIPTION SPECIFICALLY (carries from v1.1)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- THE PICKLE MOON commits to ongoing content drops (packs, themes,
  stamp variants). That commitment is real ongoing service.
- The hand-painted, considered nature of the assets means each
  addition is genuine work — not algorithmic content generation.
- The two-person partner model means the app sits in someone's
  daily life for months/years. A one-time purchase under-captures
  the value to the user; ads/utility paywalls fight against the
  calm, considered tone.
- Subscription revenue at scale (~5000 users) is "quit BHP" money
  in WA, which is a real life goal worth designing toward.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE SUBSCRIPTION MECHANIC (NEW IN v2.0)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Subscription delivers TWO things:

  1. BILATERAL LIBRARY ACCESS within a pair. If either user in a
     pair is currently subscribed, BOTH users gain access to the
     other user's owned packs for live use (calendar leader,
     palette source, completion stickers). The non-subscribing
     partner is not disadvantaged — they receive the same pair-
     wide unlock from their partner's subscription.

  2. STACKING COIN BONUS per active subscription in the pair. A
     small, ambient earn-rate boost. One sub in the pair = base
     bonus. Two subs = double bonus. Designed to be a tip, not a
     class divide.

The library access is PAIR-LEVEL (binary, on/off based on whether
anyone in the pair is subscribed). The coin bonus is PER-USER and
stacks (linear with subscription count).

A pair where both subscribe gets: pair-wide library access + double
coin bonus. The strictly best version.
A pair where one subscribes gets: pair-wide library access + single
coin bonus. The "we both benefit, one of us pays" version.
A pair where neither subscribes gets: own libraries only + no
bonus. The baseline.

WHY THIS MODEL:
  - Permanent ownership of bought content. Unsubscribing never
    strips a user of what they grinded for. The buddy-ware tone
    dies if the app feels stingy.
  - Subscription is "time-windowed richness," not "content
    ownership." During subscription, your pair's combined
    library is the unit. After lapse, you revert to your own
    library only.
  - Marketing writes itself in the buddy-ware angle: "subscribe
    to share your themes with your partner."
  - Two-subscriber pairs are materially better than one-
    subscriber pairs (double coin bonus), giving both partners
    a reason to sub independently, but the library mechanic
    means there's no FOMO if only one subs.
  - Pack catalogue stays accessible at coin prices. The
    economy serves the relationship, not the revenue extraction.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
IMMUTABLE PAST — CALENDAR HISTORY PRESERVES THEME STATE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Past days preserve the themes that were active on each day, even
after the user unsubscribes and loses live access to those themes.

If a subscriber spent three months using their partner's wizard
pack as calendar_leader and chilli pack as palette_source, those
three months of calendar history continue to render with wizard
leadership and chilli palette FOREVER, regardless of subscription
state.

The user's LIVE app (current and future days) reverts to their own
library on unsubscribe. If their current calendar_leader was a
partner-owned pack, it reverts to a sensible default (probably their
onboarding-free pack). They are gently notified at lapse:
  "Your subscription ended. Your themes have reverted to your own
   library. Resubscribe to access [Partner]'s themes again."

WHY IMMUTABLE PAST:
  - Honest. The historical record is what actually happened.
  - Psychologically generous — never punished for past
    subscription. The past stays rich forever.
  - Mechanically retentive — looking back at past richness while
    seeing your present revert to your own library is the upsell
    written by the user's own behaviour. No nag screen needed.
  - Reinforces the "subscription is time-windowed richness"
    framing. You're buying a period of richer life, not buying
    content.

SCHEMA IMPLICATION:
  `days` table gains a single JSON column:
    day_theme_state  TEXT  -- JSON map of slot → sticker ID
                              captured at day-seal time

  Written at day-seal time (per the sealing logic in the timezone
  design doc). Never modified after. Read by the historical renderer.

  This shipped as `days.day_theme_state` in the v1.0 schema
  (session 12 wholesale rewrite — not an incremental migration).

  Schema NOTE (session 7 update): the earlier sketch of two
  separate columns (day_leader + day_palette) was superseded by
  the feature-catalogue model in the theme doc rewrite (session
  7). The slot system has 10+ named slots, not 2. A single JSON
  column captures the full slot state at seal time and survives
  future slot additions without further schema migrations. See
  four_tasks_theme_design_notes.md "Schema implications" for the
  full slot map shape.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
COIN ECONOMY TUNING (NEW IN v2.0)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Numbers in this section are PLACEHOLDERS pending a full economy
balance pass against the actual prototype rates. The STRUCTURE is
locked; the exact figures are not.

REST DAY CADENCE (anchor metric):
  Free pair, perfect play (both 4T daily): 1 rest day per ~7 days.
  One-sub pair, perfect play:               1 rest day per ~6 days.
  Two-sub pair, perfect play:               1 rest day per ~5 days.
  Solo user (subbed or not):                base rate, no partner multiplier.

This means a free pair playing optimally gets roughly 4 rest days
per month. Practical play (mix of 4T days, partial days, one-
partner-only days) will land closer to 1 rest day per 9-10 days
for free pairs — tight but not punishing.

PARTNER-COMPLETION MULTIPLIER:
  Base 1.5x stays. Untouched by subscription. The pair-completion
  bonus is about playing together, not paying. Don't conflate the
  two systems.

SUBSCRIPTION COIN BONUS:
  Small per-user stacking bonus. Target: +15-20% over base
  earn rate per active subscription. Two subs in a pair = ~30-40%
  total bonus.
  Tuned so that a one-week subscription cannot earn enough coins
  to buy the entire launch pack catalogue. The bonus is a tip,
  not an accelerator.

PACK PRICING CURVE:
  First pack: FREE. Specifically, the pack the user picked at
  onboarding is unlocked in full (whichever feature elements that
  sticker ships — leader + palette + whatever theme files exist
  in its folder) as their starter. Free users live the full theme
  system at the depth their starter sticker offers from day 1.
  The upgrade path to subscription is concrete, not imagined.

  Subsequent packs: single cost curve, ramps to a cap.
    Pack 2:   ~300 coins
    Pack 3:   ~400 coins
    Pack 4:   ~500 coins
    Pack 5+:  ~600 coins (cap)

  Pricing NOTE (session 7 update): the earlier two-tier pricing
  (TIER 1-3 at ~300-600, TIER 0 at ~1500) was superseded by the
  feature-catalogue model. Stickers no longer fall into tier
  buckets — each ships whatever subset of features makes sense
  for its identity. Pricing is a single ramp, identical across
  all stickers regardless of how feature-rich any individual
  sticker is. "Value" is what the user perceives, not what the
  pack technically contains. Some 600-coin packs are minimal,
  some are maximal; both are valid art. Users choose based on
  taste, not on content quantity.

PACK COUNT IS PAIR-WIDE:
  The ramp counter is shared across the pair. Both users see the
  same pack list, the same cost curve, the same purchase history.
  Coins remain pair-shared per the prototype. One user spending
  coins is the pair spending coins.

  Buying a pack means: "this pack is now owned by us as a pair,
  available in our combined library to be used by either user
  whose subscription state allows it" — own library always
  available, partner's library only while subscribed.

  In a solo user (no partner), "the pair" is just them. They
  own everything they buy, accessible to them always.

PACK DROP CADENCE — FORTNIGHTLY ALTERNATING (LOCKED SESSION 7):
  The studio commits to a FORTNIGHTLY drop cadence post-launch,
  alternating between subscriber-only and free-user-purchasable
  packs:

    Week 1: Subscriber-only drop. Available for purchase ONLY by
            users whose pair has at least one active subscription.
            Free users see it in the picker (locked) but cannot
            spend coins on it.
    Week 3: Free-user-thank-you drop. Available for purchase by
            EVERYONE, free or subscribed. The studio's thank-you
            for free users continuing to use and recommend the
            app.
    Week 5: Subscriber-only drop. (Cycle restarts.)
    Week 7: Free-user-thank-you drop.
    ...and so on.

  Net effect over any given month: one drop for free users, one
  drop for subscribers. Free users never feel locked out for long;
  subscribers get faster catalogue growth and access to the
  full subscriber drops.

  The "thank-you drop" may be review-gated unlock — captured in
  theme doc Open work and to be designed if the mechanism is
  pursued. Defer to v1.x consideration.

  WHY FORTNIGHTLY ALTERNATING:
    - Monthly would be too slow — subscriber retention promise
      weakens if catalogue grows by only one item a month.
    - Weekly would be unsustainable for a solo painter.
    - Fortnightly is the painting-budget sweet spot: 2 packs per
      month, one of each type, allows a buffer queue and per-pack
      effort variance (one minimal pack per minimal-paint week,
      one maximal pack per maximal-paint week).
    - Alternating subscriber/free creates a rhythm where every
      user has something to look forward to every fortnight,
      regardless of subscription state.
    - Subscriber drops are SUBSCRIBER-ONLY PURCHASABLE during
      their drop window. The cohort of users who can spend coins
      on them grows over time as more users subscribe; eventually
      some free users will subscribe specifically to unlock
      retrospective access to past subscriber drops they want.

  STUDIO CONTENT BUFFER:
    Paint 3-4 packs ahead at any time. Buffer absorbs missed
    fortnights from life events, FIFO swings, family priorities.
    Per-pack effort variance (minimal vs maximal) makes the
    buffer easier to maintain than a fixed "every pack must hit
    full-theme quality" model would have allowed.

  QUEUE FAILURE MODE:
    If the buffer ever runs to zero and a fortnight is missed,
    the honest move is to communicate. A skipped drop is not
    fatal as long as users see the studio is real and the next
    drop is coming. Predator-pattern subscription apps run on
    fake scarcity; an honest miss is brand-aligned.

NON-GOAL: PER-STICKER COIN BONUS STACKING.
  The Tower / idle-clicker style "every collectible adds a tiny
  earn-rate bonus that compounds over time" mechanic was considered
  and rejected. Four Tasks is not about the economy — it's about
  doing four tasks a day with your partner. The economy is
  connective tissue between play and reward. Exponential earn-rate
  growth would dissolve the rest-day cadence we tuned, force
  invented late-game money sinks, and turn the app into "play to
  earn to play more" logic. The collector instinct, if surfaced,
  should be rewarded with non-economic cosmetics (badges, ambient
  flourishes) — not with earn-rate stacking. Defer collector-
  cosmetics to v1.x post-launch.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PRICE STRUCTURE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

FOUNDERS PRICING TIER (NEW IN v2.0):

  Launch users get a permanent founders-rate discount lasting at
  least one calendar year (or until 31 December of year 2, exact
  cut-off TBD). During this period, the price floor is the
  lowest the studio commits to charging.

  After the founders period expires, the price for those users
  rises to standard rate. Users will be reminded ahead of time at
  important beats (1 month, 1 week, 1 day before the increase).
  This is a deliberate "you got in early" reward that ages
  gracefully into standard pricing.

  Founders period applies to anyone subscribed during the launch
  window (date TBD, probably first 6-12 months after store
  submission). Solo users and paired users both eligible.

  Distinct from the FOUNDERS FLAG (below) which is for prototype-
  era users and grants permanent free access — those are
  different mechanisms for different audiences.

BASE PRICING (carries from v1.1):

  Target: $0.99 USD/week or $3.99 USD/month (Apple subscription
  tier 1 or thereabouts — exact tier locked when setting up in
  App Store Connect / Play Console).

  Pricing positioned to be INVISIBLE — cheap enough that someone
  with a partner doesn't think about it, doesn't price-compare,
  doesn't feel ripped off. Decision happens once and then
  disappears.

  This price is deliberately below market for subscription apps
  in the productivity / wellness space (which typically run
  $4.99-$9.99/month). The lower price is part of the brand
  promise — patient and respectful of the user's money. Reaches
  further, signals values, builds long-term trust.

TRIAL STRUCTURE (carries from v1.1):

  Two stacked softeners — deliberately more generous than industry
  standard.

  - Days 1-30:    Free trial. Full app access. No charge.
  - Days 31-60:   Half price (~$0.49/week or ~$1.99/month).
  - Day 61+:      Full price (or founders rate if within founders
                  window).

  Why two stacked softeners: the mobile subscription market is
  saturated with predatory patterns — auto-billing after 3-day
  trials, hidden cancellation flows, "limited time" pressure
  tactics. Four Tasks deliberately positions against this. A user
  paying their first full-price month has had TWO months of
  below-full-price exposure to the product. By that point they
  know whether the app is for them. The decision to keep paying
  is informed and fair.

  This is also a marketing position — the patience of the trial
  is itself a story. "30-day free trial, half-price first paid
  month" reads as confidence in the product and respect for the
  user.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PAIR PRICING — RESOLVED IN v2.0
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

v1.1 deferred this question. v2.0 resolves it:

EACH USER SUBSCRIBES INDEPENDENTLY through the store of their own
device. The store (Apple/Google) charges that individual user. No
shared billing, no family-account complexity, no "split the bill"
UI.

The MECHANIC of pair-wide library access (one sub in the pair
unlocks library access for both) effectively delivers the "either
user pays, both get access" option from v1.1's list — but achieved
via the architecture, not via shared billing.

Implementation:
  - Server tracks `subscription_active` boolean per USER (not per
    pair). Sourced from Apple/Google store notifications.
  - Pair-level features (library access, coin bonus) read both
    users' subscription state at query time.
  - Library access = ON if either user.subscription_active = true.
  - Coin bonus = sum of active subscriptions in the pair.

Edge cases resolved:
  - Pair breakup / partner changes: subscription state moves with
    each user individually. A user's subscription persists across
    pair-key rotations (per the pair-key doc, the user identity —
    user_id — stays; only the pair-key hash changes).
  - One user subs, other doesn't: standard expected case. Library
    access is bilateral. Coin bonus accrues only to the subscriber.
  - User leaves pair entirely (rare; defer to v1.x): subscription
    follows the user to their new pair, if any. Old partner loses
    library-access benefit at that moment.

LIBRARY ACCESS SIGNALLING:
  The library mechanic is mostly silent. Subscribers see the
  expanded catalogue available for use in the picker. Free users
  see their own library only.

  When a subscriber's lapse cuts off partner-library access mid-
  use (their current calendar_leader was partner-owned, now they
  can't use it), the lapse notification is honest:
    "Your subscription ended. Your themes have reverted to your
     own library. Resubscribe to access [Partner]'s themes again."

  The partner is NOT notified when their library access changes
  (i.e., when the other user subs or unsubs). Library access
  state is implicit — they discover it by trying to use a partner-
  owned pack and either succeeding (subscribed somewhere in pair)
  or being prompted to subscribe.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SOLO USER SUBSCRIPTION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Solo users (per the onboarding doc) can subscribe. Their value
proposition:

  - Small coin bonus (same +15-20% over base rate as paired
    subscribers' per-user bonus).
  - No library access benefit — there's no partner library to
    share with.
  - Same founders rate eligibility if within the founders window.
  - When they later pair with a real partner, their subscription
    automatically activates the bilateral library access for the
    new pair. Seamless transition.

Honest value prop to solo users: "subscribe to grind for packs
faster, and to lock in founders pricing while you wait for a
partner." Not "subscribe to unlock content" — there's no content
gated from them. Just a small grind accelerator and a price lock.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EXISTING USERS — FOUNDERS FLAG (carries from v1.1)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Distinct from the founders pricing tier above. The founders flag
is for prototype-era users — mum, Ayshi, friends — and grants
PERMANENT free access.

Implementation: a `founders` flag on user records, set server-side
at native app first-boot for users whose pair_key matches the
existing prototype-era roster. Founders skip the paywall entirely,
see no subscription UI, get all current and future content free
for life.

Founders behaviour under the v2.0 mechanic:
  - Treated as if perpetually subscribed for library and coin
    purposes.
  - Their pair gets bilateral library access permanently.
  - They get the stacking coin bonus permanently.
  - If a founder is paired with a non-founder, the bilateral
    library access applies to both — the non-founder benefits
    from the founder's perpetual "subscription."

Why this model rather than redemption codes:
  - Redemption codes expire if not used. Founders should never
    have a "redeem before X date" anxiety.
  - The relationship between Morgan and these users is permanent
    and personal, not transactional.
  - Baking it into the data model signals something to future
    users: "the people who built this with us are still here."
    That's social proof that can't be bought.
  - Costs nothing — these are friends and family, not a
    marketing-funnel cost.

Cap on founders: roughly 20-30 users from the prototype era.
Hardcoded list, not opt-in.

Founders cutoff date (OPEN — carries from v1.1): the EXACT date
that closes the founders flag list needs sharp definition.
Candidates: at native app first deployment, at v1.0 submission to
stores, at Phase 5 gate. Probably "at native app first
deployment to TestFlight/internal Play track." Lock this when
Phase 5 lands.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHAT THIS MEANS FOR WHAT WE BUILD
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  - IN-APP PURCHASE INFRASTRUCTURE (subscription only, no IAP
    packs). Apple's StoreKit 2, Google Play Billing. RevenueCat
    recommended as the abstraction layer (handles cross-platform
    state, receipts, restore flows; one SDK call instead of two).
    Server-side validation via Cloudflare Worker — RevenueCat
    notifies the Worker of subscription state changes, the Worker
    updates the user's subscription_active field. The Godot client
    never decides subscription state — it asks the server.

  - PAYWALL UI. Surfaces during onboarding (after the first month
    free expires), and at the day-21 staggered disclosure beat (a
    value summary, not a nag — see Phase 5 work).
    Designed to feel calm, not pressuring.

  - RESTORE-PURCHASES FLOW. User reinstalls app or switches
    devices → app asks Apple/Google "is this Apple/Google ID
    subscribed to Four Tasks?" → server-side state updates.
    Real feature with real edge cases.

  - LIBRARY-ACCESS RENDERER. When a user is currently entitled to
    partner-library access, the sticker picker, calendar leader
    selection, and palette source selection all show the combined
    pair catalogue. When the entitlement lapses, the renderer
    filters to the user's own library only. Live theme reverts as
    needed.

  - HISTORICAL THEME RENDERER. Past days read
    `days.day_theme_state` (JSON slot map) and render with the
    full slot configuration captured at seal time, regardless of
    current subscription state.

  - COIN BONUS APPLICATION. The morning sequence claim endpoint
    reads both users' subscription state, applies the appropriate
    bonus to the day's payout. Stacking math is server-side.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PRIVACY POLICY v3 IMPLICATIONS (carries from v1.1)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

When tile 0.10 lands, the policy gets these additions:

  - SUBSCRIPTION HANDLING. Apple StoreKit / Google Play Billing
    handle the actual transaction (potentially mediated via
    RevenueCat). THE PICKLE MOON doesn't see payment information
    (card numbers, billing addresses). We see only "user X has an
    active subscription" as a boolean flag and a renewal date.

  - WHAT WE STORE ABOUT SUBSCRIPTIONS. The subscription_active
    flag, renewal date, originating store (Apple/Google),
    founders flag if applicable. We don't store transaction IDs,
    prices paid, or refund history.

  - SUBSCRIPTION CANCELLATION. Users cancel through Apple/Google,
    not through us. We can't cancel for them. The privacy policy
    points to the standard cancellation flows.

  - REFUNDS. Handled by Apple/Google directly. We don't process
    refunds.

  - RESTORE PURCHASES. Standard mention.

  - V2.0 ADDITION: Partner library access. The policy explains
    that subscription state is used to determine whether a user
    can use their partner's purchased themes. Partner is not told
    of subscription state directly — they discover library access
    via use. No personal data is shared between partners beyond
    what's already visible in the app surface.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHAT'S NOT IN THE MODEL (carries from v1.1)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

For clarity, things explicitly chosen against:

  - NO ADVERTISING. Ever. Aesthetically incompatible with the
    product, and would force a reversal of the privacy policy's
    "no third parties" claim.

  - NO IAP "PREMIUM PACK" MODEL. All packs are bought with
    in-game coins by all users. No "buy this pack for $1.99."
    Cleaner. Subscription doesn't buy content; coins do.

  - NO UTILITY PAYWALL. Free-tier users keep using the app at
    full functionality. Subscription gates LIBRARY ACCESS
    (using partner's packs) and provides a small coin bonus.
    Core mechanics (tasks, MOTD, calendar, partner panel, own
    library) all stay free.

  - NO PRICE HIKE FOR EXISTING SUBSCRIBERS WITHOUT GRANDFATHERING.
    If the price ever goes up, existing subscribers keep their
    original rate. This is a permanent commitment of the studio.
    Founders pricing tier is the explicit early-adopter benefit.

  - NO PREDATORY TRIAL CONVERSION. No 3-day trials, no auto-charge
    surprises, no "limited time" pressure.

  - NO DATA SALE. Already covered in privacy policy; mentioning
    here for completeness.

  - NO PER-STICKER COIN BONUS STACKING. Considered and rejected
    in session 6 — see coin economy section above.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MARKETING SHADOW — GENEROUS EARLY-USER ACCESS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The buddy-ware viral mechanic depends on early users recruiting
friends. To maximise this, consider offering select early users
extended trial or free-access periods beyond the standard 30-day
trial. Candidates:

  - First N users to install after store launch get 6 months
    free trial.
  - First N pairs to fully pair-up (both users on real
    subscriptions or trial) get a permanent founders rate.
  - Specific subculture seeding cohorts (per the marketing notes
    doc) get bespoke trial extensions tied to community-specific
    promo codes.

Mechanism: a `trial_extension` field on users, server-set at
specific cohort gates. Adds N days to the standard trial. Honors
permanent-ownership semantics — anything bought with coins during
the extended trial is kept on lapse.

This is marketing optimisation, not core monetisation. Specifics
land with the marketing notes doc. Captured here so the
subscription system architecture supports flexible trial periods
without retrofit.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE CONVICTION LINE (carries from v1.1)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Morgan's stated position:

  "nothing turns me off faster than the greed i see in the mobile
   app market. i'm positive a majority of people can read between
   the lines and seeing something legitimately patient and
   respecting of their money is where i want to exist."

This is the brand promise. Every monetisation decision should be
checked against this line. If a future decision pulls in the
direction of "extract more revenue per user," it gets weighed
against the patience of the original promise. The studio's revenue
ceiling is partly self-imposed by this commitment, and that's
intentional.

The v2.0 reframe is consistent with this position. Subscription
delivers RELATIONAL value (shared library) and a SMALL accelerator
(coin bonus). Content is universal. Ownership is permanent. The
past is preserved. Marketing relies on the mechanic surfacing its
own value, not on nag screens.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE 5000-USER GOAL (carries from v1.1)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

5000 paying subscribers at ~$4/month gross = ~$20k/month gross.
After Apple/Google take their 15% (subscription year-2 rate;
first year is 30%), and after expected churn, sustainable net is
in the ~$11-13k/month range. That's "quit BHP and do this full-
time" income in WA.

Reference points for plausibility:
  - Finch (closest emotional peer) has ~5M users.
  - Streaks ($4.99 one-time) has sold ~1M+ copies.
  - Habitica (freemium) has ~4M users.

5000 is well below those peer-set sizes. The path is plausible.
The unknown is HOW LONG it takes — six months, two years, five
years. THE PICKLE MOON having Four Tasks + Sticky Steve + Beam Orb
means three concurrent revenue streams, which hedges the Four
Tasks timeline.

V2.0 NOTE: the bilateral library sharing model creates a slight
revenue trade-off. Pairs where only one partner subscribes deliver
$4/month per pair instead of $8/month. Pairs where both subscribe
deliver $8/month. The mechanic explicitly encourages both-partners-
sub via the stacking coin bonus, but doesn't force it. Net
expectation: average revenue per pair lands between one-sub and
two-sub rates, biased toward one-sub for new pairs and toward two-
sub for long-term retained pairs. 5000-user goal remains the
target; "user" in that count means roughly "subscriber," and pairs
will average ~1.3-1.5 subscribers each in practice.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OPEN QUESTIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Carried forward + new for v2.0:

  - EXACT COIN ECONOMY TUNING. The rest-day cadence, partner
    multiplier, subscription bonus rate, and pack cost curve are
    all placeholder figures. A full balance pass against the
    actual prototype rates is needed. Lands in Phase 4 once
    coin payouts, MOTDs, and pack purchases are all wired up.

  - FOUNDERS CUTOFF DATE. Carries from v1.1. Lock when Phase 5
    lands. Probably "first deployment to TestFlight / internal
    Play track."

  - FOUNDERS PRICING TIER EXPIRY. New in v2.0. The "until year
    2" framing needs a sharp date. Candidates: 31 December of
    the launch year, 31 December of year 2, anniversary of launch
    date. Probably "31 December of year 2" — gives launch users
    at least one full calendar year plus some buffer.

  - ANNUAL SUBSCRIPTION OPTION. Most apps offer monthly + annual
    at a discount (e.g. $39/year vs $48 if paid monthly). Worth
    offering. Not a v1.0 blocker.

  - FAMILY SHARING SUPPORT. Apple Family Group sharing is opt-in
    by the developer. Worth turning on. Free reach. Note: under
    the v2.0 model, family sharing means multiple users can
    benefit from one Apple ID's subscription — needs server-side
    accounting to handle this case correctly (each user record
    points at the same Apple ID's subscription, all get
    library/coin bonus benefits). Defer to Phase 5.

  - REFUND / ABUSE HANDLING. Edge cases like users sharing one
    Apple ID across many devices to dodge the per-device charge.
    Mostly Apple's problem to police, but worth knowing it
    exists.

  - V2.0 SPECIFIC: trial extension cohort mechanism. The
    "marketing shadow" section above sketches this; specific
    implementation lands with the marketing notes doc and Phase 5
    work.

  - V2.0 SPECIFIC: lapse-during-active-theme UX detail. When a
    subscriber's lapse forces a revert from partner-owned theme,
    is the revert instant (next app boot) or graceful (notify
    user, give 24 hours)? Probably instant + notification, but
    worth a UX review before Phase 5.

  - V2.0 SPECIFIC: library access entitlement caching on the
    client. Client should not need to re-query subscription state
    every render. Cache duration + invalidation strategy (e.g.
    cached for 1 hour, invalidated on app foreground or on
    explicit subscription state change). Defer to implementation.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED DOCS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  - four_tasks_theme_design_notes.md — pack architecture, feature-
    catalogue slot model (session 7 rewrite — supersedes the earlier
    TIER 0/1/2/3 framing). Library access mechanism: all
    accessible packs available in the combined library when
    subscribed; only own-purchased packs in own library when not.

  - four_tasks_staggered_disclosure_design_notes.md — day-21
    subscription reveal beat. Copy authoring concern at Phase 5.
    Reveal frames subscription as a value summary against the
    user's actual play history, not as a trial-end nag.

  - four_tasks_pair_key_design_notes.md — pair model. Each
    user has subscription_active independently. Pair-level
    features read both users' state.

  - four_tasks_onboarding_design_notes.md — solo mode users
    can subscribe but get only the coin bonus, no library
    sharing (no partner). Founders rate eligibility applies.

  - four_tasks_morning_sequence_design_notes.md — claim endpoint
    is where the coin bonus stacking is applied to the daily
    payout. Server reads both users' subscription state at claim
    time.

  - four_tasks_marketing_notes.md — generous early-user trial
    extensions live here as a marketing optimisation. Trial
    cohort mechanism captured architecturally in this doc.
