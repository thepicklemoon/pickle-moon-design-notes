# Four Tasks — System Map

Observational. Describes the system **as it currently exists at the head of the four-tasks repo**. Drift between this doc and the code is a defect in this doc — fix it here as soon as the code changes, in the same commit that changes the code.

**Currency note (2026-06-11, session 32):** full refresh. §4-§5 synced to the s29-s31 server (content-drops tables + endpoints, MOTD endpoints, ±2d coin-source bound, FK-ordered devkit wipe); §6-§7 client component inventory rewritten to as-built at session 31 (onboarding suite, morning-sequence coordinator, CoinFX/GlitchType, rest/rescue dialogs, hero beat, bug catcher, ceremony gating, sealed-beats-clock) — the long-standing "§6-7 under-describes" debt is paid; §11-§12 pruned of items that shipped. NOT YET BUILT and deliberately not described as current: tile 4.25 walkback span-sealing (doc-locked s32 — design lives in the morning-sequence + economy docs; forward-pointers flagged inline at §5.3 / §10.4 / §10.5).

Design intent and rationale live elsewhere (the design docs in `pickle-moon-design-notes/four-tasks/`). This map only describes what's wired up, what talks to what, and what invariants the running system holds.

---

## 1. Top-down overview

Four Tasks runs as a Godot 4 client that talks to a single Cloudflare Worker over HTTPS, which reads/writes a D1 SQLite database. The client is a thin mirror of server state; the server is the source of truth for everything except (a) the local `user://identity.cfg` file that names which user this device is and (b) a handful of per-device display preferences.

There are two server deployments behind two URLs, sharing one bundle of TypeScript:

- **Production** — `https://four-tasks-api.thepicklemoon.workers.dev`, backed by D1 `four-tasks` (UUID `d92e5792-61a1-42cf-969c-9b33b8cc1475`).
- **Dev** — `https://four-tasks-api-dev.thepicklemoon.workers.dev`, backed by D1 `four-tasks-dev` (UUID `0f46f48e-1549-4664-a692-87384af8ff38`). Exposes `/devkit/*` scenario endpoints. Production omits the `IS_DEV` env var, so those routes return 404 there.

The client picks dev vs prod URL per-request at runtime via `Backend._base_url()`, which checks `OS.has_feature("editor") or OS.has_feature("devkit")`. Editor runs always hit dev. Exported store builds always hit prod unless the custom `devkit` feature is set in the export preset.

---

## 1.5 The invariants to hold in your head

The whole system reduces to four sentences. If a bug contradicts one of
these, that contradiction is the bug — start there. (Formal versions: §10.)

1. **A day's life:** opened → accounted → written → sealed at the next
   morning's claim — possibly pausing once at the streak rescue gate. Sealed
   is forever. (Tile 4.25, doc-locked: the claim will extend to seal the
   whole span back to the last sealed day, never-opened days included.)
2. **Money:** every coin write is RELATIVE ("add N"), never absolute, so
   concurrent writers can't erase each other; `lifetime_coins` counts earns
   only — spends never touch it.
3. **Time:** the client owns the clock — bounded. Everything daily keys off
   the LOCAL DATE (per-day triggers re-checked on focus), never off process
   lifetime; the server accepts the client's `local_date` for the coin
   source (claim/resolve) only within ±2 days of server UTC, and a sealed
   day beats the clock on the client (a sealed TODAY is settled, not
   editable).
4. **Re-derive, never remember:** the server holds no conversational state
   between requests. Rotations re-derive keys; resolve re-derives the
   walkback and the rescue offer; a crashed flow heals because nothing
   half-happened. If a design wants the server to "remember an offer," the
   design is wrong.

## 2. Repository layout

