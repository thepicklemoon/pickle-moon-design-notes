# Four Tasks — Regression Test Matrix

Companion to `four_tasks_system_map.md`. The map describes **what exists**; this matrix describes **what should be verified** after any change.

## How this works

Every test case has a stable ID (`T-NNN`). The ID never changes. When you patch code that could affect a test, flip its checkbox back to `[ ]` and re-run that test before declaring the patch done.

The "Last verified" date is your honesty mechanism. Anything older than your last touch of the area is suspect.

You don't run the whole matrix per patch. You run the **subset** that covers the area you changed. Section §3 of this doc (`Coverage index`) maps files to test IDs so you can look up the right subset.

This is a manual-run matrix. There is no automated harness for v1.0. Some cases require curl, some require the DevKit menu, some require booting the app and observing. Each case states what's needed.

---

## Scope decisions (session 19)

**In scope** for the initial matrix:

- All server endpoints (Phase 1 work, contracts locked in `schema.sql` v1.0).
- All autoloads except `Loader.gd` and `Identity.gd`'s boot-routing path.
- App scene, calendar grid, tasks tray, task rows, streak bar, bonus pill, panel heading, polling.
- DevKit scenarios.

**Deferred** until after Phase 3 lands:

- Loader scene's three-way routing (gains a fourth branch for recovery).
- Identity's boot resolution path (recovery flow integration).
- Onboarding (scene is a stub).
- Post-onboarding pairing flow.

Test IDs in the deferred set are NOT reserved. They'll be allocated when written.

---

## 1. Conventions

**Setup column** — typical values:
- "Default DevKit" — run `POST /devkit/scenario/default` first, then boot with `identity.cfg` pointing at TestUser.
- "Solo TestUser" — like above but un-pair first via `POST /users/<TestUser>/unpair`.
- "Fresh DB" — `wrangler d1 execute --file=schema.sql --remote` to a wiped DB.
- "curl only" — no client boot required, just a terminal.

**Verification column** — what proof counts. "Observe X" means watch the running app. "curl returns Y" means use the terminal. "D1 row matches" means `wrangler d1 execute "SELECT ..."` shows the expected state.

**Last verified** — date in `YYYY-MM-DD` format. Empty until run.

---

## 2. Test cases

### 2.1 Server — health and routing

| ID | Description |
|---|---|
| **T-001** | **Health endpoint live on production** |
| | Setup: curl only |
| | Action: `curl https://four-tasks-api.thepicklemoon.workers.dev/health` |
| | Expect: `{"ok":true,"data":{"status":"ok","service":"four-tasks-api"}}` HTTP 200 |
| | Last verified: |
| **T-002** | **Health endpoint live on dev** |
| | Setup: curl only |
| | Action: `curl https://four-tasks-api-dev.thepicklemoon.workers.dev/health` |
| | Expect: `{"ok":true,"data":{"status":"ok","service":"four-tasks-api"}}` HTTP 200 |
| | Last verified: |
| **T-003** | **Unknown path returns 404 with envelope** |
| | Setup: curl only |
| | Action: `curl https://four-tasks-api.thepicklemoon.workers.dev/garbage` |
| | Expect: HTTP 404, body `{"ok":false,"error":"not_found","message":"No route matched this request."}` |
| | Last verified: |
| **T-004** | **DevKit routes 404 in production** |
| | Setup: curl only |
| | Action: `curl -X POST https://four-tasks-api.thepicklemoon.workers.dev/devkit/scenario/default` |
| | Expect: HTTP 404, `error: "not_found"`. This is the gate that protects production from accidental scenario triggers. |
| | Last verified: |
| **T-005** | **Bad user_id UUID rejected with 400** |
| | Setup: curl only |
| | Action: `curl https://four-tasks-api.thepicklemoon.workers.dev/users/not-a-uuid` |
| | Expect: HTTP 400, `error: "bad_user_id"` |
| | Last verified: |
| **T-006** | **Bad date format rejected with 400** |
| | Setup: Default DevKit |
| | Action: `curl -X PUT https://four-tasks-api-dev.thepicklemoon.workers.dev/users/aaaaaaaa-0000-4000-8000-000000000001/days/may-25 -H "Content-Type: application/json" -d '{"tasks_done":[true,false,false,false]}'` |
| | Expect: HTTP 400, `error: "bad_date"` |
| | Last verified: |

### 2.2 Server — POST /users (onboarding completion)

