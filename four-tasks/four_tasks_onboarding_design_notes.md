# Four Tasks — Onboarding Design Notes

**Status:** LOCKED. Eight-screen flow, copy, and screen ordering locked session 8. Server-side identity model (solo identity, pair join) reflects the shipped session-12 implementation: `pair_id NULL` solo model, separate `POST /users` / `join_by_values` / `resolve` endpoints.

**Implementation tiles:** 3.1 through 3.10 (Phase 3). Required reading before any Phase 3 tile.

**Cross-references:**
- `four_tasks_pair_key_design_notes.md` — identity model, collision resolution, recovery.
- `four_tasks_system_map.md` — live endpoint contracts (`§5`).
- `four_tasks_theme_design_notes.md` — slot catalogue, leader sticker, picker model.
- `four_tasks_staggered_disclosure_design_notes.md` — what onboarding teaches vs what waits.
- `four_tasks_morning_sequence_design_notes.md` — day-1 ritual, first claim.
- `four_tasks_monetisation_position.md` — library sharing plant on screen 7.

---

## Problem

Four Tasks is a two-user app by premise, but requiring both users to onboard together is an adoption-killer. Onboarding must work for two real install patterns:

1. **User installs alone**, recruits partner later (or never).
2. **User installs after their partner**, joins via the partner's three values.

A third pattern (both installing side-by-side simultaneously) is treated as rare — revisit only if friends-and-family testing surfaces it.

All paths land at the same end state: a functioning pair. Only the timing varies.

---

## Decision summary

