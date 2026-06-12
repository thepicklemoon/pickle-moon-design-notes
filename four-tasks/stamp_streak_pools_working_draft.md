# Stamp / Streak Message Pools — Working Draft (s34)

Extracted verbatim from the prototype (`index.html`). Organised
against the v1.0 tier structure (authority: stamp_tier doc — grey
is LATENT s34, no pool). Review, cut, and write into the gaps;
this file becomes the constants update at the laptop.

PROTO → v1.0 MAPPING:
  STAMP_MESSAGES_*          → PURPLE (rest)
  COMPLETE_STAMP_MESSAGES_* → GREEN (4 tasks)
  PARTIAL_STAMP_1/2/3_*     → RED / ORANGE / YELLOW
  STREAK_MESSAGES_*         → tile 4.9 streak popup (NOT stamps)
  MONTH_STAMP_RAINBOW_*     → tile 4.10 month stamp

THE AYSHI/DEFAULT SPLIT: the proto shipped two builds. v1.0 has
ONE universal pool per tier (doc-locked: single pool, no
sub-pools). Decide per message: promote into the universal pool,
or cut. The Ayshi pools carry most of the actual app voice —
"playful, varied, slightly absurd" is the doc's GREEN target and
the default pools mostly don't hit it.

REVIEW FLAGS (decisions while red-penning):
  1. PROFANITY: 'FUCK IT LOL', 'NO FUCKIN WAY' — ToS carries a
     13+ age line and these land in store builds. Cut or keep is
     a rating/positioning call, make it deliberately.
  2. YELLOW TONE VIOLATION: proto yellow is 'ALMOST' / 'SO CLOSE
     THOUGH' — the locked tone target explicitly bans near-miss
     framing ("no 'so close'"; yellow is its own result, not a
     failed green). Both proto entries are out as-written.
  3. PURPLE TONE: doc warns "'earned' without context lands as
     moralising" — 'YOU EARNED IT' and 'DESERVED' sit right on
     that line. Your call.
  4. PERSONAL/GENDERED: 'GO OFF QUEEN!!!', 'WELL DONE PRINCESS'
     can't go universal. If partner-flavoured pools are ever a
     thing, that's a v1.x sticker-pack/content-drop idea, not
     v1.0 server constants.

POOL SIZE TARGETS (doc): red/orange/yellow 5-10 each; green 40+;
purple 20.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PURPLE — rest day (target 20; proto has 12 + 10, overlapping)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

From STAMP_MESSAGES_AYSHI:
  LOL WHO CARES
  LMAO NOPE
  WOOPS MB
  RESTLINGTON
  OH WELL
  ZZZZZZZZZZ
  CHILL DAWG
  TOO SPEEPY
  NO THANKS
  NOT TODAY
  FUCK IT LOL          [flag 1]
  DESERVED             [flag 3]

From STAMP_MESSAGES_DEFAULT:
  REST DAY
  NO WORRIES
  TAKE IT EASY
  RECHARGE
  YOU EARNED IT        [flag 3]
  CHILL OUT
  BREATHE
  RESET
  NOT TODAY            (dupe)
  DESERVED             (dupe) [flag 3]

→ Unique usable base: ~18. Need ~2-5 more after cuts.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GREEN — 4 tasks (target 40+; proto has 12 + 7)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

From COMPLETE_STAMP_MESSAGES_AYSHI:
  HELL YEAH
  CRUSHED IT
  OMG WOW
  ABSOLUTELY
  NO DOUBT
  HOT DAWG
  SMASHLINGTON
  PHWOAR
  SHEEEEESH
  SO SIMLY
  YEET
  CRAIMZY              [flag 4? in-joke — your call]

From COMPLETE_STAMP_MESSAGES_DEFAULT:
  GREAT WORK
  NAILED IT
  WELL DONE
  SMASHED IT
  LETS GO
  SOLID DAY
  DONE DEAL

→ Unique base: 19. THE BIG GAP: ~21+ more to write. This is the
  roster-writing centrepiece — each entry wants its own small
  character (doc: "discoveries, not templates").

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RED — 1 task (target 5-10; proto has 2)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ROUGH ONE            (default — on-target per doc)
  NOT GREAT TBH        (ayshi)

→ Write ~4-8 more. Tone: gentle, no shame, slightly
  self-deprecating ("ouch", "we'll get 'em next time" band).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ORANGE — 2 tasks (target 5-10; proto has 2)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  HALF WAY THERE       (default)
  MID EFFORT TBH       (ayshi)

→ Write ~4-8 more. Tone: neutral "got through it" — not failure,
  not success, the honest middle.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
YELLOW — 3 tasks (target 5-10; proto has 2, BOTH OUT per flag 2)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ALMOST               [flag 2 — near-miss framing, banned]
  SO CLOSE THOUGH      [flag 2 — near-miss framing, banned]

→ Effectively write ALL 5-10 fresh. Tone: light affirmation
  ("nice", "solid effort", "good day") with zero reference to
  the missing fourth.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TILE 4.9 — STREAK POPUP MESSAGES (separate from stamps)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

No locked pool-size target yet; proto fired every-5-days with
~9 entries per build. Same universal-pool question applies.

STREAK_MESSAGES_AYSHI:
  GO OFF QUEEN!!!      [flag 4]
  YOU'RE AMAZING
  WELL DONE PRINCESS   [flag 4]
  PHWOAR WHAT A STREAK
  AMAZING WORK DARLING [flag 4]
  HOLY SHEEEESH
  OMGOSH SHE'S ON FIRE [flag 4]
  FIYAAAAAAA
  WHOAH CRAIMZY

STREAK_MESSAGES_DEFAULT:
  LETS GOOO!!!
  INCREDIBLE WORK
  WELL DONE
  WHAT A STREAK
  KEEP IT UP
  HOLY MOLY
  ON FIRE RIGHT NOW
  UNSTOPPABLE
  THAT IS WILD

Proto mechanics worth carrying to 4.9's build notes: emoji
escalates 🎉 → 🏆 (20+) → 👑 (30+); streak bonus was a flat
randInt(550, 750) with no multiplier — v1.0 bonus is a server-
side payout component per the economy doc, so the proto figure
is reference only, not authority.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TILE 4.10 — MONTH STAMP (rainbow / perfect month)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  NO FUCKIN WAY        [flag 1]
  PERFECT MONTH

→ One message each in proto. Decide pool-or-single at 4.10.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NOT EXTRACTED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

MOTD spinner wordlists — already migrated (per Morgan, s34).
Grey tier — LATENT, no pool, write nothing.