| ID | Description |
|---|---|
| **T-010** | **Create user with valid body** |
| | Setup: curl only, against dev worker |
| | Action: POST `/users` with `{name:"Test", username:"test", active_leader:"test", task_labels:["a","b","c","d"], timezone:"Australia/Perth"}` |
| | Expect: HTTP 200, returns `{user_id: <uuid>}`. D1 row exists with the supplied fields + `pair_id=NULL`, gameplay aggregates at defaults. |
| | Last verified: |
| **T-011** | **Missing required field rejected** |
| | Setup: curl only |
| | Action: POST `/users` with body missing `name` |
| | Expect: HTTP 400, `error: "bad_input"` |
| | Last verified: |
| **T-012** | **`task_labels` not array-of-4 rejected** |
| | Setup: curl only |
| | Action: POST `/users` with `task_labels: ["only","three","labels"]` |
| | Expect: HTTP 400, `error: "bad_input"` |
| | Last verified: |
| **T-013** | **Forbidden characters in name rejected** |
| | Setup: curl only |
| | Action: POST `/users` with `name: "user|name"` (pipe) and again with `name: "user__name"` (double underscore) |
| | Expect: HTTP 400, `error: "bad_input"` both times. These are pair-key delimiter chars and must be filtered. |
| | Last verified: |

### 2.3 Server — GET /users/:user_id (full payload)

| ID | Description |
|---|---|
| **T-020** | **Solo user returns self, no partner, no pair** |
| | Setup: Solo TestUser (un-pair first) |
| | Action: `GET /users/<TestUser>` |
| | Expect: `data.self` populated, `data.partner = null`, `data.pair = null`, `data.days` is array. |
| | Last verified: |
| **T-021** | **Paired user returns self, partner, pair, both users' days** |
| | Setup: Default DevKit |
| | Action: `GET /users/<TestUser>` |
| | Expect: All four fields populated. `data.days` contains rows for both TestUser and TestPartner. |
| | Last verified: |
| **T-022** | **Day window excludes anything before (currentYear-1)-01-01** |
| | Setup: Default DevKit, then manually `INSERT` a day row dated `2023-01-01` for TestUser |
| | Action: `GET /users/<TestUser>` |
| | Expect: The 2023 row is NOT in `data.days`. Other rows are. |
| | Last verified: |
| **T-023** | **Non-existent user returns 404** |
| | Setup: curl only |
| | Action: `GET /users/00000000-0000-4000-8000-000000000000` |
| | Expect: HTTP 404, `error: "user_not_found"` |
| | Last verified: |

### 2.4 Server — PUT /users/:user_id (mutable field update + rotation)

| ID | Description |
|---|---|
| **T-030** | **Solo user can change username — no rotation** |
| | Setup: Solo TestUser |
| | Action: `PUT /users/<TestUser>` with `{username: "renamed"}` |
| | Expect: 200, full user row returned. D1 `users.username` updated. `pairs` table unaffected. |
| | Last verified: |
| **T-031** | **Paired user changes username — pair_key rotates** |
| | Setup: Default DevKit, note current `pairs.pair_key` value |
| | Action: `PUT /users/<TestUser>` with `{username: "renamed_morgan"}` |
| | Expect: 200. `pairs.pair_key` has changed in D1 (re-derived from new username + partner triple). `pair_id` unchanged. |
| | Last verified: |
| **T-032** | **Paired user changes active_leader — pair_key rotates** |
| | Setup: Default DevKit |
| | Action: `PUT /users/<TestUser>` with `{active_leader: "frog"}` |
| | Expect: 200. `pair_key` rotated. |
| | Last verified: |
| **T-033** | **Updating other fields does NOT rotate** |
| | Setup: Default DevKit, note current `pair_key` |
| | Action: `PUT /users/<TestUser>` with `{task_labels: ["x","y","z","w"]}` |
| | Expect: 200. `pair_key` unchanged. |
| | Last verified: |
| **T-034** | **Pair-key collision returns 409** |
| | Setup: Default DevKit. Create a second pair via two new test users with identity values chosen to collide with the rotation target. (Manual setup; rare test.) |
| | Action: Trigger rotation that would produce a `pair_key` already in use. |
| | Expect: HTTP 409, `error: "pair_key_collision"`. No DB writes. |
| | Last verified: |
| **T-035** | **Attempting to write `name` is ignored (immutable)** |
| | Setup: Default DevKit |
| | Action: `PUT /users/<TestUser>` with `{name: "DIFFERENT_NAME"}` |
| | Expect: Either rejected (preferred — verify which) OR silently ignored with no DB change. Current behaviour: name is not in the mutable-fields list in `updateUser`, so the field is silently dropped. Document the actual behaviour observed. |
| | Last verified: |

### 2.5 Server — PUT /users/:user_id/days/:date

