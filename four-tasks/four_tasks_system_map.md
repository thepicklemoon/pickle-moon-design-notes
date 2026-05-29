# Four Tasks — System Map

Observational. Describes the system **as it currently exists at the head of the four-tasks repo** (Godot port, end of session 18, Phase 2 closed, Phase 3 not started). Drift between this doc and the code is a defect in this doc — fix it here as soon as the code changes, in the same commit that changes the code.

Design intent and rationale live elsewhere (the design docs in `pickle-moon-design-notes/four-tasks/`). This map only describes what's wired up, what talks to what, and what invariants the running system holds.

---

## 1. Top-down overview

Four Tasks runs as a Godot 4 client that talks to a single Cloudflare Worker over HTTPS, which reads/writes a D1 SQLite database. The client is a thin mirror of server state; the server is the source of truth for everything except (a) the local `user://identity.cfg` file that names which user this device is and (b) a handful of per-device display preferences.

There are two server deployments behind two URLs, sharing one bundle of TypeScript:

- **Production** — `https://four-tasks-api.thepicklemoon.workers.dev`, backed by D1 `four-tasks` (UUID `d92e5792-61a1-42cf-969c-9b33b8cc1475`).
- **Dev** — `https://four-tasks-api-dev.thepicklemoon.workers.dev`, backed by D1 `four-tasks-dev` (UUID `0f46f48e-1549-4664-a692-87384af8ff38`). Exposes `/devkit/*` scenario endpoints. Production omits the `IS_DEV` env var, so those routes return 404 there.

The client picks dev vs prod URL per-request at runtime via `Backend._base_url()`, which checks `OS.has_feature("editor") or OS.has_feature("devkit")`. Editor runs always hit dev. Exported store builds always hit prod unless the custom `devkit` feature is set in the export preset.

---

## 2. Repository layout

