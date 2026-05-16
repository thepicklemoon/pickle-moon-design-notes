# Four Tasks — Onboarding Design Notes

**Status:** LOCKED (session 8 rewrite). Supersedes the session 6 version entirely. Architecture and screen flow changed substantially; clean rewrite rather than patch.

**Implementation tiles:** 3.1 through 3.x (Phase 3). Required reading before any Phase 3 tile lands.

**Cross-references:**
- `four_tasks_pair_key_design_notes.md` — identity model, collision resolution, recovery.
- `four_tasks_theme_design_notes.md` — slot catalogue, leader sticker, picker model.
- `four_tasks_staggered_disclosure_design_notes.md` — what onboarding teaches vs what waits.
- `four_tasks_morning_sequence_design_notes.md` — day-1 ritual, first claim, MOTD context.
- `four_tasks_timezone_and_sealing_design_notes.md` — silent timezone capture at install.
- `four_tasks_write_rules_design_notes.md` — resolve + join endpoints.
- `four_tasks_monetisation_position.md` — library sharing plant on screen 7.

---

## Problem

Four Tasks is a two-user app by premise. Pair-key v2 requires all six identity values (name + username + active_leader × 2 users) to compute a real pair-key. The naive interpretation — both users must complete onboarding together — creates an adoption-killing friction point.

Onboarding must work for two real-world install patterns:

1. **User installs alone**, will recruit partner later (or never).
2. **User installs after their partner**, joins via the partner's three values.

A third pattern — both users installing simultaneously side-by-side — exists but is treated as rare (real-world frequency unknown; revisit if friends-and-family native testing surfaces it).

All paths land at the same end state: a functioning pair. The timing varies.

---

## Decision summary

1. **Solo mode is the default outcome of onboarding.** Every install completes onboarding solo. No partner fork during onboarding.

2. **Partner pairing is a post-onboarding action.** A persistent "Add my partner" button lives on the empty partner calendar UI. User taps it when ready.

3. **Pairing mechanism is typed three values.** Joining user types partner's name, username, and picks partner's leader sticker from a visual grid. No codes, no URLs, no universal links.

4. **Solo data migrates into the pair on join.** When solo user A is joined by B, A's calendar history, coins, streaks all carry over. Re-uses pair-key migration machinery.

5. **Ambiguous-match collisions resolved by username variation.** Rare collision case (multiple solo users with identical three values) is resolved by asking the inviting user to temporarily vary their username. After pairing succeeds, they can revert because the full six-value hash now includes the partner half.

6. **Theme and MOTD are introduced in onboarding.** Both are permanent UI elements; hiding them from day zero meant showing UI elements with no framing. The staggered disclosure principle yields to features whose UI is always visible.

7. **Library sharing is planted in onboarding.** Single line on the partner calendar screen. Not a pitch; a forward-reference so the eventual day-21 subscription disclosure has a mental hook.

8. **Coin grant on final screen.** Generous starter balance (sized for 2-3 stickers) animates in during the read of the closing screen. The user lands in the app already participating in the economy.

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

The three-line pitch is the brand voice in compressed form. "Four small things" is concrete; "do them with someone" carries the two-user premise without explanation; "that's the whole app" pre-empts the "wait, that's it?" reaction by making it the pitch.

No recovery link on this screen. Recovery lives on screen 1a.

### Screen 1a — First time, or returning?

```
Are you new here, or rejoining a pair?

[ First time here ]

      Recover my calendar
```

"First time here" is the primary CTA — full-width filled button. "Recover my calendar" is visually secondary — smaller, lighter weight beneath. Recovery on native stores is expected to be rare; the visual hierarchy matches expected frequency.

"Recover my calendar" rather than "Recover my account" because the calendar is the concrete object the user lost. The word also matches what they'll see throughout the app (the calendar tour, the partner calendar, etc.) — consistent vocabulary.

Recovery path branches here, owned by tile 3.4 design.

### Screen 2 — Your name