| ID | Description |
|---|---|
| **T-040** | **Write tasks_done on a fresh day creates row** |
| | Setup: Default DevKit, pick a future-but-now-today date or use today's date |
| | Action: `PUT /users/<TestUser>/days/<today>` with `{tasks_done:[true,false,false,false]}` |
| | Expect: 200, full row returned. D1 row created with `sealed_at=NULL`, other defaults. |
| | Last verified: |
| **T-041** | **Partial body (motd only) preserves other fields** |
| | Setup: Default DevKit, day with tasks_done already set |
| | Action: `PUT /users/<TestUser>/days/<date>` with `{motd: "test"}` only |
| | Expect: 200. `tasks_done` unchanged in D1. `motd` updated. |
| | Last verified: |
| **T-042** | **Sealed day rejects write with 409** |
| | Setup: Default DevKit (the seed includes sealed past days) |
| | Action: `PUT /users/<TestUser>/days/<sealed_date>` with any field |
| | Expect: HTTP 409, `error: "day_sealed"`. D1 row unchanged. |
| | Last verified: |
| **T-043** | **tasks_done not exactly 4 booleans rejected** |
| | Setup: Default DevKit |
| | Action: `PUT /users/<TestUser>/days/<today>` with `{tasks_done:[true,true,true]}` (length 3) |
| | Expect: HTTP 400, `error: "bad_input"` |
| | Last verified: |
| **T-044** | **motd over 200 chars rejected** |
| | Setup: Default DevKit |
| | Action: `PUT /users/<TestUser>/days/<today>` with `motd` 201 chars long |
| | Expect: HTTP 400, `error: "bad_input"` |
| | Last verified: |

### 2.6 Server — POST /users/:user_id/claim

The most complex endpoint. Many cases.

| ID | Description |
|---|---|
| **T-050** | **Claim with no eligible day returns null sealed_day** |
| | Setup: Solo user with no day rows at all |
| | Action: `POST /users/<user>/claim` with `{local_date:"2026-05-27"}` |
| | Expect: 200, `data.sealed_day = null`, `data.user` returned unchanged from D1. No DB writes. |
| | Last verified: |
| **T-051** | **Claim seals one-task day → red tier, +1 task's worth of coins** |
| | Setup: Manually insert one unsealed past day for TestUser with `tasks_done='[true,false,false,false]'`, all other days sealed or future |
| | Action: `POST /users/<TestUser>/claim` with today's local_date |
| | Expect: `sealed_day.stamp_tier == "red"`, `coins_earned` ∈ [900,1400], `streak == 0` after (red breaks streak), `longest_streak` unchanged from before. Day row in D1 has `sealed_at != NULL` and `stamp` populated. |
| | Last verified: |
| **T-052** | **Claim seals four-task day → green tier, streak +1** |
| | Setup: Manually insert one unsealed past day with `tasks_done='[true,true,true,true]'`, current `streak=3` |
| | Action: `POST /users/<TestUser>/claim` |
| | Expect: `stamp_tier == "green"`, `coins_earned` ∈ [3600,5600], user's `streak == 4` after, `longest_streak >= 4`. |
| | Last verified: |
| **T-053** | **Claim seals rest day → purple tier, 0 coins, streak unchanged** |
| | Setup: Insert unsealed past day with `rest_day=1`, current `streak=5` |
| | Action: claim |
| | Expect: `stamp_tier == "purple"`, `coins_earned == 0`, `streak` still 5 after. |
| | Last verified: |
| **T-054** | **Claim with empty unsealed days walks past them** |
| | Setup: Insert two unsealed past days; older has `tasks_done='[true,false,false,false]'`, newer has all-false-no-rest |
| | Action: claim |
| | Expect: The OLDER day gets sealed (newer is empty, walkback skips it). |
| | Last verified: |
| **T-055** | **Claim is atomic — failed write doesn't partial-update** |
| | Setup: Hard to fault-inject; observational. After any successful claim, verify `days.sealed_at` and `users.coins` both updated, never one without the other. |
| | Expect: Consistency. |
| | Last verified: |
| **T-056** | **Theme snapshot captured on seal** |
| | Setup: Default DevKit, manually set `users.active_theme = '{"leader":"frog"}'` |
| | Action: claim against an eligible day |
| | Expect: That day's `day_theme_state` in D1 == `'{"leader":"frog"}'` (the string value of active_theme at seal time). |
| | Last verified: |
| **T-057** | **longest_streak never decreases** |
| | Setup: User with `streak=5, longest_streak=10`. Seal a red day (which resets streak to 0). |
| | Expect: After: `streak=0, longest_streak=10`. |
| | Last verified: |

### 2.7 Server — POST /users/:user_id/unpair

