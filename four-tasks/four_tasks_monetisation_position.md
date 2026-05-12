# Four Tasks — Monetisation Position

**Last updated:** 11 May 2026
**Status:** v1.1 — locked direction, some details deferred. Studio name normalised to THE PICKLE MOON.
**Source:** decision conversation in session 1 of the Godot port.

---

## The shape

Four Tasks is a **subscription product**. ~$1 USD/week (or equivalent $4/month).

The subscription pays for **continued investment in a living space** — ongoing hand-painted asset additions (emoji packs, themes, stamp variants, customisation surfaces) that keep the app expanding for as long as the studio exists. This isn't "feature access fee." It's "the maker is still actively making, and your subscription keeps that real."

This positions Four Tasks alongside products like Finch, Animal Crossing, or Cookie Clicker — products where the value is the ongoing breath of the world, not a fixed feature set. Closest peer-set is *not* productivity habit trackers like Streaks or Habitica.

## Why subscription specifically

- THE PICKLE MOON commits to ongoing content drops (emoji packs, themes, stamp variants). That commitment is real ongoing service.
- The hand-painted, considered nature of the assets means each addition is genuine work — not algorithmic content generation.
- The two-person partner model means the app sits in someone's daily life for months/years. A one-time purchase under-captures the value to the user; ads/utility paywalls fight against the calm, considered tone.
- Subscription revenue at scale (~5000 users) is "quit BHP" money in WA, which is a real life goal worth designing toward.

## Price

**Target: $0.99 USD/week or $3.99 USD/month** (Apple subscription tier 1 or thereabouts — exact tier locked when setting up in App Store Connect / Play Console).

Pricing positioned to be **invisible** — cheap enough that someone with a partner doesn't think about it, doesn't price-compare, doesn't feel ripped off. Decision happens once and then disappears.

This price is *deliberately* below market for subscription apps in the productivity / wellness space (which typically run $4.99-$9.99/month). The lower price is part of the brand promise — patient and respectful of the user's money. Reaches further, signals values, builds long-term trust.

## Trial structure

Two stacked softeners — deliberately more generous than industry standard.

- **Days 1-30:** Free trial. Full app access. No charge.
- **Days 31-60:** Half price (~$0.49/week or ~$1.99/month).
- **Day 61 onward:** Full price.

**Why two stacked softeners:**
The mobile subscription market is saturated with predatory patterns — auto-billing after 3-day trials, hidden cancellation flows, "limited time" pressure tactics. Four Tasks deliberately positions against this. A user paying their first full-price month has had **two months** of below-full-price exposure to the product. By that point they know whether the app is for them. The decision to keep paying is informed and fair.

This is also a marketing position — the patience of the trial is itself a story. "30-day free trial, half-price first paid month" reads as confidence in the product and respect for the user.

## Pair pricing — DEFERRED

The partner-app shape opens the question: when two people share a pair, who pays?

Options on the table (not yet decided):

- **Either user pays, both get access.** $4/month covers both seats. Effective per-person price ~$2/month, genuinely cheap. Clean for couples. Edge cases: what happens at breakup, how to handle pair_key migration when the paying user changes partners.
- **Each user pays individually.** Standard. Higher revenue ceiling. But charging both individuals for an app explicitly built around shared experience may feel grasping.
- **Either-or, user choice.** Either one user pays the full pair rate, OR both users split it (each pays half). UI surfaces the choice during subscription setup.

This decision is **deferred**. Morgan wants longer to think on it. Decision goes here when it lands.

Notes for resuming the discussion:
- Apple's subscription system supports family sharing natively. A subscription bought by user A can be shared with up to 5 family members at no extra cost. *However*, this only works for users in the same Apple Family Group — which mostly applies to literal family members, not friends or unmarried partners.
- The pair-key data model already binds two users into a shared identity. Whichever pricing model lands, the subscription state needs to live on the *pair* record, not just the user record.
- Word-of-mouth dynamics: pair pricing where one user pays could double the effective install base per paying customer. Strong for growth. Weakens revenue per active user.
- Privacy implication: subscription state on the pair record means both users can see "is this pair currently subscribed" — could make the subscription state itself a tiny social/relational signal between the partners. Worth thinking about.

## Existing users (mum, Ayshi, friends)

**Founders model.** Pre-launch users get permanent free access baked into the product, not via redemption codes.

Implementation: a `founders` flag on user records, set server-side at native app first-boot for users whose pair_key matches the existing prototype-era roster. Founders skip the paywall entirely, see no subscription UI, get all current and future content drops free for life.

Why this model rather than redemption codes:
- Redemption codes expire if not used. Founders should never have a "redeem before X date" anxiety.
- The relationship between Morgan and these users is permanent and personal, not transactional.
- Baking it into the data model signals something to future users: "the people who built this with us are still here." That's social proof that can't be bought.
- Costs nothing — these are friends and family, not a marketing-funnel cost.

