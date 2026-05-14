# Write Rules — Design Notes

**Status:** LOCKED for v1.0 (session 4). This document specifies the
defensive write rules for tile 1.3 — the field-level write permissions,
state preconditions, validation logic, migration semantics, and rejection
codes for every write endpoint in the Four Tasks API.

**Implementation timing:** tile 1.3 implements these rules. The morning-
payout claim endpoint depends on the schema additions captured here.

**Supersedes:** session 3's closing note that bundled tile 1.3 design
with write-endpoint implementation. This doc is the design half; the
endpoints land in a separate implementation tile against this spec.

**References:**
- `four_tasks_pair_key_design_notes.md` — pair-key identity model,
  migration semantics, partner-app-mediated recovery
- `four_tasks_partner_reactions_design_notes.md` — field-level write
  permissions concept, immutability gate, post-seal-only rules
- `four_tasks_architectural_preference.md` — clarity-over-cleverness
  rule that governs the design choices below
- `four_tasks_timezone_and_sealing_design_notes.md` — sealing is
  lazy-on-open in the claim endpoint, NOT via nightly cron. Tile 1.4
  was reframed in session 6.
- `four_tasks_theme_design_notes.md` — `active_leader` is the
  migration-triggering identity field; the rest of theme state lives
  in JSON columns that do NOT migrate.

---

## The three write endpoints

Tile 1.3 covers three write endpoints. Each has its own rule table.

| Endpoint                                               | Purpose                          |
|--------------------------------------------------------|----------------------------------|
| `POST /pair/:key/bug_report`                           | File a bug report                |
| `PUT  /pair/:key/users/:target/days/:date[?caller=]`   | Update a day row                 |
| `PUT  /pair/:key/users/:target`                        | Update a user row (may migrate)  |

All three follow the same response envelope (locked session 2):
- Success: `{ "ok": true,  "data": <payload> }`
- Failure: `{ "ok": false, "error": "<code>", "message": "<human>" }`

HTTP status codes used: 200, 400, 404, 409, 500. No others.

---

## Endpoint 1 — POST /pair/:key/bug_report

**URL:** `POST /pair/:key/bug_report`

The pair-key in the URL identifies the reporting pair. Bug reports are
paired-only (onboarding-stage users can't reach the bug button in the
UI). The `pair_key` column in the `bug_reports` table remains nullable
as a zero-cost hedge for future "stuck-on-loader bug report" flows;
v1.0 never writes a NULL there.

**Body shape:**
```json
{
  "message": "string, required, non-empty, max 2000 chars",
  "app_version": "string, optional, no format check"
}
```

**Server fills:** `id` (autoincrement), `submitted_at` (server time), `pair_key` (from URL).

### Rejections

| Condition                                          | Status | Error code      |
|----------------------------------------------------|--------|-----------------|
| Pair key in URL not 16 hex                         | 400    | `bad_pair_key`  |
| Body not valid JSON                                | 400    | `bad_input`     |
| `message` missing / not string / empty             | 400    | `bad_input`     |
| `message` > 2000 chars                             | 400    | `bad_input`     |
| `app_version` present but not string               | 400    | `bad_input`     |

### Design decisions

**No pair-existence check.** The server does not SELECT from `pairs`
before inserting. Reasons:

1. The 16-hex regex already gates obvious garbage.
2. Bug reports are a write-only log, philosophically detached from
   referential integrity (the schema comment is explicit:
   "historical accuracy > referential integrity").
3. Adding an existence check would create a *second pair-key oracle*
   beyond GET /pair/:key — a malicious actor could enumerate valid
   pair-keys by spamming bug-report requests and watching for 200
   vs 404 responses. The defensive benefit (junk in the table) is
   small; the security cost is real.

**Spam defence belongs at the rate-limit layer**, not at per-write
existence checks. See "Deferred / parked items" below.

### Response

On success: `{ "ok": true, "data": { "id": <new_id> } }`. Just the new
row ID; the client doesn't need the message echoed back.

---

## Endpoint 2 — PUT /pair/:key/users/:target/days/:date

**URL (self-write):** `PUT /pair/:key/users/:target/days/:date`
**URL (cross-write):** `PUT /pair/:key/users/:target/days/:date?caller=:caller_name`

