# Four Tasks — Morning Sequence Design Notes

**Status:** LOCKED for v1.0. Implementation tile 4.6 (full coordinator). Related: 4.4 (slot machine), 4.5 (MOTD reroll cost), 4.7 (stamp slap), 4.8 (rest day), 4.9 (streak popup).

The claim endpoint this doc designed shipped in tile 1.3 (session 12) as `POST /users/:user_id/claim`. The endpoint contract below is corrected to match what shipped; see `four_tasks_system_map.md` §5.3 for the live contract.

The morning sequence is the centrepiece of the daily experience — the ritual that fires once per day on the user's first app-open, when there's a pending payout to claim.

---

## What it is

A ceremony that fires once per day, on first app-open, when a past day is sealable. It is **not a mode** the app enters; it is an **overlay-on-top** of the normal calendar view. Base state is always "calendar visible, no trays open, full control." The sequence is what happens *to* the user when they open at the right moment.

The user loses control deliberately for ~15-30s. The app:
1. Pays out yesterday's coins, visually.
2. Stamps yesterday with a tier-coloured judgement and message.
3. Announces today's MOTD, typewriter-glitching it into its header position.

Then control returns and the day begins. The lockout is load-bearing: the moment is about acknowledgement, not action. The user is being *given* a moment, not driving one.

---

## The beat-by-beat

The prototype implements approximately this. The Godot port replicates it with a cleaner state machine, the atomic server claim, and the slot-randomisation fix (see MOTD structure).

1. User opens app. Calendar visible. No trays open. (Q3 — calendar always opens flat; sequence is overlay-on-top.)
2. App asks the server to claim (see Claim endpoint). If nothing is sealable, no sequence — control retained immediately. If a day seals, the response drives the rest of the sequence.
3. Yesterday's day cell is tapped automatically. Tray slides out for that day, normal tray timing.
4. Tray reaches position, unlocks from docking, wobbles briefly to draw focus.
5. Tray flies up, detached, to screen centre. Brief pause.
6. Each completed task jiggles in sequence. Coins fly out, blitz onto the counter. Completed checkboxes green-tick; uncompleted get a strike-through + red cross. (Rest-day variant differs — see Q6.)
7. A stamp comes down and seals the day. Colour matches tier:
   - 1 task → red ("woops" pool)
   - 2 tasks → orange ("solid effort" pool)
   - 3 tasks → yellow ("not bad" pool)
   - 4 tasks → green ("hell yeah" pool)
   - rest day → purple (rest pool)
   Message typewriter-glitches into the tray as part of the stamp moment.
8. Tray slides back to docked position, stamp + message visible.
9. The MOTD picker appears (centred), auto-spins a new MOTD, settles. Brief pause.
10. The day's MOTD typewriter-glitches into its header position (the subtitle slot above the streak bar). This is the day's *fingerprint*, visible all day until tomorrow's sequence replaces it.
11. Yesterday's tray slides down. Today's tray slides out, signalling the day has begun.
12. User regains control.

Total: ~15-30s target. Tune timing at implementation against feel — long enough to feel ceremonial, short enough not to annoy.

---

## MOTD structure (Q1)

Four spinners: **adverb + verb + noun + emoji** (prototype: "eerily" + "ignored" + "the mind" + "👽").

The emoji slot reads from the *same pool* as the user's day-cell completion stickers — `users.active_stickers`, managed via tap-to-toggle in the picker (theme doc). Change your active pool and your MOTD emoji changes along with your completed-day stickers. Single source of truth.

**MOTD font** is locked global (theme doc, session 7) — same typography across all themes; only the emoji slot changes per pack. Brand-voice consistency over per-theme expressive typography.

**Prototype bug to fix (tile 4.4):** the prototype's spinners visibly *alternate* when spun. The port rolls each slot independently, no alternation pattern.

---

## Crash-resistance (Q2)

The sequence is **purely cosmetic**. The server's coin balance and seal state are updated by the claim transaction as data, not derived from the animation playing. The animation is the user's *experience* of the payout, not the payout itself.

Because the claim endpoint is atomic and idempotent (a sealed day is excluded from the next walkback), the failure cases all degrade safely:

- **Crash before claim fires:** nothing sealed yet. Next open re-runs the walkback, finds the same eligible day, replays the full sequence. No loss.
- **Crash after claim commits, before the animation finishes:** the day is sealed and coins are banked server-side. Next open's walkback finds nothing eligible → no sequence. The user sees yesterday already stamped and today's MOTD in place. They missed the ceremony but the data is correct. Least-good case, still consistent.

