# Four Tasks — Month Colour + Stats Popup (design notes)

Status: LOCKED s35 (mobile design conversation, Morgan + Claude).
Authority for tile 4.10 (reshaped — see SCOPE CHANGE).
Related: stamp_tier doc (cell tier semantics), economy doc (numbers,
untouched here — this feature pays out NOTHING), calendar_cells doc
(header surface), theme doc v2 (day_theme_state eternalisation).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SCOPE CHANGE (s35) — MONTH STAMP CUT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The rainbow / perfect-month STAMP is CUT. Rationale (Morgan): a
stamp slapped over the month is a visual DOWNGRADE from what the
month already is — a full calendar of the user's own stickers and
themed UI, eternalised per-day at seal (days.day_theme_state).
The month-end payoff is instead:
  1. The month-name TIER COLOUR (this doc), and
  2. The completed-month chrome rule — DESIGNED s35 (same
     night): a completed month's screen-level slots render from
     the day_theme_state of its last SEALED ACCOUNTED day —
     pure render policy, zero new data. AUTHORITY: theme doc
     v2, COMPLETED-MONTH CHROME section; builds inside 4.14b.
     Keepsake/image exporting stays v1.x. A one-shot month-close
     disclosure note ("today's look seals the month") is
     registered in the staggered-disclosure doc.

PERFECT MONTH survives as a single client-side flourish string in
the stats popup header at 100% (completed months only). It is NOT
a server pool — the month-stamp message question from the pools
working draft is closed: no pool, one constant.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MONTH-NAME TIER COLOUR
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

WHEN: completed months only — any month strictly before the
current local month. The current month's name stays stock. The
colour doubles as the long-press affordance signal ("this month
has a story now").

TIER: from completion %, same ramp as cells (sacred semantic
colours — the border values from day_cell's COLOURS dict; mirror
with a comment per the Palette.gd convention, never themed).

THE PERCENTAGE:
  % = ticked tasks / (4 × non-rest days)
  - REST DAYS ARE EXCLUDED FROM THE DENOMINATOR. A paid rest day
    never drags the month colour — consistent with rest
    protecting the streak. (Rest-days count was dropped from the
    popup as a stat, but rest still shapes the maths.)
  - Days with no row / never engaged count as 0 ticks. Honest.
  - CURRENT MONTH (popup browsing only — no colour): the
    denominator is days elapsed THROUGH TODAY, so mid-month
    progress reads as progress, not failure.

BANDS — per-day-average midpoints, so the colour reads as "your
typical day on this month":
  GREEN   >= 87.5%   (avg 3.5+ tasks/day)
  YELLOW  >= 62.5%   (avg 2.5+)
  ORANGE  >= 37.5%   (avg 1.5+)
  RED     >= 12.5%   (avg 0.5+)
  GREY    below      (the month barely happened)
Thresholds are client constants in the popup/header code —
display-only, no economy linkage, retune freely.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
LONG-PRESS → STATS POPUP
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Long-press on the MONTH TITLE (not cells) opens the popup for
that panel's user. Works on ANY month including the current one
(progress-to-date view); the colour merely marks completed months
as discoverable. Both panels — partner months are viewable
(read-only data already in the poll mirror; no new fetch).

POPUP CONTENTS (locked s35; proto screenshot is the reference):
  - Month name + year, tier-coloured on completed months.
  - Completion % — the big number. PERFECT MONTH flourish line
    at 100% on completed months.
  - Tasks X / Y (Y = 4 × non-rest denominator above).
  - Best run: N days — longest WITHIN-MONTH run of 4/4 days,
    where designated rest days BRIDGE the run (neither extend
    nor break it; matches streak-protection semantics in
    spirit). Computed client-side from tasks_done/rest_day.
    DISPLAY-ONLY APPROXIMATION, deliberately labelled "best
    run" so it can never be confused with the server's real
    cross-month streak number. Runs may cross month boundaries
    in reality; this stat truncates at the month edges — that's
    fine, it's a month sheet.
  - Coins earned: STUBBED v1.0 (renders "—"). See below.
  - In-popup month nav (< >), bounded install-month → current
    month, mirroring the calendar's bounded nav. Buttons disable
    at the bounds.
  - Dismiss.
  - DROPPED: rest-days count (Morgan s35).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
COINS EARNED — STUBBED, SERVER MECHANISM RECORDED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Per-day coin amounts DO NOT EXIST: the span seal computes each
day's payout, presents it in the ceremony, adds it to the balance,
and discards it (days rows store no coin column). Client
recomputation is wrong BY DESIGN — multipliers are baked at seal
and unknowable after. So the stat stubs ("—") until a server
mechanism ships. Two candidates, decided at the NEXT index.ts
unfreeze (post D-S6):

  (A) PREFERRED — per-day `coins` column on days, written at
      seal from the already-computed computeDayPayout value.
      Exact, attributes coins to the day they were EARNED,
      reuses existing computation, enables future stats.
      Forward-only: already-sealed history shows 0 — acceptable,
      the test pool is tiny and pre-launch.
  (B) CHEAP — lifetime_coins watermark at month rollover
      (Morgan's suggestion: diff month-start vs month-end
      lifetime values). Viable but approximate: lifetime_coins
      moves at CLAIM time, so a roster-gap span sealed after a
      month boundary attributes a whole span's coins to the
      wrong month. Also needs a watermark table anyway — same
      order of server work as (A) for a worse number.

Client-side snapshots were considered and REJECTED: lost on
reinstall/recovery, contradicts server-is-source-of-truth.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BUILD SPLIT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  1. POPUP MECHANISM — month_stats_popup.gd, self-contained
     code-built overlay (bug_catcher pattern), DevKit-hosted
     trigger. BUILT s35 (build-ahead, disjoint from all
     unverified surfaces).
  2. HEADER WIRING — month-name tint + long-press relay in
     calendar_grid: a few lines, lands AFTER D-S2/D-S3 verify
     (calendar_grid is an unverified surface until then). The
     tier/percentage helpers live in the popup script as statics
     so the grid wiring imports, never duplicates.
  3. COINS COLUMN — server, at next index.ts unfreeze (A above).
  4. COMPLETED-MONTH CHROME — designed s35 (theme doc v2 owns
     it); builds inside 4.14b's renderer. Month-close disclosure
     note registered (copy TBD, roster content list).