Two distinct write patterns hit this endpoint:

1. **Owner writes** to their own day row: ticking tasks, MOTD, rest day.
   `?caller` absent; caller is implicitly the target.
2. **Partner writes** to the *other* user's day row: dropping reactions
   on a sealed day. `?caller` present; caller != target.

Body is **partial** — client sends only fields it's changing. A single
task toggle sends `{ "tasks_done": [true, false, false, false] }`.

**Day rows auto-create on first write.** If no row exists for the given
(pair, user, date), the server inserts a fresh row with schema defaults
and then applies the partial update.

### Column rule table

**Owner-writable** (caller == target):

| Column        | State precondition  | Validation                                                  |
|---------------|---------------------|-------------------------------------------------------------|
| `tasks_done`  | Unsealed only       | JSON array of exactly 4 booleans                            |
| `diary`       | Unsealed only       | Non-empty string, max 200 chars                             |
| `rest_day`    | Unsealed only       | Integer 0 or 1                                              |
| `local_date`  | Always              | YYYY-MM-DD string matching :date param                      |

Note: `local_date` is sent with writes alongside other fields to support
the timezone localisation model (see timezone doc). Server validates
±4hr plausibility against current UTC. Used by the seal-on-open logic.

**Partner-writable** (caller != target, `?caller=` present):

| Column            | State precondition                            | Validation                       |
|-------------------|-----------------------------------------------|----------------------------------|
| `motd_reaction`   | Day is sealed AND column currently NULL       | Non-empty string, max 50 chars   |
| `tray_reaction`   | Day is sealed AND column currently NULL       | Non-empty string, max 50 chars   |

When `motd_reaction` is written, server also fills `motd_reaction_at`
with `Date.now()`. Same for `tray_reaction` / `tray_reaction_at`.

**Server-only columns** (rejected with 400 if present in body):

| Column              | Filled by                                         |
|---------------------|---------------------------------------------------|
| `stamp`             | Morning payout (server-side, derived from tasks_done tier + message pool) |
| `sealed_at`         | Lazy seal-on-open during claim endpoint transaction (per timezone doc) |
| `day_theme_state`   | Written at seal time — JSON snapshot of active theme slots |
| `motd_reaction_at`  | Server, when partner writes `motd_reaction`       |
| `tray_reaction_at`  | Server, when partner writes `tray_reaction`       |

### Note on `diary` content