The "infinite coins" mode (crash at the right moment to get paid twice) is structurally impossible — once a day's `sealed_at` is set, the walkback filter `sealed_at IS NULL` excludes it permanently.

---

## Base state is always calendar-flat (Q3)

The first frame on open is the calendar in its normal interactive state. No trays, no animation in progress. THEN the sequence fires (if a day sealed), as an overlay layered on that base state. When it ends, the user is back at the same base state, plus yesterday's stamp and today's MOTD.

Cleaner than "boot into morning-sequence mode if pending, else normal mode" — same code path always.

Implications:
- Loader (tile 2.1) handles identity + routes to App. Loader does NOT check for a pending payout.
- App boots, calendar paints, then a separate check fires the claim. A sealed day in the response starts the sequence; a null `sealed_day` does nothing.
- Mid-sequence backgrounding: the state machine decides resume/restart/skip (see Interruption model).
- Late-in-the-day opens: already claimed earlier → null `sealed_day` → standard calendar.

---

## Partner panel during the sequence (Q4)

Your morning sequence is private — the partner never sees it. The user is locked out during the sequence, so swipe is disabled and the partner panel is invisible during your ceremony.

More importantly, your sealed state is invisible to the partner **until you claim**. Until your claim commits, your previous day shows unstamped on their partner panel; their polling (tile 2.8, 30s) picks up the stamp within 30s of your claim. Conversely, if they claimed at 6am and you open at 7am, you immediately see their sealed yesterday because their claim already fired.

**Critical bug to avoid:** the prototype had a "prestamped" bug where days appeared stamped to the partner before the user had claimed. Partners must NOT see your stamp before you've experienced your own ceremony. The claim endpoint being the moment state-becomes-visible enforces this.

This "dead until claimed" property is good design, not a limitation — state moves when the partner moves it, like a "delivered → read" indicator. A private moment in a shared system.

**Polling implication:** tile 2.8 must refresh day-cell state, not just metadata, or the morning-reveal pattern doesn't surface on the partner side. (Shipped — the poll selective-merge diffs day rows.)

---

## Multi-day walkback (Q5)

"What if the user is away for a week?" — there is no backlog of payouts.