1. **Solo mode is the default outcome of onboarding.** Every install completes solo. No partner fork during onboarding.
2. **Partner pairing is a post-onboarding action.** A persistent "Add my partner" button lives on the empty partner calendar. User taps it when ready.
3. **Pairing mechanism is typed three values** — partner's name, username, and leader sticker picked from a visual grid. No codes, no URLs, no universal links.
4. **Solo data stays put on join.** Day rows are keyed by `user_id`, which never changes, so a solo user's history is already part of the pair the moment they join — nothing moves.
5. **Ambiguous-match collisions resolved by username variation.** Multiple solo users with identical three values → ask the inviting user to temporarily vary their username, pair, then revert (the six-value hash now includes the partner half).
6. **Theme and MOTD are introduced in onboarding** — both are permanent UI elements, so hiding them from day zero would mean showing unexplained UI. The staggered-disclosure principle yields to always-visible features.
7. **Library sharing is planted in onboarding** — a single line on the partner screen, a forward-reference so the day-21 subscription disclosure has a hook. Not a pitch.
8. **Coin grant on the final screen** — a generous starter balance (2-3 stickers' worth) animates in during the closing read. The user lands already participating in the economy.

---

## The eight-screen flow

### Screen 1 — Welcome

```
[ leader sticker / mascot ]

Four Tasks

Pick four small things.
Do them with someone.
That's the whole app.

[ Get started ]
```

Brand voice compressed. "That's the whole app" pre-empts the "wait, that's it?" reaction by making it the pitch. No recovery link here — recovery lives on 1a.

### Screen 1a — First time, or returning?

```
Are you new here, or rejoining a pair?

[ First time here ]

      Recover my calendar
```

"First time here" is the primary CTA (full-width filled). "Recover my calendar" is visually secondary — recovery is expected to be rare. "Calendar" not "account" because the calendar is the concrete object the user lost, and it matches vocabulary used throughout the app.

Recovery branches here → tile 3.10 (six-value entry → `POST /resolve`).

### Screen 2 — Your name

```
What is your name?

[ text input ]

Write this correctly, as it needs to be
exact for your partner to connect with you.

[ Continue ]
```

Question framing, not a UX scold. The body gives a *reason* to type carefully (partner connection); immutability is named on the next screen where it's load-bearing.

Validation: non-empty, no `|` or `__` (pair-key delimiter chars). Continue greys until valid.

### Screen 3 — Confirm your name

```
Unlike your username which you will choose next,
this cannot be changed later. Are you sure
this is how you spell your name?

[ Jamie ]

[ Back ] [ Continue ]
```

The forward-reference to username removes hesitation ("you'll get a flexible handle in a moment"). Back is essential — this is the moment regret strikes.

The confirmed value is displayed **exactly as typed** — no all-caps or other transform. Names are case-sensitive in the pair-key hash (no casefold), so the displayed string must match the stored string character-for-character. Name commits as immutable on Continue.

### Screen 4 — Your username

```
What username would you like?

[ text input ]

This, unlike the name you just entered,
can be changed as often as you like.

[ Continue ]
```

Parallel structure to screen 2, explicitly reversing the immutability stake. No confirmation screen — mutable, low stakes. The user is not told that changing username triggers a pair-key rotation (leaky abstraction) or that partners see the username (they'll learn it on the partner panel).

### Screen 5 — Pick your starting avatar

Pre-picker popup, before the picker becomes interactive:

```
Pick your starting avatar

It'll give you a bunch of UI elements to
decorate your calendar.

[ Continue ]
```

"Avatar" names the function (the sticker is who the user is in the app, visible to their partner). Continue dismisses the popup and activates the picker.

**Picker layout:** picker takes 30-50% of the screen on the left; calendar is visible on the right, updating in real time as stickers are tapped. The picker shows a **curated launch starter subset** — every sticker here ships full UI coverage so every tap is a visibly transformative reveal.

**Starter subset selection criteria:** ships many slots (taps feel meaningful), looks distinct from the others (shows theming range), reads well on marketing screenshots (the store gallery uses this picker). The subset is **fixed across all installs** and is also the seed pool for screen 5b. Not the full catalogue — that's reserved for post-onboarding discovery.

**Continue (off the picker) commits the chosen sticker** as `active_leader` and fires the theme assembly theatre.

### Theme assembly theatre (transition, ~2s)

Loading-screen moment. Chosen leader centred, glitch-typewriter loading text, themed UI elements assemble onto the calendar in z-index order with small bounce-and-twist animations. Skippable on tap. Onboarding-only — mid-app theme changes apply instantly per the theme doc.

### Screen 5b — Themed companion pool

Popup over the themed calendar (calendar visible underneath, MOTD area present in header):

```
Here are some stickers that match your theme,
to get you started. You'll collect the ones
you want in time.

[ 5ish themed companion stickers display ]

These have a chance to appear each day you
complete.

[ animated 7-day strip: 6/7 days green with
  themed stickers placed ]

They also complete your message of the day
— let's see what that is.

[ Continue ]
```

**Themed companion architecture:** each curated launch leader has a small bundle of companion stickers (Frog → swamp companions, etc.), data-driven alongside the curated subset. **Art budget note:** every curated launch leader needs full-slot UI coverage AND 4-5 companions — larger than one sticker per leader.

Screen 5b commits the leader + companions as the user's `active_stickers` pool. The animated strip teaches the per-day rotation mechanic visually. The trailing line sets up screen 6 without explaining MOTD yet.

### Screen 6 — Your message of the day

The popup dismisses; attention is drawn to the MOTD area under the header. A new popup overlays:

```
Your message of the day

Each day a new message spins up to set the
vibes for the day. Give it a few spins if you
don't like the first one — here's some coins
to get you started.

[ Coins: 750 ]    [ More coins ]

[ Spinner: ADVERB | VERB | NOUN | STICKER ]

                              [ Spin ]
                              [ Save ]
```

**MOTD mechanic:** four spinners — adverb / verb / noun / sticker. First three are server-side word lists; the fourth pulls from the user's `active_stickers` pool. Output is a four-element phrase ("Quietly | tended | the swamp | 🐸"). Dark-Souls-death-message energy.

**Tutorial coin loop:** the "More coins" button arms whenever balance falls below the reroll threshold (70-110); tapping tops back up to 750. Infinite tutorial rerolling. The user learns the economy by playing it, not by reading about it.

**Save commits the MOTD** (typewriters onto the header) and advances. **Tutorial coins do not persist** — the carryover balance is discarded; screen 8 grants the real starting amount.

### Screen 7 — Your partner's calendar

The typewriter MOTD finishes; a subtle swipe-right prompt appears; the user swipes to the empty, muted partner calendar. A popup overlays:

```
Your partner's calendar

This is where your partner's days will live
once you pair up. Their theme, their message
of the day, their progress — all here.

You can pair anytime using the button on this
screen.

Subscribing later lets you share your sticker
libraries — they'll have access to your stickers
and the UI they bring, and you'll have access
to theirs.

[ Continue ]
```

**The persistent "Add my partner" button lives on the partner calendar UI itself, not in this popup.** The popup teaches; the button is the always-available action.

**Subscription plant:** one line introducing bilateral library sharing as the value of subscription. "Subscribing later" defuses sales pressure by naming it a future option. "UI they bring" establishes vocabulary reused on screen 8 and in the picker — a sticker is a multi-faceted object, without naming slots.

### Screen 8 — Coins and the picker

User swipes back to their own (fully themed) calendar. Popup overlay. **A coin blitz fires during the read** — the counter ticks up to a clean generous starting balance (2-3 stickers' worth at expected pricing).

```
One more thing.

Here's some coins to play with — you'll earn
more every day you finish your tasks.

Have a play with the sticker picker before
you head into your first day. Save some coins
for later or grab a new sticker if you like
it or the UI elements it features.

It's your app.

[ Start the day ]
```

The animation is the announcement — "some coins" is understated so the size feels surprising. "It's your app" names the pattern every prior screen demonstrated through behaviour. "Start the day" names the daily rhythm.

**After "Start the day":** popup dismisses, user is on their themed calendar with their saved MOTD, granted coins, and picker accessible. No calendar tour overlay — the partner panel and coins each got their own screen, and the UI assembled in front of the user during the theatre. They're dropped onto the calendar and trusted to use it.

**Open question:** does "Start the day" fire the day-1 morning sequence immediately, or drop the user on a flat calendar with the sequence firing on next open? Morning sequence assumes a sealed previous day; day 1 has none. Resolve in the morning sequence doc's next pass.

### The four-tasks entry screen (placement open)

The user must enter their four tasks somewhere in the flow — this screen is real, but its placement (between 6 and 7, or 7 and 8) and presentation (full screen, loading-screen style, whatever) are **implementer's choice at tile 3.x**. Not locked, no strong preference. Working copy:

```
Your four tasks

[ input 1, placeholder from curated pool ]
[ input 2, placeholder ]
[ input 3, placeholder ]
[ input 4, placeholder ]

Four small things you'd like to do each day.
Keep them small enough that you'll actually
do them. You can change them later.

[ Continue ]
```

"Keep them small enough that you'll actually do them" is the most important sentence in onboarding. Validation: each label non-empty, no `|` / `__`, UX cap ~30 chars (server caps at 49).

---

## Solo mode — app behaviour

**Works in solo mode:** full task list / tick / MOTD / calendar; stamp animation; coin economy at base rate (no 1.5x — that needs a partner); morning sequence (partner-panel beat replaced by a recruitment beat); full theme system; streaks, rest days, milestones; the day-2/3/4 staggered reveals (day-4 partner-reactions reveal is deferred with the feature — see staggered disclosure doc).

**Does not work:** partner reactions (no partner), 1.5x multiplier, partner panel content.

**Partner panel as recruitment surface:** muted, gently styled empty calendar (visibly not broken, just empty), centred placeholder copy ("Your partner's space is waiting." / "Pair with them to see their tasks here, and they'll see yours."), persistent `[ Add my partner ]` button. Tapping it opens the join-by-values flow.

---

## Server — identity and pairing (shipped session 12)

### Solo identity model

A solo user has a `user_id` (server-generated UUID v4, assigned when the `users` row is created at the end of onboarding), a `users` row holding identity + personal state with `pair_id = NULL`, and a `days` row per day of use attached to that `user_id`. **No `pairs` row** — solo is `users.pair_id IS NULL`. This is the same state as post-un-pair: solo is solo regardless of history.

### When the user row is created

`POST /users` fires at the **end of onboarding** (screen 8 "Start the day"), returning the user's `user_id`, which the client writes to `identity.cfg`. Pairing is always a later action against an already-existing `user_id`.

### Pair join flow

`POST /users/:user_id/join_by_values` — body `{partner: {name, username, active_leader}}`. The caller's own triple is read from their stored `users` row, not sent in the body.

1. Caller must be solo (`pair_id IS NULL`), else 409 `already_paired`.
2. Search solo users (`pair_id IS NULL`, excluding self) matching `partner` exactly (case-sensitive, NFC-normalised, whitespace-trimmed per pair-key Section 12).
3. **Exactly one match:** derive pair-key from both triples, UNIQUE-check (409 `pair_key_collision` on clash), then atomically INSERT the `pairs` row and UPDATE both users' `pair_id`. Returns `{pair_id}`. Days don't move — they're already `user_id`-keyed.
4. **Zero matches:** 404 `partner_not_found`. No phantom-pair creation — the session-8 phantom-pair sketch was dropped (it produced orphaned `users` rows). The client surfaces "couldn't find your partner — check they've installed and their three values are correct."
5. **Two-plus matches:** 409 `ambiguous_match` → collision-resolution UX below.

### Recovery (separate endpoint)

`POST /resolve` — body is both triples. Matches the six-value pair-key, disambiguates the recovering user by `self.name`, returns `{user_id, pair_id}`. 404 `pair_not_found` if no pair matches. This is **not** part of `join_by_values`; recovery and fresh-join are distinct endpoints.

Solo users have no `pair_key` and no partner-mediated recovery — they fall back to OS-level `identity.cfg` backup (iCloud Keychain / Google Drive) per pair-key Section 17. A solo user with no OS backup who loses their device re-installs fresh as a new `user_id`; prior solo data is unrecoverable. Deliberate accept — solo users have less to lose, and the partner-mediated mechanism requires a partner.

### Three failure modes, kept distinct

- **Ambiguous-match** (this doc): multiple solo users match one set of partner values. Counted before any hash; 409, prevents the bad join.
- **Wrong-attach** (pair-key Section 9.3): joiner's six values happen to hash to a real but unintended pair. Detected only post-hash; server returns 200, human verification at the UI catches it.
- **Hash collision** (pair-key Section 13): two distinct intended pairs hash to the same `pair_key`. UNIQUE constraint catches it at write time.

---

## Ambiguous-match resolution UX

On 409 `ambiguous_match`, shown to the joiner:

```
We couldn't find your partner uniquely.

There's more than one person matching the
details you typed. Ask your partner to change
their username slightly, then try again.

They can change it back once you're paired.
```

The fix is on the partner's side, not the joiner's — the copy routes them correctly. Partner changes username → pair-key rotates → the solo match becomes unique → joiner retries → pair forms. After pairing, the partner can revert: the six-value hash now includes the joiner's half, so even an identical-to-the-other-person username produces a different pair-key. Post-pair revert is an explicitly supported flow.

The user absorbs three lessons without being told: usernames are mutable, pair identity survives username changes after pairing, and the collision was a join-time issue not a permanent one. Frequency is expected to be vanishingly low; the UX exists for the rare case.

---

## Staggered disclosure intersection

Onboarding teaches: name, username, starting avatar; themed companion pool + per-day rotation; MOTD spin/save + reroll cost; partner calendar location + post-onboarding pairing; library sharing concept; four tasks; coins as live, earnable currency.

Onboarding does NOT teach: rest days (day-3 reveal), long-press philosophy + week mode (day-2 reveal + cascade), picker context menu (trigger-gated on first picker open), streak milestones, coin name personalisation, subscription specifics, leaderboard. (Partner reactions deferred to v1.x with the feature.)

The load-bearing rule: onboarding may absorb a feature only if its UI is permanently visible from day zero. Everything else waits for the staggered schedule.

---

## Edge cases

- **Joint in-person install** — two users onboarding side-by-side may cross-talk. The solo-default architecture makes this mild (neither proxy-onboards the other). Frequency unknown; watch during friends-and-family testing.
- **iOS install-referrer** — not relevant. No invite links; all users onboard fresh then type partner values manually. No platform asymmetry. AASA / assetlinks files are now optional (only for marketing deep-links, not pairing).
- **Two-solo-users pair-merge** — if both installed solo independently and later want to combine, one must un-pair/reboot and join the other; the rebooted user's solo history is lost. Rare; the join flow requires one party solo at join time. Future v1.x: a pair-merge flow that picks which history to keep.

---

## Open work

- **Four-tasks entry screen** — placement + presentation, implementer's choice at tile 3.x (see above).
- **"Start the day" trigger** — fire day-1 morning sequence immediately or defer to next open? Morning sequence doc's next pass.
- **Curated launch starter subset** — concrete leader + companion-bundle list. Art + content authoring.
- **Placeholder task pool** — ~50 examples for the four-tasks inputs. Copy authoring.
- **MOTD word pools** — adverb / verb / noun lists, server-side. Tonally consistent, never bleak, slightly off-kilter.
- **Partner panel empty-state copy** — placeholder above needs a polish pass.
- **Recovery flow detail** — masked-hint mechanism, rate limiting, hint-eligible fields. Captured in pair-key doc; lands at tile 3.10.
- **Screen 3 confirmation UX** — single lock button vs modal vs inline accordion. Implementation decision.
- **"Is this your partner?" sanity-check screen** — optional post-resolve confirmation (partner name + sticker) before committing. v1.x polish.
