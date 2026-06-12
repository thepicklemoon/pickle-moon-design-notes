# Four Tasks — Wire Regression Pack

The durable home for server wire tests. Folds the s29 + s30 session packs
(`session_29_wire_tests.md`, `session_30_wire_tests.md` — DELETED, superseded
by this file) into one re-runnable reference. Run any block after a dev deploy
that touches its endpoint; run the lot after anything structural.

Status key: each W-row notes when it last PASSED. Open rows are in the todo's
test block — the todo gates, this file carries the commands.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STANDING CONVENTIONS (read once, apply to every block)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

PowerShell, from inside `server/`.

- Dev worker:  https://four-tasks-api-dev.thepicklemoon.workers.dev
- TestUser:    aaaaaaaa-0000-4000-8000-000000000001
- TestPartner: aaaaaaaa-0000-4000-8000-000000000002
- Deploy dev:  npx wrangler deploy --env dev   (bare deploy ALSO lands on dev
  post-s29 toml, but say what you mean)
- D1 remote:   ALWAYS pass --remote or you're talking to the local emulator.
- Bodies: inline JSON escaping fails — write to file, send with curl.exe:
      '{"k":"v"}' | Out-File -Encoding ascii -NoNewline body.json
      curl.exe -s -X POST "<url>" -H "Content-Type: application/json" --data-binary "@body.json"
- Invoke-RestMethod hides nested objects — pipe through ConvertTo-Json -Depth 5.
- Fresh-deploy curl may throw SEC_E_ILLEGAL_MESSAGE for ~30min (edge cert
  propagation). Browser succeeds first; wait curl out.
- Session setup (AWST), once per session:
      $BASE = "https://four-tasks-api-dev.thepicklemoon.workers.dev"
      $U = "aaaaaaaa-0000-4000-8000-000000000001"
      $TODAY = (Get-Date).ToString("yyyy-MM-dd")
      '{"local_date":"' + $TODAY + '"}' | Out-File -Encoding ascii -NoNewline claim.json
      '{"accept":true,"local_date":"' + $TODAY + '"}' | Out-File -Encoding ascii -NoNewline accept.json
      '{"accept":false,"local_date":"' + $TODAY + '"}' | Out-File -Encoding ascii -NoNewline decline.json
- Seeded baselines (every devkit scenario reseed): TestUser coins 4,250,
  lifetime 12,800, streak 3, longest 7. Coins sit BELOW the 50,000
  rescue/rest cost ON PURPOSE — 409 paths test first; top up when a block
  says so:
      npx wrangler d1 execute four-tasks-dev --remote --command "UPDATE users SET coins = 60000 WHERE user_id = 'aaaaaaaa-0000-4000-8000-000000000001'"
- Backslash-escaped quotes inside PS double-quoted SQL sometimes mangle —
  if a payload rejects, write the SQL to a file and use --file.
- sealed_at diagnostic: CLAIM-sealed rows share ONE identical sealed_at
  (the atomicity marker); SEED-sealed rows step by exactly 86,400. Instant
  tell for "did the scenario reseed actually run before this SELECT".

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SCHEMA APPLY (safe to re-run)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

npx wrangler d1 execute four-tasks-dev --remote --file=schema.sql

