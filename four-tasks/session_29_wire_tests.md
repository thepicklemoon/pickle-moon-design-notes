# Session 29 — Wire Test Pack (run after `npx wrangler deploy --env dev`)

PowerShell, from inside `server/`. Substitute `<TODAY>` with the actual test
date (path-date and body-date just have to match) and `<SEALED_DATE>` from
step 4's output. Return outputs to Claude numbered, one at a time, for
grading. Device tests D1-D11 are in the todo's TEST LEDGER block.

Gotchas: fresh-deploy curl may throw SEC_E_ILLEGAL_MESSAGE for ~30min
(browser-test, wait); steps 18/22 use backslash-escaped quotes inside a
PS double-quoted string — if D1 rejects the payload, write the SQL to a
file and use --file.

## 1 — apply schema (safe to re-run; creates the 3 content-drop tables)
npx wrangler d1 execute four-tasks-dev --remote --file=schema.sql

## 2 (W1) — generic day-write to a fresh date
'{"tasks_done":[true,false,false,false]}' | Out-File -Encoding ascii -NoNewline body.json
curl.exe -s -X PUT "https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001/days/2026-07-01" -H "Content-Type: application/json" --data-binary "@body.json"

## 3 (W1 verify) — new row must show accounted_for = 1
npx wrangler d1 execute four-tasks-dev --remote --command "SELECT date, accounted_for, tasks_done FROM days WHERE user_id='aaaaaaaa-0000-4000-8000-000000000001' AND date='2026-07-01'"

## 4 (W2 prep) — find a sealed date
npx wrangler d1 execute four-tasks-dev --remote --command "SELECT date FROM days WHERE user_id='aaaaaaaa-0000-4000-8000-000000000001' AND sealed_at IS NOT NULL ORDER BY date DESC LIMIT 1"

## 5 (W2) — write to it; EXPECT 409 day_sealed
curl.exe -s -X PUT "https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001/days/<SEALED_DATE>" -H "Content-Type: application/json" --data-binary "@body.json"

## 6 (W3) — rest_day via day-write; EXPECT 400 naming the rest endpoint
'{"rest_day":true}' | Out-File -Encoding ascii -NoNewline body.json
curl.exe -s -X PUT "https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001/days/<TODAY>" -H "Content-Type: application/json" --data-binary "@body.json"

## 7 (W4 prep) — balance check (need >= 50,000)
npx wrangler d1 execute four-tasks-dev --remote --command "SELECT coins FROM users WHERE user_id='aaaaaaaa-0000-4000-8000-000000000001'"

## 8 (W4 prep, only if step 7 < 50100) — top up
npx wrangler d1 execute four-tasks-dev --remote --command "UPDATE users SET coins=100000 WHERE user_id='aaaaaaaa-0000-4000-8000-000000000001'"

## 9 (W4) — rest ON; EXPECT ok, coins -50,000
'{"rest_day":true,"local_date":"<TODAY>"}' | Out-File -Encoding ascii -NoNewline body.json
curl.exe -s -X POST "https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001/days/<TODAY>/rest" -H "Content-Type: application/json" --data-binary "@body.json"

## 10 (W4) — rest OFF; EXPECT ok, coins back to step-9 start
'{"rest_day":false,"local_date":"<TODAY>"}' | Out-File -Encoding ascii -NoNewline body.json
curl.exe -s -X POST "https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001/days/<TODAY>/rest" -H "Content-Type: application/json" --data-binary "@body.json"

## 11 (W5) — 41-char username; EXPECT 400 value_too_long
'{"username":"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"}' | Out-File -Encoding ascii -NoNewline body.json
curl.exe -s -X PUT "https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001" -H "Content-Type: application/json" --data-binary "@body.json"

## 12 (W6) — unpair TestUser (destructive; step 14 restores); EXPECT ok
curl.exe -s -X POST "https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001/unpair"

## 13 (W6 verify) — initiator notice NULL, partner carries initiator's username
npx wrangler d1 execute four-tasks-dev --remote --command "SELECT username, pair_id, pending_unpair_notice FROM users"

## 14 (W6 cleanup) — reseed the canonical pair
curl.exe -s -X POST "https://four-tasks-api-dev.thepicklemoon.workers.dev/devkit/scenario/default"

## 15 (W7) — economy container; EXPECT rest_cost 50000, reroll_cost 100
(curl.exe -s "https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001" | ConvertFrom-Json).data.economy | ConvertTo-Json -Depth 5

## 16 (W8) — paid reroll; EXPECT ok, coins -100, roll has adverb/verb/noun
curl.exe -s -X POST "https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001/motd/reroll"

## 17 (W8 broke path) — drain, reroll (EXPECT 409 insufficient_coins), restore
npx wrangler d1 execute four-tasks-dev --remote --command "UPDATE users SET coins=50 WHERE user_id='aaaaaaaa-0000-4000-8000-000000000001'"
curl.exe -s -X POST "https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001/motd/reroll"
npx wrangler d1 execute four-tasks-dev --remote --command "UPDATE users SET coins=100000 WHERE user_id='aaaaaaaa-0000-4000-8000-000000000001'"

## 18 (W9 seed) — insert one live test drop
npx wrangler d1 execute four-tasks-dev --remote --command "INSERT INTO content_drops (drop_id, shape, status, subscriber_only, payload, released_at, created_at) VALUES ('bbbbbbbb-0000-4000-8000-000000000001','patch','live',0,'{\"title\":\"wire test\",\"body\":\"hello\"}',strftime('%s','now'),strftime('%s','now'))"

## 19 (W9) — fetch; EXPECT the test drop, payload parsed
curl.exe -s "https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001/drops"

## 20 (W9) — delivered ack; EXPECT ok
curl.exe -s -X POST "https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001/drops/bbbbbbbb-0000-4000-8000-000000000001/delivered"

## 21 (W9) — re-fetch; EXPECT empty list (once-only)
curl.exe -s "https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001/drops"

## 22 (W9 forward-only) — pre-join drop must NOT appear
npx wrangler d1 execute four-tasks-dev --remote --command "INSERT INTO content_drops (drop_id, shape, status, subscriber_only, payload, released_at, created_at) VALUES ('bbbbbbbb-0000-4000-8000-000000000002','patch','live',0,'{\"title\":\"ancient\",\"body\":\"pre-join\"}',0,0)"
curl.exe -s "https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001/drops"

## 23 (W9 cleanup) — remove test drops + delivery row
npx wrangler d1 execute four-tasks-dev --remote --command "DELETE FROM content_drop_deliveries WHERE drop_id LIKE 'bbbbbbbb%'; DELETE FROM content_drops WHERE drop_id LIKE 'bbbbbbbb%'"

## 24 (W10, optional) — cron via local emulator; or let a real Thursday prove it
npx wrangler d1 execute four-tasks-dev --file=schema.sql
npx wrangler dev --env dev --test-scheduled
# second terminal:
curl.exe -s "http://localhost:8787/__scheduled?cron=0+0+*+*+4"