| ID | Description |
|---|---|
| **T-060** | **Symmetric un-pair clears both users + deletes pair row** |
| | Setup: Default DevKit |
| | Action: `POST /users/<TestUser>/unpair` |
| | Expect: 200. D1: both users have `pair_id=NULL`, both have `pending_unpair_notice` set to the other's username. `pairs` row deleted. |
| | Last verified: |
| **T-061** | **Un-pair on solo user is idempotent** |
| | Setup: Solo TestUser |
| | Action: `POST /users/<TestUser>/unpair` |
| | Expect: 200, `data: null`. No D1 changes. |
| | Last verified: |
| **T-062** | **Day rows survive un-pair** |
| | Setup: Default DevKit, note count of TestUser's day rows |
| | Action: un-pair |
| | Expect: TestUser's day row count unchanged. (Days belong to user_id, not pair_id.) |
| | Last verified: |
| **T-063** | **dismiss_unpair_notice clears the flag** |
| | Setup: A user with `pending_unpair_notice` populated (from T-060) |
| | Action: `POST /users/<user>/dismiss_unpair_notice` |
| | Expect: 200. D1: `pending_unpair_notice = NULL`. |
| | Last verified: |
| **T-064** | **dismiss_unpair_notice on user with no notice is idempotent** |
| | Setup: User with `pending_unpair_notice = NULL` |
| | Action: dismiss |
| | Expect: 200, no DB change. |
| | Last verified: |

### 2.8 Server — POST /users/:user_id/join_by_values

| ID | Description |
|---|---|
| **T-070** | **Single unambiguous match creates pair** |
| | Setup: Two solo users in D1 with distinct identities |
| | Action: One calls `join_by_values` with the other's triple |
| | Expect: 200, returns `{pair_id}`. Both users' `pair_id` set in D1. New `pairs` row exists with fresh pair_key. |
| | Last verified: |
| **T-071** | **Already-paired caller rejected with 409** |
| | Setup: Default DevKit (caller is paired) |
| | Action: `join_by_values` |
| | Expect: HTTP 409, `error: "already_paired"`. |
| | Last verified: |
| **T-072** | **No match returns 404** |
| | Setup: Solo TestUser |
| | Action: `join_by_values` with a triple matching no user |
| | Expect: HTTP 404, `error: "partner_not_found"`. |
| | Last verified: |
| **T-073** | **Ambiguous match returns 409** |
| | Setup: Three solo users in D1, two of them with identical `(name, username, active_leader)` |
| | Action: third user calls `join_by_values` with that shared triple |
| | Expect: HTTP 409, `error: "ambiguous_match"`. |
| | Last verified: |
| **T-074** | **Self-join attempts are filtered (cannot join with own values)** |
| | Setup: Solo TestUser (only user in DB) |
| | Action: `join_by_values` with TestUser's own triple |
| | Expect: HTTP 404, `error: "partner_not_found"`. Self is excluded by query. |
| | Last verified: |
| **T-075** | **Paired users are not findable as partners** |
| | Setup: Default DevKit (TestUser + TestPartner paired). Add a third solo user. |
| | Action: third user `join_by_values` with TestPartner's triple |
| | Expect: HTTP 404 — already-paired users filtered out. |
| | Last verified: |

### 2.9 Server — POST /resolve (recovery)

| ID | Description |
|---|---|
| **T-080** | **Valid six-tuple returns user_id + pair_id** |
| | Setup: Default DevKit |
| | Action: `POST /resolve` with both triples (TestUser as self) |
| | Expect: 200, returns `{user_id: <TestUser>, pair_id: <TestPair>}`. |
| | Last verified: |
| **T-081** | **Wrong values return 404** |
| | Setup: Default DevKit |
| | Action: `POST /resolve` with garbage triples |
| | Expect: HTTP 404, `error: "pair_not_found"`. |
| | Last verified: |
| **T-082** | **Resolve disambiguates self by name** |
| | Setup: Default DevKit |
| | Action: `POST /resolve` with the same triples but `self.name = "TestPartner"` (treat TestPartner as the recovering user) |
| | Expect: 200, returns TestPartner's user_id (not TestUser's). |
| | Last verified: |

### 2.10 Server — POST /users/:user_id/bug_report

| ID | Description |
|---|---|
| **T-090** | **Bug report inserts row with pair_id_snapshot** |
| | Setup: Default DevKit |
| | Action: `POST /users/<TestUser>/bug_report` with `{message:"test"}` |
| | Expect: 200, returns `{report_id}`. D1 `bug_reports` row exists with `pair_id_snapshot = <TestPair>`. |
| | Last verified: |
| **T-091** | **Empty message rejected** |
| | Setup: any user |
| | Action: bug_report with `{message: ""}` |
| | Expect: HTTP 400, `error: "bad_input"`. |
| | Last verified: |
| **T-092** | **501-char message rejected** |
| | Setup: any user |
| | Action: bug_report with 501-char message |
| | Expect: HTTP 400. |
| | Last verified: |
| **T-093** | **Solo user can report a bug — snapshot is NULL** |
| | Setup: Solo user |
| | Action: bug_report |
| | Expect: 200. D1 row exists with `pair_id_snapshot = NULL`. |
| | Last verified: |