`diary` stores the MOTD (Message Of The Day) — a space-delimited string
in the shape `"${adverb} ${verb} ${noun} ${emoji}"` built by the client
from four wordlists. The wordlists live client-side (mirror of the web
build's `MSG_ADVERBS` / `MSG_VERBS` / `MSG_NOUNS` constants), with the
emoji slot drawn from the user's active sticker pool per morning sequence
doc Q1. The picker follows the same preview-on-press / commit-on-exit
pattern as the v2 identity picker (NOT the theme slot picker, which is
instant-commit).

The server does not validate `diary` content against wordlists. The
defence-in-depth argument that justified server-side validation of
`/resolve` input (no `|`, no `__`) doesn't apply here — duplicating
~300 words across client and server creates two sources of truth that
will drift. Trust-the-client + length cap is the right tier of
robustness for this column.

Future tightening (server-side enum validation when the slot-machine
wordlists stabilise) is a backward-compatible change. Not v1.0 scope.

### Note on `stamp`

Stamps are NOT user-written. They land server-side during the morning
payout, with a tier determined by `tasks_done` count (1-4 completed
tasks → different colour + different message pool). Users cannot
un-stamp a day, cannot choose their stamp, cannot pre-set it. The
column is server-only on PUT day.

### Rejection table — preconditions

| Condition                                                | Status | Error code              |
|----------------------------------------------------------|--------|-------------------------|
| Pair key in URL not 16 hex                               | 400    | `bad_pair_key`          |
| `:target` empty / contains `\|` or `__`                  | 400    | `bad_input`             |
| `:date` not `YYYY-MM-DD` shape                           | 400    | `bad_input`             |
| `?caller` present but empty / contains `\|` or `__`      | 400    | `bad_input`             |
| `?caller` present and equals `:target`                   | 400    | `bad_input`             |
| Body not valid JSON                                      | 400    | `bad_input`             |
| Body contains a server-only column                       | 400    | `bad_input`             |
| Pair row doesn't exist                                   | 404    | `not_found`             |
| Target user doesn't exist in pair                        | 404    | `not_found`             |
| `?caller` present and caller doesn't exist in pair       | 404    | `not_found`             |
| Owner trying to write a partner-only column              | 404    | `not_found`             |
| Partner trying to write an owner-only column             | 404    | `not_found`             |

The 404 vs 409 split: **404** = "this combination of caller + column
doesn't exist as a writable thing" (permission). **409** = "this is
normally writable but the current state forbids it" (timing).

### Rejection table — state conflicts (409)

| Condition                                                | Status | Error code              |
|----------------------------------------------------------|--------|-------------------------|
| Owner writing `tasks_done`/`diary`/`rest_day` on sealed day | 409 | `day_sealed`            |
| Partner writing reaction on unsealed day                 | 409    | `day_not_sealed`        |
| Partner writing reaction on column already non-NULL      | 409    | `reaction_already_set`  |

Permission-denied always folds into 404 (auth convention locked
session 2). State-denied surfaces as 409 so the client can render the
right user-facing message ("this day is locked" vs "reaction already
placed").

### Response

On success: `{ "ok": true, "data": { "day": <updated_day_row> } }`.

---

## Endpoint 3 — PUT /pair/:key/users/:target

**URL:** `PUT /pair/:key/users/:target`

PUT user is always a self-write. No `?caller=` — there's nothing in the
`users` table that a partner can legitimately change about the other
user. (Partner reactions write to the `days` table, not `users`.)

Body is partial. Send only fields being changed.

### Column rule table

**Owner-writable (no migration):**

| Column                       | Validation                                                          |
|------------------------------|---------------------------------------------------------------------|
| `task_labels`                | JSON array of exactly 4 strings, each non-empty, each max 49 chars  |
| `reaction_confirm_dismissed` | Integer 0 or 1                                                      |
| `tutorial_progress`          | JSON object, MERGED into stored value (not replaced), overall max 4000 chars |
| `timezone`                   | IANA timezone string (e.g. 'Australia/Perth')                       |
| `active_theme`               | JSON map of slot → sticker ID for non-leader theme slots            |
| `active_stickers`            | JSON array of sticker IDs (pool members)                            |

**Owner-writable (triggers pair-key migration):**

| Column           | Validation                                                  |
|------------------|-------------------------------------------------------------|
| `username`       | Non-empty string, max 40 chars, no `\|`, no `__`            |
| `active_leader`  | Sticker ID string, max 50 chars, no `\|`, no `__`           |

`active_leader` (renamed from prototype-era `emoji` / `icon`) is the
chosen visual identity sticker. It participates in the pair-key
identity hash — changing it triggers migration. All other theme slots
(palette, background, cell treatments, effects, audio) live in the
`active_theme` JSON map and are non-identity writes.

The `|` and `__` restrictions on `username` and `active_leader` mirror
the canonical-form rules at `/resolve`. They cost nothing and protect
the pair-key derivation from ambiguous inputs.

**Server-only** (rejected with 400 if present in body):

| Column                       | Filled by                                                  |
|------------------------------|------------------------------------------------------------|
| `name`                       | Set at creation only — immutable per identity model        |
| `coins`                      | Morning payout + reroll endpoint + future subscription     |
| `streak`                     | Computed server-side from days table at seal time          |
| `coin_name`                  | Server-derived from username via generator (post-v1.0)     |
| `coin_name_reroll_count`     | Changed via separate reroll endpoint (post-v1.0)           |
| `morning_payout_due_for`     | Set during seal-on-open; cleared by morning-payout claim   |

Name immutability is **server-enforced**, not just an onboarding
convention. It is the foundation of partner-side migration recovery
working: if name could change, recovery would be impossible because
the partner wouldn't know what name to type. PUT user rejects `name`
in body with 400.

### `tutorial_progress` merge semantics

Unlike all other columns, `tutorial_progress` is **merged** into the
stored value rather than replaced. Client sends only the keys it wants
to add or update:

```json
{ "tutorial_progress": { "coins_intro": 1715472000 } }
```

Server reads the stored JSON, merges in the submitted keys, writes back.
Keys not present in the submitted object are left untouched.

Reasons:
- The state is accretive — features get marked "shown" over time.
- A replace would clobber updates made between the client's last poll
  and its current write (small race window, but real).
- Merge matches the natural usage: "I just saw this feature reveal,
  mark it as shown."

### Pair-key migration

When `username` or `active_leader` changes, the pair-key changes. The
migration is the dense part of this endpoint.

#### Trigger detection

After validation, server reads current user state, compares submitted
`username` / `active_leader` against stored values. If either differs,
migration is needed. If neither differs (or neither was in the body),
no migration.

#### Atomic transaction shape

Inside a single D1 transaction:

1. Re-derive new pair-key from post-write six-tuple
   `(name_A, username_A, active_leader_A, name_B, username_B,
   active_leader_B)` sorted by name lexicographically. Uses the same
   `derivePairKey()` function `/resolve` uses.
2. INSERT into `pairs` (new_key, copied `created_at` and `tutorial_done`
   from old row).
3. INSERT into `users` for both users under new_key (target user with
   new username/active_leader values; partner copied unchanged).
4. INSERT into `days` — copy every existing days row from old_key to
   new_key for both users.
5. DELETE from `days` WHERE pair_key = old_key.
6. DELETE from `users` WHERE pair_key = old_key.
7. DELETE from `pairs` WHERE key = old_key.
8. Commit.

If any step fails, the transaction rolls back. State is either fully
old or fully new — never partial.

We use insert-new-then-delete-old rather than `UPDATE pair_key = ?` in
place because D1 (SQLite) doesn't handle primary-key column updates
cleanly. The copy-then-delete pattern is the natural fit for D1's
transaction model.

Migrations are **re-entrant by design** (locked v2 design notes): server
always re-derives the key from current authoritative state, never trusts
cached client keys. Two simultaneous migrations converge to the same
correct final state.

#### Hash collision (409)

The pair-key is SHA-256 truncated to 16 hex chars = 64 bits of entropy.
Birthday-paradox math: ~5 billion pairs before collisions are likely.
Not a practical concern at any realistic user count, but the server
must handle it correctly when it does happen.

Detection: the INSERT into `pairs` with the new key will fail with a
primary-key violation if the new key already exists. The transaction
rolls back automatically.

Response: 409 `pair_key_collision`. Client surfaces this as a routine
"that username is taken, sorry try another" — the same UX as a
username-collision in onboarding. The user has no need to know about
the underlying hash-collision event.

No pre-emptive collision check is performed. Letting the INSERT fail
naturally saves a query and is information-theoretically cleaner
(though the leak from a pre-check would be near-zero — the user would
need to know the partner's full six-tuple to construct a collision,
at which point they already have recovery credentials for that pair).

#### Response shape

Response always includes a `migrated` boolean flag, regardless of which
path was taken.

**Case A — no migration:**

```json
{
  "ok": true,
  "data": {
    "migrated": false,
    "pair": { ... full pair payload, same shape as GET /pair/:key ... }
  }
}
```

**Case B — migration happened:**

```json
{
  "ok": true,
  "data": {
    "migrated": true,
    "new_pair_key": "def456...",
    "pair": { ... full pair payload under new key ... }
  }
}
```

Same shape either way (`migrated` + `pair`), with `new_pair_key` present
only in the migration case. Client always reads `data.migrated` and
branches on it — no special case for "did the response include a new
key?"

The full pair payload is returned in both cases so the client has the
post-write state in one round-trip. Returning just the new key would
force a follow-up GET /pair/:new_key, doubling the migration cost.

### Partner experience during migration

The migration is **silent and automatic** from the partner's perspective:

1. Partner's next poll of GET /pair/:old_key returns 404.
2. Partner's client falls into the recovery path: re-derives the key
   from the six values it already holds (the five visible values plus
   the partner's own name).
3. Re-derived key is the new one. Client updates its stored identity,
   re-polls, gets the updated state including new username/active_leader.
4. UI re-themes seamlessly because the new active_leader is in the response.

No popup. No error. The partner experiences this as: brief poll fail
→ automatic recovery → updated partner panel with new name/identity.

This is the same code path that handles any 404 on a stored pair-key,
not a migration-specific recovery flow. The robustness was already
required for v2 identity recovery; migration just happens to also
exercise it.

### Client-side defensive write requirements

The server-side rules above guarantee state integrity under any
sequence of failures. But the client must hold up its half of the
contract to avoid lockout scenarios. These requirements apply to
tile 1.7 (Godot identity layer) and tiles 4.14 / 4.15 (active_leader
and username edit flows):

**Requirement 1 — persist pending change before sending request.**

Before sending a migration-triggering PUT request, the client writes
the new username/active_leader to a "pending" slot in
`user://identity.cfg`. This slot persists across crashes, app kills,
OS terminations, and device reboots.

**Requirement 2 — recover from pending state on 404.**

If a subsequent GET /pair/:key returns 404 and a pending change is
in the slot, the client uses the pending values (not the canonical
old values) when deriving the recovery key. This handles the case
where the migration request was processed server-side but the
response never reached the client.

**Requirement 3 — clear pending slot on confirmed success.**

When the migration response confirms `migrated: true` and returns
the new pair-key, the client commits the pending values to canonical
state, updates the stored pair-key, and clears the pending slot.

**Requirement 4 — concurrent migrations are handled by retry.**

If two clients in the same pair migrate simultaneously, one will hit
404 because the other's migration committed first. The 404 recovery
path discovers the new state via /resolve using pending values,
then retries the original PUT against the new key. Re-entrant by
design.

### Rejection table — preconditions

| Condition                                                | Status | Error code              |
|----------------------------------------------------------|--------|-------------------------|
| Pair key in URL not 16 hex                               | 400    | `bad_pair_key`          |
| `:target` empty / contains `\|` or `__`                  | 400    | `bad_input`             |
| Body not valid JSON                                      | 400    | `bad_input`             |
| Body contains a server-only column                       | 400    | `bad_input`             |
| Pair row doesn't exist                                   | 404    | `not_found`             |
| Target user doesn't exist in pair                        | 404    | `not_found`             |
| Migration would collide with existing pair               | 409    | `pair_key_collision`    |

---

## Schema deltas captured

Three migrations land alongside tile 1.3 implementation. Commutative —
apply in any order.

**migration_003_morning_payout.sql:**
```sql
ALTER TABLE users ADD COLUMN morning_payout_due_for TEXT;
```

NULL = no payout pending. `"YYYY-MM-DD"` = payout pending for that
date, set during seal-on-open in the claim endpoint, cleared at the
end of that same transaction.

**migration_004_user_timezone.sql** (from session 6 part 1 timezone doc):
```sql
ALTER TABLE users ADD COLUMN timezone TEXT NOT NULL DEFAULT 'UTC';
```

User's IANA timezone string. Mutable, not in pair-key hash. Used by
the seal-on-open logic to determine the user's local "yesterday."

**migration_005_theme_state.sql** (from session 7 theme doc rewrite):
```sql
ALTER TABLE users ADD COLUMN active_leader TEXT;
ALTER TABLE users ADD COLUMN active_theme TEXT;       -- JSON map
ALTER TABLE users ADD COLUMN active_stickers TEXT;    -- JSON array
ALTER TABLE days ADD COLUMN day_theme_state TEXT;     -- JSON map
```

Per-user theme state + per-day theme snapshot. `active_leader`
participates in pair-key hash. The rest are JSON for flexibility —
adding new theme slots later does not require schema migrations.

If the prototype-era `users.icon` column still exists, drop it — its
role is now split across `active_leader` and the `active_theme` JSON's
`palette` key.

---

## Deferred / parked items

The following surfaced during this design conversation and are
captured here so they don't get lost:

### Rate limiting (ACTIVE tier)

Bug-report and migration-triggering writes both surfaced rate-limit
concerns. The right layer for spam / abuse defence is Cloudflare's
rate-limiting (Workers Rate Limiting API or per-IP at the platform
edge), not per-endpoint existence checks or per-write counters.

Generous limits: real users hitting catastrophic bugs may legitimately
file three bug reports in five minutes; real users adjusting username
may try a few combinations before settling. Limits should protect
against floods, not annoy real usage.

Belongs in its own design conversation after a few more endpoints
exist to protect collectively. Not blocking tile 1.3.

### Migration UX popup (DEFERRED to tile 4.15)

Nice-to-have client-side UX layer: when a username or active_leader
change fires, show a small dismissible popup —
"username changes may take a minute to take effect" with a "don't
show again" checkbox. Mirrors the partner-reaction confirmation
popup pattern. Sits on top of the defensive write requirements as
a courtesy to the user, not as a correctness layer.

### Morning-payout claim endpoint (specced in morning sequence doc)

Approximate shape — full spec in morning_sequence design notes:

- `POST /pair/:key/users/:target/claim_morning`
- Server-side atomic transaction:
  - Performs seal-on-open if user's local_date has advanced
  - Reads pending date from `users.morning_payout_due_for`
  - Computes stamp tier from `days.tasks_done` count for that date
  - Picks a random message from the tier-appropriate pool
  - Writes `days.stamp` for that date
  - Captures `days.day_theme_state` JSON snapshot at seal time
  - Pays out coins (tasks_done count → coin amount)
  - Updates `users.streak`
  - Clears `users.morning_payout_due_for`
  - Returns the new state for client animation playback
- Empty request body (server already knows what's pending)

Belongs in tile 4.6 design pass alongside morning sequence
implementation.

### Stamp / reaction / active_leader enum validation (DEFERRED v1.x+)

Server-side validation of these against hard-coded enums is a
backward-compatible tightening. Adding it later doesn't require
schema changes or client changes (well-behaved clients already send
valid values). Holding off until the catalogues stabilise.

---

## Rejection code reference

Comprehensive list of error codes used across all three endpoints
in this doc, for client implementers:

| Code                     | Status | Meaning                                                          |
|--------------------------|--------|------------------------------------------------------------------|
| `bad_pair_key`           | 400    | Pair key in URL not 16 hex chars                                 |
| `bad_input`              | 400    | Body malformed, missing/wrong-shape field, server-only column attempted, `?caller` redundant or invalid |
| `not_found`              | 404    | Pair, user, or writable target doesn't exist (also: permission denied) |
| `day_sealed`             | 409    | Owner trying to write live-only column on sealed day             |
| `day_not_sealed`         | 409    | Partner trying to react on unsealed day                          |
| `reaction_already_set`   | 409    | Partner trying to overwrite an existing reaction                 |
| `pair_key_collision`     | 409    | Migration would produce an existing key (rare; UX as "username taken") |
| `server_error`           | 500    | Server-side bug (parse failure, etc.) — see /pair/:key precedent |

---

## Implementation notes for tile 1.3

When the implementation tile lands (separate session from this design):

1. Extract validation helpers shared with `/resolve` (the `|` / `__` /
   non-empty checks). They already exist as `validateUser()` in
   `index.ts`; expand or factor as needed.
2. Re-use `derivePairKey()` for the migration step. Already extracted.
   Update to consume `active_leader` instead of legacy `emoji` column.
3. Re-use `getPair()` for response payloads (both endpoints return full
   pair state on success, same shape as GET /pair/:key).
4. The migration transaction is the substantive new code. Use D1's
   `batch()` for the multi-statement transaction; D1 wraps batched
   statements in an implicit transaction.
5. Server-only column rejection: define a constant set per endpoint
   listing writable columns, reject body keys outside the set with 400.
   Defence-in-depth pattern — even if new columns are added later
   without updating the set, the default is reject.
6. The claim endpoint (POST claim_morning) lands in this tile bundle
   alongside seal-on-open logic (timezone doc) — they share the same
   transaction surface.
7. Curl-verify every rejection path the way tile 1.2's reads were
   verified — each error code on each endpoint gets a positive test.

---

## Closing note

This document is the design half of tile 1.3. The implementation tile
remains `[ ]` in the todo and will reference this doc when written.
Splitting the design from the implementation follows the architectural
preference doc's pattern: capture the design when it's fresh, implement
against a fixed spec.
