# Session 30 wire tests — streak rescue + amended streak rule (SERVER)

Status: UNVERIFIED — written 2026-06-11 (Morgan mobile), run at the laptop
alongside the session-29 pack (`session_29_wire_tests.md`, W1-W10 — still
owed, run FIRST; this pack assumes the s29 deploy steps already happened).

Deploy first (from `server/`):
    npx wrangler deploy --env dev
No schema change this session — no d1 execute needed.

Conventions (s29 gotchas apply):
- Dev worker: https://four-tasks-api-dev.thepicklemoon.workers.dev
- TestUser id: aaaaaaaa-0000-4000-8000-000000000001
- PowerShell: write bodies to file (`Out-File -Encoding ascii -NoNewline`),
  send with `curl.exe --data-binary "@body.json"`. Pipe Invoke-RestMethod
  output through `ConvertTo-Json -Depth 5` or nested objects print as blanks.
- $env:TODAY below = today's AWST date, YYYY-MM-DD. Set once:
    $TODAY = (Get-Date).ToString("yyyy-MM-dd")
    '{"local_date":"' + $TODAY + '"}' | Out-File -Encoding ascii -NoNewline claim.json
    '{"accept":true,"local_date":"' + $TODAY + '"}' | Out-File -Encoding ascii -NoNewline accept.json
    '{"accept":false,"local_date":"' + $TODAY + '"}' | Out-File -Encoding ascii -NoNewline decline.json

Seeded baselines (every scenario reseed): TestUser coins 4250, lifetime 12800,
streak 3, longest 7. NOTE: coins are BELOW the 50,000 rescue cost — W11/W13
accept paths need a coin top-up after seeding (step listed inline). That's
deliberate: W11a tests the 409 first.

---

W11 — GREY rescue: pending shape, 409 floor, accept converts + holds streak
  1. POST /devkit/scenario/morning_grey
  2. POST /users/<id>/claim  @claim.json
     EXPECT 200: sealed_day = null; pending_rescue { target_date = yesterday,
     target_tier = "grey", kind = "grey", rescue_date = target_date,
     streak_at_stake = 3, cost = 50000 }; user.coins = 4250 (nothing moved).
  3. (W11a) POST /users/<id>/claim/resolve  @accept.json
     EXPECT 409 insufficient_coins (4250 < 50000). Nothing sealed — verify:
     re-POST claim → same pending_rescue again (self-heal property).
  4. Top up:
     npx wrangler d1 execute four-tasks-dev --remote --command "UPDATE users SET coins = 60000 WHERE user_id = 'aaaaaaaa-0000-4000-8000-000000000001'"
  5. POST /users/<id>/claim/resolve  @accept.json
     EXPECT 200, claim-shaped: sealed_day { rest_day = true, stamp_tier =
     "purple", coins_earned = 0, tasks_completed_count = 0 };
     user.coins = 10000 (60000 - 50000); user.lifetime_coins = 12800
     (UNCHANGED — spends never count); user.streak = 3 (HELD);
     longest_streak = 7.
  6. Re-POST claim → sealed_day null, no pending (one-shot via seal).

W12 — GREY rescue: decline seals grey, streak resets
  1. POST /devkit/scenario/morning_grey
  2. POST /users/<id>/claim → pending_rescue (as W11 step 2).
  3. POST /users/<id>/claim/resolve  @decline.json
     EXPECT 200: sealed_day { stamp_tier = "grey", coins_earned = 0,
     rest_day = false }; user.streak = 0; user.coins = 4250 (no charge for
     declining); longest_streak = 7.
  4. Re-POST claim → sealed_day null. The grey day is gone for good.

W13 — GAP rescue: accept creates the missing day, continuity holds through
  1. POST /devkit/scenario/morning_gap
  2. Top up coins (same SQL as W11 step 4).
  3. POST /users/<id>/claim  @claim.json
     EXPECT 200: pending_rescue { kind = "gap", target_date = yesterday,
     target_tier = "orange", rescue_date = day-before-yesterday,
     streak_at_stake = 3, cost = 50000 }.
  4. POST /users/<id>/claim/resolve  @accept.json
     EXPECT 200: sealed_day = the TARGET (yesterday): stamp_tier = "orange",
     tasks_completed_count = 2, coins_earned > 0 (2 tasks × 900-1400 ×
     partner mult); user.streak = 4 (3 held through the bought rest, +1 for
     the worked target); user.coins = 60000 - 50000 + coins_earned;
     lifetime_coins = 12800 + coins_earned (earn counts, spend doesn't).
  5. Verify the bought day exists sealed purple:
     npx wrangler d1 execute four-tasks-dev --remote --command "SELECT date, rest_day, stamp, sealed_at FROM days WHERE user_id = 'aaaaaaaa-0000-4000-8000-000000000001' ORDER BY date DESC LIMIT 3"
     EXPECT the -2 date: rest_day = 1, stamp non-empty, sealed_at set.

W14 — Unrescuable 2-day gap: no offer, immediate seal, streak break applied
  1. POST /devkit/scenario/morning_gap
  2. Widen the gap — delete the -3 sealed day too:
     npx wrangler d1 execute four-tasks-dev --remote --command "DELETE FROM days WHERE user_id = 'aaaaaaaa-0000-4000-8000-000000000001' AND date = DATE('now','+8 hours','-3 days')"
     (Last sealed is now -4 → two missing days before the -1 target.)
  3. POST /users/<id>/claim  @claim.json
     EXPECT 200 with NO pending_rescue: sealed_day { stamp_tier = "orange" };
     user.streak = 1 (gap zeroed the 3, then the worked day counted);
     coins_earned > 0 BUT with NO streak bonus regardless of sub state
     (bonus reads the post-break 0 — only visible if subs are on; note only).

W15 — Amended streak rule: a partial day now ADVANCES (was: reset)
  1. POST /devkit/scenario/morning_payout   (2-task yesterday, -2 sealed, no gap)
  2. POST /users/<id>/claim  @claim.json
     EXPECT 200, no pending: sealed_day { stamp_tier = "orange" };
     user.streak = 4 (3 + 1 — under the OLD rule this was 0; this row is the
     rule-change proof); longest = 7.

W16 — resolve with nothing pending degrades to plain claim
  1. POST /devkit/scenario/default          (everything sealed)
  2. POST /users/<id>/claim/resolve  @accept.json
     EXPECT 200: sealed_day = null, user returned, no error, no charge
     (coins 4250). Re-entrancy floor.

W17 — economy payload carries rescue_cost
  1. GET /users/<id>
     EXPECT economy { rest_cost = 50000, reroll_cost = 100,
     rescue_cost = 50000 }.

---

Ledger lines for the todo (add under the s29 wire rows):
  [ ] W11 grey rescue: pending → 409 floor → accept (purple, -50k, streak held)
  [ ] W12 grey rescue: decline (grey, streak 0, no charge)
  [ ] W13 gap rescue: accept (bought day sealed purple, streak 4, debit+earn)
  [ ] W14 2-day gap: no offer, seal + break (streak 1)
  [ ] W15 streak rule: partial day advances (3 → 4)
  [ ] W16 resolve no-pending: plain-claim degradation
  [ ] W17 economy.rescue_cost shipped

Sync owed at verify (not before): system map §5.3 (claim contract gains
pending_rescue + the resolve endpoint; streak rule line).