The data model enforces this for free: a day is only sealable if tasks were ticked (or it's a rest day), and task-ticking only happens on a day the user had the app open. Days during an absence are empty — never ticked, never sealable. So "away for a week" produces at most **one** pending payout: the most recent day they actually ticked something.

**The walkback** (runs inside the claim transaction): from the request's `local_date`, find the most recent past day where `sealed_at IS NULL AND (rest_day = 1 OR tasks_done != '[false,false,false,false]')`. That's the day to seal. If none exists, no sequence fires. App-open semantics mean at most one such row exists at a time.

**Streak consequences:** empty days during an absence break the streak (no four-of-four). Rest days cover the gap only if pre-placed. The ceremony on return is for the *last actually-played day*; the user is dropped into today with the streak in whatever state the gap left it. No "you broke your streak" announcement — the dropped counter reveals it naturally.

**Push notifications** (a daily nudge) would partially solve absence. Deferred to v1.x. v1.0 ships "no open, no play" as a feature.

---

## Rest day variant (Q6)

Rest days play the SAME sequence with content swaps — same beats, timing, lockout. Only the content pools differ. The state machine doesn't branch on rest-vs-worked; it loads different content. (Clarity over cleverness — one parameterised sequence, not two parallel ones.)

Differences vs a worked day:
- **Beat 6:** tasks jiggle, but labels typewriter-glitch into rest-vocabulary phrases ("relaxed", "had a cheat meal", "called your mum"). Checkboxes get purple ticks (or hearts/hyphens — art decision, 4.7). No coins fly — rest cost was pre-paid when the day was designated (4.8).
- **Beat 7:** purple stamp (sacred semantic — not themed against palette). Message from the rest pool, tone is affirmation of the choice to rest ("good on you for the rest", "well-deserved", "soft day, well chosen"), not celebration of effort.

**New content to author:** rest-day task labels (~15-30 phrases, four drawn per rest day; separate pool from onboarding's example tasks — 4.8) and rest-day stamp messages (small affirmation pool — 4.7/4.6).

---

## MOTD reroll cost (Q8)

**Locked:** the first reroll (the auto-roll during the sequence) is free. Each subsequent reroll costs a flat random **90-110 coins**, no escalation. The 90-110 spread (vs a flat 100) adds a tiny "what'll it be" moment per press for free.

This supersedes the prototype's doubling-cost mechanic. Doubling was wrong here: uncapped, an inattentive user could burn ~8000 coins in ~12 presses, an entire month's payouts on one annoyed evening. MOTD reroll is a **vent**, not a sink — small spend to express a preference about transient text (today's MOTD, gone tomorrow). The real sinks are sticker packs (300-600/pack per monetisation v2.0), rest days (pre-paid), and coin name rerolls (post-v1.0).

**General principle from this:** doubling cost belongs on features whose output should ESCALATE in significance (coin name — a persistent identity element); flat cost belongs on features that should stay casual (MOTD).

**Open:** whether the 90-110 range scales with the user's task multiplier (proportional pain across progression) or stays flat forever (high-streak players earn the casualness). Either is defensible — decide at tile 4.5 when the broader economy is tuned.

---

## Claim endpoint (shipped tile 1.3)

`POST /users/:user_id/claim`, body `{local_date: "YYYY-MM-DD"}`. The walkback finds what's sealable; there is no server-side "payout due" flag. Sealing happens inside the claim transaction (lazy seal-on-open per the timezone doc) — no cron.

Transaction:
1. Walkback (above) finds the most recent unsealed eligible day for this user. If none, return 200 with `sealed_day: null` and the user row unchanged. No writes.
2. Derive the stamp tier from the day's `tasks_done` count, or purple if `rest_day = 1`.
3. Pick a random message from the tier's server-held pool.
4. Coin payout: 0 on rest days; otherwise each completed task rolls `[900, 1400]` and the rolls sum. Server-side randomness — the client can't predict it.
5. Streak: green → +1; purple (rest) → unchanged; anything else → reset to 0. `longest_streak = max(longest_streak, new streak)`.
6. Theme snapshot: copy `users.active_theme` into the day's `day_theme_state` (immutable past — drives the historical theme renderer per monetisation v2.0).
7. Single atomic batch updates the day row (`stamp`, `sealed_at`, `day_theme_state`) and the user row (`coins`, `lifetime_coins`, `streak`, `longest_streak`).

**Response:** `{sealed_day, user}` — `sealed_day` carries the sealed day row (stamp, tier, etc.), `user` carries the updated user row (coins, streak). The client drives the animation from this. The sequence is cosmetic; the data is the truth.

**Idempotency:** a second claim after the first finds the day already sealed (`sealed_at` set), walks past it, returns `sealed_day: null`. No double-payout.

**Today's MOTD storage:** the MOTD the user saves during the sequence writes to **today's** `days.motd` via the normal day-write path — no `users`-level field, no schema change. It stays mutable all day (that's the reroll mechanic). When the next day opens and this day seals, `motd` is frozen by the same `sealed_at` gate that freezes `tasks_done` and `rest_day` (`PUT /days/:date` returns 409 on a sealed day). The MOTD then lives permanently in that day's record.

**Two surfaces, two values (don't conflate them):**
- The **panel header subtitle** is *today-anchored* — it always shows today's MOTD (the live daily fingerprint), regardless of which day's tray is open. Falls back to yesterday's only in the boot-before-first-sequence edge. (Shipped in the Godot port, tile 2.A.)
- The **tray subtitle** is *per-day* — each day's tray carries its own `days.motd` in a subtitle slot beneath the day label, mirroring the panel-heading title+subtitle block. Populated only once that day has a saved MOTD; empty on an unsealed today before the MOTD is set. (Confirmed in the PWA prototype; NOT yet built in the Godot port — the port's tray currently renders task rows only per system map §7.5. Lands with tile 4.6 / the morning sequence work.)

So on a given day the header and that day's tray show the same string, but opening a *past* day's tray shows *that* day's sealed MOTD while the header keeps showing today's. The MOTD is NOT on the cell face — the cell shows the stamp; the MOTD is header + tray. Cell-vs-tray split locked session 14.

**Client call timing — locked Option A:** the claim fires at the start of beat 3 (as yesterday's tile is auto-tapped). The server response is authoritative *before* any payout animation plays, so the client never animates something the server might disagree with. On network/500 failure the client retries on next open — nothing sealed, sequence replays. Finalise at tile 4.6.

---

## Interruption model

If the user backgrounds mid-sequence, the claim-at-beat-3 timing makes it collapse cleanly:
- **Claim already committed before backgrounding:** day is sealed. Foregrounding finds nothing eligible → no replay. The user sees the calendar with yesterday stamped and today's MOTD in place. Missed the moment, data correct.
- **Claim not yet fired (opened then backgrounded within ~200ms):** nothing sealed. Foregrounding replays from the start.

In practice the claim fires ~1s after open, so the "background before claim" window is tiny. Degradation: network/500 failure on claim → nothing sealed → replay on next open. Background during the pre-claim partial animation → cancel-and-replay or queue-and-finish (implementation choice, 4.6).

---

## Rotation window property

The sequence's locked-out window has been called "the natural rotation window" in the pair-key doc (pair-key changes were formerly called "migrations"; terminology locked to "rotation" session 10). This doc constrains it with numbers.

During the sequence the user has no interactive controls, so the server has predictable uninterruptible time for identity-tied writes.

Budget: ~15-30s of uninterruptible animation. What fits comfortably: the claim transaction (<100ms), one pair-key rotation (<1s), partner-side polling catch-up. What doesn't fit: bulk operations (don't bundle weeks-of-absence processing here), user-facing rotation UX moments (if it needs to explain itself to the user, that's outside the sequence).

Hard upper bound: if an operation can't complete in the window, either split it pre-claim/post-claim or accept a follow-up sync after control returns. Don't treat the window as a magic bucket — it's a real budget, measure and respect it.

---

## Related tiles

- **4.4** slot machine — MOTD spinner; fix the alternating bug, randomise truly.
- **4.5** MOTD reroll cost — flat 90-110, supersedes doubling (Q8).
- **4.6** morning sequence coordinator — this doc is the reference.
- **4.7** stamp slap — the stamp-down beat; tier colours + message pools authored here.
- **4.8** rest day — sets up the Q6 variant; rest-day task pool authored here.
- **4.9** streak popup — fires AFTER the sequence if a milestone was hit; layered after, not part of it.
- **1.3** claim endpoint + lazy seal — shipped (session 12).
- **2.8** partner polling — picks up day-cell state after the user claims (shipped).
- **4.16** partner reactions — DEFERRED to v1.x (see below).

---

## What this doc leaves open

Deferred to tile 4.6 implementation:
- Per-beat timing values (wobble, pauses, total length).
- Stamp message pools (red/orange/yellow/green/purple, ~10-20 each). Content authoring.
- Rest-day task label pool (~15-30) and rest-day stamp pool. Content authoring.
- MOTD reroll price range tuning + whether it scales with multiplier (Q8 open).
- Background-mid-sequence handling (cancel+restart vs queue+finish).

---

## Partner reaction surfacing (Q7) — DEFERRED with the feature

> Partner reactions are deferred to v1.x (tile 4.16; doc moved to `deferred/` at session 12). This section is the surfacing design for when the feature lands — NOT v1.0 spec. Retained because it locked a principle that affects how partner data surfaces generally.

**Locked principle:** partner reactions are TIME-AGNOSTIC, not event-driven. A reaction is a backlog feature, not fresh news — either side engages with the backlog whenever, in any quantity, or not at all. So: no push when a partner reacts, no morning-sequence beat for "your partner left this", no urgent badge.

Instead, a benign ambient marker on past-day cells, symmetric in both directions:
- **Your view of their panel:** their sealed days you haven't reacted to get a soft marker ("you can do something here if you want"). Clears when you react. The reaction itself stays on the day.
- **Their view of your panel:** your days that received a reaction get a marker drawing your eye. Clears once viewed. The reaction stays forever.

Both directions use the same quiet visual vocabulary — asymmetric handling would feel off. A cell could carry both states at once (you haven't reacted AND they reacted to it); the UI resolves that at tile 4.16 (different sides/shapes, or border-vs-mark). Visual style is an art-tile decision (NOT a red dot, NOT a notification "1" badge).

When it ships, the tutorial reveal is scheduled (staggered disclosure), not purely trigger-gated, so the user has the vocabulary to notice the marker. This supersedes the push-based "Surfacing reactions to the receiver" section in the partner reactions doc.

---

## Related docs

- `four_tasks_pair_key_design_notes.md` — identity model; the rotation-window property is constrained here.
- `four_tasks_system_map.md` — live claim endpoint contract (§5.3) and seal invariants.
- `four_tasks_timezone_and_sealing_design_notes.md` — seal-on-open; shares the claim transaction surface.
- `four_tasks_staggered_disclosure_design_notes.md` — partner reactions reveal (deferred) lives in that schedule.
- `four_tasks_theme_design_notes.md` — Q1 MOTD emoji slot reads `active_stickers`; MOTD font locked global.
- `four_tasks_monetisation_position.md` — theme snapshot drives the immutable-past historical renderer.
- `four_tasks_architectural_preference.md` — one parameterised sequence for rest-vs-worked (Q6) follows from clarity-over-cleverness.