```
What is your name?

[ text input ]

Write this correctly, as it needs to be
exact for your partner to connect with you.

[ Continue ]
```

Question framing, not instructional label. The body sentence gives the user a *reason* to type carefully (partner connection) rather than a UX scold (immutability). Immutability is named on the next screen where it's load-bearing.

Validation: non-empty, no `|` or `__` characters (server rules). Continue greys until valid. Edge-case validation surfaces as inline server errors when triggered, not as pre-emptive copy.

### Screen 3 — Confirm your name

```
Unlike your username which you will choose next,
this cannot be changed later. Are you sure
this is how you spell your name?

[ JAMIE ]

[ Back ] [ Continue ]
```

The forward-reference to username does psychological work: it tells the user "you'll get a flexible handle in a moment, this isn't your last chance to express yourself." Removes hesitation that would otherwise stall the name commit.

Back button is essential — the confirmation is the moment regret strikes. Without Back, a user staring at a typo is trapped.

Name is committed as immutable on Continue. No app-UI path to change it afterward.

### Screen 4 — Your username

```
What username would you like?

[ text input ]

This, unlike the name you just entered,
can be changed as often as you like.

[ Continue ]
```

Parallel structure to screen 2 ("What is your name?" / "What username would you like?"). The body sentence explicitly contrasts against the previous screen, reversing the immutability stake the user was just primed on. No confirmation screen — mutable, low stakes.

The user is not told that changing username triggers a pair-key migration. That's leaky abstraction. The user is not told that partners see the username. They'll figure that out the first time they see the partner panel.

### Screen 5 — Pick your starting avatar

Pre-picker popup, before the picker becomes interactive:

```
Pick your starting avatar

It'll give you a bunch of UI elements to
decorate your calendar.

[ Continue ]
```

The popup is the framing. "Avatar" names the function (the sticker is who the user will be in the app, visible to their partner forever). "It'll give you a bunch of UI elements to decorate your calendar" names what's about to happen — the sticker brings things, you receive them.

Continue dismisses the popup and activates the picker.

**Picker layout:** sticker picker takes 30-50% of the screen on the left. Calendar UI is visible on the right, updating in real time as the user taps stickers. The picker shows a **curated launch starter subset** — every sticker on this screen ships full UI element coverage (background, palette, cell variants, etc.) so every tap is a visibly transformative reveal.

**Selection criteria for the launch starter subset:**
- Stickers that ship many slots (so taps are meaningful).
- Stickers that look distinct from each other (showing the range of theming).
- Stickers that read well on marketing screenshots (the install-store gallery uses this picker).

**The starter subset is fixed across all installs.** It is also the seed pool for screen 5b. Same selection, same purpose. Not the full launch catalogue — that's reserved for post-onboarding discovery via the picker.

**Continue (off the picker) commits the chosen sticker** as `active_leader`, fires the theme assembly theatre.

### Theme assembly theatre (transition, ~2 seconds)

Loading-screen-style moment. The chosen leader sticker is centred. Glitch-typewriter loading text. Themed UI elements assemble onto the calendar in z-index order with small bounce-and-twist animations. The pair-key (or solo-pair-key) hashes during this window.

Skippable on tap.

This is reserved for onboarding only. Mid-app theme changes apply instantly or near-instantly (per the theme doc).

### Screen 5b — Themed companion pool

Popup over the themed calendar (calendar visible underneath, MOTD area present in the header):

```
Here are some stickers that match your theme,
to get you started. You'll collect the ones
you want in time.

[ 5ish themed companion stickers display ]

These have a chance to appear each day you
complete.

[ animated 7-day strip: 6/7 days green with
  themed stickers placed; the strip updates
  as the user could imagine toggling pool
  members, though no toggle interaction here ]

They also complete your message of the day
— let's see what that is.

[ Continue ]
```

