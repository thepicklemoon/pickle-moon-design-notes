# Stamp / Streak Message Pools — Working Draft (s34, red-penned s35)

Extracted from the prototype s34; red-penned with Morgan s35
(mobile). All four review flags RESOLVED. Pools below are the
ACCEPTED entries — this file becomes the constants update at the
laptop once green reaches 40+. (Authority: stamp_tier doc — grey
is LATENT s34, no pool.)

FLAG RESOLUTIONS (s35):
  1. PROFANITY: CUT everywhere (FUCK IT LOL, NO FUCKIN WAY).
     Keeps the content-rating questionnaire clean.
  2. YELLOW near-miss framing: confirmed banned. Proto entries +
     Morgan's NEARLY / SO CLOSE / ALMOST THERE / 99% all cut.
     Yellow is its own result, not a failed green.
  3. EARNED/DESERVED: DESERVED + YOU EARNED IT cut. EARNED IT
     kept in purple only — "earned rest" has context there.
  4. GENDERED/IN-JOKE: all cut from universal pools (QUEEN /
     PRINCESS / DARLING / SHE'S ON FIRE / CRAIMZY / RESTLINGTON /
     SMASHLINGTON / SO SIMLY / YEET / TOO SPEEPY / WHOAH CRAIMZY).
     Partner-flavoured pools remain a v1.x content-drop idea, not
     v1.0 server constants.

Also cut s35: BE PROUD (instructional — tells the user what to
feel), GOOD TRY (condescending), TRY AGAIN (game-over flavour).

POOL SIZE TARGETS (doc): red/orange/yellow 5-10 each; green 40+;
purple 20.

PROTO → v1.0 MAPPING (reference): STAMP_* → purple,
COMPLETE_STAMP_* → green, PARTIAL_1/2/3 → red/orange/yellow,
STREAK_* → 4.9 (DEFERRED), MONTH_STAMP_* → 4.10. Proto's
Ayshi/default split is dead — one universal pool per tier.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PURPLE — rest day (18 / target 20 — need ~2)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  NO THANKS
  NOPE
  NAH IM GOOD
  ZZZZZZZ
  CHILL DAWG
  NOT TODAY
  REST UP
  NAAAAH
  TAKE IT EASY
  RECHARGE
  CHILL OUT
  BREATHE
  RESET
  RELAX
  EXHALE
  EARNED IT
  TOMORROW
  DAY OFF

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GREEN — 4 tasks (25 / target 40+ — need ~15)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  HECK YEAH
  CRUSHED IT
  OMG WOW
  ABSOLUTELY
  NO DOUBT
  HOT DAWG
  PHWOAR
  SHEEEEESH
  GREAT WORK
  NAILED IT
  WELL DONE
  SMASHED IT
  LETS GO
  SOLID DAY
  DONE DEAL
  WOOOO
  PHENOMENAL
  BEAUTIFUL
  INCREDIBLE
  SENSATIONAL
  IMMACULATE
  YOU DID IT
  AMAZING
  FANTASTIC
  PERFECT

→ THE REMAINING GAP. ~15 more; doc target is "discoveries, not
  templates" — each wants its own small character.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
YELLOW — 3 tasks (4 / target 5-10 — need 1+)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  NOT BAD AT ALL
  GOOD WORK
  PRETTY GOOD
  NICE ONE

→ Tone: light affirmation, ZERO reference to the missing fourth.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ORANGE — 2 tasks (3 / target 5-10 — need ~2-7)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  HALF AINT BAD
  50%
  A-OK

→ Tone: neutral "got through it" — the honest middle.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RED — 1 task (3 / target 5-10 — need ~2-7)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  NEXT TIME
  YOU CAN DO IT
  DONT GIVE UP

→ Tone: gentle, no shame, no game-over flavour.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TILE 4.9 — STREAK POPUP — DEFERRED v1.x (s35)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Tile 4.9 deferred whole (Morgan, s35): streak bar already shows
the number; the milestone moment ships v1.x when the fuller
milestone vision is designed, not as a popup for its own sake.
Pool below is post-red-pen REFERENCE ONLY — not a v1.0 constant,
do not include in the constants update.

  YOU'RE AMAZING
  PHWOAR WHAT A STREAK
  HOLY SHEEEESH
  FIYAAAAAAA
  LETS GOOO!!!
  INCREDIBLE WORK
  WELL DONE
  WHAT A STREAK
  KEEP IT UP
  HOLY MOLY
  ON FIRE RIGHT NOW
  UNSTOPPABLE
  THAT IS WILD

Proto mechanics, carried as reference for the v1.x revival:
fired every-5-days; emoji escalates 🎉 → 🏆 (20+) → 👑 (30+);
streak bonus was a flat randInt(550, 750) with no multiplier —
any revived bonus is a server-side payout component per the
economy doc, proto figure is reference only.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TILE 4.10 — MONTH STAMP (rainbow / perfect month)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  PERFECT MONTH

→ Proto's second entry cut (flag 1). Decide pool-or-single
  at 4.10.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NOT EXTRACTED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

MOTD spinner wordlists — already migrated (per Morgan, s34).
Grey tier — LATENT, no pool, write nothing.
