# Four Tasks — State.gd Offset Verification (batch closeout)

One job: confirm the timezone offset in State.gd is correct on a real
Android device, then close the session-22 code batch. Delete this doc once
the bias sign is verified.

---

## What changed (already done, on disk)

`State.gd` — `today_iso()` / `yesterday_iso()` / `install_iso()` no longer
hardcode `+ 8 * 3600`. They read the live OS offset via a new helper:

```
func _local_offset_seconds() -> int:
    return int(Time.get_time_zone_from_system().get("bias", 0)) * 60
```

Assumption baked in: Godot's `bias` is in MINUTES and POSITIVE east of UTC
(AWST = +480). The date functions ADD this to a UTC epoch then format as
UTC → local date. If that sign is wrong on Android, every date is off near
midnight.

THE FIX IF IT'S WRONG: negate the return in `_local_offset_seconds()`.
One line, one place. That's the entire contingency.

---

## Why it can't be tested on the laptop

In the Godot editor on Windows, `get_time_zone_from_system()` returns
Windows' offset — also +8 in WA. So the editor shows a correct date whether
the sign is right or wrong. Useless as a test. Needs a real device, ideally
one whose timezone can be changed.

---

## Tonight (laptop) — build the APK

State.gd reaches the phone via an Android EXPORT, not `wrangler deploy`
(that's the server). Steps:

1. Save State.gd + Bonus.gd into the Godot project `scripts/`.
2. Godot → Project → Export → Android → Export Project → debug `.apk`.
   (If Android export was never set up: one-time export-templates +
   debug-keystore setup. Do it tonight, not at work.)
3. Stash the APK where work-you can reach it (email / Drive / USB).

---

## At work tomorrow — install + test

Install the APK. Three checks, fastest first:

1. **Sanity (any time):** open app, look at today's date (calendar "today"
   cell, or fire a claim and read the row). Is it the correct WA date right
   now? If WRONG even here → sign is backwards → negate the helper return,
   rebuild. Stop, it's broken.

2. **Changed-timezone (lunch is fine):** phone Settings → Date & time →
   turn off automatic → set to **Hawaii (UTC-10)** or **LA (UTC-8)**.
   Reopen app. Today's date still correct for that zone? This exercises the
   NEGATIVE-bias path AWST never hits — proves the sign works both
   directions. Set phone back to automatic after.

3. **Midnight band (WA evening, after ~4-5pm — decisive):** open app in the
   WA evening. UTC has rolled to tomorrow's date but local is still today.
   - Correct: shows TODAY's WA date.
   - Buggy: shows TOMORROW's date.
   This is the exact condition a wrong sign breaks on. If the date's right
   at 8pm WA, the offset is verified.

Pass = today's date matches the wall calendar wherever you're standing.

---

## Batch status (session 22 code batch)

- Item 1 — pair-key `||` separator: shipped + verified (dev). DONE.
- Item 2 — `accounted_for` + grey 0-task tier: shipped + 4 tests passed
  (dev DB). DONE. (Client grey-stamp RENDER is tile 4.7, not this batch.)
- State.gd offset: code-complete, THIS DOC verifies it on-device.
- `_dict_equal` (capture Item 8): STRUCK — not a bug. JSON.stringify
  sort_keys defaults true (confirmed vs engine source + case sweep), so
  it's already order-safe. Comment added in State.gd. No code change.
- Bonus.gd `yesterday_iso` → `today_iso`: DONE. Pill is a LIVE forecast of
  today's earning rate (partner-today), applied at tomorrow's seal. NOTE:
  server multiplier is still ×1.0 STUB (gate 3) — pill computes the real
  1.0–1.5 but coins pay ×1.0 until the server side ships. Pill-vs-payout
  mismatch during testing is the stub, not a bug.
- index.ts duplicate-comment tidy: DONE.

## After verification passes

1. Log session 22/23 close in devlog + todo (batch shipped; offset
   verified on device; Item 8 struck).
2. Delete `four_tasks_session_19_capture.md` — everything in it has landed.
3. Next tile: back into Phase 3 onboarding, or Phase 4 hero moments.

## Still open (NOT this batch — don't scope-creep)

- Economy redesign doc (`four_tasks_economy_redesign_notes.md`) — owed
  before any coin number or the server multiplier build. Banked verbally,
  nothing locked. See todo PARKED.
- Same-day-only rest + future-cell non-tappability (client change).
- Morning-sequence rest-purchase escape hatch (design doc, parked).
