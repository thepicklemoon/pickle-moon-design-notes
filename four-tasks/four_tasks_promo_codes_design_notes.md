# Four Tasks — Promo Codes Design Notes

**Status:** FORMALISED session 29 from the ad-hoc decisions the schema
was built against (session 12). Schema is LIVE (`promo_codes` +
`redemption_attempts` shipped in the v1.0 wholesale rewrite); no
redemption endpoint exists yet. Required before subreddit outreach
(quiet-launch seeding, Phase 5.15). Implementation: redemption
endpoint + client entry sit with Phase 5 commercial work; DevKit
scenario `promo` is stubbed in the menu.

## What a promo code is

A short human-typeable string handed out in a specific community
context (subreddit thread, Discord, a founder DM) that grants a
benefit on redemption. v1.0 benefits are flags, not currency: no code
ever grants coins (the economy doc's sinks/sources stay closed).

Benefit types:
  - `founders_rate` — permanent discounted subscription rate. PERMANENT
    once granted: a later lapse or unsubscribe does not revoke it; the
    rate re-applies on resubscribe. (Immutable-past principle applied
    to commercial state — same shape as baked payouts.)
  - `trial_extension` — adds N days to the subscription trial
    (schema: `users.trial_extension_days`).

## Code shape

`FIFOFOUNDER20`-style: an uppercase A-Z0-9 string, 6-20 chars,
readable, theme-able per outreach context (the code itself is part of
the marketing — a subreddit gets a code that names them). No ambiguous
characters policy needed at this length (codes are copy-pasted more
than typed). Codes are created by hand (SQL-seed pre-launch; the
publishing web tool inherits code creation post-launch if volume ever
justifies it).

Each code row carries: the string (PK), benefit type, benefit value,
optional max_redemptions (NULL = uncapped), optional expires_at,
active flag.

## Redemption rules

  - **One code per user, ever.** `users.redeemed_code` is write-once;
    a second redemption attempt on any code returns 409
    `already_redeemed`. This is the anti-stacking rule — codes are a
    welcome gift, not a coupon book.
  - A code at `max_redemptions` or past `expires_at` or inactive
    returns 409 `code_exhausted` / `code_expired` / 404 `code_not_found`
    (inactive folds into not-found — no information leak about which
    codes exist).
  - Redemption is transactional: flag grant + `users.redeemed_code` +
    redemption count in one batch.

## Attempt logging

`redemption_attempts` logs EVERY attempt — successes and failures,
with the code string as typed, user_id, timestamp, and outcome. The
failures are the valuable half: they show typo patterns (is the code
too fiddly?), guessing/enumeration (does rate limiting need
tightening?), and which outreach contexts actually convert. No
pruning; volume is trivial at this scale.

## Interactions

  - **Founders FLAG vs founders RATE** are distinct: the flag
    (`founders_flag`) is the hardcoded ~20-30-person list cut off at
    first TestFlight/internal-track deploy (todo: founders cutoff
    locks); the RATE is what codes grant to launch-window users. A
    flagged founder doesn't need a code.
  - Codes never touch coins, streaks, or days — commercial columns
    only. The claim handler never reads code state.
  - Rate limiting: redemption is a write endpoint and falls under the
    standard per-IP edge limits (rate-limiting design notes) — generous
    enough for a fat-fingered retry, tight enough to make enumeration
    pointless on top of the 36^N search space.

## Out of scope

Referral revenue-share is a SEPARATE system with its own design pass
(moves real money — see todo). Promo codes grant product benefits
only; nothing here creates a payout obligation.