**Themed companion architecture:** each curated launch leader has a small set of companion stickers attached as a bundle. Frog → swamp companions (lily pad, fly, swamp light, swamp flower). Vampire → castle companions. Cat → cat companions. Each leader's bundle is data-driven, defined alongside the curated launch subset.

**Content authoring constraint:** every curated launch leader needs full-slot UI coverage AND 4-5 companion stickers in its world. The art budget for the launch starter set is larger than a single sticker per leader.

**Pool seeding:** screen 5b commits the chosen leader + its themed companions as the user's `active_stickers` pool.

The animated 7-day strip teaches the per-day rotation mechanic visually. No copy needed beyond the supporting sentence.

The trailing line ("they also complete your message of the day") sets up the next screen's reveal without explaining MOTD here. The user reads "let's see what that is" and taps Continue genuinely curious.

### Screen 6 — Your message of the day

The popup dismisses, the user's attention is drawn to the MOTD area under the `[username]'s calendar` header. A new popup overlays the calendar:

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

**MOTD mechanic:** four spinners, each pulling from its own pool. Adverb / verb / noun / sticker. The first three are server-side word lists. The fourth pulls from the user's `active_stickers` pool. Output assembles into a four-element phrase ("Quietly | tended | the swamp | 🐸"). Dark-Souls-death-message energy.

**Tutorial coin loop:** the "More coins" button arms whenever the user's balance falls below the reroll cost threshold (70-110). Tapping it tops the balance back up to 750. Infinite tutorial rerolling. The user can spin freely without spending real economy.

**Why this works as the coin introduction:** the user implicitly learns "coins are spent on rerolls" by spending them. They learn "coins have a balance" by watching it deplete and replenish. They learn the *feel* of the economy through play, not text.

**Save commits the MOTD.** It typewriters onto the header in the position where it belongs. Onboarding moves to the next screen.

**Note on coin state at end of MOTD screen:** the carryover balance is discarded. Whatever the user has when they tap Save is irrelevant — screen 8 will grant a fresh coin amount. Tutorial coins do not persist as live economy currency.

### Screen 7 — Your partner's calendar

Setup: typewriter MOTD finishes animating onto the header. A subtle swipe-right prompt (arrow, chevron, or animated indicator) appears. The user swipes. The view slides to the partner calendar — empty, muted styling, neutral header. A popup overlays.

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

**The subscription plant:** single line introducing bilateral library sharing as the value proposition of subscription. Not a pitch, not a pricing surface. The framing "subscribing later" defuses sales pressure by naming the action as a future option. The bilateral phrasing ("they'll have access to your stickers... you'll have access to theirs") is precise about what library sharing actually does.

**"UI they bring"** establishes vocabulary the user will encounter again on screen 8 and in the picker. "Stickers" plus "the UI they bring" tells the user that a sticker is a multi-faceted object without naming slots or the feature-catalogue model — which they don't need yet.

### Screen 8 — Coins and the picker

User swipes back to their own calendar. The themed UI is fully present. Popup overlay. **A coin blitz animation fires during the read** — coin counter ticks up from current state to a clean generous starting balance (sized for 2-3 sticker purchases at expected pricing).

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

**The animation is the announcement.** The copy doesn't need to say "here's some coins" because the user is watching them arrive. The phrase "here's some coins to play with" is deliberately understated — calling the gift "some" lets the size feel surprising rather than oversold.

**Weight comes from animation time, not language.** ~2 seconds of slow coin counter climb, soft audio if appropriate, final balance lands with a small settling beat. Long enough to feel the size of the gift; short enough not to be waiting.

**"It's your app."** Brand thesis stated plainly. Every previous screen showed this through behaviour (you can change usernames, you can spend everything, you can keep whatever stickers you want). This sentence names the pattern.

**"Start the day"** as the button label. Echoes the previous sentence's "head into your first day." Names the daily rhythm the app will use forever. Stronger than "Continue" or "Start my first day."

### After "Start the day"

The popup dismisses. The user is on their themed calendar with the MOTD they saved, the coins they were granted, and the picker accessible (wherever the picker lives on the UI — deferred design decision, internally parked).

**No calendar tour overlay.** The original session 6 design had a screen 12 with arrow annotations pointing at today, the partner panel, and the coin balance. Removed in session 8 because the partner panel got its own screen, coins got their own screen, and the calendar UI assembled in front of the user during the theme theatre. The user is dropped onto the calendar and trusted to use it.

**Open question:** does "Start the day" trigger the day-1 morning sequence immediately, or does it just drop the user on a flat calendar with the morning sequence firing on next app open? Morning sequence design assumes a sealed previous day; on day 1 there isn't one. Real design question for the morning sequence doc's next pass.

---

## Solo mode — app behaviour

What works in solo mode:

- Full task list, tick, MOTD, calendar.
- Stamp animation on completion.
- Coin economy at base rate (no 1.5x multiplier — that requires a partner).
- Morning sequence — fires normally. Partner panel section of the sequence is replaced with a recruitment beat.
- Theme system (active_stickers pool, active_leader, active_theme slots) all functional. Post-fork features (per-slot context menu on long-press) trigger-gated on first picker open per staggered disclosure.
- Streaks, rest days, milestones — all work.
- Day-2 long-press philosophy reveal fires. Day-3 rest day intro fires. Day-4 partner reactions reveal fires (framed prospectively for solo users: "when you pair up, you'll be able to react to each other's days").

What does not work:

- Partner reactions. No partner exists.
- 1.5x multiplier on daily payout.
- Partner panel content. Empty calendar + persistent "Add my partner" button.

**The partner panel as recruitment surface:**

- Muted, gently styled empty calendar — visibly not broken, just empty.
- Centred copy (placeholder, copy job parked):
  - "Your partner's space is waiting."
  - "Pair with them to see their tasks here, and they'll see yours."
- Persistent button: `[ Add my partner ]`.

Tapping Add my partner opens the join-by-values flow (see Server section below).

---

## Server — solo identity model

A solo user has:
- A `solo_uuid` (server-generated, UUIDv4, unguessable by construction).
- A `pairs` row keyed by `solo:<uuid>` (not by the six-value hash — no six values yet).
- A single `users` row attached to that pair-key. Partner half is absent.
- A `days` row each day they use the app, attached to the solo pair-key.

The solo pair-key has a distinguishable prefix (`solo:`) so server logic can branch on solo-vs-real cheaply without an extra column:

```
pairs.key = "solo:" + uuid_v4_hex
```

Real pair-keys remain the SHA-256-truncated-16-hex format from pair-key v2. The two formats are non-overlapping by prefix.

Schema: no new columns required. Existing pair-key column handles solo via prefix convention. The `users` table may have one or two rows per pair-key.

---

## Server — pair join flow

**Single endpoint replacing the session 6 `join_by_link` / `join_by_values` pair.**

`POST /join_by_values`

```
Body: {
  partner_values: { name, username, active_leader },
  joiner_values:  { name, username, active_leader }
}
```

**Logic:**

1. Search solo pairs (pair-key starts with `solo:`) whose attached user matches `partner_values` exactly (case-sensitive, whitespace-normalised — exact normalisation rules TBD as part of pair-key doc's outstanding work).

2. **If exactly one solo match:**
   - Compute new real pair-key from all six values.
   - Open D1 transaction.
   - INSERT new `pairs` row with real pair-key.
   - UPDATE existing `users` row to point to new pair-key.
   - INSERT new `users` row for joiner, pointing to new pair-key.
   - UPDATE all `days` rows from solo pair-key to new pair-key.
   - DELETE old solo pairs row.
   - Commit. Return new pair-key + full pair state to joiner.

3. **If zero solo matches:** search real pairs whose A or B user matches `partner_values` AND whose other user matches `joiner_values` exactly. This handles the recovery case (existing real pair, joiner is reinstalling). If found, return pair state to joiner without creating anything new. If not found, create a new real pair with both users (the no-such-partner case — partner is a phantom until they install).

4. **If more than one solo match:** return 409 `ambiguous_match`. Client surfaces the collision-resolution UX (see below).

---

## Ambiguous-match resolution UX

When B types A's three values and the server returns 409 `ambiguous_match`:

Copy shown to joiner:

```
We couldn't find your partner uniquely.