The Godot project and Cloudflare Worker live in **one repo**, `four-tasks`, on `@thepicklemoon`. Local path: `C:\Users\Necruccio\Documents\four-tasks\`.

Inside that repo (paths inferred from the files in this Project; the actual repo may use `scripts/` and `scenes/` subfolders — note the `preload("res://scenes/...")` references in code):

- `server/src/index.ts` — Worker entry point.
- `server/src/stamp_messages.ts` — stub pools, one per stamp tier.
- `server/schema.sql` — single source of truth for the D1 schema.
- `server/seed_devkit.sql` — re-runnable seed for the two DevKit test users.
- `server/wrangler.toml` — two environments (production, dev) in one file.
- `scripts/*.gd` — autoloads (PascalCase: `Backend.gd`, `State.gd`, etc.) and scene scripts (snake_case: `day_cell.gd`, `task_row.gd`, etc.).
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

Seven tables. All times are unix epoch seconds (`INTEGER`).

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

**Week mode** — `week_mode_weekdays` (INTEGER NN, default 0; bitmask, bit N = ISO weekday N).

**Subscription / loyalty / promo** (server-set only) — `subscription_active`, `founders_flag`, `founders_rate_eligible`, `trial_extension_days` (all INTEGER NN, default 0), `redeemed_code` (TEXT, nullable).

Index: `idx_users_pair_id` on `pair_id`.

### 4.3 `user_weekday_overrides`
Per-user-per-weekday task label override (for week mode, deferred to Phase 4/v1.x).
PK: composite `(user_id, weekday)`. Columns: `user_id` (FK), `weekday` (INTEGER, 1–7), `task_labels` (TEXT, JSON array).

### 4.4 `days`
PK: composite `(user_id, date)`.

**Owner-writable, immutable after seal** — `tasks_done` (TEXT NN, default `'[false,false,false,false]'`, JSON array of 4 bools), `motd` (TEXT NN, default `''`), `rest_day` (INTEGER NN, default 0).

**Server-written at seal time only** — `stamp` (TEXT NN, default `''`), `sealed_at` (INTEGER, nullable), `day_theme_state` (TEXT NN, default `'{}'`).

### 4.5 `promo_codes` and `redemption_attempts`
Schema landed in session 12, no design doc written, no endpoints implemented. Scope-deferred to Phase 5 (`/redeem` endpoint + `migration_006`). Currently inert — no read or write path touches them.

### 4.6 `bug_reports`
Append-only operational log. PK: `id` (AUTOINCREMENT). Columns: `user_id` (FK), `pair_id_snapshot` (TEXT, value of `users.pair_id` at submission time), `submitted_at`, `message` (1–500 chars), `app_version` (TEXT, ≤50 chars, default `''`).

---

## 5. Server endpoints (`server/src/index.ts`)

All production responses use the uniform envelope: `{ok: true, data: ...}` on success, `{ok: false, error: "<code>", message: "..."}` on failure. Status codes are restricted to **200, 400, 404, 409, 500**. No 201/202/204/401/403/429.

`?caller=` is reserved and unused — all v1.0 writes are self-writes addressed by URL.

The router is order-dependent and prefix-matched — each branch tests that the path starts with `/users/` and ends with a given suffix. This holds only while suffixes stay mutually non-prefixing and no path segment carries a user-controlled value (paths carry only server-generated UUIDs and validated dates). Adding a route with a colliding suffix, or routing on a user-supplied value, breaks these assumptions. No bug today — a constraint on future routes.

### 5.1 Production endpoints

| Method | Path | Body | Success data | Failure codes |
|---|---|---|---|---|
| GET | `/health` | — | `{status:"ok", service:"four-tasks-api"}` | — |
| POST | `/resolve` | `{self:{name,username,active_leader}, partner:{name,username,active_leader}}` | `{user_id, pair_id}` | 400 bad_input · 404 pair_not_found |
| POST | `/users` | `{name, username, active_leader, task_labels[4], timezone}` | `{user_id}` | 400 bad_input · 500 server_error |
| GET | `/users/:user_id` | — | `{self, partner, pair, days}` (days windowed to `>= (currentYear-1)-01-01`) | 400 bad_user_id · 404 user_not_found · 500 server_error |
| PUT | `/users/:user_id` | partial `{username?, active_leader?, task_labels?, timezone?, tutorial_progress?, active_theme?, active_stickers?, week_mode_weekdays?}` | full user row | 400 bad_user_id/bad_input · 404 user_not_found · 409 pair_key_collision · 500 server_error |
| POST | `/users/:user_id/bug_report` | `{message (1-500), app_version?}` | `{report_id}` | 400 · 404 |
| POST | `/users/:user_id/dismiss_unpair_notice` | `{}` | `null` (idempotent) | 400 · 404 |
| POST | `/users/:user_id/unpair` | `{}` | `null` (idempotent on solo) | 400 · 404 · 500 |
| POST | `/users/:user_id/join_by_values` | `{partner:{name,username,active_leader}}` | `{pair_id}` | 400 bad_input · 404 partner_not_found · 409 already_paired/ambiguous_match/pair_key_collision · 500 |
| POST | `/users/:user_id/claim` | `{local_date:"YYYY-MM-DD"}` | `{sealed_day, user}` (sealed_day null if nothing owed) | 400 bad_input/bad_date · 404 user_not_found · 500 |
| PUT | `/users/:user_id/days/:date` | partial `{tasks_done[4]?, motd?, rest_day?}` | full day row | 400 bad_user_id/bad_date/bad_input · 404 user_not_found · 409 day_sealed · 500 |

### 5.2 Dev-only endpoints (gated by `env.IS_DEV === "true"`)

| Method | Path | Behaviour |
|---|---|---|
| POST | `/devkit/scenario/default` | Wipe both test users + pair; reseed canonical spread (~14 days install age, ~6 days of day rows per user covering every cell state) |
| POST | `/devkit/scenario/fresh_install` | Server-side teardown only; client wipes `identity.cfg` independently |
| POST | `/devkit/scenario/recovery_flow` | Same server-side teardown; client behaviour differs (recovery UI not yet built) |

Stable DevKit identifiers (hardcoded; never rotate):

- TestUser `user_id`: `aaaaaaaa-0000-4000-8000-000000000001`
- TestPartner `user_id`: `aaaaaaaa-0000-4000-8000-000000000002`
- TestPair `pair_id`: `bbbbbbbb-0000-4000-8000-000000000001`
- TestPair `pair_key`: `"devkit_test_pair_key"` (literal, not a hash)

### 5.3 Endpoint-specific contracts that matter for testing

**`POST /users/:user_id/claim`** — the most complex endpoint by far.

Walkback query (as built): most recent past day (`date < local_date`) for this user where `sealed_at IS NULL AND (rest_day = 1 OR tasks_done != '[false,false,false,false]')`. App-open semantics mean at most one such row exists.

*Locked design, not yet built:* this predicate becomes `sealed_at IS NULL AND accounted_for = 1` — a write-once engagement boolean (new `days.accounted_for` column) that retires the fragile JSON string-match and lets a genuinely-engaged 0-task day seal as a disappointment instead of being skipped. See the morning-sequence doc and the day-state taxonomy (§10.5); tracked in §11.

Stamp tier from `(tasks_completed_count, rest_day)`:
- `rest_day=1` → purple, regardless of count
- count=4 → green
- count=3 → yellow
- count=2 → orange
- count=1 → red
- count=0 + not rest → *as built:* walkback excludes it, no claim possible. *Locked design:* an `accounted_for` 0-task day seals with a new **disappointment** tier (colour/pool TBD) — judgement can't be dodged for a day you showed up to. Not yet built.

Coin payout: 0 on rest days; otherwise per completed task roll `Math.floor(Math.random() * 501) + 900` (so each task ∈ [900, 1400]) and sum. Server-side randomness — client cannot predict.

*Partner multiplier (specced, not built — currently ×1.0):* when built, the summed payout is multiplied by the partner's engagement on the **same date as the sealed row** (graduated 1.0–1.5; partner rest = 1.5), read against the sealed row's own date — never a hardcoded today/yesterday. The client preview pill (§6.6 `Bonus.gd`) reads the partner's *current* day; the server applies the final factor at seal against the sealed date. See the morning-sequence doc.

Streak update:
- green → `streak + 1`
- purple → unchanged
- anything else → reset to 0

`longest_streak = max(longest_streak, new_streak)`.

Theme snapshot: `users.active_theme` (the string, not parsed) copied into `days.day_theme_state` for the sealed day.

Single atomic batch updates the day row (stamp, sealed_at, day_theme_state) and the user row (coins, lifetime_coins, streak, longest_streak).

**`PUT /users/:user_id`** — paired users with `username` or `active_leader` changes trigger a rotation transaction: re-derive `pair_key` from new self + current partner, UNIQUE-check against other pairs, then atomic batch updates `pairs.pair_key` + `users` row. Solo users skip rotation.

**`POST /users/:user_id/join_by_values`** — caller must be solo (`pair_id IS NULL`), or 409 `already_paired`. Search restricted to other solo users (`pair_id IS NULL`) by exact `(name, username, active_leader)` match. Zero matches → 404. Two-plus matches → 409 `ambiguous_match`. One match → derive pair_key, check collision, batch INSERT pair + UPDATE both users.

**`POST /resolve`** — pair_key lookup. After matching, additionally filter by self's `name` to disambiguate the two users in the matched pair. Returns 404 `pair_not_found` if the pair exists but neither user matches the supplied self.name.

---

## 6. Godot client — autoloads

Order of registration (matters; later autoloads depend on earlier ones at `_ready`):

```
Backend → State → Identity → Poll → DevKit → Bonus → Layout → Palette → Fonts → Prefs
```

(Order verified from the load-order discussion in session 13/16 devlogs. Exact `project.godot` ordering should match.)

### 6.1 `Backend.gd` — stateless HTTP layer
The only node that makes network calls. Spawns a fresh `HTTPRequest` child per call, awaits, frees it. Stateless: callers pass `user_id` into every method that needs one.

Public methods, one per endpoint:
`check_health, create_user, load_user, poll_user, update_user, write_day, claim, submit_bug_report, unpair, dismiss_unpair_notice, join_by_values, resolve_pair, devkit_run_scenario`

Signals, one success + one failure per endpoint (22 total at last count; verify against `Backend.gd` signal declarations):

- `health_checked` / `health_check_failed`
- `user_created` / `user_creation_failed`
- `user_loaded` / `user_load_failed`
- `user_updated` / `user_update_failed`
- `day_written` / `day_write_failed`
- `claim_completed` / `claim_failed`
- `bug_reported` / `bug_report_failed`
- `unpair_completed` / `unpair_failed`
- `unpair_notice_dismissed` / `unpair_notice_dismiss_failed`
- `joined_by_values` / `join_by_values_failed`
- `pair_resolved` / `pair_resolve_failed`
- `poll_completed` / `poll_failed` (reuses `GET /users/:user_id` so State can apply selective merge)
- `devkit_scenario_ran` / `devkit_scenario_failed`

URL switching: `_base_url()` returns `BASE_URL_DEV` or `BASE_URL_PROD` per request, picked by `OS.has_feature("editor") or OS.has_feature("devkit")`. Called per-request — not cached.

### 6.2 `State.gd` — in-memory mirror
Source-of-truth invariant: **server is canonical**. State writes optimistically and reconciles on Backend response.

Fields:
- `self_user: Dictionary` — current self payload
- `partner_user: Dictionary` — current partner payload, `{}` if solo
- `pair: Dictionary` — current pair row, `{}` if solo
- `days: Dictionary` — nested `{user_id: {date: row}}` for O(1) lookup
- `_writes_in_flight: Dictionary` — per-(user_id, date) flag for in-flight `PUT /days/:date`
- `_pending_snapshot: Dictionary` — single rollback target for `set_motd` / `toggle_rest_day` failures
- Cached helpers: `install_iso(user_id)` (per-user cache), `today_iso()`, `yesterday_iso()`

Signals: `self_changed`, `partner_changed`, `pair_changed`, `day_changed(user_id, date)`.

**Merge policies** — both important for diagnosing race conditions:

`_on_day_written` (response to `PUT /days/:date`): take server's value for `stamp`, `sealed_at`, `day_theme_state`. Preserve local copy of `tasks_done`, `motd`, `rest_day` (server's view was canonical *at write time*, but rapid taps may have advanced local state past the response).

`_on_poll_completed` (response to background poll): per-day diff with the same merge rule for self days that have an in-flight write; otherwise take server's row whole. `self_user`, `partner_user`, `pair` taken whole from server (brief stomp risk on in-flight `PUT /users` accepted for v1.0). Days that vanish from the response are dropped (defensive — shouldn't happen in practice). Emits `day_changed` only for rows that actually differ.

`_on_claim_completed`: replaces `self_user` with the returned user row; if `sealed_day != null`, triggers a full `Backend.load_user(self_user.user_id)` to refresh the calendar.

**Concurrency model (session 16 simplification)** — Task taps fire optimistic update + server write immediately, with NO in-flight guard. Server uses `INSERT ON CONFLICT REPLACE` so last write wins. There is NO rollback path for task taps. `set_motd` and `toggle_rest_day` retain the single-snapshot rollback via `_pending_snapshot`.

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

`run_scenario(name)`:
1. Per-scenario client prep. `fresh_install` and `recovery_flow` call `Identity.clear()` before the server call.
2. `Backend.devkit_run_scenario(name)`.
3. On `devkit_scenario_ran` → `_reboot()` swaps to `loader.tscn` via `call_deferred`. Real boot path re-executes with whatever server state the scenario produced.
4. On `devkit_scenario_failed` → push warning, do NOT reboot.

### 6.6 `Bonus.gd` — partner coin multiplier
Reads `State.partner_user`'s yesterday day row and returns a float multiplier (1.0 / 1.2 / 1.3 / 1.4 / 1.5). Same six-line rule the server's claim endpoint will apply at payout time when partner-bonus mechanics formally land — currently a display-only computation. Solo users return 1.0.

### 6.7 `Layout.gd`, `Palette.gd`, `Fonts.gd`, `Prefs.gd` — single sources of truth
- `Layout.gd` — every layout dimension (cell sizes, paddings, font sizes, panel widths). Scenes carry literals; a comment block at the top of each `.tscn` maps the literals back to `Layout.*` names so they can be kept in sync by hand. The `.tscn` cannot reference autoload constants directly.
- `Palette.gd` — non-sacred colours (panel backgrounds, button states, etc.). **Sacred** colours (cell-tier ramp, task-row state palette) remain hardcoded in `day_cell.gd` and `task_row.gd` because they're theme-locked.
- `Fonts.gd` — Departure Mono (project default), JetBrains Mono (MOTD subtitle). Set globally via Project Settings → GUI → Theme, plus an explicit override on the subtitle node.
- `Prefs.gd` — per-device local prefs (e.g. which heading-name variant is showing on each panel). Stored locally; never round-tripped to server.

---

## 7. Scenes

### 7.1 `loader.tscn` — boot router
Main scene of the project. Centred splash texture + hidden `ErrorLabel`. `Loader.gd` connects to `Identity.boot_resolved` and routes:

- `(false, false)` — no identity → `change_scene_to_file("res://scenes/onboarding.tscn")` (deferred)
- `(true, true)` — identity loaded, payload received → `change_scene_to_file("res://scenes/app.tscn")` (deferred)
- `(true, false)` — identity stored, server rejected → show error label, hook `gui_input` for tap-to-retry. Retry disconnects the handler, hides the label, calls `Identity.load_identity()` again.

Boot diagnostic printed on `_ready`: `[Loader] editor=<bool> devkit=<bool>`.

### 7.2 `onboarding.tscn` — STUB
Single `Control` with one placeholder `Label`. Phase 3 tiles 3.1–3.10 will replace.

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

**Swipe behaviour** (session 16 simplification — `App.gd`):
- `DRAG_DEADZONE = 8.0` Euclidean. Below threshold = treated as a tap, container does not move, child taps fire cleanly.
- Once past deadzone, drag commits to horizontal tracking. Vertical movement is ignored from that point.
- Live 1:1 drag with `EDGE_GIVE = 50.0` soft over-extension clamp.
- Always-snap on release: drag > `COMMIT_THRESHOLD = 0.12 * viewport_width` commits to the other panel; anything less snaps back.
- `SNAP_DURATION = 0.4`, `TRANS_QUART`, `EASE_OUT`.

**Cross-panel day-carry**: when the user swipes between panels while a tray is open, the currently-open day is carried to the other panel's tray after the snap settles (`SNAP_DURATION + 0.02s` timer). The source tray closes first; the target tray opens with the same date.

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

Nav: prev disabled if displayed month ≤ install month; next disabled if displayed month ≥ today's month. No future navigation.

Cell visuals: `_apply_active_colours` for the active six states (paints `bg_panel` with a `StyleBoxFlat` carrying the cell-tier palette + applies `font_color` to the date label). `_apply_dead_visual` for the three dead states (inert grey StyleBox + `DEAD_INERT_TEXT` colour; for `DEAD_HISTORICAL` only, additionally desaturate via `modulate = DEAD_DESATURATE` on `bg_panel`, `date_label`, `sticker_slot`).

Cell input: `_gui_input` handles taps and long-presses via `_start_press` / `_end_press` + `_process` timer for long-press detection. Tap distance threshold `Layout.TAP_DRAG_THRESHOLD`; long-press duration `Layout.LONG_PRESS_DURATION`. Emits `cell_tapped(date)` or `cell_long_pressed(date)`. Dead-pre-install and dead-future cells are non-interactive (early-return on `is_interactive()`).

Listens for `State.day_changed(user_id, date)` to recompute affected cells without re-rendering the whole grid.

### 7.5 `tasks_tray.tscn` + `task_list.tscn` + `task_row.tscn`
One `TasksTray` per panel, scoped to that panel's user via `set_user(user_id)`. API: `show_for_date(date)`, `close()`, `is_open()`, `current_date()`. Signals: `opened(date)`, `closed(date)`, `close_pressed(date)`.

Tap-same-day on a cell = retract. Tap-different-day = close-then-open (animated).

`TaskList` is instantiated as a child of the tray body on `_ready`, then re-bound to a new `(user_id, date)` each `show_for_date`. Renders four `TaskRow` instances.

`TaskRow` states (`task_row.gd::RowState`):
- `UNDONE` — empty circle, label normal
- `DONE` — filled green circle with checkmark
- `MISSED` — past-day undone, red ✕
- `REST_DONE` — purple ✓ when day is a rest day, all rows render in this state

`configure(idx, text, state, interactive)`. Tap emits `toggled(idx)` on interactive rows only. Past days and partner rows are non-interactive.

Self-day taps trigger `State.toggle_task(user_id, date, idx)` which:
1. Mutates the local day row in `State.days` (optimistic, creates a fresh row via `_new_day_row` if absent).
2. Emits `day_changed(user_id, date)`.
3. Calls `Backend.write_day(user_id, date, {tasks_done: ...})`.
4. On response, the merge policy in section 6.2 applies. No rollback for task taps.

### 7.6 `streak_bar.tscn` + `bonus_pill.tscn`
Apply styling at runtime in `_apply_styles` + `_apply_fonts` from `Palette` and `Layout` so the `.tscn` carries no literals.

`streak_bar.gd` reads `State.self_user.streak` + `State.self_user.coins`. Comma formats coins.

`bonus_pill.gd` reads `Bonus.partner_multiplier()` and shows "✨ ×N.N coin bonus active". Hidden when multiplier == 1.0 (slot remains in layout to preserve panel height parity).

### 7.7 `panel_heading.tscn`
Same scene instanced into each panel, inspector-set `side: "self" | "partner"` selects which user it reads from. Title reads `"{display_label}'s four tasks"` where `display_label` flips between `user.name` and `user.username.toLowerCase()` on tap (per-device, persisted in `Prefs`). Subtitle is the user's `motd`. *As built (tile 2.A):* a two-row fallback — today's `motd`, else yesterday's. *Locked design:* carry-forward — the most recent non-empty `motd` at or before today (walk back until found), gap-day-safe where a one-row fallback is not. Same pixels most days; they diverge in the pre-morning-sequence window and across multi-day gaps. Reconciliation deferred to tile 4.6. See §11 and the MOTD design note.

### 7.8 `devkit_button.tscn` + `devkit_menu.tscn`
**`devkit_button.tscn`** — parented inside PanelYou below `Layout`. Yellow `[DEV]` button. On `_ready`, checks `OS.has_feature("editor") or OS.has_feature("devkit")`; hides/queue_frees itself if false. Tap → instantiate `devkit_menu.tscn` as a full-screen overlay.

**`devkit_menu.tscn`** — full-screen overlay. Scenario list grouped by category, defined in the `SCENARIOS` const in `devkit_menu.gd`:

| Category | Rows |
|---|---|
| Default | Default state (live) |
| Identity flows | Fresh install (live), Recovery flow (live), Unpair received (stub) |
| Daily rituals | Morning sequence + payout (stub), Rest-day morning variant (stub), MOTD entry prompt due (stub) |
| Staggered disclosure | Day-2 philosophy reveal, Gesture dismissal pattern, Picker hint trigger, Week mode cascade (all stubs) |
| ...further categories | (additional stubs) |

Live rows tap-fire `DevKit.run_scenario(id)`. Stub rows render with `(stub)` suffix, disabled, no-op except for an info print.

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

---

## 9. Write flows (representative)

### 9.1 Task tap (self panel)
1. User taps a `TaskRow` on an interactive day.
2. `TaskRow` emits `toggled(idx)`.
3. `TaskList` handler calls `State.toggle_task(user_id, date, idx)`.
4. `State.toggle_task`:
   a. Creates day row in `State.days[user_id][date]` if absent (`_new_day_row`).
   b. Toggles `tasks_done[idx]` in-place.
   c. Emits `day_changed(user_id, date)`.
   d. Calls `Backend.write_day(user_id, date, {tasks_done: <new array>})`.
5. `CalendarGrid` and `TasksTray`'s `TaskList` both re-render the cell/row from `day_changed`.
6. Server returns `{ok:true, data: <full day row>}`. Backend emits `day_written(data)`.
7. `State._on_day_written` merges per policy. If the row was modified server-side (e.g. seal raced — won't happen for unsealed days), the visible state updates.

### 9.2 Day claim (morning)
Not yet wired to a UI trigger in the current build — claim is implemented server-side at tile 1.3 and reachable via `Backend.claim(user_id, local_date)`, but no client code calls it yet. Phase 4 (morning sequence) will wire the trigger.

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
Server endpoint exists (`POST /users/:user_id/unpair`). Symmetric: both users get `pair_id = NULL`, `pending_unpair_notice = <other's username>`, and the `pairs` row is deleted. Idempotent — if caller is already solo, returns `ok(null)`. UI trigger lands in Phase 4 tile 4.21 (long-press partner header → menu → Un-pair).

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
- `subscription_active`, `founders_flag`, `founders_rate_eligible`, `trial_extension_days`, `redeemed_code` are never client-set. (Currently no endpoint writes them — they remain at schema defaults.)

### 10.2 Client-side
- `State.self_user.user_id == Identity.user_id` after boot completes successfully.
- `State.partner_user.is_empty()` ⟺ `State.self_user.pair_id` is null/empty.
- `State.pair.is_empty()` ⟺ `State.self_user.pair_id` is null/empty.
- `State.days[user_id][date]` is never present for a date in the future or before that user's `install_iso`.
- `Poll._armed` is true only after `State.self_user.user_id` has been set at least once.
- `Identity.user_id == ""` ⟺ no usable `identity.cfg` on disk.
- DevKit corner button is absent in store-build trees.

### 10.4 Day row existence

A row in `days` exists if and only if at least one write has happened against that `(user_id, date)` pair. Writes mean: a `tasks_done` toggle, a `motd` set, or a `rest_day` toggle. Opening the app on a date is NOT a write — there is no "viewed" tracking.

Consequences:

- An empty unsealed past day and a never-opened past day are indistinguishable in the database. Both render as `UNMARKED` in the calendar.
- Claim's walkback filter `(rest_day = 1 OR tasks_done != '[false,false,false,false]')` (as built) correctly skips both cases — neither has data, neither needs sealing, neither breaks or extends a streak. (The locked `accounted_for` design changes this — see §10.5: a day the user *opened* becomes accounted-for and will seal, distinguishing it from a day never opened.)
- This means the system handles "user skipped a day entirely" and "user opened the app but ticked nothing" identically. That is the intended behaviour for v1.0 (consistent with `tracking_design_notes` being DEFERRED).
- Distinguishing these cases needs a new field. That field is now designed — the `accounted_for` column, locked this session, not yet built; until it ships, the distinction does not exist in the database.

The `accounted_for` design clarifies the mechanics: any rest-designation sets `accounted_for` and creates the row, so a marked day seals as rest. Whether the morning flow actively *offers* a retroactive "mark yesterday as rest" remains a tile-4.6 UX decision — resolve against `four_tasks_morning_sequence_design_notes.md`.

### 10.5 Day-state taxonomy

"Empty day" is not one concept — it is per-consumer, and each feature has historically invented its own inline predicate. They are individually correct but undocumented, so every new day-history walker re-derives them and risks drift. This is the canonical list; new walkers reuse a named predicate rather than rolling their own.

States (currently collapsed by the schema; the `accounted_for` design splits state 2):

1. **Never touched** — no row in `days`.
2. **Row exists, not engaged** — row created but all-false tasks, empty motd, not rest, and (under the locked design) `accounted_for = 0`. Indistinguishable from (1) to every current query by design (§10.4).
3. **Engaged, unsealed** — real activity (or, under the design, any deliberate engagement incl. app-open) pre-seal. What claim hunts.
4. **Sealed** — immutable history. What the carry-forward subtitle reads.
5. *(arriving with week mode)* — an opted-out weekday that does not count; interacts with streak. Unresolved; tracked in the week-mode doc + todo.

Per-consumer predicates (use these; do not re-derive):

| Consumer | Predicate | Hunts |
|---|---|---|
| Calendar `_classify` | row empty OR absent → `UNMARKED` | treats 1 + 2 alike |
| Claim walkback (as built) | `sealed_at IS NULL AND (rest_day=1 OR tasks_done != all-false)` | state 3 |
| Claim walkback (locked design) | `sealed_at IS NULL AND accounted_for = 1` | state 3, incl. engaged 0-task |
| Subtitle (locked design) | most recent `motd != ''`, sealed-or-not, ≤ today | 3 or 4 |

**Structural guard.** Document the taxonomy; do NOT add structure to model it gratuitously — no `is_empty` column, no `status` enum, no create-rows-on-open-as-a-feature. Those are the premature-structure move the architectural-preference rule refuses. The ONE structural addition that earns its place is `accounted_for`: a single write-once boolean that *retires* a fragile JSON string-match (clarity/robustness, not cleverness — the preferred side of the meta-rule) and is what splits state 2 from state 3 for an opened-but-idle day.

**Consequence (locked this session).** Under this design, app-open sets `accounted_for` on today's row, so opening the app on a date becomes a write (it creates/touches that row) — a deliberate change from §10.4's current "opening is not a write". Confirmed effect: an opened-then-idle day seals as a disappointment rather than vanishing as never-touched.

### 10.3 Cross-system
- Every `Backend.<x>_completed` signal has a corresponding `<x>_failed` and exactly one of the two fires per call. (Network errors → failure signal.)
- DevKit scenario URLs return 404 in production. (Verified manually at session 18.)
- The `four-tasks-api` worker bundle is identical between prod and dev; differentiation is purely in `env.IS_DEV` and the D1 binding.

---

## 11. Known fragility and tech debt

Captured here because the system map is also the place that should make these visible to whoever's reading.

- **Hardcoded timezone offset** in `State.gd` (`+ 8 * 3600` in `install_iso`/`today_iso`/`yesterday_iso`; `calendar_math.gd` does UTC-only and defers the offset to `State.gd`). v1.0 fix: use the OS offset `Time.get_time_zone_from_system().bias` — DST-correct for "now", no library. Full IANA-name capture is deferred to v1.x (needs a native plugin). See the timezone doc's V1.0 SCOPE. Verify the bias sign on a real Android device.
- **`DeadOverlay` is a placeholder `Panel`** on `day_cell.tscn`. Theme overlay wiring deferred to tile 4.14b.
- **Layout values duplicated as literals in `.tscn` files** with hand-maintained comment-block mappings to `Layout.gd` constants. Drift risk.
- **`tasks_done` parses to JSON but the field on a fresh `_new_day_row` is a native array; `rest_day` similarly arrives as bool from client and int (0/1) from server.** Code casts via `bool(...)` defensively. Source of past bugs in `_classify`.
- **Solo-user `_on_day_written` for partner days has no defined behaviour** — partner days only exist when partner exists. Currently no code path can produce a `partner` day write on the client side. Future endpoints (e.g. cross-user writes if partner reactions are reinstated) would need to revisit this.
- **`State.self_user` being whole-replaced by polls** can briefly stomp local optimistic updates from an in-flight `PUT /users`. Accepted for v1.0 — rare condition, next poll resettles.
- **Promo code tables exist with no read/write path**. `subscription_active`, `founders_flag`, etc. are schema-present but inert. Phase 5 work.
- **No client-side validation on `task_labels` length** (server caps at 49 chars per label, UX cap is 30). Onboarding will need to enforce the 30 client-side.
- **`onboarding.tscn` is a stub.** Every boot with no identity lands on a placeholder Label. Phase 3 work.
- **No retry/backoff in Backend.gd**. Single attempt per call; failure signals straight through. Tolerable while users live in Australia on home wifi; reconsider before broader testing.
- **DevKit menu shows ~15 stub scenarios** with `(stub)` suffix and no behaviour. Visual noise during testing; cosmetic.
- **Focus-in deferred poll suspected non-functional in editor** (session 19, observed). Print logs show only the 30s interval ticks, no `Poll: focus OUT` / `Poll: focus IN` lines. Most likely cause: Godot editor's run window doesn't reliably propagate `NOTIFICATION_APPLICATION_FOCUS_OUT/IN`. If `_is_focused` never flips, the focus-in handler is never re-armed and the 10s deferred poll never schedules. Diagnostic: alt-tab away from the run window and watch the editor output for the focus lines. If absent in editor, verify the behaviour on a real phone before treating as a real bug — focus events tend to work correctly on Android/iOS even when editor builds drop them.
- **Optimistic-write guarding in `State.gd` is inert under last-write-wins.** `_writes_in_flight`, `_pending_snapshot`, and the per-field merge in `_on_day_written` do real work only on the failure-rollback path. On the success path the three user-controlled fields are always kept local — safe for `tasks_done` (LWW), `motd` (single-writer, write-on-exit), and `rest_day` (refundable, not time-critical), each for a different reason. Safe to delete; kept only against a future endpoint that genuinely rejects user writes. The in-code comment currently explains only the rapid-tap (task) reason.
- **`panel_heading` subtitle uses a one-row-back fallback** (today's `motd`, else yesterday's). Intended rule is most-recent-non-empty-`motd` ≤ today (gap-day safe). Reconcile when tile 4.6 lands. See §7.7.
- **`State._dict_equal` is key-order-sensitive** — it compares via `JSON.stringify(a) == JSON.stringify(b)`, which preserves insertion order, so two dicts with identical content but different key order miscompare as unequal and fire a spurious `*_changed` re-render on every poll. Dormant in Phase 2 (flat cells redraw invisibly); becomes visible once cells carry sticker art + theme overlays. Fix before Phase 4: replace with an order-independent recursive deep-equal (arrays stay order-sensitive). Laptop + device test.
- **Claim seal predicate + payout are mid-redesign** (locked, not built): the `accounted_for` walkback replacing the `tasks_done` string-match, a 0-task disappointment tier, and the same-date partner multiplier. §5.3 documents both the built and design states; reconcile when the schema column + walkback land.

---

## 12. What this map does not describe

- Future phases (3, 4, 5, 5b) and their tiles. The map updates *as* those tiles land, not before.
- Pixel art, sticker assets, theme palettes — those live in `pickle-moon-assets` repo and the theme design notes.
- Promo code scope — schema is present, no behaviour exists yet.
- Morning sequence and claim UI — server endpoint exists, no client trigger wired.
- Partner reactions — explicitly killed for v1.0; doc moved to `deferred/`.
- Recovery flow UI — endpoint exists (`POST /resolve`), DevKit scenario stub clears identity, but no UI screen prompts for the six values yet.

---

## 13. Change log for this document

| Date | Session | Change |
|---|---|---|
| 2026-05-27 | 19 (mobile) | Initial draft. Snapshot at end of session 18 (Phase 2 closed, Phase 3 pending). |
| 2026-05-27 | 19 (mobile) | Added §10.4 (day row existence invariant + walkback gap), flagged focus-in poll suspected-non-functional in §11. |
| 2026-05-29 | 21 | Capture propagation: §5 router-convention note; §5.3 + §10.4 annotated with the locked `accounted_for` walkback / 0-task tier / partner-multiplier redesign; new §10.5 day-state taxonomy; §7.7 panel_heading built-vs-design; §11 added inert-guard, panel_heading, `_dict_equal`, and claim-redesign debt, and corrected the timezone-offset entry to `State.gd` / OS-offset. |