### 2.11 DevKit scenarios

| ID | Description |
|---|---|
| **T-100** | **`default` scenario re-seeds idempotently** |
| | Setup: Any state |
| | Action: Run `default` twice in a row from DevKit menu |
| | Expect: Both runs succeed. D1 ends in canonical state both times (two test users, paired, ~6 days each covering full state ramp). |
| | Last verified: |
| **T-101** | **`fresh_install` server-side teardown + client identity wipe** |
| | Setup: Default DevKit, app running on TestUser |
| | Action: Tap "Fresh install" in DevKit menu |
| | Expect: Identity.cfg wiped, server-side day rows deleted, user rows reset to defaults but UUIDs preserved. App re-boots through Loader to onboarding. |
| | Last verified: |
| **T-102** | **`recovery_flow` server-side teardown, client wipes identity** |
| | Setup: Default DevKit |
| | Action: Tap "Recovery flow" |
| | Expect: Same server-side teardown as fresh_install. Client wipes identity. (Recovery UI not yet built — currently routes to onboarding stub.) |
| | Last verified: |
| **T-103** | **DevKit button absent in store build** |
| | Setup: Export a non-DevKit build (no `devkit` custom feature) |
| | Action: Boot |
| | Expect: No `[DEV]` button visible on PanelYou. DevKit autoload prints "flag off, autoload dormant" at boot. |
| | Last verified: |

### 2.12 Client — Backend autoload

| ID | Description |
|---|---|
| **T-110** | **Backend.check_health → health_checked signal** |
| | Setup: Any boot |
| | Action: From the running app or a test scene, call `Backend.check_health()` |
| | Expect: `health_checked` signal fires with `{status:"ok", service:"four-tasks-api"}`. |
| | Last verified: |
| **T-111** | **Network failure fires `_failed` not `_completed`** |
| | Setup: Block network at OS level (or point Backend at unreachable URL) |
| | Action: Call any Backend method |
| | Expect: The corresponding `_failed` signal fires with `error: "network_error"`. The `_completed` signal does NOT fire. |
| | Last verified: |
| **T-112** | **Editor build hits dev URL** |
| | Setup: Run from Godot editor |
| | Action: Any Backend call |
| | Expect: Network log / wrangler tail shows request landing on `four-tasks-api-dev.workers.dev`, not prod. |
| | Last verified: |
| **T-113** | **HTTPRequest children cleaned up after each call** |
| | Setup: Editor scene tree visible |
| | Action: Fire 5 Backend calls in succession |
| | Expect: No `HTTPRequest_*` children remain under Backend after all calls complete. |
| | Last verified: |

### 2.13 Client — State autoload

