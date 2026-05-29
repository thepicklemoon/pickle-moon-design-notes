# Four Tasks — Message of the Day (MOTD) Design Notes

**Status:** LOCKED for v1.0. Implementation spread across tiles: 3.6 (onboarding MOTD screen), 4.4 (slot machine — real wheels), 4.5 (reroll cost + lock/confirm), 4.6 (morning sequence reveal + memorial). The panel-header subtitle that displays it shipped in tile 2.A (session 18), with a known fallback bug reconciled at 4.6 (see Display model).

This doc pulls the MOTD design into one home. It was previously scattered: the structure + storage in `four_tasks_morning_sequence_design_notes.md` (§ MOTD structure + the claim contract's MOTD notes), the cost rule across the tile list, and the disclosure question in `four_tasks_staggered_disclosure_design_notes.md`. Authored from the session-20 capture (Item 2 / Item 3).

The MOTD is the day's **fingerprint** — a short, randomly-spun line the user generates each day, that lives in the header all day and is then pressed into that day's record as a permanent keepsake. It is the soft, expressive counterpart to the hard judgement of the stamp.

---

## What it is

One widget, three trigger contexts, one display rule.

The message is four spun slots: **adverb + verb + noun + emoji**. Prototype example: *"eerily" + "ignored" + "the mind" + "👽"*. The result is deliberately nonsensical and personal — a daily piece of flavour, not a motivational quote.

It is generated fresh each day, displayed in the panel header as the current day's fingerprint, and on the next morning's open it is sealed onto its day as a permanent memorial while a new one is spun for the new day.

---

## The widget (MOTD structure)

Four spinners, spun together:

- **adverb · verb · noun** — three text slots from server-held word pools.
- **emoji** — reads from the *same pool* as the user's day-cell completion stickers (`users.active_stickers`, managed via tap-to-toggle in the picker — see theme doc). Change your active sticker pool and your MOTD emoji slot changes with it. Single source of truth: the MOTD does not carry its own emoji set.

**Font is locked global** (theme doc, session 7). Same typography across every theme — only the emoji slot changes per pack. Brand-voice consistency over per-theme expressive type. Displayed in the secondary font (JetBrains Mono in the port) to set it apart from the UI font.

**Prototype bug to fix (tile 4.4):** the prototype's spinners visibly *alternate* when spun — a fake-random pattern the eye catches. The port rolls each of the four slots independently, no alternation.

---

## The three triggers

The same widget is reached three ways. The widget is identical; the framing and the cost differ.

### 1. Onboarding (tile 3.6, screen 6)

The full widget is handed to the user with cost-suppressed tutorial coins — capped at 750, with a "More coins" arming button that appears once the balance drops below the reroll threshold so the user can never get stuck. Spin and reroll freely until satisfied: learn-by-play. The tutorial coins are discarded on transition out of onboarding — they are not real balance.

This is where the MOTD concept *and* the fact that rerolls cost coins are taught. (See Disclosure below — this is the only place either is explicitly introduced.)

### 2. Morning sequence (tile 4.6)

As a beat of the daily ceremony, the picker appears centred, auto-spins today's new MOTD, and settles. The first reroll *within the sequence* is free (the user is being given the moment, not charged for it). The widget then leaves; the new message typewriter-glitches into the header. See the morning sequence doc for beat placement.

### 3. Anytime long-press (tile 4.5)

Long-press the MOTD subtitle on the panel to reopen the same widget outside any flow. Rerolls here cost the flat rate (see Reroll cost). This is the "I want to change today's line right now" path.

**Disclosure — locked, no dedicated reveal.** Onboarding plants the concept (the MOTD exists, rerolls cost coins); the day-2 long-press philosophy reveal makes long-press a known deliberate-decision gesture; the user infers that the live subtitle is long-pressable. The gesture is functionally live from day zero, just unadvertised until day 2 makes long-press click. This keeps the staggered schedule lean and matches the app's rule that reveals *teach*, they do not *unlock*. No dedicated "long-press your MOTD" hint (unlike the picker context-menu hint, which does get one). Revisit only if discoverability proves weak in testing.

---

## Reroll cost

**Flat 90–110 coins per press, no doubling.** Supersedes the prototype's doubling mechanic (locked S5 extended: doubling is reserved for escalating-significance features like the coin-name reroll; the MOTD is a casual feature and takes a flat cost). The reroll is a **vent, not a sink** — a flat cost means it never punishes a user for fiddling with their daily line.

- **Onboarding:** cost-suppressed (tutorial coins).
- **Morning sequence:** first reroll free, then flat rate.
- **Anytime long-press:** flat rate from the first press.

Tile 4.5 owns the lock button + confirm popup so a paid reroll is never an accident.

**Header renders on picker-close only.** The header never re-animates live during a reroll — only when the picker commits and closes. The picker is the dopamine surface (the user watches it roll); the header is the *committed* line. Re-rendering it on every reroll yo-yos the user's attention, so it is deliberately not built.

---

## Display model — carry-forward-until-seal

The header subtitle shows the **current message: the most recent message on or before today.** This is a walk-back-until-found lookup, *not* a fixed today-or-yesterday fallback.

The message is stored on the day it is generated (`days.motd`) and **never moves**. "Carry forward" is a property of the display query, not a write:

- From the moment a message is written until a later morning sequence rolls a fresh one, the header shows that message.
- The instant the morning sequence generates a new message onto today's row, the header flips to it.

**Never-empty invariant.** The subtitle is never blank in v1.0. This holds because of *two* things together:

1. Onboarding seeds the very first message, so at least one always exists.
2. The unbounded walk-back guarantees the most recent one can always be found.

Both halves are required. The onboarding seed alone does not survive a gap (the seed's day is far back); a depth-1 fallback alone (today → yesterday → blank, as the shipped tile-2.A code currently does) breaks the instant a user skips two days. Walk-back-until-found is what makes the invariant true.

**Worked gap example.** User onboards on day 0 (message M0 written to day 0's row), then doesn't open the app for a month. On day 30:

- The header walks back, skips the message-less days 1–29, lands on M0, and shows it.
- The day-30 morning sequence then generates M30 onto day 30's row; the header flips to M30.
- Days 1–29 stay genuinely message-less — there was no day for them to be the message *of*. They render as plain cells.

**Generation cadence: once per new-day open.** It is the message *of the day*. A month-long gap generates exactly one new message (today's), not one per skipped day. The morning sequence does not run thirty times.

---

## The memorial (handoff at seal)

When a day seals, the message that lived in the header all day becomes that day's **permanent memorial** — an additional keepsake on the day, alongside the stamp, that the user can look back on.

**Where it lands: that day's tray** (the tray that slides out of the cell when tapped), in the tray subtitle beneath the day label. **Not the cell face.** The cell face shows the stamp; the MOTD lives in header + tray (cell-vs-tray split locked session 14). The calendar grid never renders "this day has a message" — that axis has only two consumers, the tray and the header.

**Handoff, not a flight.** The message does not move from the header into the tray. It is *already* resident on the day's tray — it is that day's saved value — and it is shown there throughout the seal (prototype behaviour: present on the day's tray as the coins blitz, before and during the stamp). The header is showing the same string at the same time, via carry-forward. The "transferral of permanence" is the **header letting go**: once the morning sequence rolls the new message and the picker closes, the header swaps to the new one and the old message simply stays on its day, where it always lived. The data was already frozen by the `sealed_at` gate at claim time (the same gate that freezes `tasks_done` and `rest_day`) — the visible handoff is the user's *experience* of that permanence, cosmetic over an atomic freeze, same principle as the coin payout. **No flying-object animation is built.**

**Open note (4.6 polish, parked).** On a gap return there is no on-screen "yesterday" tray to anchor the moment — the memorial is simply already in its day's tray when the user navigates there. Whether a gap-return deserves its own "welcome back, here's where you left off" beat is a morning-sequence-beat question, deferred to tile 4.6 and tied to the beat-collision work.

---

## Storage and write rules

- **Field:** `days.motd`. Written via the normal day-write path (`PUT /users/:user_id/days/:date`). No `users`-level field, no schema change.
- **Mutable all day** — that is the reroll. A long-press reroll **always composes TODAY's message**, regardless of what is currently displayed in the header (which may be a carried-forward older message). It never edits a past day.
- **Frozen at seal.** When the next day opens and this day seals, `motd` is frozen by the same `sealed_at` gate as the rest of the row — `PUT /days/:date` returns 409 on a sealed day. The message then lives permanently in that day's record, consistent with the immutable-past monetisation invariant.
- **Yesterday is never editable.** Once sealed, a day's message is read-only forever.

---

## Two surfaces, two values (do not conflate)

- **Header subtitle** — the *current* message (most-recent-on-or-before-today walk-back). The live daily fingerprint, shown regardless of which day's tray is open.
- **Tray subtitle** — *per-day*. Each day's tray carries its own `days.motd`. Opening a past day's tray shows *that day's* sealed message while the header keeps showing the current one.

So on a given day the header and that day's tray show the same string; opening a *past* day's tray shows that day's frozen message while the header is unchanged.

**Build status:** the header subtitle shipped in tile 2.A but currently uses the depth-1 fallback — reconcile to walk-back-until-found at 4.6. The per-day tray subtitle is **not yet built** in the port (the tray renders task rows only, system map §7.5); it lands with tile 4.6.

**Stamp message ≠ MOTD.** The stamp message is a *separate string* — the tier judgement drawn from the red/orange/yellow/green/purple pools — that types into the tray at the stamp beat. The MOTD is the fingerprint. Two strings, two pools, two purposes; they share the tray surface but are never the same value.

---

## Day-state interaction

The MOTD's "has a message" state is an axis independent of the day's progress tier (untouched / partial / complete / rest). A blank day can carry a message; a complete day may not. Its only consumers are the header and the tray — the calendar render and the claim walkback both ignore it. Full taxonomy lives in the day-state notes / system map (capture Item 6); this doc owns only the message axis.

---

## Related tiles

- **3.6** onboarding MOTD screen — trigger 1; tutorial coins, "More coins" arming button, discard-on-transition.
- **4.4** slot machine — the widget's real wheels; fix the alternation bug, randomise each slot independently.
- **4.5** reroll cost — flat 90–110, lock button + confirm; first morning-sequence reroll free.
- **4.6** morning sequence coordinator — trigger 2; the auto-spin beat, the header reveal, the memorial transferral; reconcile the header walk-back here.

## Related docs

- `four_tasks_morning_sequence_design_notes.md` — beat placement of the MOTD reveal + memorial; the claim contract that freezes `motd`. (Its §5.3 header wording still says "today's MOTD, fall back to yesterday's" — the same depth-1 framing this doc supersedes; correct it to walk-back-until-found when propagating.)
- `four_tasks_staggered_disclosure_design_notes.md` — the day-2 long-press philosophy reveal the anytime trigger rides on; confirms no dedicated MOTD reveal.
- `four_tasks_theme_design_notes.md` — emoji slot reads `active_stickers`; MOTD font locked global.
- `four_tasks_monetisation_position.md` — immutable-past invariant the seal-freeze upholds.
- `four_tasks_system_map.md` — live claim contract; day-state taxonomy home (capture Item 6).