There's more than one person matching the
details you typed. Ask your partner to change
their username slightly, then try again.

They can change it back once you're paired.
```

**Why this works:**

- The fix is on A's side, not B's. B can't fix this by retyping; the copy routes them correctly.
- A goes into settings, changes username to something more distinctive, taps save. The pair-key migrates. The solo match becomes unique.
- B retries. Pair forms.
- After pairing, A can revert username if they want — the full six-value hash now includes B's half, so even reverting to identical-to-the-other-Sarah's-username produces a different pair-key. Post-pair username revert is an explicit supported flow.

**Vocabulary lesson the user absorbs without explanation:**

- Usernames are mutable.
- Pair identity is robust to username changes after pairing.
- The collision was a join-time issue, not a permanent issue.

The frequency of this case is expected to be vanishingly low. Two solo users with identical name + username + leader sticker is unlikely at any real scale. The UX exists for the rare case, not as a common flow.

---

## Recovery path

Triggered from screen 1a's "Recover my calendar" button.

Owned by tile 3.4 design. Recovery flow:

- User enters all six values (their own three + partner's three, from memory or from the partner's visible app state).
- Server resolves to existing pair-key.
- If match: data loads, user lands in main app, skips onboarding.
- If no match: collision popup with masked-hint mechanism (see pair-key v2 doc's "Masked-hint recovery aid" section — exact rate limiting and which fields are hint-eligible are pair-key doc's outstanding work).

Recovery does NOT touch solo pairs — only real pairs. A user who was previously solo and lost their data has the same recovery options as any other user, but their solo pair-key (`solo:<uuid>`) was server-generated and not memorable. They can't recover a solo state. They can re-install fresh as solo and re-invite their (future or never-existent) partner.

This is a deliberate accept. Solo users have less to lose than paired users — no streak with another person, no shared history. Recovery prioritises paired data.

---

## Staggered disclosure intersection

The onboarding floor (what gets taught on day zero) is now defined in this doc's eight-screen flow. The staggered disclosure design notes hold the principle and the day-2-onward schedule.

Onboarding teaches:
- Name, username, starting avatar.
- Themed companion pool and per-day rotation mechanic.
- MOTD spin/save mechanic and coin reroll cost.
- Partner calendar location and post-onboarding pairing.
- Library sharing concept (subscription plant).
- Four tasks (placeholder pool, mutability).
- Coins as live currency, earnable daily.

Onboarding does NOT teach:
- Rest days (day-3 reveal).
- Long-press philosophy and week mode (day-2 reveal + cascade).
- Picker context menu (trigger-gated on first picker open).
- Partner reactions (day-4 reveal).
- Streak milestones, coin name personalisation, subscription specifics, leaderboard.

This is the load-bearing intersection. Onboarding may absorb more features over time if their UI is permanent; anything not permanent in the UI waits.

---

## Edge cases

### Joint in-person install dissonance

Two users installing side-by-side at the same time may experience cross-talk as they each navigate their own onboarding. The current solo-default architecture makes this less acute than the session 6 design — neither user is proxy-onboarding the other; they each set up their own app, and pairing happens after via the post-onboarding button.

Real-world frequency unknown. Worth watching during friends-and-family native testing. If joint install proves common and produces UX issues, revisit. Currently not blocking.

### iOS install-referrer

Not relevant in the current design. The session 6 design relied on universal links carrying invite parameters; we no longer use invite links. iOS users install fresh, run through full onboarding, then enter their partner's three values manually. Android users do the same. No platform asymmetry.

The pickle-moon-public repo's universal link / app link configuration (AASA file, assetlinks.json) is now optional — only needed if Four Tasks later wants deep-link support for marketing URLs (e.g. "open the app to this specific page"). Not v1.0 blocking.

### Two-solo-users pair-merge

If both A and B installed solo independently and later want to combine, one of them has to reboot their install and join the other via the join-by-values flow. The rebooted user's solo history is lost.

This is an accept. The scenario is rare (two people in a relationship both happening to install the app independently and never tell each other), and the workaround is straightforward. Solo data is meant to migrate into a pair on join, but the join flow requires one user to be solo at join time — there's no clean way to merge two existing histories.

Future v1.x consideration: a pair-merge flow that asks "which user's history should be preserved?" and discards the other. Not v1.0 work.

---

## Open work

Defer to implementation or future design passes:

- **Placeholder task pool** — ~50 examples for the four-tasks input on screen 7 (which the design doc doesn't currently include — see Note below). Curated copy authoring tile.
- **Curated launch starter subset definition** — concrete list of leader stickers and companion bundles. Art + content authoring tile.
- **MOTD word pools** — adverb, verb, noun lists. Server-side content authoring. Tonally consistent, never bleak, slightly off-kilter.
- **Partner panel empty state copy** — placeholder above; needs polish pass.
- **Recovery flow detail** — masked hint mechanism, rate limiting, fields eligible for hinting. Captured in pair-key design doc; lands at tile 3.4 implementation.
- **Confirmation modal exact UX (screen 3)** — single button to lock vs full-screen modal vs inline accordion. Implementation decision.
- **"Is this your partner?" sanity-check screen** — after manual pair resolve, optionally show partner name + active_leader sticker for confirmation before fully committing. Defer to v1.x polish.
- **Whether "Start the day" triggers day-1 morning sequence immediately or defers it.** Real design question for morning sequence doc's next pass.

### Note on missing screen

The original session 6 design had a "Your four tasks" screen as part of the partner-fork flow (screen 11 in the old numbering). The session 8 rewrite organised around eight screens centred on identity + theme + MOTD + partner intro + coin grant — the four-tasks entry was implicitly subsumed but not given its own explicit screen number in this rewrite.

**This is a gap.** The user needs to enter their four tasks somewhere. The likely placement is between screen 6 (MOTD) and screen 7 (partner calendar), or between screen 7 and screen 8. To be resolved in the next design pass.

Working copy from session 8:

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

"Keep them small enough that you'll actually do them" is the most important sentence in the entire onboarding. The four-tasks placement question is real and needs resolution.

---

## Session 8 changes summary (vs session 6 doc)

- 12 screens → 8 screens (plus the four-tasks screen still to be placed).
- Partner fork removed. Solo is the default outcome of onboarding.
- Partner-entry screens (old 7-9) removed.
- Solo confirm screen (old 10) removed.
- Manual fallback flow removed.
- Invite link flow (old A1-A7) removed entirely.
- MOTD moved into onboarding (was explicitly excluded in session 6).
- Theme depth moved into onboarding (was deferred to day-7).
- Leader pick now has a curated launch starter subset with themed companion bundles.
- Theme assembly theatre added as a transition moment.
- Themed companion pool reveal added (screen 5b).
- MOTD tutorial coin loop added.
- Coin blitz on closing screen added.
- Library sharing plant added (screen 7).
- Server endpoints: `join_by_link` and `join_by_values` collapse into a single `join_by_values`. No URL parameters, no codes.
- Universal links / app links / install-referrer SDK decisions: no longer relevant. The pickle-moon-public AASA / assetlinks files become optional (deep-link support for marketing URLs, not pairing).
- Ambiguous-match resolution UX added (ask partner to vary username temporarily).
- Calendar tour overlay removed (old screen 12). Replaced by per-screen-context introductions across screens 5b, 6, 7, 8.
- "Add my partner" button now lives on the partner calendar UI as a persistent action, not in the partner-entry onboarding screens (which no longer exist).