(`IF NOT EXISTS` everywhere — it will NOT restructure an existing table; a
reshape like 4.D2's override table needs an explicit drop first.)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
W1-W3 — writeDay rules (PASSED s31)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

W1 — generic day-write accounts the day:
  '{"tasks_done":[true,false,false,false]}' | Out-File -Encoding ascii -NoNewline body.json
  curl.exe -s -X PUT "$BASE/users/$U/days/2026-07-01" -H "Content-Type: application/json" --data-binary "@body.json"
  Verify accounted_for = 1:
  npx wrangler d1 execute four-tasks-dev --remote --command "SELECT date, accounted_for, tasks_done FROM days WHERE user_id='aaaaaaaa-0000-4000-8000-000000000001' AND date='2026-07-01'"

W2 — write to a sealed day; EXPECT 409 day_sealed:
  Find one: npx wrangler d1 execute four-tasks-dev --remote --command "SELECT date FROM days WHERE user_id='aaaaaaaa-0000-4000-8000-000000000001' AND sealed_at IS NOT NULL ORDER BY date DESC LIMIT 1"
  Then PUT the same body.json at that date.

W3 — rest_day via writeDay; EXPECT 400 naming the rest endpoint:
  '{"rest_day":true}' | Out-File -Encoding ascii -NoNewline body.json
  curl.exe -s -X PUT "$BASE/users/$U/days/$TODAY" -H "Content-Type: application/json" --data-binary "@body.json"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
W4 — rest endpoint toggle (PASSED s31)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Needs ≥ 50,000 coins (top-up SQL in conventions).
ON  (EXPECT ok, coins −50,000):
  '{"rest_day":true,"local_date":"' + $TODAY + '"}' | Out-File -Encoding ascii -NoNewline body.json
  curl.exe -s -X POST "$BASE/users/$U/days/$TODAY/rest" -H "Content-Type: application/json" --data-binary "@body.json"
OFF (EXPECT ok, refund to start):
  same with "rest_day":false (rebuild body.json with the $TODAY concat)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
W5 — identity length caps (PASSED s31)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

41-char username; EXPECT 400 value_too_long naming the field:
  '{"username":"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"}' | Out-File -Encoding ascii -NoNewline body.json
  curl.exe -s -X PUT "$BASE/users/$U" -H "Content-Type: application/json" --data-binary "@body.json"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
W6 — unpair, recipient-only notice (PASSED s31; destructive, reseed after)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  curl.exe -s -X POST "$BASE/users/$U/unpair"
Verify (initiator notice NULL, partner carries initiator's username):
  npx wrangler d1 execute four-tasks-dev --remote --command "SELECT username, pair_id, pending_unpair_notice FROM users"
Reseed:
  curl.exe -s -X POST "$BASE/devkit/scenario/default"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
W7 + W17 — economy container (PASSED s31)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  (curl.exe -s "$BASE/users/$U" | ConvertFrom-Json).data.economy | ConvertTo-Json -Depth 5
EXPECT { rest_cost = 50000, reroll_cost = 100, rescue_cost = 50000 }.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
W8 — MOTD paid reroll (PASSED s31)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Happy (EXPECT ok, coins −100, roll has adverb/verb/noun):
  curl.exe -s -X POST "$BASE/users/$U/motd/reroll"
Broke (EXPECT 409 insufficient_coins):
  drain → reroll → restore:
  npx wrangler d1 execute four-tasks-dev --remote --command "UPDATE users SET coins=50 WHERE user_id='aaaaaaaa-0000-4000-8000-000000000001'"
  curl.exe -s -X POST "$BASE/users/$U/motd/reroll"
  npx wrangler d1 execute four-tasks-dev --remote --command "UPDATE users SET coins=100000 WHERE user_id='aaaaaaaa-0000-4000-8000-000000000001'"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
W9 — content drops: fetch, ack, once-only, forward-only (PASSED s31)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Seed one live drop:
  npx wrangler d1 execute four-tasks-dev --remote --command "INSERT INTO content_drops (drop_id, shape, status, subscriber_only, payload, released_at, created_at) VALUES ('bbbbbbbb-0000-4000-8000-000000000001','patch','live',0,'{\"title\":\"wire test\",\"body\":\"hello\"}',strftime('%s','now'),strftime('%s','now'))"
Fetch (EXPECT the drop):        curl.exe -s "$BASE/users/$U/drops"
Ack   (EXPECT ok):              curl.exe -s -X POST "$BASE/users/$U/drops/bbbbbbbb-0000-4000-8000-000000000001/delivered"
Re-fetch (EXPECT empty):        curl.exe -s "$BASE/users/$U/drops"
Forward-only — a released_at=0 (pre-join) drop must NOT appear:
  npx wrangler d1 execute four-tasks-dev --remote --command "INSERT INTO content_drops (drop_id, shape, status, subscriber_only, payload, released_at, created_at) VALUES ('bbbbbbbb-0000-4000-8000-000000000002','patch','live',0,'{\"title\":\"ancient\",\"body\":\"pre-join\"}',0,0)"
  curl.exe -s "$BASE/users/$U/drops"
Cleanup:
  npx wrangler d1 execute four-tasks-dev --remote --command "DELETE FROM content_drop_deliveries WHERE drop_id LIKE 'bbbbbbbb%'; DELETE FROM content_drops WHERE drop_id LIKE 'bbbbbbbb%'"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
W10 — Thursday cron (OPEN, optional — or let a real Thursday prove it)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Local emulator:
  npx wrangler d1 execute four-tasks-dev --file=schema.sql
  npx wrangler dev --env dev --test-scheduled
  # second terminal:
  curl.exe -s "http://localhost:8787/__scheduled?cron=0+0+*+*+4"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
W11-W17 — streak rescue + amended streak rule (PASSED s31; W11/W12 RE-PROVEN s33 under span code)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

W11 — GREY rescue: pending → 409 floor → accept:
  1. POST /devkit/scenario/morning_grey
  2. POST $BASE/users/$U/claim @claim.json → EXPECT pending_rescue { kind="grey",
     rescue_date = target_date = yesterday, target_tier="grey",
     streak_at_stake=3, cost=50000 }, sealed_day null, coins untouched (4250).
  3. resolve @accept.json → EXPECT 409 insufficient_coins; re-claim → same
     pending again (self-heal).
  4. Top up (conventions SQL) → resolve @accept.json → EXPECT sealed_day
     { rest_day=true, stamp_tier="purple", coins_earned=0 }; coins 10,000;
     lifetime UNCHANGED 12,800; streak HELD 3.
  5. Re-claim → sealed_day null, no pending (one-shot via seal).

W12 — GREY rescue decline: sealed grey, streak 0, NO charge (coins 4250).
W13 — GAP rescue accept: scenario morning_gap + top-up → claim → pending
  { kind="gap", rescue_date = -2 day } → accept → sealed_day = TARGET
  (orange, 2 tasks, coins_earned > 0); streak 4 (held through bought rest,
  +1 target); bought -2 day exists sealed purple (SELECT to verify).
W14 — 2-day gap unrescuable: scenario morning_gap, then delete the -3
  sealed day (DATE('now','+8 hours','-3 days')) → claim → NO pending;
  target seals orange; streak 1 (break applied, then +1); no streak bonus.
W15 — amended streak rule: scenario morning_payout → claim → streak 3→4
  (a 2-task day ADVANCES; old rule reset — the rule-change proof row).
W16 — resolve with nothing pending: scenario default → resolve @accept.json
  → 200, sealed_day null, no charge. Re-entrancy floor.
W17 — economy.rescue_cost present (folded into W7 above).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
W18 — coin-source ±2d bound (OPEN — s31 ship, 2 min)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Far date; EXPECT 400 bad_date:
  '{"local_date":"2026-07-30"}' | Out-File -Encoding ascii -NoNewline far.json
  curl.exe -s -X POST "$BASE/users/$U/claim" -H "Content-Type: application/json" --data-binary "@far.json"
Real date; EXPECT 200 (normal claim behaviour, whatever the seed state):
  curl.exe -s -X POST "$BASE/users/$U/claim" -H "Content-Type: application/json" --data-binary "@claim.json"
(Also proves the s31 index.ts deploy landed on dev.)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
4.25 RE-SPEC — W13/W14 superseded shapes + new cases
(NOT RUNNABLE until tile 4.25 ships; doc-locked s32. On 4.25 verify,
these REPLACE the W13/W14 rows above.)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

W13r — gap rescue, span shape (PASSED s33):
  accept  → bought -2 day sealed purple + target sealed orange in ONE batch
            (shared sealed_at — SELECT both rows, compare); streak 4.
  decline → the -2 day gets a row CREATED sealed GREY with accounted_for=0
            + target sealed orange, same batch, shared sealed_at; streak 1
            (zeroed at the gap, +1 at the target); no charge.
            (Old W13 had no decline row-creation — gaps just broke.)

W14r — 2+ day gap, span shape (PASSED s33):
  Seed: scenario morning_gap, then widen the hole:
    npx wrangler d1 execute four-tasks-dev --remote --command "DELETE FROM days WHERE user_id='aaaaaaaa-0000-4000-8000-000000000001' AND date = DATE('now','+8 hours','-3 days')"
  claim → NO pending; EVERY missing day gets a grey row (accounted_for=0,
  sealed_at set, coins 0) + target seals; streak 1. SELECT the span —
  contiguous rows from last-seal to target, no holes. (Old W14 left the
  holes rowless.)

W19 — first-contact hole, no payout target (PASSED s33):
  Seed: scenario morning_grey, then make yesterday a pure hole:
    npx wrangler d1 execute four-tasks-dev --remote --command "DELETE FROM days WHERE user_id='aaaaaaaa-0000-4000-8000-000000000001' AND date = DATE('now','+8 hours','-1 days')"
  Claim with $TODAY:
  EXPECT pending_rescue { kind="gap", rescue_date = yesterday,
  target_date = null, target_tier = null }.
  decline → 200, sealed_day = null (no ceremony), yesterday sealed grey,
  streak 0. accept → yesterday sealed purple rest, streak held,
  sealed_day = null.

W20 — multi-deficit suppression (gap + grey target) (PASSED s33):
  Seed: morning_grey, then delete the -2 sealed day (one hole + a grey
  target = two zero-task days in the span):
    npx wrangler d1 execute four-tasks-dev --remote --command "DELETE FROM days WHERE user_id='aaaaaaaa-0000-4000-8000-000000000001' AND date = DATE('now','+8 hours','-2 days')"
  Claim:
  EXPECT NO pending (pre-scan counts 2); hole seals grey + target seals
  grey, one batch; streak 0. No chained offers, no two-purchase saves.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DEVICE RECIPE — rescue affordability dance (reference)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Scenarios reseed coins to 4,250 (below cost) so the 409 path tests first.
To exercise the accept path ON DEVICE: run scenario → open app → KILL at the
rescue dialog (nothing sealed — pending is stateless) → top-up SQL
(conventions) → relaunch → the offer re-fires → accept with funds.
