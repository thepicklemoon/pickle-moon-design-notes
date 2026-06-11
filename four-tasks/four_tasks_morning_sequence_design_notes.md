# Four Tasks — Morning Sequence Design Notes

**Status:** LOCKED for v1.0. Implementation tile 4.6 (full coordinator, shipped s26-28). Related: 4.4 (slot machine), 4.5 (MOTD reroll cost), 4.7 (stamp slap), 4.8 (rest day), 4.9 (streak popup), 4.25 (span-sealing). Interruption model rewritten session 29 (freeze/resume — see that section); trigger hardened to per-local-day semantics the same session. Beat 0b stage normalization added s31. STREAK RESCUE GATE added s30, AMENDED s32 with WALKBACK SPAN-SEALING — tile 4.25, doc-locked ahead of build. The Q5 walkback section and the rescue gate section describe the 4.25 TARGET; deltas vs the shipped wire are flagged inline.

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

Two pre-beats were added after the original numbering locked, kept as a 0-series to avoid renumbering the beats referenced throughout (5a, 6, 7, 9):

- **Beat 0b — stage normalization (s31).** Runs in the coordinator immediately after the trigger raises App's input blocker, BEFORE the claim fires: close any open tray and `await` CLOSED, serialized deliberately (running it parallel to the claim invites a mid-close race). `show_for_date` is a TOGGLE — a tray already open ON the seal date would make beat 3's awaited `opened` never fire, deadlocking the coordinator with the blocker stuck (found by device ledger D2; never seen in five sessions of testing because DevKit always reboots through Loader, a clean stage). App's side of the same normalization: `_snap_to(0)` on `started` (the ceremony must not play off-screen if the app was backgrounded on the partner panel), and the day-carry's deferred tray re-open is gated on the ceremony being active.
- **Beat 0 — streak rescue gate (s30, amended s32).** When the claim returns `pending_rescue` instead of sealing, the rescue dialog presents over the flat calendar — PRE-ceremony, so the ceremony lockout stays acknowledgement-not-action and the decision is taken before any choreography. The resolve response is claim-shaped and drives beats 3+ exactly as a plain claim would. See STREAK RESCUE GATE below.