Cap on founders: roughly 20-30 users from the prototype era. Hardcoded list, not opt-in.

## What this means for what we build

- **In-app purchase infrastructure (subscription only, no IAP packs).** Apple's StoreKit 2, Google Play Billing. Server-side validation via Cloudflare Worker — Apple/Google notify the Worker of subscription state changes, the Worker updates the user's `subscription_status` field. The Godot client never decides subscription state — it asks the server.
- **Paywall UI.** Surfaces during onboarding (after the first month free expires), and as a soft block on certain features (TBD which features — see below). Designed to feel calm, not pressuring.
- **Restore-purchases flow.** User reinstalls app or switches devices → the app asks Apple/Google "is this Apple ID subscribed to Four Tasks?" → server-side state updates. Real feature with real edge cases.
- **Subscription-state-aware features.** TBD which features sit behind the paywall after the trial period. The current working assumption: **app stays fully usable, but content drops (new emoji packs, new themes, new stamp variants released after launch) are subscriber-only.** Free-tier users keep using the original launch set forever. This is the cleanest version of "subscription = ongoing value" — non-paying users keep what they've got, paying users get the world expanding around them.

## Privacy policy v3 implications

When tile 0.10 lands, the policy gets these additions:

- **Subscription handling.** Apple StoreKit / Google Play Billing handle the actual transaction. THE PICKLE MOON doesn't see payment information (card numbers, billing addresses). We see only "user X has an active subscription" as a boolean flag and a renewal date.
- **What we store about subscriptions.** The `subscription_status` flag, renewal date, originating store (Apple/Google), `founders` flag if applicable. We don't store transaction IDs, prices paid, or refund history.
- **Subscription cancellation.** Users cancel through Apple/Google, not through us. We can't cancel for them. The privacy policy points to the standard cancellation flows.
- **Refunds.** Handled by Apple/Google directly. We don't process refunds.
- **Restore purchases.** Standard mention.

## What's NOT in the model

For clarity, things explicitly chosen against:

- **No advertising.** Ever. Aesthetically incompatible with the product, and would force a reversal of the privacy policy's "no third parties" claim.
- **No IAP "premium pack" model.** All content drops are part of the subscription. No "buy this pack for $1.99." Cleaner.
- **No utility paywall.** Free-tier users keep using the app. The paywall is on *new* content, not on *existing* utility.
- **No price hike for existing subscribers without grandfathering.** If the price ever goes up, existing subscribers keep their original rate. This is a permanent commitment of the studio.
- **No predatory trial conversion.** No 3-day trials, no auto-charge surprises, no "limited time" pressure.
- **No data sale.** Already covered in privacy policy; mentioning here for completeness.

## The conviction line

Morgan's stated position:

> "nothing turns me off faster than the greed i see in the mobile app market. i'm positive a majority of people can read between the lines and seeing something legitimately patient and respecting of their money is where i want to exist."

This is the brand promise. Every monetisation decision should be checked against this line. If a future decision pulls in the direction of "extract more revenue per user," it gets weighed against the patience of the original promise. The studio's revenue ceiling is partly self-imposed by this commitment, and that's intentional.

## The 5000-user goal

5000 paying subscribers at ~$4/month gross = ~$20k/month gross. After Apple/Google take their 15% (subscription year-2 rate; first year is 30%), and after expected churn, sustainable net is in the ~$11-13k/month range. That's "quit BHP and do this full-time" income in WA.

Reference points for plausibility:
- Finch (closest emotional peer) has ~5M users.
- Streaks ($4.99 one-time) has sold ~1M+ copies.
- Habitica (freemium) has ~4M users.

5000 is well below those peer-set sizes. The path is plausible. The unknown is *how long* it takes — six months, two years, five years. THE PICKLE MOON having Four Tasks + Sticky Steve + Beam Orb means three concurrent revenue streams, which hedges the Four Tasks timeline.

## Open questions to resolve later

- **Pair pricing structure** (deferred — see above).
- **Specific feature gating after trial.** Working assumption is "subscribers get new content drops, free users keep launch content forever." Needs pressure-testing once the customisation surface is fully built.
- **Annual subscription option.** Most apps offer monthly + annual at a discount (e.g. $39/year vs $48 if paid monthly). Worth offering. Not a v1.0 blocker.
- **Family sharing support.** Apple Family Group sharing is opt-in by the developer. Worth turning on. Free reach.
- **Refund / abuse handling.** Edge cases like users sharing one Apple ID across many devices to dodge the per-device charge. Mostly Apple's problem to police, but worth knowing it exists.
- **The "founders" cutoff date.** Specifically *when* in the timeline does someone stop being a founder? At launch? At v1.0 submission to stores? When the native app gets first deployment? Needs a sharp definition.
