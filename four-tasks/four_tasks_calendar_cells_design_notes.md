# Calendar Cells — Design Notes

Last edit: 2026-06-12 AWST (session 34)

**Status:** LOCKED (session 34, tile 4.24 decision set). Supersedes
the session-14 comment-level lock in `calendar_grid.gd` as the
authority for cell classification, treatment, and tap behaviour.

**References:**
- `four_tasks_stamp_tier_design_notes.md` — grey stamp tier LATENT
  (written at seal, never surfaced); amended s34 alongside this doc.
- `four_tasks_timezone_and_sealing_design_notes.md` /
  morning-sequence Q5 — span-sealing model (s32/s33). The span walk
  is why sealed 0-tick days exist as rows at all.
- `four_tasks_theme_design_notes.md` — sacred tier colours
  (non-themeable), sticker-on-cell / stamp-on-tray split.

---

## Cell populations

| State        | Meaning                                   | Treatment            | Tap |
|--------------|-------------------------------------------|----------------------|-----|
| Tier (R/O/Y/G)| Sealed worked day, 1-4 tasks             | Sacred tier colour + sticker | Opens sealed tray |
| REST (purple)| Sealed rest day (chosen or rescue-bought) | Purple + sticker     | Opens sealed tray |
| UNMARKED     | Unsealed day, 0 ticks                     | `#f1f5f9`, dark date text | Opens live tray |
| DEAD         | See unified family below                  | Dead-inert: same bg, muted date text (`#cbd5e1`), dead overlay | See below |
| DEAD_FUTURE  | Date > today                              | Dead-inert           | None |

Under span-sealing, every day before today is sealed the moment the
app opens (the claim seals the whole span). **In practice only TODAY
is ever UNMARKED** — the muted-text + overlay difference between
UNMARKED and DEAD is the only distinction needed, and it already
exists in the dead treatment.

## The unified DEAD family (s34)

One light-grey dead-inert treatment, no internal distinction,
applied to:

1. **Pre-install days** — date < install. No rows exist; render-only.
2. **Sealed 0-tick days** — skips, declined rescue targets,
   first-contact holes. Rows exist (sealed, grey tier, no payout).
3. **Off-month cells** — the displayed grid's leading/trailing
   overflow, REGARDLESS of the day's real state underneath. A
   worked green May 31 renders the same flat grey in June's grid
   as a skipped May 30 — deliberate flattening; the real state
   lives in its own month, one tap away.

Rationale: days the app didn't exist for and days you didn't show
up for should look equally dead. A dead gap interrupting a streak
run is the reminder — open the app or pay the rescue price. No
stamp, no tray, no judgement text on top (stamp_tier doc: only
positives are memorialised; 0 falls silent).

## Interactivity rules

- **In-month DEAD cells (pre-install, sealed 0-tick): uninteractable.**
  No tap, no tray. The grey stamp the server wrote is never shown.
  Applies identically on the partner panel.
- **Off-month cells: tap = navigate to the cell's home month, then
  behave as an in-month tap on that day.** Direction-agnostic —
  leading overflow navigates back, trailing overflow on a past
  month navigates forward. After navigation, the day's own rules
  apply: worked/rest days open their sealed tray; grey/pre-install
  days do nothing further (navigation was the whole interaction).
  This keeps one rule with no special case: a day behaves the same
  whether reached directly or via overflow.
- **DEAD_FUTURE: fully dead**, including trailing overflow showing
  future dates. Unchanged from session 14.
- Long-press (rest toggle) is today-only and unaffected by any of
  this.

## Implementation pointers (tile 4.24 build)

- `calendar_grid._derive_active_state`: new branch — row sealed
  (`sealed_at != null`), not rest, 0 ticks → DEAD (sealed-grey).
  Currently falls through to UNMARKED, which is the bug 4.24 fixes.
- `day_cell`: sealed-grey state shares the existing dead-inert
  visual; rename `DEAD_HISTORICAL` → `OFF_MONTH` (it no longer
  means "lived but dimmed" — drop the desaturated-real-state
  rendering, flatten to dead-inert).
- App tap handler: off-month tap → `calendar.show_month(home)` then
  delegate to the normal in-month tap path for that date. Ceremony
  gate (`_morning_active`) applies as for any tap.
- Server: no changes. Grey sealing, grey stamp writes, and the
  span walk are untouched — latency is purely a client surface
  decision.

## Explicitly accepted costs

- Grey stamp tier is latent art/plumbing (stamp_tier doc). A v1.x
  keepsake export could surface it; nothing is removed.
- Off-month flattening hides real state until tapped (information
  deferred one tap, not lost).
- A declined rescue day has no revisitable record beyond the dead
  cell itself. The gap is the record.

## Week mode

No interaction — week mode is label-overrides only (s29 lock) and
does not alter cell classification or treatment.