1. User opens app. Calendar visible. No trays open. (Q3 — calendar always opens flat; sequence is overlay-on-top.)
2. App asks the server to claim (see Claim endpoint). If nothing is sealable, no sequence — control retained immediately. If the response is `pending_rescue`, beat 0 runs first. If a day seals, the response drives the rest of the sequence.
3. Yesterday's day cell is tapped automatically. Tray slides out for that day, normal tray timing.
4. Tray reaches position, unlocks from docking, wobbles briefly to draw focus.
5. Tray flies up, detached, to screen centre. Brief pause.
5a. **The applied coin-bonus multiplier typewriter-glitches into the tray subtitle slot** (beneath the day label): `coin bonus = ×1.4`. This is the AUTHORITATIVE multiplier the server applied at seal — returned in the claim response (`sealed_day.multiplier`), NOT the live preview pill's reading. It holds through the payout beat so the user can read it while the coins fly. (Why this exists: the coin payout is deliberately obfuscated — the user can't reverse-engineer the multiplier from a fuzzy coin delta. And the live preview pill on the partner panel reads the partner's *current* day, which at morning-open is usually empty or mid-completion, so it gives the WRONG number for the payout that just landed. The only honest place to state "this is the multiplier you actually earned" is here, at the payout, sourced from the seal. ×1.0 is still shown — an honest "no bonus this time" — so the line is always present, never conditionally hidden.)
6. Each completed task jiggles in sequence. Coins fly out, blitz onto the counter. Completed checkboxes green-tick; uncompleted get a strike-through + red cross. (Rest-day variant differs — see Q6.)
7. A stamp comes down and seals the day. Colour matches tier:
   - 0 tasks → **grey** (the disappointment tier; pool per the stamp-tier doc). The "you showed up and did nothing" stamp; a day you opened is a day you're judged on.
   - 1 task → red ("woops" pool)
   - 2 tasks → orange ("solid effort" pool)
   - 3 tasks → yellow ("not bad" pool)
   - 4 tasks → green ("hell yeah" pool)
   - rest day → purple (rest pool)
   The **stamp message** glitches in on the stamp itself (tier-coloured pool, see Claim endpoint). At the same moment, **the sealed day's own MOTD glitches into the tray subtitle slot, OVERWRITING the bonus line** from 5a — one subtitle Label, used twice: the bonus line first, then the day's MOTD as the stamp lands. The glitch churn consumes the old text as it resolves the new (a single overwrite, no glitch-out pass). This is the per-day diary MOTD (yesterday's, the day being reviewed) per the Diary/header split below — NOT the new MOTD the picker spins at beat 9. Two distinct strings: stamp message ≠ MOTD (both ≠ the beat-9 new MOTD). The claim response returns the sealed day's `motd` so the client has it without an extra read.
8. Tray slides back to docked position, stamp + message + MOTD subtitle visible.
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
- "Calendar-flat" is normalized, not assumed (s31): on a same-process focus-in trigger the stage may NOT be flat — a tray can be open, possibly on the seal date itself, and the app may be resting on the partner panel. Beat 0b + App's snap-home (see beat-by-beat) force the base state before the claim fires, rather than trusting it.
- Mid-sequence backgrounding: the sequence freezes and resumes where it left off (see Interruption model).
- Late-in-the-day opens: already claimed earlier → null `sealed_day` → standard calendar.

---

## Partner panel during the sequence (Q4)

Your morning sequence is private — the partner never sees it. The user is locked out during the sequence, so swipe is disabled and the partner panel is invisible during your ceremony.

More importantly, your sealed state is invisible to the partner **until you claim**. Until your claim commits, your previous day shows unstamped on their partner panel; their polling (tile 2.8, 30s) picks up the stamp within 30s of your claim. Conversely, if they claimed at 6am and you open at 7am, you immediately see their sealed yesterday because their claim already fired.

**Critical bug to avoid:** the prototype had a "prestamped" bug where days appeared stamped to the partner before the user had claimed. Partners must NOT see your stamp before you've experienced your own ceremony. The claim endpoint being the moment state-becomes-visible enforces this.

This "dead until claimed" property is good design, not a limitation — state moves when the partner moves it, like a "delivered → read" indicator. A private moment in a shared system.

**Polling implication:** tile 2.8 must refresh day-cell state, not just metadata, or the morning-reveal pattern doesn't surface on the partner side. (Shipped — the poll selective-merge diffs day rows.)

---

## Walkback span-sealing (Q5 — AMENDED s32, tile 4.25)

"What if the user is away for a week?" — there is no backlog of payouts, and as of the s32 amendment there is no backlog of *judgement* either: the claim adjudicates the entire span.

A day is engageable-with via **`accounted_for`** — a write-once boolean set the first time the user deliberately engages with that date (opening the app on it, ticking a task, saving a MOTD, or designating it a rest day — all four implemented server-side as of session 29: the claim's app-open upsert, writeDay's upsert (task tick / MOTD save; the gap closed session 29), the rest endpoint, and createUser's day-0 row). Days during an absence are never engaged with, so they're never `accounted_for` — they carry no payout. But they are still *days*, and the streak rule (economy doc) says zero-task days break the chain whether opened or not. Span-sealing is where that judgement actually lands.

**The span walk** (runs inside the claim transaction. DOC-LOCKED s32, ships at tile 4.25 — until it lands, the wire runs the SUPERSEDED single-target walkback recorded below): from the request's `local_date`, take every calendar day after the most recent sealed day and strictly before `local_date`, oldest first. Every day in the span ends terminal:

- **Accounted day** → seals at its natural tier. The most recent of these is the **payout target**: it pays per the economy doc and is the `sealed_day` the ceremony presents. (The rare shadowed accounted day earlier in the span — it takes two consecutive claim failures across separate days to create one — seals at its natural tier WITH its natural payout, silently batched into the same claim; economy doc, Earn. The user did the work and keeps the coins; the ceremony still presents only the most recent day.)
- **Never-opened day** → seals **grey**: `accounted_for` stays 0 (honest — they never engaged), zero payout, no ceremony. The grey seal is the judgement of the hole, rendered on the calendar (cell treatment: tile 4.24 [DECIDE]), not a moment. "Batch-seal grey silently."
- **Streak math runs per-day in walk order** (economy doc is authority): ≥1 task advances, rest holds, a zero-task day zeroes the streak at the day it happened; later days in the walk build from the post-break value.
- **Rescue interception:** before any of the above seals, the pre-scan runs (STREAK RESCUE GATE below). If it fires, the claim returns `pending_rescue` and NOTHING in the span seals until resolve.
- No sealed day exists yet (fresh install) → the span starts at the oldest existing day row. No judgement of pre-install days.
- Empty span and no accounted target (the normal same-day re-open) → `sealed_day: null`, no writes beyond the app-open upsert.

**Why first contact matters — the wart this kills:** under the superseded mechanism, holes were only checked *behind the payout target*, so judgement of a gap waited for the NEXT seal. Skip Tuesday, open Wednesday: Wednesday's claim found nothing sealable and said nothing; the streak displayed live all day; Thursday's claim then broke it retroactively — the streak advanced past a hole on day N and collapsed on day N+1. Under span-sealing, Tuesday is adjudicated Wednesday morning, rescue offer included if applicable — possibly on a claim that pays nothing. The streak the user sees is always already adjudicated, and history is **contiguous**: every day between install and the last seal ends terminal.

**SUPERSEDED single-target walkback (s12 → s31, recorded for history):** find the most recent past day where `sealed_at IS NULL AND accounted_for = 1`; seal only that day; detect gaps via date-adjacency against the most recent previously-sealed day at the moment of that seal. Two faults: (a) the retroactive collapse above; (b) an older stranded accounted day was shadowed *forever* — each day produced one new unsealed accounted row and the claim consumed only the most recent, so the earlier claim in this doc that the backlog "drains most-recent-first, one per claim, self-correcting" was wrong in steady state. Span-sealing retires both.

**Why `accounted_for` (and not a tasks_done string-match) — unchanged:** the original predicate `tasks_done != '[false,false,false,false]'` was fragile — it depended on the exact JSON serialisation of the array, so any drift in how `tasks_done` was written would mis-seal or 500 — and it wrongly excluded legitimately-engaged days where the user ticked nothing. `accounted_for` is a boolean set once on first engagement and never cleared; **it is monotonic once the day has arrived** — you can untick every task, but you cannot un-account a day you showed up to. A day you open and complete nothing on still seals as a **disappointment** (grey tier), with its own ceremony. You can't dodge judgement for a day you turned up to; and as of s32 you can't dodge it for a day you didn't, either — the only difference is the no-show seals silently.

**Streak consequences on return from absence:** gap days seal grey at first contact and the break lands then, with the rescue gate as the one escape hatch (single miss only — see the gate). No "you broke your streak" announcement beyond the rescue dialog when it applies; the dropped counter reveals it naturally. The ceremony on return is for the *last actually-played day* if one is in the span; a pure-absence return seals its greys silently, presents nothing, and drops the user into today.

**Push notifications** (a daily nudge) would partially solve absence. Deferred to v1.x. v1.0 ships "no open, no play" as a feature.

---

## Streak rescue gate (added s30, amended s32 — tile 4.25)

The RULES — eligibility, price, no frequency cap, the exactly-one pre-scan — live in `four_tasks_economy_redesign_notes.md` (Streak rescue), which is the AUTHORITY. This section is the mechanism.

**Two-phase claim.** When the span walk's pre-scan finds the streak (≥1) about to break and exactly ONE zero-task day in the span, the claim seals NOTHING and returns pending:

```
{ ok: true, data: {
    sealed_day: null,
    pending_rescue: {
      rescue_date,            // the zero-task day a purchase converts/creates as rest
      kind: "grey" | "gap",   // accounted-grey vs never-opened
      target_date,            // the payout target's date — null when no accounted day follows (s32)
      target_tier,            // its would-be tier — null likewise (s32)
      streak_at_stake,
      cost },                 // = economy payload rescue_cost
    user } }
```

s32 delta vs the shipped s30 shape: `target_date`/`target_tier` become **nullable** — first-contact anchoring means a rescue can fire on a claim with no payout target at all (the first open after a single skipped day). The s30 shape had them always set because the offer could only ever fire in front of a target.

**Resolve.** `POST /users/:user_id/claim/resolve`, body `{accept: bool, local_date}`. Fully re-entrant: re-runs the span walk + pre-scan from CURRENT state, never trusting the client's memory of the offer — if re-evaluation no longer yields a rescue (already sealed, day edited, second deficit appeared), it degrades to plain-claim behaviour. Decline → the zero-task day seals grey; accept → 409 `insufficient_coins` below `rescue_cost`, otherwise the day converts (grey kind) or is created (gap kind) as a sealed rest day with the debit. Either answer seals the WHOLE span in one atomic batch — rescue day, any other greys, the payout target, the payout, and the user update share the transaction (and the seal rows share one `sealed_at`, the wire-provable atomicity marker). The response is claim-shaped; the client treats it exactly like a claim response.

**One-shot, stateless.** Both answers end with the breaker sealed; `sealed_at` is the only memory — there is nothing to re-offer and no offer state tracked anywhere (the house rule: the server re-derives, never remembers). Pending = nothing sealed, so app death mid-dialog self-heals into a re-offer on the next open. Not a loophole: re-offering an unanswered question is correct.

**Beat placement + dialog (built s30, device-verified s31).** The dialog is **beat 0**: pre-ceremony, over the flat calendar — the ceremony lockout stays acknowledgement-not-action, and the decision is taken before any choreography starts. `rescue_confirm` follows the `rest_confirm` pattern with two deliberate deviations: the **scrim does NOT cancel** (declining is irreversible; no accidental answers) and **every exit is an answer**. It mounts ABOVE App's input blocker (later in the tree + z-index 300) — the blocker is already up when the dialog appears, and the dialog is the one interactive surface allowed through it. `State` routes `pending_rescue` to its own signal (`claim_rescue_pending`); the ceremony coordinator's parked one-shot `morning_claimed` connection picks up the resolve's claim-shaped response, so `morning_sequence.gd` carries zero rescue awareness — the rescue path and the plain path are one code path from the coordinator down. A failed resolve re-arms the per-day claim marker (retry on next focus-in; the server sealed nothing).

**Client deltas owed at tile 4.25** (the shipped s30 client is built to the s30 shape):
- Handle null `target_date`/`target_tier` — dialog copy without the "and yesterday seals as ..." framing, and accept that a successful resolve may return `sealed_day: null` (rescue-only morning, no ceremony follows).
- Re-verify the W13/W14 wire pair against the span shape (wire_regression_pack.md carries the re-spec).

---

## Rest day variant (Q6)

Rest days play the SAME sequence with content swaps — same beats, timing, lockout. Only the content pools differ. The state machine doesn't branch on rest-vs-worked; it loads different content. (Clarity over cleverness — one parameterised sequence, not two parallel ones.)

Differences vs a worked day:
- **Beat 6:** tasks jiggle, but labels typewriter-glitch into rest-vocabulary phrases ("relaxed", "had a cheat meal", "called your mum"). Checkboxes get purple ticks (or hearts/hyphens — art decision, 4.7). No coins fly — rest cost was pre-paid when the day was designated (4.8).
- **Beat 5a (bonus line):** SKIPPED on a rest day. A rest day pays 0 coins, so there is no multiplier to display (`×anything` of 0 is 0). The subtitle slot stays empty until beat 7 fills it with the rest day's MOTD. Showing a bonus line on a zero-payout day would be incoherent.
- **Beat 7:** purple stamp (sacred semantic — not themed against palette). Message from the rest pool, tone is affirmation of the choice to rest ("good on you for the rest", "well-deserved", "soft day, well chosen"), not celebration of effort.

**New content to author:** rest-day task labels (~15-30 phrases, four drawn per rest day; separate pool from onboarding's example tasks — 4.8) and rest-day stamp messages (small affirmation pool — 4.7/4.6).

---

## MOTD reroll cost (Q8)

**Locked:** the first reroll (the auto-roll during the sequence) is free. Each subsequent reroll costs a flat random **90-110 coins**, no escalation. The 90-110 spread (vs a flat 100) adds a tiny "what'll it be" moment per press for free.

**Shipped note (s29):** the server half shipped with `REROLL_COST = 100` as a flat placeholder, delivered via the economy payload (`reroll_cost`) — the 90-110 client-side spread and any final number land with the 4.5 client half. The fee-is-vent-not-protection rationale is recorded at the constant.

This supersedes the prototype's doubling-cost mechanic. Doubling was wrong here: uncapped, an inattentive user could burn ~8000 coins in ~12 presses, an entire month's payouts on one annoyed evening. MOTD reroll is a **vent**, not a sink — small spend to express a preference about transient text (today's MOTD, gone tomorrow). The real sinks are sticker packs (20k-100k per `four_tasks_economy_redesign_notes.md` — the old 300-600 figure was the superseded early sketch), rest days (pre-paid), and coin name rerolls (post-v1.0).

**General principle from this:** doubling cost belongs on features whose output should ESCALATE in significance (coin name — a persistent identity element); flat cost belongs on features that should stay casual (MOTD).

**Open:** whether the 90-110 range scales with the user's task multiplier (proportional pain across progression) or stays flat forever (high-streak players earn the casualness). Either is defensible — decide at tile 4.5 when the broader economy is tuned.

---

## Claim endpoint (shipped tile 1.3; span-sealing target = tile 4.25)

`POST /users/:user_id/claim`, body `{local_date: "YYYY-MM-DD"}`. The walkback finds what's sealable; there is no server-side "payout due" flag. Sealing happens inside the claim transaction (lazy seal-on-open per the timezone doc) — no cron. `local_date` is bounded to ±2 days of server UTC (400 `bad_date` outside it) — the coin-source bound, s31; see the timezone doc.

Transaction (as 4.25 specifies; the shipped wire still runs the superseded single-target step 1 — see Q5):
1. **Span walk** (Q5) gathers every unsealed day after the most recent sealed day and before `local_date`, oldest first. If the span is empty, return 200 with `sealed_day: null` and the user row unchanged. No writes beyond the app-open upsert.
2. **Rescue pre-scan** (STREAK RESCUE GATE): exactly one zero-task day in the span AND streak ≥1 → return `pending_rescue`, seal nothing.
3. Per day in walk order, derive the stamp tier from the day's completed-task count — 0 → grey, 1 → red, 2 → orange, 3 → yellow, 4 → green — or purple if `rest_day = 1`. Never-opened days are grey by construction. Pick a random message from the tier's server-held pool for each sealing day.
4. Coin payout for the **payout target** (the most recent accounted day; also any shadowed accounted day, silently): 0 on rest/grey days; otherwise each completed task rolls `[900, 1400]` and the rolls sum. Server-side randomness — the client can't predict it.
   - **Partner multiplier (BUILT, session 28).** The summed base payout is multiplied by a factor derived from the *partner's* engagement **on the same date as the day being sealed** — i.e. the sealed row's own date, read at that date. Graduated: partner 0 tasks → 1.0, 1 → 1.2, 2 → 1.3, 3 → 1.4, 4 → 1.5, **partner rest day → 1.5** (rest days impart the full bonus — a partner's rest must not penalise your earn; rest is sacred). **Guardrail: never hardcode "today" or "yesterday" here.** At seal time (the morning after) the sealed day is "yesterday," but the rule is "the partner's completion of *that day's date*," read against the row being sealed — not a fixed offset. The live preview pill on the client (tile 2.7) reads the partner's *current* day so it previews the multiplier the user is presently earning and flips when the partner completes; the server applies the final factor at seal against the sealed row's date. Same date, two read-points. **IMPLEMENTATION NOTE:** the pill (`Bonus.gd`) and the server handler MUST agree on every case, including partner-rest → 1.5 — they are the same rule at two read-points. (Session 28: handler's first cut read rest as 0 tasks → 1.0, contradicting this locked line and the pill; corrected to 1.5.)
   - **Subscription + streak (economy redesign doc, AUTHORITY).** Two-sub pairs (both `subscription_active`): the partner multiplier is boosted ×1.5 (1.5 → 2.25 cap), and a streak bonus of `+1%` per streak day is added (`round(base × mult × streak × 0.01)`), reading the streak as the span walk delivers it at the payout day — any break earlier in the walk has already zeroed it. One-sub pairs earn identically to free (tier-1 is library-only, economy-neutral). Subscribed-solo users get a phantom ×1.5 (no partner, no streak). See `four_tasks_economy_redesign_notes.md` for the full model and the order of application.
5. Streak, per day in walk order (economy doc is authority — amended 2026-06-11): ≥1 task → +1; purple (rest) → hold; zero-task (grey, opened or not) → 0. `longest_streak = max(longest_streak, new streak)` after the walk. (The original green-only/reset-on-partials rule is SUPERSEDED — recorded in the economy doc.)
6. Theme snapshot: copy `users.active_theme` into each sealing day's `day_theme_state` (immutable past — drives the historical theme renderer per monetisation v2.0).
7. Single atomic batch updates every sealing day row (`stamp`, `sealed_at`, `day_theme_state`) and the user row (`coins`, `lifetime_coins`, `streak`, `longest_streak`).

**Response:** `{sealed_day, user}` — `sealed_day` carries the payout target's row (stamp, tier, etc.) plus the **applied `multiplier`** (the factor from step 4, for the beat-5a bonus line) and the sealed day's **`motd`** (for the beat-7 tray subtitle glitch); it is null when the span held no accounted day (pure-absence return — greys sealed silently, no ceremony). OR `{sealed_day: null, pending_rescue, user}` per the gate. `user` carries the updated user row (coins, streak). The client drives the animation from this. The sequence is cosmetic; the data is the truth.

**Idempotency:** a second claim after the first finds the span empty (everything sealed), returns `sealed_day: null`. No double-payout — once a day's `sealed_at` is set it is excluded permanently.

**Today's MOTD storage:** the MOTD the user saves during the sequence writes to **today's** `days.motd` via the normal day-write path — no `users`-level field, no schema change. It stays mutable all day (that's the reroll mechanic). When the next day opens and this day seals, `motd` is frozen by the same `sealed_at` gate that freezes `tasks_done` and `rest_day` (`PUT /days/:date` returns 409 on a sealed day). The MOTD then lives permanently in that day's record.

**Two surfaces, two values (don't conflate them):**
- The **panel header subtitle** shows the **most recent MOTD on or before today**, found by walking back day-by-day until one turns up — not a fixed depth-1 fallback. On a normal day that's today's MOTD (set during the sequence); before today's sequence has run, or after a multi-day gap, it's the last day that has one. A plain depth-1 "fall back to yesterday" breaks across a 2+-day gap (yesterday is empty, so it shows nothing) — hence walk-back-until-found. The message lives on the `days.motd` of its generation day and never moves; the header simply stops pointing at it once a newer one commits. Never empty in practice: the onboarding seed sets a floor and the walkback is unbounded. (Shipped in the Godot port, tile 2.A — currently the depth-1 version; correct it to walk-back-until-found per `four_tasks_motd_design_notes.md`.)
- The **tray subtitle** is *per-day* — each day's tray carries its own `days.motd` in a subtitle slot beneath the day label, mirroring the panel-heading title+subtitle block. Populated only once that day has a saved MOTD; empty on an unsealed today before the MOTD is set. (Confirmed in the PWA prototype; NOT yet built in the Godot port — the port's tray currently renders task rows only per system map §7.5. Lands with tile 4.6 / the morning sequence work.) **During the morning ceremony this same slot is used twice (see beats 5a/7):** first it shows the applied coin-bonus multiplier, then the sealed day's MOTD overwrites it as the stamp lands. After the ceremony it settles on the MOTD — the resting state. The bonus line is ceremony-transient; the MOTD is the persistent per-day content.

So on a given day the header and that day's tray show the same string, but opening a *past* day's tray shows *that* day's sealed MOTD while the header keeps showing today's. The MOTD is NOT on the cell face — the cell shows the stamp; the MOTD is header + tray. Cell-vs-tray split locked session 14.

**Client call timing — locked Option A, finalised (4.6 + session 29):** the claim fires at the start of beat 3 (as yesterday's tile is auto-tapped), after beat 0b has normalized the stage. The server response is authoritative *before* any payout animation plays, so the client never animates something the server might disagree with. On network/500 failure the client re-arms and retries on the next app focus-in (not merely the next cold open) — nothing sealed, sequence replays. The same re-arm applies to a failed resolve (rescue gate).

**Trigger semantics (hardened session 29):** "first app-open of the day" means first per **local date**, not per process. The client triggers on application focus-in whenever the local date has advanced past the last ceremonied date — so a process resumed from recents on a new day fires the ceremony without a restart. A per-process trigger was the original implementation and silently stopped the daily loop on any device that keeps the app alive overnight (most Android phones). Accepted limitation: an app that stays *focused* across midnight waits for its next focus event; the day stays sealable regardless (writeDay accounts engagement server-side).

---

## Interruption model (rewritten session 29 — freeze/resume)

**Backgrounding mid-sequence freezes the ceremony; foregrounding resumes it exactly where it left off.** This is platform behaviour, not coordinator code: on Android, backgrounding suspends the engine's main loop — timers stop, tweens stop, the sequence coroutine freezes at its current await. Input stays blocked throughout (App's blocker persists), and the focus-in morning trigger is gated on the active ceremony, so the resume moment cannot double-fire a second run. The user doesn't miss the ceremony by glancing at a notification — they pause it.

**Process KILL while frozen** lands on the crash cases (Q2), which apply verbatim:
- **Killed pre-claim:** nothing sealed → next open replays from the start.
- **Killed post-claim:** day sealed, coins banked → next open finds nothing eligible → no replay; calendar shows yesterday stamped, today's MOTD in place. Missed the moment, data correct — the accepted small loss. (Known consequence, s31: a kill before beat 9 also permanently loses today's MOTD pick — the picker never ran. The 4.5 client half doubles as the recovery surface for this; noted on that tile.)

**There is no abort API.** The original coordinator shipped one ("background → App calls abort()"); session 29 established it could not actually stop a suspended GDScript coroutine — the "aborted" sequence would resume as a zombie at its next await and drive the tray over live input. Rather than harden it (a guard after every beat, defending against a scenario the platform already prevents by freezing), it was deleted. The earlier "cancel-and-replay or queue-and-finish" implementation choice resolves as **neither**: freeze/resume, for free, with zero added states.

No replay on a *completed* sequence — unchanged. The morning sequence is a **daily** event; replay/preservation effort is reserved for rarer, higher-stakes moments (monthly/yearly keepsakes), not a thing that recurs every morning.

---

## Post-sequence presentation queue ("beat collision")

The morning sequence is not the only thing that wants the screen on app-open. Content drops, patch notes, event popups, milestone popups (4.9), and staggered-disclosure reveals all compete for the user's attention at boot. "Beat collision" is the risk that these stack on top of, or interrupt, the ceremony. The resolution is **strict temporal separation**, not arbitration *within* the sequence:

**The morning sequence is sovereign and always first.** It owns the screen from boot, uninterruptible, until it cedes control. It is NOT a member of the queue — it is the gate everything else waits behind. Nothing is allowed into its window. (This is why there is no collision to arbitrate: nothing else is ever on screen at the same time as the sequence.) The rescue dialog (beat 0) sits inside that sovereign window, before the choreography.

**The queue runs only after the sequence cedes control.** That post-control margin — calendar flat, ceremony done — is the single place non-sequence presentations are allowed to appear.

**The trigger is "control settled to the user," fired once per app-open.** Entry points that feed it:
- After a morning sequence completes and hands control back (the normal daily first-open).
- Immediately after onboarding boots into the app (no sequence runs on the onboarding open — day-0 has no prior day to seal; see the span walk + day-0 seed).
- A pure-absence return (s32): the claim seals the span's greys silently, presents no ceremony, and control settles immediately — the queue fires then.

The only no-presentation re-open is a **same-day second open** (already claimed → span empty, idempotent). The queue has already drained earlier that day and its items are one-shot, so nothing re-shows. The queue trigger therefore does NOT depend on seal outcome — it fires when control settles, and the seal outcome is deterministic from open type.

**Queue items are simple pops, not animated.** Unlike the sequence (a choreographed ceremony), a queued item just appears, holds for a short parse delay, then arms its Continue button so the user can dismiss once they've had a beat to read it. No flight, no glitch, no choreography — these are informational surfaces, not moments.

**Ordering is a simple index.** Items present in priority order, one at a time, each dismissed before the next. If more than one is ever queued on the same open, they are designed to handle each other at that point — per-item rules authored when the items themselves exist. Not designed speculatively now; the queue is the frame, the items fill it later.

**Why this unblocks content drops.** The content-drops pipeline tile was blocked on "beat collision resolution." It is resolved: content drops are queue items, they appear *after* the sequence in the post-control margin, they never touch the ceremony's window. The pipeline can build against this contract.

---

## Rotation window property

The sequence's locked-out window has been called "the natural rotation window" in the pair-key doc (pair-key changes were formerly called "migrations"; terminology locked to "rotation" session 10). This doc constrains it with numbers.

During the sequence the user has no interactive controls, so the server has predictable uninterruptible time for identity-tied writes.

Budget: ~15-30s of uninterruptible animation. What fits comfortably: the claim transaction (<100ms), one pair-key rotation (<1s), partner-side polling catch-up. What doesn't fit: bulk operations (don't bundle weeks-of-absence processing here), user-facing rotation UX moments (if it needs to explain itself to the user, that's outside the sequence). (Span-sealing note: a week of absence is a handful of grey-seal rows in the one batch — well inside the budget; it is not a bulk operation in this sense.)

Hard upper bound: if an operation can't complete in the window, either split it pre-claim/post-claim or accept a follow-up sync after control returns. Don't treat the window as a magic bucket — it's a real budget, measure and respect it.

---

## Related tiles

- **4.4** slot machine — MOTD spinner; fix the alternating bug, randomise truly.
- **4.5** MOTD reroll cost — server half shipped s29 (flat 100 placeholder); client half doubles as the mid-ceremony-kill MOTD recovery surface.
- **4.6** morning sequence coordinator — this doc is the reference. Shipped s26-28; beat 0b added s31.
- **4.7** stamp slap — the stamp-down beat; tier colours + message pools authored here.
- **4.8** rest day — sets up the Q6 variant; rest-day task pool authored here.
- **4.9** streak popup — fires AFTER the sequence if a milestone was hit; layered after, not part of it.
- **4.25** walkback span-sealing — brings the wire to this doc's Q5 + rescue gate.
- **1.3** claim endpoint + lazy seal — shipped (session 12).
- **2.8** partner polling — picks up day-cell state after the user claims (shipped).
- **4.16** partner reactions — DEFERRED to v1.x (see below).

---

## What this doc leaves open

Deferred to implementation:
- Per-beat timing values (wobble, pauses, total length) — in-context tuning owed on 4.6.
- Stamp message pools (red/orange/yellow/green/purple + grey, ~10-20 each). Content authoring (roster-week list).
- Rest-day task label pool (~15-30) and rest-day stamp pool. Content authoring.
- MOTD reroll price range tuning + whether it scales with multiplier (Q8 open).
- Sealed-grey cell rendering (4.24 [DECIDE]) — a silently-sealed never-opened day currently renders unmarked, indistinguishable from untouched.

Resolved (no longer open):
- Beat collision — resolved by the Post-sequence presentation queue (sovereign sequence first, queued pops after control returns). Unblocks the content-drops pipeline.
- Background-mid-sequence handling — no replay on a completed sequence; it is a daily event and a missed ceremony is an accepted small loss. Pre-claim background still replays on next open (nothing sealed).
- Retroactive streak collapse — resolved by span-sealing (Q5, s32).

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

- `four_tasks_economy_redesign_notes.md` — AUTHORITY for payout, streak rule, and rescue rules; this doc carries the mechanism.
- `four_tasks_pair_key_design_notes.md` — identity model; the rotation-window property is constrained here.
- `four_tasks_system_map.md` — live claim endpoint contract (§5.3) and seal invariants.
- `four_tasks_timezone_and_sealing_design_notes.md` — seal-on-open; shares the claim transaction surface; carries the ±2d coin-source bound + sealed-beats-clock client rule.
- `four_tasks_staggered_disclosure_design_notes.md` — partner reactions reveal (deferred) lives in that schedule.
- `four_tasks_theme_design_notes.md` — Q1 MOTD emoji slot reads `active_stickers`; MOTD font locked global.
- `four_tasks_monetisation_position.md` — theme snapshot drives the immutable-past historical renderer.
- `four_tasks_architectural_preference.md` — one parameterised sequence for rest-vs-worked (Q6) follows from clarity-over-cleverness.
