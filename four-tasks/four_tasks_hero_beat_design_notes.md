# Four Tasks — Seal-Day Hero Beat Design Notes

**Status:** LOCKED 2026-06-11 (Morgan, mobile — design conversation answers
recorded verbatim below). BUILD-NOW for v1.0 (Morgan's call, against the
forecast-review note that suggested deciding after design lock — the design IS
locked, so the decision resolved with it). Implementation: new client-only
overlay (`hero_beat.gd`, code-built on the rest_confirm precedent); art passes
in Phase 4b (sticker pop is 4.D-adjacent).

Related: `four_tasks_morning_sequence_design_notes.md` (the morning ceremony —
this beat's evening counterpart; NO ceremony changes required, see below),
`four_tasks_economy_redesign_notes.md` (payout obfuscation — why no amount is
shown), `four_tasks_staggered_disclosure_design_notes.md` (this beat is NOT
staggered — day-1, always).

---

## What it is

The evening celebration. When the user completes their day — fourth task
ticked, or a rest day confirmed — the moment is marked: the day's cell flies
up from the calendar and enlarges, the sticker pops onto it, and a popup
frames the payout as **arriving tomorrow morning**. Then it dismisses and the
day is done.

Purpose: decouple the SEAL-shaped moment (you finished) from the PAYOUT moment
(tomorrow's ceremony), and make the gap itself the hook — come back tomorrow
to collect. The morning sequence keeps all the numbers; the hero beat keeps
the feeling.

Key-terms note: this is the "STICKER fires as its OWN hero moment when the 4th
task is ticked" line from the devlog key terms, now fully designed. Sticker =
this moment (calendar cell). Stamp = the morning ceremony's tray artefact. Two
surfaces, two moments — do not conflate.

---

## Locked decisions (2026-06-11)

1. **No amount shown.** The popup never states or hints the coin figure —
   "that's for the morning to be excited about." The multiplier isn't even
   knowable yet (the partner's day is unfinished), and the morning's bonus
   line (beat 5a) + count-up (beat 6) are the reveal. The hero beat sells the
   APPOINTMENT, not the number.
2. **Rest days get it too.** Confirming a rest day fires the same moment with
   the purple variant — the rest sticker/cell pop IS the main visual component
   of the rest designation, landing after the rest dialog closes and the
   server confirms. Two ceremonies in a row on that path (dialog → hero) is
   accepted; the dialog is a decision, the hero is the reward.
3. **Day-1, fires always.** Not staggered, no disclosure gate. The first
   4-task completion is the user's first win; it gets the full moment.
4. **No morning-ceremony changes.** The s29 worry ("reshapes ceremony beat
   ordering + disclosure timing") dissolved under decision 1: with no amount
   shown in the evening, the morning keeps its entire reveal intact. The two
   moments share zero code and zero beats.

---

## The beat-by-beat

1. Trigger fires (see Trigger rules). Today's day cell is located on the grid.
2. A full-screen scrim fades in; a proxy of the cell (same colour state, date,
   sticker slot) tweens from the cell's global rect toward screen centre,
   enlarging (tray-lift family, but a proxy — the real cell never reparents).
3. The sticker pops onto the enlarged cell — scale-overshoot pop (wheel-jiggle
   family). Worked day: the user's active_leader sticker (placeholder art
   until 4.D — tinted panel + glyph, CoinFX-stub precedent). Rest day: the
   purple rest treatment.
4. The popup card appears beneath the cell. Copy (refine at paint, principles
   locked):
   - Worked: header "FOUR FOR FOUR." body "Sealed in the morning — your coins
     arrive with tomorrow's open."
   - Rest: header "REST TAKEN." body "Good call. Tomorrow settles the day."
   No coin figure, no multiplier, no streak number.
5. Tap anywhere dismisses: popup out, proxy flies back to the cell's rect and
   shrinks away, scrim fades, control returns. No timed auto-dismiss — the
   user closes their own moment.

Target feel: shorter and lighter than the morning ceremony (~3-5s plus
read time). It's a punctuation mark, not a second ceremony.

---

## Trigger rules

- **Worked:** the transition to 4 completed tasks on TODAY's own day (the
  optimistic local write — the moment of the tick, not the server round
  trip; a server reject rolls the tick back AFTER the moment, accepted as a
  cosmetic edge on the same terms as every optimistic write).
- **Rest:** rest designation confirmed (server response, since the rest flow
  is already server-gated by the coin debit) — fires after the rest dialog
  closes and the coin tick-down completes.
- **Once per local date, persisted** (Prefs, `hero_shown_date`). Untick/retick
  games don't re-fire it; an app restart doesn't re-fire it. Toggling rest
  OFF and back ON the same day doesn't re-fire it either — one hero moment
  per day, full stop. The flag is client-side cosmetic state, deliberately
  NOT server data (losing it to a reinstall costs at worst one duplicate
  celebration).
- **Never during the morning ceremony** (`State.ceremony_active` guard —
  belt-and-braces; the input blocker already prevents ticks mid-ceremony).
- **Today only.** Past days can't gain a fourth tick (sealed or not-toggleable)
  and future days aren't interactive (4.14c) — the guard is structural, but
  the trigger checks the date anyway.

---

## What it is NOT

- NOT a payout. No coins move, nothing seals, no server call exists for it.
  The seal still happens lazily at tomorrow's claim. Purely cosmetic, like
  the morning animation — crash mid-beat loses the moment, never data.
- NOT part of the morning sequence, the presentation queue, or the staggered
  disclosure schedule. It is its own surface with its own trigger.
- NOT a streak announcement. The streak popup (4.9) is a separate milestone
  surface; if both ever fire on the same tick (a 4th task that also hits a
  milestone), the hero beat plays first, 4.9 follows — resolve the handoff at
  4.9's build.

---

## Implementation notes

- `hero_beat.gd` — code-built, self-contained (scrim + proxy + popup card,
  explicit anchors+offsets per the rest_confirm centering lesson), styled at
  runtime from Palette, ONE_SHOT/teardown discipline on every exit path (the
  rest-anim leak family).
- Trigger wiring lives in App (it already owns day_changed/rest signals and
  the ceremony flag). The overlay exposes `play(date, kind)`; App decides
  when.
- Proxy, not reparent: the real day_cell stays in the grid (the tray-lift
  detach pattern was right for the tray because the tray returns; the cell
  proxy just dies).
- Sticker art placeholder until 4.D; swap point is the proxy's sticker slot
  render only.
- DevKit: a Dialogs row to fire the moment on demand (both variants) for
  tuning without ticking four tasks.