The Godot project and Cloudflare Worker live in **one repo**, `four-tasks`, on `@thepicklemoon`. Local path: `C:\dev\four-tasks\` (same on both machines; repos deliberately live outside OneDrive).

Inside that repo:

- `server/src/index.ts` — Worker entry point (all endpoints + devkit scenarios).
- `server/src/stamp_messages.ts` — stub pools, one per stamp tier (incl. grey).
- `server/src/motd_words.ts` — MOTD word pools (adverb/verb/noun).
- `server/schema.sql` — single source of truth for the D1 schema.
- `server/seed_devkit.sql` — re-runnable seed for the two DevKit test users (FK-ordered wipe, synced to `devkitWipeStatements()` in index.ts).
- `server/wrangler.toml` — two environments in one file. **Top-level mirrors DEV** (s29 layout): bare `wrangler deploy` lands on the dev worker; prod requires `--env production`.
- `scripts/*.gd` — autoloads (PascalCase: `Backend.gd`, `State.gd`, `DevKit.gd`, etc.) and scene scripts (snake_case: `day_cell.gd`, `task_row.gd`, etc.).
- `scenes/*.tscn` — scene files (snake_case).
- `assets/ui/the_pickle_moon.png` — splash image, 128×128.
- `data/notes.md` — dev-time scratchpad with test UUIDs.

Design docs live in a separate repo, `pickle-moon-design-notes`. Public-facing docs (privacy policy, future ToS) in `pickle-moon-public`. Expense records in `pickle-moon-ledger`.

---

## 3. Identity model

Three identifiers, each with a distinct lifetime and purpose. The model is locked in `four_tasks_pair_key_design_notes.md`; this section is the operational view.

| Identifier | Scope | Lifetime | Stored where |
|---|---|---|---|
| `user_id` | Per-person | Forever | `users.user_id` (PK), `identity.cfg` on the device |
| `pair_id` | Per-relationship | Stable across rotations; deleted on un-pair | `pairs.pair_id` (PK), `users.pair_id` (FK, nullable) |
| `pair_key` | Recovery lookup | Recomputed on every identity change | `pairs.pair_key` (UNIQUE) |

`pair_key` is a hash over six values: `(name, username, active_leader)` × 2 users, normalised and sorted. It is not used for routing — only for `/resolve`, the recovery flow. Day rows belong to `user_id`, not `pair_id`, so day history survives un-pair and re-pair.

`name` is immutable per user. `username` and `active_leader` are mutable. Any change to a mutable field on a paired user triggers a **pair-key rotation**: the server re-derives the hash from current state and updates `pairs.pair_key` atomically with the user update. Rotations are re-entrant; the server never trusts a client-supplied key.

Solo users have `pair_id = NULL` and no corresponding row in `pairs`. Username changes on solo users skip rotation.

---

## 4. Database schema (D1, single source of truth: `server/schema.sql`)

Ten tables. All times are unix epoch seconds (`INTEGER`).

### 4.1 `pairs`
| Column | Type | Notes |
|---|---|---|
| `pair_id` | TEXT PK | UUID v4 |
| `pair_key` | TEXT UNIQUE NOT NULL | First 16 hex chars of SHA-256 over the canonical six-tuple |
| `created_at` | INTEGER NOT NULL | When the pair was first formed |

### 4.2 `users`
PK: `user_id` (TEXT, UUID v4).

**Identity** — `name` (TEXT NN, immutable), `username` (TEXT NN), `active_leader` (TEXT NN), `pair_id` (TEXT, FK pairs, nullable).

**Localisation** — `timezone` (TEXT NN, IANA, default `'UTC'`).

**Install marker** — `created_at` (INTEGER NN). Used as the per-user calendar floor on the client.

**Gameplay aggregates** — `coins` (INTEGER NN, default 0), `lifetime_coins` (INTEGER NN, default 0), `streak` (INTEGER NN, default 0), `longest_streak` (INTEGER NN, default 0), `task_labels` (TEXT NN, default `'[]'`, JSON array of 4 strings).

**Theme state** — `active_theme` (TEXT NN, default `'{}'`, JSON), `active_stickers` (TEXT NN, default `'[]'`, JSON).

**UX progress** — `tutorial_progress` (TEXT NN, default `'{}'`, JSON).

**State flags** — `pending_unpair_notice` (TEXT, nullable; presence = banner to show; value is the ex-partner's username).

**Week mode** — `week_mode_weekdays` (INTEGER NN, default 0; bitmask). RETIRED-UNUSED as of the s30 Model-B fork resolution — week mode is a popup editor with per-slot overrides; the per-weekday toggle died. Column stays until 4.D2 reshapes the override table.

**Subscription / loyalty / promo** (server-set only) — `subscription_active`, `founders_flag`, `founders_rate_eligible`, `trial_extension_days` (all INTEGER NN, default 0), `redeemed_code` (TEXT, nullable).

Index: `idx_users_pair_id` on `pair_id`.

### 4.3 `user_weekday_overrides`
Per-user-per-weekday task label override (week mode, tile 4.D2).
PK: composite `(user_id, weekday)`. Columns: `user_id` (FK), `weekday` (INTEGER, 1–7), `task_labels` (TEXT, JSON array). Reshapes at 4.D2 (per-slot text + active_slots bitmask; explicit drop-and-recreate — `IF NOT EXISTS` won't restructure it).

### 4.4 `days`
PK: composite `(user_id, date)`.

**Owner-writable, immutable after seal** — `tasks_done` (TEXT NN, default `'[false,false,false,false]'`, JSON array of 4 bools), `motd` (TEXT NN, default `''`), `rest_day` (INTEGER NN, default 0).

**Engagement flag** — `accounted_for` (INTEGER NN, default 0). Write-once-to-1, monotonic; see §10.4.

**Server-written at seal time only** — `stamp` (TEXT NN, default `''`), `sealed_at` (INTEGER, nullable), `day_theme_state` (TEXT NN, default `'{}'`).

### 4.5 `promo_codes` and `redemption_attempts`
Schema landed session 12; design doc written session 29 (`four_tasks_promo_codes_design_notes.md` — one-code-per-user-ever, permanent founders rate, attempt logging incl. failures, codes never grant coins). Redemption endpoint + client entry remain Phase-5 commercial work; tables currently inert.

### 4.6 `bug_reports`
Append-only operational log. PK: `id` (AUTOINCREMENT). Columns: `user_id` (FK), `pair_id_snapshot` (TEXT, value of `users.pair_id` at submission time), `submitted_at`, `message` (1–500 chars), `app_version` (TEXT, ≤50 chars, default `''`). Write path LIVE: `POST /users/:id/bug_report` + the 4.12 client composer; 2000-row live cap enforced at submit.

### 4.7 `content_drops`, `content_drop_assets`, `content_drop_deliveries` (s29)
The content-drops receive side (tile 4.22 server half). `content_drops` — drop catalogue rows (release scheduling, `subscriber_only` flag); a Thursday cron (`scheduled()` + wrangler `[triggers]`) promotes the next queued drop. `content_drop_assets` — per-drop asset records (`r2_keys` recorded; no client can fetch/render them yet — runtime sticker-asset loading is UNDESIGNED, joined to the theme-prep conversation). `content_drop_deliveries` — per-user delivery acks (explicit, idempotent, never implicit on read). All three are users-FK tables covered by `devkitWipeStatements()` (§5.2).

---

## 5. Server endpoints (`server/src/index.ts`)

All production responses use the uniform envelope: `{ok: true, data: ...}` on success, `{ok: false, error: "<code>", message: "..."}` on failure. Status codes are restricted to **200, 400, 404, 409, 500**. No 201/202/204/401/403; 429 reserved for future rate limiting only.

`?caller=` is reserved and unused — all v1.0 writes are self-writes addressed by URL.

The router is order-dependent and prefix-matched — each branch tests that the path starts with `/users/` and ends with a given suffix. This holds only while suffixes stay mutually non-prefixing and no path segment carries a user-controlled value (paths carry only server-generated UUIDs and validated dates). Adding a route with a colliding suffix, or routing on a user-supplied value, breaks these assumptions. No bug today — a constraint on future routes.

### 5.1 Production endpoints

| Method | Path | Body | Success data | Failure codes |
|---|---|---|---|---|
| GET | `/health` | — | `{status:"ok", service:"four-tasks-api"}` | — |
| GET | `/motd/roll` | — | 3 rolled MOTD words (adverb/verb/noun) | — |
| GET | `/motd/pool` | — | full MOTD word pools (client caches per session) | — |
| POST | `/resolve` | `{self:{name,username,active_leader}, partner:{...}}` | `{user_id, pair_id}` | 400 bad_input · 404 pair_not_found |
| POST | `/resolve_solo` | `{self:{name,username,active_leader}}` | `{user_id}` (solo users only) | 400 bad_input · 404 user_not_found |
| POST | `/users` | `{name, username, active_leader, task_labels[4], timezone, today, motd}` | `{user_id}` (+ day-0 seed row, `accounted_for=1`, MOTD set) | 400 bad_input · 500 server_error |
| GET | `/users/:user_id` | — | `{self, partner, pair, days, economy}` (days windowed to `>= (currentYear-1)-01-01`; `economy = {rest_cost, reroll_cost, rescue_cost}`) | 400 bad_user_id · 404 user_not_found · 500 server_error |
| PUT | `/users/:user_id` | partial `{username?, active_leader?, task_labels?, timezone?, tutorial_progress?, active_theme?, active_stickers?, week_mode_weekdays?}` | full user row | 400 bad_user_id/bad_input/value_too_long · 404 user_not_found · 409 pair_key_collision · 500 server_error |
| POST | `/users/:user_id/bug_report` | `{message (1-500), app_version?}` | `{report_id}` | 400 · 404 |
| POST | `/users/:user_id/dismiss_unpair_notice` | `{}` | `null` (idempotent) | 400 · 404 |
| POST | `/users/:user_id/unpair` | `{}` | `null` (idempotent on solo; recipient-only notice) | 400 · 404 · 500 |
| POST | `/users/:user_id/join_by_values` | `{partner:{name,username,active_leader}}` | `{pair_id}` | 400 bad_input · 404 partner_not_found · 409 already_paired/ambiguous_match/pair_key_collision · 500 |
| POST | `/users/:user_id/claim` | `{local_date:"YYYY-MM-DD"}` | `{sealed_day, user}` (sealed_day null if nothing owed) — OR `{sealed_day:null, pending_rescue, user}` when the streak rescue gate fires (§5.3) | 400 bad_input/bad_date (incl. ±2d bound) · 404 user_not_found · 500 |
| POST | `/users/:user_id/claim/resolve` | `{accept:bool, local_date}` | claim-shaped `{sealed_day, user}` (§5.3) | 400 bad_input/bad_date (incl. ±2d bound) · 404 user_not_found · 409 insufficient_coins · 500 |
| PUT | `/users/:user_id/days/:date` | partial `{tasks_done[4]?, motd?}` (`rest_day` 400s — rest endpoint only) | full day row | 400 bad_user_id/bad_date/bad_input · 404 user_not_found · 409 day_sealed · 500 |
| POST | `/users/:user_id/days/:date/rest` | `{local_date}` (same-day-only) | full day row | 400 · 404 · 409 insufficient_coins/day_sealed · 500 |
| POST | `/users/:user_id/motd/reroll` | `{}` | `{user, roll}` (relative debit of `reroll_cost`) | 400 · 404 · 409 insufficient_coins · 500 |
| GET | `/users/:user_id/drops` | — | live + forward-only + undelivered drops (`subscriber_only` returned, NOT filtered — sub-pack news is conversion pull; release-day local-time gating is client-side) | 400 · 404 |
| POST | `/users/:user_id/drops/:drop_id/delivered` | `{}` | `null` (explicit, idempotent ack — never implicit on read) | 400 · 404 |

### 5.2 Dev-only endpoints (gated by `env.IS_DEV === "true"`)

| Method | Path | Behaviour |
|---|---|---|
| POST | `/devkit/scenario/default` | Wipe both test users + pair; reseed canonical spread (~14 days install age, ~6 days of day rows per user covering every cell state) |
| POST | `/devkit/scenario/fresh_install` | Server-side teardown only; client wipes `identity.cfg` independently |
| POST | `/devkit/scenario/recovery_flow` | Same server-side teardown; client routes into the recovery UI |
| POST | `/devkit/scenario/morning_payout` | Default spread but TestUser's yesterday is an unsealed 2-task worked day → next open seals orange + runs the worked ceremony (s26) |
| POST | `/devkit/scenario/morning_rest` | Yesterday an unsealed REST day → seals purple, rest ceremony variant (s28) |
| POST | `/devkit/scenario/morning_grey` | Yesterday an unsealed accounted ZERO-task day (streak 3 at stake, no gap) → next claim returns pending_rescue kind=grey (s30) |
| POST | `/devkit/scenario/morning_gap` | Yesterday an unsealed 2-task day with the day-before OMITTED (last sealed = -3) → next claim returns pending_rescue kind=gap (s30) |

All wipe sites route through the shared `devkitWipeStatements()` helper (s31): users-FK tables (`bug_reports`, `redemption_attempts`, `content_drop_deliveries`) are deleted BEFORE `users`, or the wipe FK-fails. Maintenance rule pinned at the helper: a new users-FK table adds its delete there, only there. `seed_devkit.sql` mirrors the same order.

Stable DevKit identifiers (hardcoded; never rotate):

- TestUser `user_id`: `aaaaaaaa-0000-4000-8000-000000000001`
- TestPartner `user_id`: `aaaaaaaa-0000-4000-8000-000000000002`
- TestPair `pair_id`: `bbbbbbbb-0000-4000-8000-000000000001`
- TestPair `pair_key`: `"devkit_test_pair_key"` (literal, not a hash)

### 5.3 Endpoint-specific contracts that matter for testing

**`POST /users/:user_id/claim`** — the most complex endpoint by far.

> **Forward note (s32):** tile 4.25 (walkback SPAN-SEALING, doc-locked — morning-seq doc Q5 + economy doc) supersedes the single-target walkback below ON BUILD: the claim will adjudicate every day back to the last seal (never-opened days seal grey silently; rescue pre-scans the span and anchors to the zero-task day itself; `pending_rescue.target_date/target_tier` become nullable). Until 4.25 lands, THIS section describes the wire.

**`local_date` bound (s31):** claim and resolve reject a `local_date` outside ±2 days of server UTC with 400 `bad_date` (`LOCAL_DATE_BOUND_DAYS`). Time travel stays legal and self-spoiling; farming the ONLY coin source is closed. Sinks and day writes are deliberately unbounded — they create no coins (rationale recorded at the constant). Wire test W18.

Before the walkback, the claim **upserts today's row with `accounted_for = 1`** (app-open IS engagement, session 23) — write-once-to-1, touches no owner-writable field.

Walkback query (as built, session 23): most recent past day (`date < local_date`) for this user where `sealed_at IS NULL AND accounted_for = 1`. The old JSON string-match predicate is retired. `accounted_for` is set by every documented engagement trigger: the claim's app-open upsert, `writeDay`'s upsert (task tick / MOTD save — gap closed session 29), the rest endpoint, and createUser's day-0 row. In normal use at most one eligible row exists. **Known fault in the soft case (s32 observation):** when multiple stranded rows exist (two consecutive claim failures), the most-recent-first LIMIT 1 does NOT drain them — each day produces one new unsealed row and the claim consumes only the newest, so an older stranded day is shadowed indefinitely and its payout lost. 4.25's span walk retires this along with the retroactive-streak-collapse wart (a hole is only detected behind the NEXT payout target, one day late).

Stamp tier from `(tasks_completed_count, rest_day)`:
- `rest_day=1` → purple, regardless of count
- count=4 → green
- count=3 → yellow
- count=2 → orange
- count=1 → red
- count=0 + not rest → **grey** (built session 23): an accounted 0-task day seals as a **disappointment** — judgement can't be dodged for a day you showed up to.

Coin payout (economy LIVE as of session 28, per the economy redesign doc — AUTHORITY): 0 base on rest/grey days; otherwise per completed task roll `Math.floor(Math.random() * 501) + 900` (each task ∈ [900, 1400]), summed, then:
- × partner multiplier — explicit lookup table `[1.0, 1.2, 1.3, 1.4, 1.5]` by the partner's task count **on the sealed row's own date** (never a hardcoded today/yesterday); partner rest day = full 1.5. Table, not arithmetic — float drift.
- × 1.5 again if BOTH users `subscription_active` (two-sub boost; 2.25 effective cap). Solo-but-subscribed = phantom ×1.5.
- + two-sub-only streak bonus: `round(base*mult * streak * 0.01)` using the PRE-claim streak.
Partner/sub reads are gated behind `base > 0` (rest/grey skip them). Payout is baked immutably at seal — no recompute on later sub lapse. The claim response carries `multiplier` and the sealed day's `motd`. The client preview pill (§6.6 `Bonus.gd`) reads the partner's *current* day and mirrors the table (incl. partner-rest = 1.5) but does NOT show the two-sub boost (Phase-5 note).

**Coin writes are RELATIVE** (`coins = coins + ?`, session 29) — same as the rest endpoint, so the two coin writers commute and a concurrent rest debit can't be swallowed by a stale read. Streak/longest stay absolute (no concurrent writer).

Streak update (AMENDED session 30 — economy doc is authority):
- FIRST, the continuity check: if the day being sealed is not date-adjacent
  to the most recent previously-sealed day (`lastSealedDateBefore`, derived —
  no column), the streak zeroes BEFORE this day counts, and the streak BONUS
  above reads the zeroed value (no bonus across a broken streak). As
  originally shipped, no-show gaps never broke streaks at all (unsealed days
  never ran streak math) — this check closed that hole.
- then: ≥1 task → `streak + 1` (the bare minimum keeps the chain alive)
- purple (rest) → unchanged (holds; a rescued day is rest, so holds through)
- grey (0 tasks) → reset to 0

`longest_streak = max(longest_streak, new_streak)`. SUPERSEDED rule (s12-s29):
green-only advance, reset on partials.

**Streak rescue gate (session 30).** Before sealing, the claim evaluates
whether this seal would break a streak ≥ 1 in a way ONE rest purchase saves:
the target itself sealing grey with no gap (kind=grey), or the target
maintaining but exactly one never-opened day sitting behind it (kind=gap).
If so the claim returns `{sealed_day:null, pending_rescue:{target_date,
target_tier, kind, rescue_date, streak_at_stake, cost}, user}` and seals
NOTHING. Anything unrescuable by one purchase (2+ day gaps; grey-behind-a-gap)
gets no offer — the seal proceeds with the break applied.

**`POST /users/:user_id/claim/resolve`** carries the answer
(`{accept, local_date}`) and is fully RE-ENTRANT: it re-runs the walkback +
rescue evaluation from current DB state, never trusting the client's memory
of the offer. Decline (or stale offer) → seal as-is via the same path.
Accept → 409 `insufficient_coins` below `RESCUE_COST` (= REST_COST = 50,000,
own constant, shipped as `economy.rescue_cost`); otherwise ONE atomic batch:
grey case converts the target to rest then seals it purple; gap case INSERTs
the missing day as an already-sealed purple rest (silent — no ceremony of its
own; reaches the calendar via poll merge) then seals the target with
continuity intact. The debit rides the same relative coin write as the payout
(`coins + (earned - cost)`); `lifetime_coins` gains earns only. The ONE-SHOT
property is enforced by the seal itself — declining IS sealing, accepting IS
converting-then-sealing, so there is no offer state to track and an
unanswered offer (app death) self-heals by re-offering at the next claim.

Both claim and resolve end in the same shared `sealTarget()` — tier, economy,
streak, batch, response live exactly once (s30 refactor); `deriveTier` /
`evaluateRescue` / `lastSealedDateBefore` / `isoShiftDays` / `isoDaysBetween`
are its pure helpers.

Theme snapshot: `users.active_theme` (the string, not parsed) copied into `days.day_theme_state` for the sealed day.

Single atomic batch updates the day row (stamp, sealed_at, day_theme_state) and the user row (coins, lifetime_coins, streak, longest_streak). A post-batch re-read supplies the response, so it reflects any concurrent movement.

**`POST /users/:user_id/days/:date/rest`** (built session 28) — the ONLY rest writer; SAME-DAY-ONLY (`date` must equal the request's `local_date`). Toggle on: debits REST_COST (50,000; relative write), sets `rest_day = 1` + `accounted_for = 1`; 409 `insufficient_coins` if short. Toggle off: refunds (relative), clears the flag (`accounted_for` stays — engagement is monotonic). Idempotent, transactional. `writeDay` REJECTS `rest_day` in its body with 400 (session 29) — a free rest day via the generic day-write was the hole.

**`PUT /users/:user_id/days/:date`** (writeDay) — writes `tasks_done` and/or `motd` only. Sets `accounted_for = 1` on insert AND conflict (session 29). Sealed days 409 `day_sealed`.

**`POST /users/:user_id/motd/reroll`** (s29) — relative debit of `REROLL_COST` (100, placeholder, shipped via the economy payload), 409 `insufficient_coins`, returns the updated user + a fresh word roll. The fee is vent-pricing on the legitimate flow, not resource protection — words stay free via `GET /motd/roll`, MOTD writes free via writeDay; bypassing the fee is self-spoiling only (decision recorded at the constant). Client half (picker lock + confirm) is tile 4.5.

**`GET /users/:user_id`** — serves both boot load and the 30s poll. Payload: `{self, partner, pair, days, economy}`. The `economy` container (session 29) ships server-authoritative numbers the client needs pre-request — `{rest_cost, reroll_cost, rescue_cost}` as of s30; the rest/rescue dialogs' afford gates read it via `State.rest_cost()` / `State.rescue_cost()`. Retunes are one server constant.

**`GET /users/:user_id/drops` + delivered ack** (s29) — live, forward-only, undelivered drops; release-day local-time crossing is CLIENT-side (client-owns-the-clock); `subscriber_only` rows are returned, not filtered. The ack is explicit and idempotent, never implicit on read (decoupling principle). Per-drop parse isolation on `getEligibleDrops` (sweep-chat patch, carried in the s30 build).

Identity fields (`name`/`username`/`active_leader`) are length-capped server-side at 49/40/50 post-normalisation (session 29); violations return 400 `value_too_long` naming the field. All write paths covered, including the rotation route.

**`PUT /users/:user_id`** — paired users with `username` or `active_leader` changes trigger a rotation transaction: re-derive `pair_key` from new self + current partner, UNIQUE-check against other pairs, then atomic batch updates `pairs.pair_key` + `users` row. Solo users skip rotation. (Known cosmetic: a simultaneous-write TOCTOU race returns 500 not 409; `pair_key` UNIQUE keeps correctness — s29 finding, fix opportunistically.)

**`POST /users/:user_id/join_by_values`** — caller must be solo (`pair_id IS NULL`), or 409 `already_paired`. Search restricted to other solo users (`pair_id IS NULL`) by exact `(name, username, active_leader)` match. Zero matches → 404. Two-plus matches → 409 `ambiguous_match`. One match → derive pair_key, check collision, batch INSERT pair + UPDATE both users.

**`POST /resolve`** — pair_key lookup. After matching, additionally filter by self's `name` to disambiguate the two users in the matched pair. Returns 404 `pair_not_found` if the pair exists but neither user matches the supplied self.name. **`POST /resolve_solo`** — the solo variant: one triple, `pair_id IS NULL` candidates only.

---

## 6. Godot client — autoloads

Order of registration (matters; later autoloads depend on earlier ones at `_ready`):

```
Backend → State → Identity → Poll → DevKit → Bonus → CoinFX → GlitchType → Layout → Palette → Fonts → Prefs
```

(Verify exact `project.godot` ordering on the canonical machine; the constraint is Backend/State/Identity first, presentation utilities after.)

### 6.1 `Backend.gd` — stateless HTTP layer
The only node that makes network calls. Spawns a fresh `HTTPRequest` child per call, awaits, frees it. Stateless: callers pass `user_id` into every method that needs one.

Public methods, one per endpoint (verified s32):
`check_health, create_user, load_user, poll_user, update_user, write_day, set_rest_day, claim, resolve_claim, submit_bug_report, unpair, dismiss_unpair_notice, join_by_values, resolve_pair, resolve_solo, motd_roll, fetch_motd_pool, devkit_run_scenario`

Signals: one success + one failure per endpoint, 38 declarations at s32. Notables beyond the obvious pairs:
- `claim_resolved` / `claim_resolve_failed` (s30 — claim-shaped by contract; State funnels both into the claim handlers)
- `motd_rolled` / `motd_roll_failed`, `motd_pool_ready` / `motd_pool_failed`, `motd_pool_fetched` / `motd_pool_fetch_failed` (s28-29)
- `poll_completed` / `poll_failed` (reuses `GET /users/:user_id` so State can apply selective merge)
- `devkit_scenario_ran` / `devkit_scenario_failed`

URL switching: `_base_url()` returns `BASE_URL_DEV` or `BASE_URL_PROD` per request, picked by `OS.has_feature("editor") or OS.has_feature("devkit")`. Called per-request — not cached.

### 6.2 `State.gd` — in-memory mirror
Source-of-truth invariant: **server is canonical**. State writes optimistically and reconciles on Backend response.

**Ceremony gate (s27/s28):** `State.ceremony_active` (single shared flag) + `ceremony_changed(active)` signal replace per-surface mid-ceremony repaint guards. The claim handler is the only writer; surfaces only read it and listen for the signal.

**Rescue routing (s30):** `_on_claim_completed` detects `pending_rescue` and emits `claim_rescue_pending(pending)` INSTEAD of `morning_claimed` — the ceremony coordinator's parked one-shot stays armed; the resolve's claim-shaped response re-enters the same handler and emits `morning_claimed` as normal, so `morning_sequence.gd` is rescue-blind by design. `resolve_claim(accept)` reuses the claim's remembered `local_date`. Resolve failures route through `_on_claim_failed` (coordinator winds down on `morning_claimed({})`; App re-arms the per-day trigger).

**DevKit mock clock (s29):** `set_mock_today(date_iso)` overrides `today_iso()` everywhere it flows — including the claim's `local_date` — so a mocked "tomorrow" runs the full real seal path against the dev server. Cleared on scenario reboot so a forgotten mock can't corrupt the next scenario. (Server-side, the ±2d bound limits how far a mock can travel — by design.)

**Economy getters:** `rest_cost()`, `rescue_cost()` (and the reroll figure rides the same payload) read the server-shipped `economy` container — afford gates never hardcode prices.

Fields:
- `self_user: Dictionary` — current self payload
- `partner_user: Dictionary` — current partner payload, `{}` if solo
- `pair: Dictionary` — current pair row, `{}` if solo
- `days: Dictionary` — nested `{user_id: {date: row}}` for O(1) lookup
- `_writes_in_flight: Dictionary` — per-(user_id, date) flag for in-flight `PUT /days/:date`
- `_pending_snapshot: Dictionary` — single rollback target for `set_motd` / `toggle_rest_day` failures
- Cached helpers: `install_iso(user_id)` (per-user cache), `today_iso()`, `yesterday_iso()`

Signals: `self_changed`, `partner_changed`, `pair_changed`, `day_changed(user_id, date)`, `cleared` (logout/reboot — UI drops cached state), `ceremony_changed(active)`, `morning_claimed(sealed_day)`, `claim_rescue_pending(pending)`, plus per-write failure signals (e.g. `rest_day_set_failed`).

Key write entry points: `toggle_task(date, index)` (self-only — no user_id param; optimistic, LWW, no rollback), `set_motd(date, motd)` + `toggle_rest_day(date)` (single-snapshot rollback), `claim_morning(local_date)`, `resolve_claim(accept)`.

**Merge policies** — both important for diagnosing race conditions:

`_on_day_written` (response to `PUT /days/:date`): take server's value for `stamp`, `sealed_at`, `day_theme_state`. Preserve local copy of `tasks_done`, `motd`, `rest_day` (server's view was canonical *at write time*, but rapid taps may have advanced local state past the response).

`_on_poll_completed` (response to background poll): per-day diff with the same merge rule for self days that have an in-flight write; otherwise take server's row whole. `self_user`, `partner_user`, `pair` taken whole from server (brief stomp risk on in-flight `PUT /users` accepted for v1.0). Days that vanish from the response are dropped (defensive — shouldn't happen in practice). Emits `day_changed` only for rows that actually differ.

`_on_claim_completed` (ceremony path): merges the banked user SILENTLY and seals the day onto `State.days` with NO `day_changed` (source-side suppression — the calendar can't spoil the reveal), emitting `morning_claimed(sealed_day)` instead. Beat 12 refreshes via `Backend.poll_user` (selective merge preserves the in-flight beat-10 MOTD write).

**Concurrency model (session 16 simplification)** — Task taps fire optimistic update + server write immediately, with NO in-flight guard. Server uses `INSERT ON CONFLICT REPLACE` so last write wins. There is NO rollback path for task taps. `set_motd` and `toggle_rest_day` retain the single-snapshot rollback via `_pending_snapshot`. (s29 hardening made rest rollback per-operation.)

### 6.3 `Identity.gd` — local user_id ownership
Owns `user://identity.cfg` via `ConfigFile`. Section `[identity]`, key `user_id`. No version field for v1.0.

Public API: `load_identity()` (async, emits `boot_resolved` when chain completes), `save(user_id)`, `clear()`, `has_identity()`.

`load_identity()` flow:
1. Read `identity.cfg`. File missing or unreadable → emit `boot_resolved(false, false)`, return.
2. Read `user_id` from section. Empty/missing/non-string → emit `boot_resolved(false, false)`, return.
3. Store `user_id` in memory.
4. Connect `Backend.user_loaded` and `Backend.user_load_failed` as one-shot handlers.
5. Call `Backend.load_user(user_id)`.
6. On `user_loaded` → `boot_resolved(true, true)`. On `user_load_failed` → `boot_resolved(true, false)`.

Used by Loader for routing and by DevKit for `fresh_install` / `recovery_flow` scenarios (which call `Identity.clear()` before triggering server teardown and re-boot).

### 6.4 `Poll.gd` — background refresh
Owns one interval timer (30s) and one one-shot focus-in timer (10s deferred poll).

State machine:
- **Disarmed** until `State.self_user` first appears (via `State.self_changed`). Then `_arm()` and start the interval timer if foreground.
- **Foreground / background** via `NOTIFICATION_APPLICATION_FOCUS_IN/OUT`. Background stops both timers.
- **Focus-in**: start the 10s deferred poll. If focus is lost again before it fires, the timer's `stop()` cancels it. Never stacks.
- **Every poll** (interval OR focus-in) calls `_interval_timer.start()` to reset the 30s window, so cadence is "30s since last poll" not "30s since boot."

Failure is silent — next tick retries. Backend.poll_user uses the same `GET /users/:user_id` endpoint as bootstrap but routes through `poll_completed` so State applies the selective merge rather than the full reset.

### 6.5 `DevKit.gd` — scenario coordinator
`is_active = OS.has_feature("editor") or OS.has_feature("devkit")`. False in store builds — autoload stays dormant.

**Filename (s31):** renamed `devkit.gd` → `DevKit.gd` — the one autoload that violated the PascalCase convention; Windows resolves the registration case-insensitively, the Android PCK does not (autoload Nil on device, menu dead). **Repo trap (s32):** a case-only rename on Windows does not register as a git change by default (`core.ignorecase`); land it via `git mv devkit.gd DevKit.gd` or it silently never commits.

`run_scenario(name)`:
1. Per-scenario client prep. `fresh_install` and `recovery_flow` call `Identity.clear()` before the server call.
2. `Backend.devkit_run_scenario(name)`.
3. On `devkit_scenario_ran` → `_reboot()` swaps to `loader.tscn` via `call_deferred`. Real boot path re-executes with whatever server state the scenario produced.
4. On `devkit_scenario_failed` → push warning, do NOT reboot.

### 6.6 `Bonus.gd` — partner coin multiplier
Reads `State.partner_user`'s current day row and returns a float multiplier via the same explicit lookup table the server's claim handler applies at seal (incl. partner-rest → 1.5). Display-only preview — the server applies the authoritative factor against the sealed row's date. Solo users return 1.0. Does NOT show the two-sub boost (lands with paywall UI, Phase 5).

### 6.7 `CoinFX.gd` — coin payout visuals (tile 4.3, s27)
The animated *experience* of a payout, never the payout itself — coins are banked server-side at claim; CoinFX only flies them. Stub coin glyph until 4.A (swap `_make_coin` only). Consumed by the ceremony's beat 6.

### 6.8 `GlitchType.gd` — glitch-typewriter factory (tile 4.1, s25)
THIN by design: animates nothing itself; spawns self-contained `glitch_reveal` workers that churn-resolve text on a target Label. Consumed by the ceremony (beats 5a/7/10), panel headings, and (at 4.2) onboarding popups.

### 6.9 `Layout.gd`, `Palette.gd`, `Fonts.gd`, `Prefs.gd` — single sources of truth
- `Layout.gd` — every layout dimension (cell sizes, paddings, font sizes, panel widths). Scenes carry literals; a comment block at the top of each `.tscn` maps the literals back to `Layout.*` names so they can be kept in sync by hand. The `.tscn` cannot reference autoload constants directly.
- `Palette.gd` — non-sacred colours (panel backgrounds, button states, etc.). **Sacred** colours (cell-tier ramp, task-row state palette) remain hardcoded in `day_cell.gd` and `task_row.gd` because they're theme-locked.
- `Fonts.gd` — Departure Mono (project default), JetBrains Mono (MOTD subtitle). Set globally via Project Settings → GUI → Theme, plus an explicit override on the subtitle node.
- `Prefs.gd` — per-device local prefs (heading-name variant per panel; `hero_shown_date`, s30 — the local date the seal-day hero beat last fired, once per day full stop; client-only cosmetic state, deliberately not server data). Stored locally; never round-tripped to server.

---

## 7. Scenes

### 7.1 `loader.tscn` — boot router
Main scene of the project. Centred splash texture + hidden `ErrorLabel`. `Loader.gd` connects to `Identity.boot_resolved` and routes:

- `(false, false)` — no identity → `change_scene_to_file("res://scenes/onboarding.tscn")` (deferred)
- `(true, true)` — identity loaded, payload received → `change_scene_to_file("res://scenes/app.tscn")` (deferred)
- `(true, false)` — identity stored, server rejected → show error label, hook `gui_input` for tap-to-retry. Retry disconnects the handler, hides the label, calls `Identity.load_identity()` again.

Boot diagnostic printed on `_ready`: `[Loader] editor=<bool> devkit=<bool>`.

### 7.2 Onboarding suite (Phase 3, shipped s24-25)
`onboarding.tscn` hosts `onboarding.gd` — the step machine (enum identity tags; the order the user sees is `_sequence` in `_ready`, not enum order; Back stack; accumulating result payload, committed once at the end). Spine: welcome (first-time/recover fork) → path choice → name → immutable name confirm (displays the typed value exactly) → username → avatar/leader → companions (commits leader + 5 to `active_stickers`) → MOTD (4.4 spinner, economy-fenced, real as of s26) → four-tasks entry (UX cap 30 chars/label client-side; server caps 49) → boot into App. `create_user` sends `today` (client ISO) + the composed `motd`; the server writes the users row AND the day-0 row (`accounted_for = 1`) in one batch, so no sequence fires on the onboarding open.

Per-screen scripts: `onboarding_welcome/path_choice/name/name_confirm/username/avatar/companions/motd/tasks/recovery/placeholder.gd`. Recovery (3.10B, prod-verified both directions s25) is the two-tier flow: paired six-tuple via `/resolve`, solo triple via `/resolve_solo`. Pairing post-onboarding: `pair_dialog.gd` (join-by-values) + `partner_empty_state.gd` on the partner panel (3.10A). A `devkit_button` instance is mounted in onboarding too — fresh devices have no `identity.cfg` and can't reach the App scene's button.

Validation everywhere: no `|`, non-empty; UX caps under server caps.

### 7.3 `app.tscn` — main UI
Two-panel horizontal scene. Side-by-side, swipe-driven.

**Per-panel layout** (PanelYou and PanelPartner are structurally identical except PanelYou has a UtilBar and the streak bar; PanelPartner has the bonus pill):

```
Panel{You,Partner}
├── Background        — palette-source tint (placeholder for now)
├── Layout (VBox)
│   ├── HeaderArea    — panel_heading instance (title + MOTD subtitle)
│   ├── StatusArea    — streak_bar (PanelYou) OR bonus_pill (PanelPartner)
│   ├── CalendarCard  — calendar_grid instance
│   └── TasksTray     — tasks_tray instance (which holds a TaskList)
└── UtilBar           — reserved for reboot/bug/help icons (PanelYou only; visible only when tray closed)
```

**Ceremony hosting (s26-31):** App owns the morning trigger (once per LOCAL DATE, re-checked on every focus-in; failed claim/resolve re-arms the per-day marker), a full-screen input blocker (z 200, shown only while `_morning_active`), and snap-home on ceremony `started` (`_snap_to(0)` — the ceremony must not play off-screen on the partner panel; the day-carry deferred re-open is gated on `_morning_active`). The rescue dialog (`rescue_confirm`) mounts ABOVE the blocker — later in the tree + z 300 — as the one interactive surface allowed through it. The seal-day hero beat (`hero_beat.gd`, s30) is an App-triggered overlay: trigger policy in `_maybe_fire_hero` off `day_changed`; the overlay only animates.

**Rest designation:** long-press today's cell → `rest_confirm` dialog → `State.toggle_rest_day`. Bails on sealed days (sealed-beats-clock, s31) and during the ceremony.

**Swipe behaviour** (session 16 simplification — `App.gd`):
- `DRAG_DEADZONE = 8.0` Euclidean. Below threshold = treated as a tap, container does not move, child taps fire cleanly.
- Once past deadzone, drag commits to horizontal tracking. Vertical movement is ignored from that point.
- Live 1:1 drag with `EDGE_GIVE = 50.0` soft over-extension clamp.
- Always-snap on release: drag > `COMMIT_THRESHOLD = 0.12 * viewport_width` commits to the other panel; anything less snaps back.
- `SNAP_DURATION = 0.4`, `TRANS_QUART`, `EASE_OUT`.

**Cross-panel day-carry**: when the user swipes between panels while a tray is open, the currently-open day is carried to the other panel's tray after the snap settles (`SNAP_DURATION + 0.02s` timer). The source tray closes first; the target tray opens with the same date. Suppressed while the ceremony is active (s31).

**Input plumbing**: parent Controls have `Mouse Filter = Ignore` so input falls through to `App._unhandled_input`. The swipe handler reads `InputEventMouseButton + InputEventMouseMotion` (desktop) and `InputEventScreenTouch + InputEventScreenDrag` (mobile/touch).

**Calendar binding**: `App.gd` listens to `State.self_changed` and `State.partner_changed`; on each, calls `calendar_{you,partner}.set_user(uid)` and `tray_{you,partner}.set_user(uid)`.

### 7.4 `calendar_grid.tscn` + `day_cell.tscn`
7×6 grid (always 6 rows — short months get trailing dead-future cells; locked session 14). Header has `MonthLabel` + `PrevBtn` + `NextBtn`.

Cell state classification (`calendar_grid.gd::_classify`):
1. `date < install_iso(user_id)` → `DEAD_PRE_INSTALL`
2. `date > today_iso()` → `DEAD_FUTURE`
3. `not in_displayed_month` → `DEAD_HISTORICAL` (lived day but in adjacent month — tappable, read-only)
4. Else → derive from `State.get_day(user_id, date)`:
   - empty row → `UNMARKED`
   - `rest_day` truthy → `REST`
   - count of `tasks_done` true bools: 1=RED, 2=ORANGE, 3=YELLOW, 4=GREEN, 0=UNMARKED

(Known gap, tile 4.24 [DECIDE]: a sealed zero-task day classifies UNMARKED — indistinguishable from untouched. Grey cell tier vs stamp-only grey is the open call; 4.25 will make silently-sealed never-opened days common enough to force it.)

Nav: prev disabled if displayed month ≤ install month; next disabled if displayed month ≥ today's month. No future navigation.

Cell visuals: `_apply_active_colours` for the active six states (paints `bg_panel` with a `StyleBoxFlat` carrying the cell-tier palette + applies `font_color` to the date label). `_apply_dead_visual` for the three dead states (inert grey StyleBox + `DEAD_INERT_TEXT` colour; for `DEAD_HISTORICAL` only, additionally desaturate via `modulate = DEAD_DESATURATE` on `bg_panel`, `date_label`, `sticker_slot`).

Cell input: `_gui_input` handles taps and long-presses via `_start_press` / `_end_press` + `_process` timer for long-press detection. Tap distance threshold `Layout.TAP_DRAG_THRESHOLD`; long-press duration `Layout.LONG_PRESS_DURATION`. Emits `cell_tapped(date)` or `cell_long_pressed(date)`. Dead-pre-install and dead-future cells are non-interactive (early-return on `is_interactive()`; future cells hardened 4.14c).

Listens for `State.day_changed(user_id, date)` to recompute affected cells without re-rendering the whole grid; ceremony-gated via `State.ceremony_active`.

### 7.5 `tasks_tray.tscn` + `task_list.tscn` + `task_row.tscn`
One `TasksTray` per panel, scoped to that panel's user via `set_user(user_id)`. API: `show_for_date(date)` (a TOGGLE — see ceremony beat 0b), `close()`, `is_open()`, `current_date()`. Signals: `opened(date)`, `closed(date)`, `close_pressed(date)`.

Tap-same-day on a cell = retract. Tap-different-day = close-then-open (animated).

**Subtitle slot** (built with 4.6; the repurposed "TASKS FOR" eyebrow — no scene change): ONE Label above the date, state-driven — unsealed day → `TASKS FOR` (resting label); sealed day with MOTD → the day's own MOTD; sealed without → `TASKS FOR`. In **pre-payout mode** (`set_task_pre_payout` before `show_for_date`) the slot stays neutral on open and the ceremony fills it itself (beat 5a bonus line → beat 7 MOTD overwrite).

`TaskList` is instantiated as a child of the tray body on `_ready`, then re-bound to a new `(user_id, date)` each `show_for_date`. Renders four `TaskRow` instances. **Sealed-beats-clock (s31):** `is_sealed` (`sealed_at != null`) is threaded through `_render` / `_resolve_row_state` — a sealed day renders settled and is never toggleable REGARDLESS of date, including a sealed TODAY (backwards clock travel can produce one; the server held throughout, every tap 409'd, but the client rendered it editable). Pre-payout mode explicitly wins over the sealed render (the ceremony day is always already sealed by the time beats play). Rest days render UNDONE in pre-payout (the reveal is the ceremony's).

`TaskRow` states (`task_row.gd::RowState`):
- `UNDONE` — empty circle, label normal
- `DONE` — filled green circle with checkmark
- `MISSED` — past-day undone, red ✕
- `REST_DONE` — purple ✓ when day is a rest day, all rows render in this state

`configure(idx, text, state, interactive)`. Tap emits `toggled(idx)` on interactive rows only. Past days, sealed days, and partner rows are non-interactive.

Self-day taps trigger `State.toggle_task(date, idx)` which:
1. Mutates the local day row in `State.days` (optimistic, creates a fresh row via `_new_day_row` if absent).
2. Emits `day_changed(user_id, date)`.
3. Calls `Backend.write_day(user_id, date, {tasks_done: ...})`.
4. On response, the merge policy in section 6.2 applies. No rollback for task taps.

### 7.6 `streak_bar.tscn` + `bonus_pill.tscn`
Apply styling at runtime in `_apply_styles` + `_apply_fonts` from `Palette` and `Layout` so the `.tscn` carries no literals.

`streak_bar.gd` reads `State.self_user.streak` + `State.self_user.coins`. Comma formats coins. Ceremony-gated (guards on `State.ceremony_active`, settles on `ceremony_changed(false)`) so the payout reveal isn't spoiled.

`bonus_pill.gd` reads `Bonus.partner_multiplier()` and shows "✨ ×N.N coin bonus active". Hidden when multiplier == 1.0 (slot remains in layout to preserve panel height parity).

### 7.7 `panel_heading.tscn`
Same scene instanced into each panel, inspector-set `side: "self" | "partner"` selects which user it reads from. Title reads `"{display_label}'s four tasks"` where `display_label` flips between `user.name` and `user.username.toLowerCase()` on tap (per-device, persisted in `Prefs`). Subtitle is the user's `motd` — *as built:* today's, else yesterday's (depth-1). *Locked design:* most recent non-empty `motd` ≤ today (walk back until found, gap-day-safe). Still unreconciled — see §11 and the MOTD design note. Ceremony-gated: the claim-merge `self_changed` and the beat-10 `set_motd` `day_changed` can't clobber the beat-10 glitch mid-animation.

(Display note, recorded: the title lowercases the username at render while recovery matches STORED case — the recovery-footgun item in the todo/ACTIVE list.)

### 7.8 `devkit_button.tscn` + `devkit_menu.tscn`
**`devkit_button.tscn`** — instanced in PanelYou AND in onboarding (fresh devices can't reach App). Yellow `[DEV]` button. On `_ready`, checks `OS.has_feature("editor") or OS.has_feature("devkit")`; hides/queue_frees itself if false. Tap → instantiate `devkit_menu.tscn` as a full-screen overlay.

**`devkit_menu.tscn`** — full-screen overlay. Scenario list grouped by category (`SCENARIOS` const in `devkit_menu.gd`), live rows tap-fire `DevKit.run_scenario(id)`, stubs render disabled with `(stub)`:

| Category | Live rows | Stub rows |
|---|---|---|
| Default | Default state | — |
| Identity flows | Fresh install, Recovery flow | Unpair received |
| Daily rituals | Morning sequence + payout, Rest-day variant, Streak rescue — grey day, Streak rescue — 1-day gap | MOTD entry prompt due |
| Staggered disclosure | — | Day-2 reveal, Gesture dismissal, Picker hint, Week mode cascade |
| Content drops | — | Patch announcement, Pack (free), Pack (subscriber) |
| Monetisation | — | Founders copy, Promo redemption, Day-21 disclosure |
| Dialogs | Bug catcher trigger; Hero beat — worked variant; Hero beat — rest variant (gate-bypassing, for tuning) | — |

Plus a **mock clock section** (s29): sets/clears `State.set_mock_today`; cleared automatically on scenario reboot.

### 7.9 `morning_sequence.gd` — ceremony coordinator (tile 4.6, s26-28; beat 0b s31)
App-owned, code-built (no scene). Runs the 12-beat ceremony per the morning-sequence doc: beat 0b stage normalization (close any open tray + await CLOSED, serialized, BEFORE the claim — `show_for_date` is a toggle and an already-open seal-date tray deadlocks beat 3's awaited `opened`), claim at beat 2/3 via `State.claim_morning`, tray flight + bonus line + CoinFX payout + stamp + MOTD picker + header glitch, control return at beat 12 (refresh via `Backend.poll_user`, NOT load_user — the selective merge preserves the in-flight beat-10 MOTD write). Rescue-blind: the pending_rescue path is State + App's concern (§6.2, §7.10); the coordinator's parked one-shot just receives the eventual claim-shaped response. Interruption = freeze/resume (no abort API — deleted s29). Owed: in-context tuning pass (wobble dir, coin size, picker card width, MOTD font, beat-5a hold).

### 7.10 Dialog + overlay components (code-built unless noted)
- **`rest_confirm.gd`** (4.8, s28) — rest designation confirm; afford gate reads `State.rest_cost()`; scrim cancels (designation is reversible).
- **`rescue_confirm.gd`** (s30, device-verified s31) — the streak rescue dialog; rest_confirm pattern with two deliberate deviations: scrim does NOT cancel (declining is irreversible) and every exit is an answer. Mounted above App's input blocker (tree-later + z 300). Emits `answered(accept)` → `State.resolve_claim`.
- **`hero_beat.gd`** (s30, device-verified s31) — seal-day evening celebration: 4th tick (or rest confirm) → cell PROXY flies up + enlarges (the real cell never reparents), sticker pops (placeholder glyph until 4.D), card frames the payout as arriving tomorrow — NO amount (the morning keeps the reveal; zero ceremony/server changes). Once per local date via `Prefs.hero_shown_date`, day-1 gated, rest variant delays ~0.8s for the coin tick-down. Self-contained; App owns trigger policy.
- **`bug_catcher.gd`** (4.12 mechanism, s29; D11-verified s31) — self-contained compose → sending → thanks/failed against the live endpoint; 2000-cap live counter; ONE_SHOT teardown on every exit path. Trigger currently DevKit → Dialogs; the real one-line trigger lands with 4.11's help menu; confetti waits for 4.9's SHARED system.
- **`pair_dialog.gd`** + **`partner_empty_state.gd`** (3.10A) — post-onboarding join-by-values flow on the partner panel.
- **`msg_picker.gd`** + **`msg_wheel.gd`** (4.4) — the MOTD spinner: `open(emoji_override)` for onboarding injection; `reroll()` (snappy jiggle) split from `spin()` (dramatic velocity wrap-scroll). Tuning deferred (column widths, font, selector band; EMOJI reel renders sticker IDs as TEXT — Label-based wheel needs a TextureRect column).
- **`glitch_reveal.gd`** — the per-Label worker GlitchType spawns (§6.8).

---

## 8. Bootstrap sequence (cold start, identity present + valid)

```
Loader scene loads (main scene)
  └── Loader._ready
        ├── print build flags
        ├── Identity.boot_resolved.connect(_on_boot_resolved)
        ├── ErrorLabel.hide
        └── Identity.load_identity
              ├── ConfigFile.load("user://identity.cfg") → OK
              ├── user_id = "..." (in memory)
              ├── Backend.user_loaded one-shot connected
              ├── Backend.user_load_failed one-shot connected
              └── Backend.load_user(user_id)
                    └── HTTP GET .../users/:user_id
                          └── State._on_user_loaded(data)
                                ├── self_user = data.self
                                ├── partner_user = data.partner or {}
                                ├── pair = data.pair or {}
                                ├── days re-indexed
                                ├── caches invalidated
                                ├── self_changed.emit
                                ├── partner_changed.emit (if applicable)
                                └── pair_changed.emit (if applicable)
                          └── (returns to Identity's one-shot handler)
                                └── boot_resolved.emit(true, true)
                                      └── Loader._on_boot_resolved
                                            └── _goto_app (call_deferred)
                                                  └── change_scene_to_file("res://scenes/app.tscn")
                                                        └── App._ready
                                                              ├── builds tree, wires signals
                                                              ├── _bind_self (calendar.set_user, tray.set_user)
                                                              └── _bind_partner (if partner present)
                                                                    └── Poll arms via State.self_changed
                                                                          └── Poll._interval_timer.start (30s)
```

(Then: App's per-local-date morning trigger checks whether today has been ceremonied; if not, the claim fires per §9.2.)

---

## 9. Write flows (representative)

### 9.1 Task tap (self panel)
1. User taps a `TaskRow` on an interactive day.
2. `TaskRow` emits `toggled(idx)`.
3. `TaskList` handler calls `State.toggle_task(date, idx)`.
4. `State.toggle_task`:
   a. Creates day row in `State.days[user_id][date]` if absent (`_new_day_row`).
   b. Toggles `tasks_done[idx]` in-place.
   c. Emits `day_changed(user_id, date)`.
   d. Calls `Backend.write_day(user_id, date, {tasks_done: <new array>})`.
5. `CalendarGrid` and `TasksTray`'s `TaskList` both re-render the cell/row from `day_changed`.
6. Server returns `{ok:true, data: <full day row>}`. Backend emits `day_written(data)`.
7. `State._on_day_written` merges per policy. If the row was modified server-side (e.g. seal raced — won't happen for unsealed days), the visible state updates.

### 9.2 Day claim (morning) — WIRED (sessions 26-31; rescue branch s30: pending_rescue → App's rescue_confirm dialog above the input blocker → State.resolve_claim → claim-shaped response resumes the parked coordinator)
1. App triggers at most once per **local date** (not per process — session 29): on boot once self loads, and on every application focus-in where `today_iso()` has advanced past the last-ceremonied date. A failed claim OR resolve re-arms via the failure signals and retries on the next focus-in.
2. `morning_sequence.gd` (coordinator, App-owned) normalizes the stage (beat 0b — tray closed, App snapped home, carry gated) then runs the 12-beat ceremony: `State.claim_morning(local_date)` fires at beat 2/3; State merges the banked user silently and seals the day onto `State.days` with NO signal (source-side suppression — the calendar can't spoil the reveal), emitting `morning_claimed(sealed_day)` instead.
3. Surfaces that must keep receiving their trigger signals (streak bar, panel heading) guard on `State.ceremony_active` + settle on `ceremony_changed(false)` — the second suppression mechanism.
4. Beat 12 refreshes through `Backend.poll_user` (NOT load_user — the poll's selective merge preserves the in-flight beat-10 MOTD write; session 29).
5. Interruption: backgrounding freezes the engine loop; the ceremony resumes where it left off. Process kill lands on the crash cases (pre-claim → replay; post-claim → no replay; a kill before beat 9 also loses today's MOTD pick — recovery surface is the 4.5 client half). No abort API (deleted session 29). See the morning-sequence doc.

### 9.3 Background poll
1. Every 30s while foreground, or 10s after focus-in:
2. `Poll._poll_now` checks armed + focused + has user_id, restarts interval, calls `Backend.poll_user(user_id)`.
3. `GET /users/:user_id` returns full payload.
4. `Backend.poll_completed` fires.
5. `State._on_poll_completed` applies the selective merge:
   - `self_user`, `partner_user`, `pair` taken whole from server if `!_dict_equal(local, server)`.
   - Per-day: if write in flight for `(user_id, date)`, preserve user-controlled fields; otherwise take server's row.
   - Days absent from response are dropped from local state.
6. `day_changed` emitted per actually-changed day. UI re-renders only the changed cells/rows.

### 9.4 Un-pair (entry point not yet wired)
Server endpoint exists (`POST /users/:user_id/unpair`). Both users get `pair_id = NULL` and the `pairs` row is deleted; the un-pair NOTICE is **recipient-only** (session 29, per tile 4.21): the partner gets `pending_unpair_notice = <initiator's username>`, the initiator's notice is set NULL (also clearing any stale notice from a prior relationship). Idempotent — if caller is already solo, returns `ok(null)`. UI trigger lands in Phase 4 tile 4.21 (long-press partner header → menu → Un-pair).

---

## 10. Invariants

These are always true for a healthy system. A violation indicates a bug.

### 10.1 Server-side
- Every `users` row's `pair_id` is either `NULL` or references an existing `pairs` row.
- Every `pairs` row has exactly two `users` rows referencing it. (Currently enforced by application logic, not a DB constraint.)
- `users.name` for a given `user_id` never changes after creation.
- `pairs.pair_key` is a fresh re-derivation from current user state. The server never trusts client-supplied keys.
- Sealed day rows (`sealed_at IS NOT NULL`) never have their owner-writable fields (`tasks_done`, `motd`, `rest_day`) modified. `writeDay` returns 409 `day_sealed` for these.
- `lifetime_coins >= coins` is NOT enforced (a future grant could plausibly add to `coins` without `lifetime_coins`); but `lifetime_coins` is monotonically non-decreasing per the claim endpoint.
- `longest_streak >= streak` is enforced by `Math.max(currentLongest, newStreak)` in claim.
- Streak continuity (s30): a sealed day not date-adjacent to the previously-sealed day zeroes the streak before counting, and zeroes that payout's streak bonus. The continuity anchor is DERIVED (max sealed date before target), never stored.
- Rescue one-shot (s30): there is NO stored offer state anywhere. Decline = seal grey now; accept = convert/create rest + seal now; `sealed_at` is the enforcement. Resolve is re-entrant and degrades to plain-claim behaviour when nothing is pending.
- `lifetime_coins` gains EARNS only — the rescue debit (and any spend) rides the relative `coins` write and never reduces or touches lifetime.
- Coin-source bound (s31): claim and resolve reject `local_date` outside ±2 days of server UTC (400 `bad_date`). Sinks and day writes are deliberately unbounded.
- `subscription_active`, `founders_flag`, `founders_rate_eligible`, `trial_extension_days`, `redeemed_code` are never client-set. (Currently no endpoint writes them — they remain at schema defaults. The claim READS `subscription_active` for the two-sub boost as of session 28.)
- `days.accounted_for` is monotonic write-once-to-1; writers: claim app-open upsert, writeDay upsert, rest endpoint, createUser day-0. Nothing clears it.
- Both `coins` writers (claim payout, rest debit/refund) use RELATIVE SQL arithmetic — they commute under concurrency (session 29).
- Identity fields are length-capped (49/40/50 post-normalisation) on every write path; violations are 400 `value_too_long` (session 29).
- `rest_day` is writable ONLY via the rest endpoint; writeDay 400s it (session 29).
- Drop deliveries are acked explicitly (`POST .../delivered`), never implicitly on read (s29).

### 10.2 Client-side
- `State.self_user.user_id == Identity.user_id` after boot completes successfully.
- `State.partner_user.is_empty()` ⟺ `State.self_user.pair_id` is null/empty.
- `State.pair.is_empty()` ⟺ `State.self_user.pair_id` is null/empty.
- `State.days[user_id][date]` is never present for a date in the future or before that user's `install_iso`.
- `Poll._armed` is true only after `State.self_user.user_id` has been set at least once.
- `Identity.user_id == ""` ⟺ no usable `identity.cfg` on disk.
- DevKit corner button is absent in store-build trees.
- Sealed beats the clock (s31): a day row with `sealed_at` set renders settled and is never toggleable, regardless of what the local clock claims today is. Pre-payout mode (the ceremony's own render of the already-sealed day) is the single deliberate exception.
- Exactly one of `morning_claimed` / `claim_rescue_pending` fires per claim response; the resolve's response re-enters the same funnel.

### 10.4 Day row existence

A row in `days` exists if and only if at least one write has happened against that `(user_id, date)` pair. Writes mean: a `tasks_done` toggle, a `motd` set, a rest designation, or — since session 23 — **opening the app on that date** (the claim's app-open upsert creates today's row with `accounted_for = 1`). "Opening is not a write" is DEAD.

Consequences (as built):

- A never-engaged past day has no row and renders `UNMARKED`; it is invisible to the walkback. Its streak break is detected via the adjacency check at the NEXT seal (s30). **(Tile 4.25, doc-locked, changes this:** the span walk will CREATE rows for never-engaged days, sealed grey with `accounted_for = 0` and zero payout, at the first claim after them — history becomes contiguous and the break lands at first contact. `accounted_for = 0` + `sealed_at` set will then be the canonical "never-opened, judged" state.)
- An engaged-but-idle day (opened, or ticked-then-unticked, or MOTD'd) has a row with `accounted_for = 1` and WILL seal — as grey if 0 tasks. "Skipped entirely" and "showed up but did nothing" are now distinct, and judged differently.
- Retroactive "mark yesterday as rest" = the STREAK RESCUE GATE, built session 30 (server wire-verified; client device-verified s31): pre-ceremony beat 0, two-phase claim + re-entrant resolve. See §5.3.

### 10.5 Day-state taxonomy

"Empty day" is not one concept — it is per-consumer, and each feature has historically invented its own inline predicate. They are individually correct but undocumented, so every new day-history walker re-derives them and risks drift. This is the canonical list; new walkers reuse a named predicate rather than rolling their own.

States (the `accounted_for` split is LIVE since session 23):

1. **Never touched** — no row in `days`. (4.25 will retire this for past dates — every past day gains a terminal row at the next claim; only TODAY and the pre-install region stay rowless.)
2. **Row exists, not engaged** — `accounted_for = 0`. Rare as-built (every current write path accounts); becomes the canonical never-opened-sealed-grey state at 4.25.
3. **Engaged, unsealed** — `accounted_for = 1`, `sealed_at IS NULL`. What claim hunts.
4. **Sealed** — immutable history. What the carry-forward subtitle reads.
5. ~~(arriving with week mode) an opted-out weekday~~ — RESOLVED MOOT 2026-06-10: week mode is label-overrides only; there is no opted-out-day concept, so no fifth state arrives. See the week-mode doc.

Per-consumer predicates (use these; do not re-derive):

| Consumer | Predicate | Hunts |
|---|---|---|
| Calendar `_classify` | row empty OR absent → `UNMARKED` | treats 1 + 2 alike (4.24 [DECIDE] will split sealed-grey out) |
| Claim walkback (as built, s23) | `sealed_at IS NULL AND accounted_for = 1 AND date < local_date` | state 3, incl. engaged 0-task |
| Header MOTD fallback (locked design, s29) | today's `motd`, else most recent prior day with `motd != ''` (full walkback) | 3 or 4 (as built: depth-1 — see §7.7/§11) |
| Sealed-beats-clock (client, s31) | `sealed_at != null` → settled, never toggleable | state 4, any date |

**Structural guard.** Document the taxonomy; do NOT add structure to model it gratuitously — no `is_empty` column, no `status` enum, no create-rows-on-open-as-a-feature. Those are the premature-structure move the architectural-preference rule refuses. The ONE structural addition that earns its place is `accounted_for`: a single write-once boolean that *retires* a fragile JSON string-match (clarity/robustness, not cleverness — the preferred side of the meta-rule) and is what splits state 2 from state 3 for an opened-but-idle day. (4.25's grey rows are not new structure — they reuse the existing seal columns; the walk is behaviour, not schema.)

**Consequence (live since session 23, extended session 29).** App-open sets `accounted_for` on today's row, and as of session 29 so does any writeDay (task tick / MOTD save) — every documented engagement trigger is implemented. Confirmed effect: an opened-then-idle day seals as a disappointment rather than vanishing as never-touched, and a day worked entirely inside a long-lived app session (no fresh open on that date) is still sealable.

### 10.3 Cross-system
- Every `Backend.<x>_completed` signal has a corresponding `<x>_failed` and exactly one of the two fires per call. (Network errors → failure signal.)
- DevKit scenario URLs return 404 in production. (Verified manually at session 18.)
- The `four-tasks-api` worker bundle is identical between prod and dev; differentiation is purely in `env.IS_DEV` and the D1 binding.

---

## 11. Known fragility and tech debt

Captured here because the system map is also the place that should make these visible to whoever's reading.

- **Hardcoded timezone offset** in `State.gd` (`+ 8 * 3600` in `install_iso`/`today_iso`/`yesterday_iso`; `calendar_math.gd` does UTC-only and defers the offset to `State.gd`). v1.0 fix: use the OS offset `Time.get_time_zone_from_system().bias` — DST-correct for "now", no library. Full IANA-name capture is deferred to v1.x (with the s29-parked `yesterday_iso` epoch−86400 DST edge — same family, same fix window). See the timezone doc's V1.0 SCOPE. Verify the bias sign on a real Android device.
- **Stranded-day shadowing + retroactive streak collapse** in the single-target walkback — known, doc-locked fix is tile 4.25 (see §5.3 forward note). Until it ships, a hole breaks the streak one day late and a double-claim-failure strands a payout.
- **No App reaction to backwards local-date moves** — stale paint + a now-future tray can stay open (s31). Designed fix is tile 4.23 (focus-in date watcher); forward moves are already claim-handled.
- **`DeadOverlay` is a placeholder `Panel`** on `day_cell.tscn`. Theme overlay wiring deferred to tile 4.14b.
- **Layout values duplicated as literals in `.tscn` files** with hand-maintained comment-block mappings to `Layout.gd` constants. Drift risk.
- **`tasks_done` parses to JSON but the field on a fresh `_new_day_row` is a native array; `rest_day` similarly arrives as bool from client and int (0/1) from server.** Code casts via `bool(...)` defensively. Source of past bugs in `_classify`.
- **Solo-user `_on_day_written` for partner days has no defined behaviour** — partner days only exist when partner exists. Currently no code path can produce a `partner` day write on the client side. Future endpoints (e.g. cross-user writes if partner reactions are reinstated) would need to revisit this.
- **`State.self_user` being whole-replaced by polls** can briefly stomp local optimistic updates from an in-flight `PUT /users`. Accepted for v1.0 — rare condition, next poll resettles.
- **Promo code tables have a design doc (s29) but no read/write path**. `subscription_active`, `founders_flag`, etc. remain server-set-only and inert. Phase 5 work.
- **No retry/backoff in Backend.gd**. Single attempt per call; failure signals straight through. Tolerable while users live in Australia on home wifi; reconsider before broader testing. (The per-day claim/resolve re-arm is the one deliberate retry mechanism.)
- **DevKit menu carries ~14 stub scenarios** with `(stub)` suffix. Visual noise during testing; cosmetic.
- **Focus-in deferred poll suspected non-functional in editor** (session 19, observed). Print logs show only the 30s interval ticks. Most likely cause: Godot editor's run window doesn't reliably propagate `NOTIFICATION_APPLICATION_FOCUS_OUT/IN`. Verified working on device for the morning trigger (s29-31 ledger); treat editor-only oddities as editor artefacts.
- **Optimistic-write guarding in `State.gd` is inert under last-write-wins.** `_writes_in_flight`, `_pending_snapshot`, and the per-field merge in `_on_day_written` do real work only on the failure-rollback path. Safe for `tasks_done` (LWW), `motd` (single-writer, write-on-exit), and `rest_day` (refundable), each for a different reason. Kept against a future endpoint that genuinely rejects user writes. Residual (s29 #7 / s31): rapid-tick untick misbehaves ONLY under mock-clock hopping — the shared per-day flight flag lacks request identity. PARKED; fix = per-request write identity, only if it bites without DevKit clock games.
- **`panel_heading` subtitle uses a one-row-back fallback** (today's `motd`, else yesterday's). Intended rule is most-recent-non-empty-`motd` ≤ today (gap-day safe). Still unreconciled post-4.6. See §7.7.
- **`State._dict_equal` is key-order-sensitive** — it compares via `JSON.stringify(a) == JSON.stringify(b)`, which preserves insertion order, so two dicts with identical content but different key order miscompare as unequal and fire a spurious `*_changed` re-render on every poll. Dormant while cells are flat colour; becomes visible once cells carry sticker art + theme overlays. Fix before the art tiles: replace with an order-independent recursive deep-equal (arrays stay order-sensitive). Laptop + device test.
- **Rotation/join TOCTOU returns 500 not 409** on a simultaneous-write race (s29). `pair_key` UNIQUE keeps correctness. Cosmetic; fix opportunistically if ever touching those handlers.
- **Runtime sticker-asset loading is UNDESIGNED** — `content_drop_assets.r2_keys` are recorded but no client can fetch/render them; until the theme-prep conversation lands, drops deliver announcements + catalogue data, not new art.

---

## 12. What this map does not describe

- Future tiles. The map updates *as* tiles land, not before — tile 4.25 is the one deliberate exception, flagged as doc-locked forward notes (§5.3, §10.4, §10.5) because its design supersedes statements this map would otherwise make unqualified.
- Pixel art, sticker assets, theme palettes — those live with the theme design notes; nothing is painted yet (Phase 4b).
- Promo code behaviour — schema + design doc exist, no endpoints.
- Subscription plumbing (RevenueCat, webhook, paywall) — Phase 5 (tile 5.s); the claim already reads `subscription_active`.
- Partner reactions — deferred to v1.x; doc in `deferred/`.

---

## 13. Change log for this document

| Date | Session | Change |
|---|---|---|
| 2026-05-27 | 19 (mobile) | Initial draft. Snapshot at end of session 18 (Phase 2 closed, Phase 3 pending). |
| 2026-05-27 | 19 (mobile) | Added §10.4 (day row existence invariant + walkback gap), flagged focus-in poll suspected-non-functional in §11. |
| 2026-05-29 | 21 | Capture propagation: §5 router-convention note; §5.3 + §10.4 annotated with the locked `accounted_for` walkback / 0-task tier / partner-multiplier redesign; new §10.5 day-state taxonomy; §7.7 panel_heading built-vs-design; §11 added inert-guard, panel_heading, `_dict_equal`, and claim-redesign debt, and corrected the timezone-offset entry to `State.gd` / OS-offset. |
| 2026-06-11 | 30 | §1.5 added; §5.x/§9.2/§10.x refreshed to s30 as-built (amended streak rule, rescue gate + resolve, gap continuity, hero beat, grey/gap devkit scenarios). |
| 2026-06-11 | 32 | Full refresh: §4 to ten tables (content drops), §5.1 full endpoint surface (MOTD endpoints, drops, rest, reroll, resolve_solo), §5.2 FK-ordered wipe, §5.3 ±2d bound + stranded-day fault + 4.25 forward note, §6-§7 complete client inventory rewrite (sessions 24-31), §10 sealed-beats-clock + bound invariants + 4.25 notes, §11/§12 pruned of shipped items, repo path corrected. |