| ID | Description |
|---|---|
| **T-120** | **user_loaded triggers full reset** |
| | Setup: Boot fresh; observe `State.days` is empty |
| | Action: `Backend.load_user(TestUser)` |
| | Expect: After `user_loaded` fires, `State.self_user`, `State.partner_user`, `State.pair`, `State.days[TestUser]`, `State.days[TestPartner]` all populated. Caches invalidated. |
| | Last verified: |
| **T-121** | **Optimistic task tap updates local immediately** |
| | Setup: Default DevKit |
| | Action: Tap a task on today's row. Watch State.days in debugger. |
| | Expect: `tasks_done[i]` flips locally BEFORE server response arrives. `day_changed` signal fires immediately. |
| | Last verified: |
| **T-122** | **Day write merge preserves rapid taps** |
| | Setup: Default DevKit |
| | Action: Tap task 0, then tap task 1 within 100ms (before first response returns). |
| | Expect: Both taps persist locally. Final state shows both flipped. No flicker back to "only task 0 done". |
| | Last verified: |
| **T-123** | **Poll merge preserves in-flight write** |
| | Setup: Default DevKit. Use breakpoints or sleep injection to hold the response to a `PUT /days` while poll fires. |
| | Expect: Poll response does NOT clobber the local optimistic value. Hard to test cleanly — observational, may defer. |
| | Last verified: |
| **T-124** | **State.install_iso() per-user cache** |
| | Setup: Default DevKit |
| | Action: Call `State.install_iso(TestUser)` and `State.install_iso(TestPartner)`. Verify they differ if their `created_at` differ. |
| | Expect: Different ISO dates if users have different install times. Both cached (subsequent calls don't re-compute). |
| | Last verified: |
| **T-125** | **Cache invalidates on `clear()` and `user_loaded`** |
| | Setup: After T-124, call `State.clear()` |
| | Expect: Subsequent `install_iso(uid)` re-computes (cache miss). |
| | Last verified: |

### 2.14 Client — Identity autoload (non-boot paths)

Boot-path tests deferred. These are the non-routing paths.

| ID | Description |
|---|---|
| **T-130** | **save() writes identity.cfg** |
| | Setup: Wipe `user://identity.cfg` first |
| | Action: `Identity.save("<uuid>")` |
| | Expect: File exists at user data path, contains `[identity]` section with `user_id = "<uuid>"`. `Identity.user_id` matches. |
| | Last verified: |
| **T-131** | **save() with empty string is a no-op + push_error** |
| | Setup: Existing identity.cfg |
| | Action: `Identity.save("")` |
| | Expect: File unchanged, error printed. |
| | Last verified: |
| **T-132** | **clear() removes the file and resets memory** |
| | Setup: Valid identity present |
| | Action: `Identity.clear()` |
| | Expect: File deleted. `Identity.user_id == ""`. `Identity.has_identity() == false`. |
| | Last verified: |
| **T-133** | **clear() on missing file is safe** |
| | Setup: No identity file |
| | Action: `Identity.clear()` |
| | Expect: No errors. State remains "no identity". |
| | Last verified: |

### 2.15 Client — Poll autoload

| ID | Description |
|---|---|
| **T-140** | **Polling arms when self_user first loads** |
| | Setup: Fresh boot, watch console |
| | Expect: `Poll: armed=false ...` at autoload _ready. After `user_loaded`, the interval timer starts. |
| | Last verified: |
| **T-141** | **30s interval poll fires** |
| | Setup: Boot, leave running for 90s with wrangler tail open |
| | Expect: 3 GET requests at ~30s intervals. Console prints `Poll: firing for <uuid>` each time. |
| | Last verified: |
| **T-142** | **Focus-out stops the interval timer** |
| | Setup: Running app |
| | Action: Tab away / minimise the window |
| | Expect: `Poll: focus OUT` printed. No further polls fire until focus returns. **Suspected non-functional in editor — see system map §11.** Verify on real device. |
| | Last verified: |
| **T-143** | **Focus-in schedules a 10s deferred poll** |
| | Setup: App in background |
| | Action: Bring to foreground |
| | Expect: `Poll: focus IN` printed. A poll fires ~10s later (not immediately). Subsequent interval ticks resume from that point. **Same caveat as T-142.** |
| | Last verified: |
| **T-144** | **Focus-in poll cancelled if focus lost again** |
| | Setup: App in foreground |
| | Action: Quickly tab away, then back, then away again — all within 10s |
| | Expect: No deferred poll ever fires while not focused. Single deferred poll fires after the final stable focus-in. |
| | Last verified: |
| **T-145** | **Poll failure does not stop subsequent polls** |
| | Setup: Block network mid-session |
| | Action: Wait for a poll, observe failure, restore network |
| | Expect: Next interval tick fires anyway and succeeds. |
| | Last verified: |

### 2.16 Client — App scene (two-panel swipe)

| ID | Description |
|---|---|
| **T-150** | **Tap on cell does NOT trigger swipe** |
| | Setup: Default DevKit, app on PanelYou |
| | Action: Quick tap (under 8px movement) on a day cell |
| | Expect: Tray opens for that day. Panel does not pan. |
| | Last verified: |
| **T-151** | **Drag past 12% commits to other panel** |
| | Setup: PanelYou |
| | Action: Drag left ~15% of viewport width, release |
| | Expect: Snaps to PanelPartner. 400ms ease-out. |
| | Last verified: |
| **T-152** | **Drag under 12% snaps back** |
| | Setup: PanelYou |
| | Action: Drag left ~8% viewport, release |
| | Expect: Snaps back to PanelYou. |
| | Last verified: |
| **T-153** | **Edge over-extension feels soft** |
| | Setup: PanelYou (left edge of swipe range) |
| | Action: Drag right (would go past edge) |
| | Expect: Container moves up to 50px past edge but no further. Releases snaps back. |
| | Last verified: |
| **T-154** | **Cross-panel day-carry on swipe** |
| | Setup: PanelYou with tray open on some day |
| | Action: Swipe to PanelPartner |
| | Expect: PanelYou tray closes, PanelPartner tray opens on the same date. |
| | Last verified: |
| **T-155** | **Vertical drag during swipe ignored** |
| | Setup: PanelYou |
| | Action: Drag diagonally — start horizontal then move vertically |
| | Expect: Once past 8px deadzone, only horizontal motion is tracked. Vertical movement does not affect anything. |
| | Last verified: |
| **T-156** | **set_swipe_enabled(false) blocks swipes** |
| | Setup: Call `App.set_swipe_enabled(false)` from console |
| | Action: Attempt swipe |
| | Expect: No drag occurs. Cell taps still work. |
| | Last verified: |

### 2.17 Client — Calendar grid

| ID | Description |
|---|---|
| **T-160** | **Today's cell shows correct active state** |
| | Setup: Default DevKit, today |
| | Expect: Today's cell is UNMARKED (no row) OR matches `tasks_done` count. |
| | Last verified: |
| **T-161** | **Past sealed day with 4 tasks renders GREEN** |
| | Setup: Default DevKit (-1 day TestUser has all-true seed) |
| | Expect: That cell is GREEN. |
| | Last verified: |
| **T-162** | **Day before install renders DEAD_PRE_INSTALL** |
| | Setup: Default DevKit (install was 14 days ago) |
| | Action: Navigate calendar to 30+ days ago |
| | Expect: Cells dated before install render with inert grey, no desaturation, non-interactive. |
| | Last verified: |
| **T-163** | **Future day renders DEAD_FUTURE** |
| | Setup: Today |
| | Expect: Cells dated after today are inert grey, non-interactive. Next-month nav disabled. |
| | Last verified: |
| **T-164** | **Off-month days from previous month render DEAD_HISTORICAL with desaturation** |
| | Setup: Navigate to a month where leading days fall in previous month |
| | Expect: Those cells show correct day number, desaturated, tappable. Tap shows the tray (read-only display). |
| | Last verified: |
| **T-165** | **Prev nav disabled at install month** |
| | Setup: Navigate to install month |
| | Expect: Prev button disabled (greyed). |
| | Last verified: |
| **T-166** | **Next nav disabled at current month** |
| | Setup: Navigate to current month |
| | Expect: Next button disabled. |
| | Last verified: |
| **T-167** | **Grid always renders 6 rows** |
| | Setup: Navigate to a short month (February non-leap, etc.) |
| | Expect: 6 rows visible, trailing rows are DEAD_FUTURE next-month cells. |
| | Last verified: |
| **T-168** | **State.day_changed triggers single-cell re-render, not full grid** |
| | Setup: Default DevKit |
| | Action: Tap a task on today's row, watch grid |
| | Expect: Only the cell representing today re-paints. Other cells don't flash. |
| | Last verified: |
| **T-169** | **Partner calendar uses partner's install date as floor** |
| | Setup: A pair where users have different install dates |
| | Expect: Each panel's calendar shows DEAD_PRE_INSTALL based on the displayed user's install date, not self's. |
| | Last verified: |

### 2.18 Client — Tasks tray + task rows

| ID | Description |
|---|---|
| **T-170** | **Tap cell opens tray** |
| | Setup: Default DevKit |
| | Action: Tap today's cell |
| | Expect: Tray slides in, shows 4 task rows. |
| | Last verified: |
| **T-171** | **Tap same cell closes tray** |
| | Setup: Tray open on some day |
| | Action: Tap that day's cell again |
| | Expect: Tray slides out. |
| | Last verified: |
| **T-172** | **Tap different cell swaps tray** |
| | Setup: Tray open |
| | Action: Tap a different cell |
| | Expect: Current tray slides out, new tray slides in with new date's tasks. |
| | Last verified: |
| **T-173** | **Tap ✕ closes tray** |
| | Setup: Tray open |
| | Action: Tap close button |
| | Expect: Tray slides out. |
| | Last verified: |
| **T-174** | **Today's task tap toggles state + writes server** |
| | Setup: Default DevKit, tray on today |
| | Action: Tap row 0 |
| | Expect: Circle fills green with ✓. Server PUT logged in wrangler tail. D1 row updated. |
| | Last verified: |
| **T-175** | **Past day rows render as post-payout** |
| | Setup: Default DevKit, navigate to a sealed day |
| | Action: Open tray |
| | Expect: Done rows show green ✓, undone rows show red ✕. Rows not interactive. |
| | Last verified: |
| **T-176** | **Rest day rows all render purple ✓** |
| | Setup: Default DevKit (-5 day is rest for TestUser) |
| | Action: Open tray |
| | Expect: All 4 rows purple ✓. |
| | Last verified: |
| **T-177** | **Partner tray rows non-interactive** |
| | Setup: Swipe to PanelPartner, open any day's tray |
| | Action: Tap a row |
| | Expect: Nothing happens. No server write. |
| | Last verified: |

### 2.19 Client — Streak bar + bonus pill

| ID | Description |
|---|---|
| **T-180** | **Streak bar shows self.streak and self.coins** |
| | Setup: Default DevKit (TestUser streak=3, coins=4250) |
| | Expect: Bar shows "3" with flame, coin pill shows "4,250" (comma-formatted). |
| | Last verified: |
| **T-181** | **Streak bar updates when self.streak changes** |
| | Setup: Trigger a claim that updates streak |
| | Expect: Bar updates immediately on `self_changed`. |
| | Last verified: |
| **T-182** | **Bonus pill hidden at multiplier 1.0** |
| | Setup: Partner with empty yesterday |
| | Expect: Bonus pill node exists in tree but `visible=false`. Slot space preserved. |
| | Last verified: |
| **T-183** | **Bonus pill shows ×1.5 on partner rest day** |
| | Setup: Partner yesterday `rest_day=1` |
| | Expect: Pill visible, text "✨ ×1.5 coin bonus active". |
| | Last verified: |
| **T-184** | **Bonus pill multiplier matches the six-line rule** |
| | Setup: Iterate partner yesterday through 0/1/2/3/4 tasks done + rest variant |
| | Expect: 0=hidden, 1=×1.2, 2=×1.3, 3=×1.4, 4=×1.5, rest=×1.5. |
| | Last verified: |

### 2.20 Client — Panel heading

| ID | Description |
|---|---|
| **T-190** | **Title shows `{name}'s four tasks` by default** |
| | Setup: Default DevKit |
| | Expect: PanelYou heading "TestUser's four tasks". (Note: name is case-preserved.) |
| | Last verified: |
| **T-191** | **Tap title flips to lowercase username** |
| | Setup: PanelYou heading on name view |
| | Action: Tap title |
| | Expect: Heading flips to "testuser's four tasks" (lowercased username). Prefs persists this. |
| | Last verified: |
| **T-192** | **Heading per-panel preference is independent** |
| | Setup: Flip PanelYou to username, leave PanelPartner on name |
| | Action: Swipe between panels |
| | Expect: Each panel remembers its own choice. |
| | Last verified: |
| **T-193** | **Subtitle shows today's motd** |
| | Setup: Today has a motd set |
| | Expect: Subtitle displays it. |
| | Last verified: |
| **T-194** | **Subtitle falls back to yesterday's motd if today empty** |
| | Setup: Today has no motd, yesterday does |
| | Expect: Subtitle displays yesterday's. |
| | Last verified: |

---

## 3. Coverage index — file → test IDs

When you edit a file, run at minimum these tests.

| File | Tests |
|---|---|
| `server/src/index.ts` (routing) | T-001..T-006 |
| `server/src/index.ts` (createUser) | T-010..T-013 |
| `server/src/index.ts` (getUserPayload) | T-020..T-023 |
| `server/src/index.ts` (updateUser) | T-030..T-035 |
| `server/src/index.ts` (writeDay) | T-040..T-044 |
| `server/src/index.ts` (claim) | T-050..T-057 |
| `server/src/index.ts` (unpair / dismiss) | T-060..T-064 |
| `server/src/index.ts` (joinByValues) | T-070..T-075 |
| `server/src/index.ts` (resolve) | T-080..T-082 |
| `server/src/index.ts` (submitBugReport) | T-090..T-093 |
| `server/src/index.ts` (devkit scenarios) | T-004, T-100..T-103 |
| `server/schema.sql` | Re-run all server tests T-001..T-103 |
| `server/seed_devkit.sql` | T-100 |
| `Backend.gd` | T-110..T-113 |
| `State.gd` | T-120..T-125 |
| `Identity.gd` | T-130..T-133 (boot tests deferred) |
| `Poll.gd` | T-140..T-145 |
| `App.gd` | T-150..T-156 |
| `calendar_grid.gd` / `day_cell.gd` / `calendar_math.gd` | T-160..T-169 |
| `tasks_tray.gd` / `task_list.gd` / `task_row.gd` | T-170..T-177 |
| `streak_bar.gd` / `bonus_pill.gd` / `Bonus.gd` | T-180..T-184 |
| `panel_heading.gd` / `Prefs.gd` | T-190..T-194 |
| `Layout.gd` | Visual pass on any panel — no dedicated tests |
| `Palette.gd` | Visual pass on any panel — no dedicated tests |
| `Fonts.gd` | Visual pass on heading + MOTD — no dedicated tests |
| `wrangler.toml` | T-001, T-002, T-004 |

---

## 4. Areas without tests (intentional gaps)

For the record:

- **Boot routing (Loader.gd, Identity.load_identity full path)** — deferred until Phase 3 lands. Will add as `T-200..T-209` once the recovery branch exists.
- **Onboarding** — scene is a stub. Will add as `T-210..T-249` after Phase 3 implementation.
- **Post-onboarding pairing flow + recovery UI** — deferred.
- **Theme-based visual rendering** — slot catalogue not yet wired; placeholder palette in use. Will add when tile 4.14b lands.
- **Layout / Palette / Fonts** — single-source-of-truth autoloads. Visual inspection only; no behavioural tests warranted.
- **`promo_codes`, `redemption_attempts` tables** — no endpoints touch them. Tests added when `/redeem` lands in Phase 5.

---

## 5. Change log

| Date | Session | Change |
|---|---|---|
| 2026-05-27 | 19 (mobile) | Initial draft. ~70 cases covering server endpoints, autoloads, and stable client UI. Boot routing + onboarding explicitly deferred. |
