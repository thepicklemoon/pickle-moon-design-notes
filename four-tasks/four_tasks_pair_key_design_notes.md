# Pair-Key Identity — Design Notes

**Status:** LOCKED (session 11 close — un-pair UX detail pass
complete). Canonical identity model for Four Tasks v1.0+.

**Schema status (session 12):** the schema described throughout this
doc SHIPPED in the v1.0 wholesale rewrite — `user_id` PK, `pair_id`
(nullable FK), `pair_key` (UNIQUE), `pending_unpair_notice`, and the
rest are all live in `server/schema.sql`. References to "migration_005"
below are historical (that was the working name for this delta before
session 12 collapsed all migrations into a single wholesale schema);
read them as "the v1.0 schema," not as a pending migration. Server
endpoints (`/resolve`, `join_by_values`, `unpair`, rotation on
`PUT /users/:user_id`) shipped in tile 1.3. See `four_tasks_system_map.md`
§3–§5 for the live operational view.

Implementation landed in tile 1.3 (server-side write rules + rotation),
with tile 4.14b (theme manager / picker UX) and tile 4.21 (un-pair
flow — full UX spec) still ahead.

This doc supersedes the v1 two-name model. The v1 doc
(`four_tasks_pair_key_identity_design_notes.md`) is referenced for
historical context only and is no longer authoritative.

**Session 11 changes (what's new vs session 10 fourth-pass close):**

- **Un-pair UX detail pass complete.** Architecture from session 10
  retained unchanged; the UX detail was filled in. Section 17
  rewritten to capture: long-press-on-partner-header entry point,
  minimal `Un-pair?` confirmation popup (no body copy), inline
  dismissable banner on partner side replacing the popup
  notification, change-username follow-up popup on initiator side,
  no cooldown, identical-to-fresh-solo post-un-pair empty state.
- **`pending_unpair_notice TEXT NULL` column on users.** New schema
  delta in migration_005 (alongside session-10 changes). Stores
  the ex-partner's last-known username; populated by the un-pair
  transaction on both rows; cleared by a dismiss endpoint when
  the banner is tapped.
- **Terminology in UI: "un-pair".** Backend / internal naming
  remains "leave pair" / "pair break" — same operation. User-facing
  copy never says "leave" or "break."
- **Tile 4.21 scope expanded.** Estimate raised from 2-3h to 3-4h
  to absorb the change-username follow-up popup, the banner
  mechanism, and the dismiss endpoint.

**Session 10 changes (what's new vs the prior LOCKED state):**

- **user_id locked as the operational identifier for a person.**
  Every user gets a stable server-generated UUID at the moment their
  `users` row is created (end of onboarding for fresh users, end of
  recovery for returning ones). user_id never changes for the life
  of the user. Personal data (day rows, tasks, coins, lifetime
  stats, theme state, tutorial progress, subscription state) hangs
  off user_id, NOT off pair_id. This is the single biggest change
  in this pass — it decouples personal data from pair lifecycle and
  lays the architectural groundwork for leaving / re-pairing /
  future group expansion.
- **Three-identifier model made explicit.** user_id (who you are,
  forever), pair_id (which relationship you're currently in),
  pair_key (the hash that lets a partner find you again from
  scratch). Three concepts, three jobs, clean separation. See
  Section 6.
- **Option B for pair structure retained.** Stable pair_id +
  mutable pair_key lookup. Rotation is O(1) on the pairs table.
  Day rows are now user_id-owned so they don't enter the picture
  at all during rotation.
- **Casefold removed from name normalisation.** Names are
  case-sensitive in the canonical string. The previous version
  casefolded names with a recovery-convenience justification that
  didn't survive scrutiny — partner-app-mediated recovery means
  the partner reads the *current* spelling off their own screen
  anyway, so user memory of exact case isn't load-bearing. Whitespace
  trim + collapse and NFC Unicode normalisation are retained for
  both name and username; casefold applies to neither.
- **pair_key reframed as recovery-only credential.** It is NOT a
  partition key, NOT an operational identifier, NOT used in URLs
  for ongoing work. It exists solely so that a user who has lost
  local state can recover via the six-value partner-mediated flow.
  Once recovery completes, the client gets back user_id and pair_id
  and uses those for everything else.
- **Pair lifecycle: symmetric-break model locked (fourth pass).**
  Either user initiating "leave pair" puts BOTH users into solo
  state. No asymmetric leaving. Personal data preserved on both
  sides (it's user_id-owned). New tile 4.21 sketched for the
  leave-pair flow. Solo recovery handled via OS-level
  identity.cfg backup, NOT pair_key — pair_key is now framed as
  a bonus recovery layer specific to paired users. Group / triple
  expansion remains NOT designed and explicitly not in v1.0
  scope. See Section 17.
- **Terminology cleanup retained.** "Migration" reserved for schema
  migrations only. The pair-key change event is **pair-key rotation**.
- **Concurrency analysis updated.** Day-tick writes go straight to
  user_id and never touch pair_key. Rotation only touches pair_id +
  the rotating user's row. Race conditions become even simpler
  because day writes and rotation writes don't share rows at all.
- **Threat model retained.** Three scenarios (shoulder-surfing,
  brute-force enumeration, accidental wrong-attach) get explicit
  treatment. Privacy policy alignment flagged.
- **Hash collision and wrong-attach kept separated.** Different
  failure modes, different responses.

Terminology note (session 7, retained): the third identity value
was previously called `emoji` or `icon`. It is now consistently
called `active_leader` to match the theme system's slot catalogue
language. The semantics are identical.

---

## Table of contents

1. What changed from v1
2. Why the change
3. The six values
4. The recovery argument
5. **Data model — Option B + user_id locked**
6. The three-identifier model
7. Pair-key rotation semantics
8. Concurrency analysis
9. Threat model
10. Picker UX (impacts tile 4.14b)
11. Client-side defensive write requirements
12. Pair-key format and canonical string spec
13. Hash collision and wrong-attach
14. Post-pair username revert
15. What this changes about the schema
16. Outstanding questions (not blocking)
17. **Pair lifecycle (symmetric break locked; groups deferred)**
18. Bug reports — out-of-band lifecycle
19. Cross-references

---

## 1. What changed from v1

**v1 (locked in session 13):** pair-key = `{min(name,partner)}__{max(name,partner)}`.
Two values per user. Recovery = remember both names.

**v2 (this doc):** pair-key = hash of normalised six-tuple
`(name_A, username_A, active_leader_A, name_B, username_B, active_leader_B)`,
where A/B are sorted by `name` lexicographically.

The shift: identity is now anchored in **three values per user**, not
one, and recovery is **partner-app-mediated** — five of six values are
visible on the partner's screen at all times.

The session 10 pass adds two further shifts on top of that:
- Personal data attaches to a stable per-user `user_id`, not to the
  pair. This means pair changes don't move user data around.
- The `pair_key` is reframed as a recovery-only credential rather
  than an operational identifier.

## 2. Why the change

The v1 model was driven by collision avoidance — "two real Morgans +
two real Ayshis won't collide because the *pair* is the unit." Correct,
but it collapsed the wrong concern. The actual concern was **recovery**
in the case of local data loss (which has happened to mum and to Ayshi
on the web build).

v1 recovery requires the user to remember what they typed at
onboarding. That's fragile across long timelines: mum recovering 8
months later may not remember whether she typed `Sandra` or `sandra`
or `mum`.

v2 recovery doesn't depend on memory of input. The credential **is**
the current visible state on the partner's app. As long as one
partner has working data, the other can always rebuild from it.

The session 10 user_id addition extends this further: a user's own
data belongs to them, not to the relationship. The relationship can
change (rotate, dissolve, re-pair) without disturbing the personal
data underneath. This isn't a recovery concern — it's a
flexibility-of-life concern. Real relationships change. The data
model should accommodate that.

## 3. The six values

| Field           | Mutability | Rotation trigger? | Visible to partner? |
|-----------------|------------|-------------------|---------------------|
| name            | immutable  | n/a               | yes (label on panel)|
| username        | mutable    | yes               | yes (display name)  |
| active_leader   | mutable    | yes               | yes (theme avatar)  |

Same row, both users. So six values total per pair.

**Why name is immutable:** name change in v1 was the gnarliest case,
deferred to v1.x. Locking name as immutable in v2 keeps that
complexity off the table permanently. Onboarding copy enforces the
choice with a confirmation step before locking.

Username covers all the "I want to change how I show up" cases that
name would have covered. Username is freely mutable. No regret risk.

**Why username and active_leader are mutable:** they're meant to be
expressive. Username is what you display; active_leader drives the
chosen visual identity sticker. Locking them would kill the
personalisation features. They mutate, and the pair-key mutates with
them.

**Note on `active_leader`:** this is the sticker ID string for the
user's chosen visual identity sticker (e.g. `"frog_01"`). It is ONE
of many theme slots, but it is the only theme slot that participates
in the pair-key identity hash. All other theme slots (palette source,
background, cell treatments, effects, audio, etc.) live in the
`users.active_theme` JSON column and are non-identity writes — they
freely change without rotation. See the theme doc for the full slot
catalogue.

**Important under user_id:** the six values together form the
pair_key for recovery purposes. They are NOT the user's identity in
the database — that's `user_id`, generated server-side, never visible
to users. The six values are *display state* plus *recovery
credential*. They identify a person within a relationship; user_id
identifies a person across all their relationships.

## 4. The recovery argument

Failure scenarios for v2 recovery:

1. **User loses data; partner still has working app.** ✅ Partner's
   screen shows current username and active_leader for both users.
   User reads off five values, remembers own name (sixth), types all
   six. Server hashes, finds pair, hands back user_id + pair_id +
   full state.

2. **Both users lose data simultaneously.** ❌ Same failure case as
   v1. No mitigation. This is the "freak accident" tail. Note: in
   practice users may have backed up identity.cfg to iCloud Keychain
   or Google Drive — those backups would restore user_id
   transparently and avoid this case. We don't *require* backup but
   we don't prevent it.

3. **User loses data; partner has rotated since user last saw their
   values.** ⚠️ Partner-app source-of-truth still works, but the
   recovering user has to *contact* the partner to learn the current
   values rather than typing them from memory. The partner reads
   their current displayed values aloud (or via text); recovering
   user types those plus their own three. Server finds pair, recovery
   works.

4. **Pair-key hash collision (two real pairs hash to the same key).**
   ✅ Onboarding second pair gets prompted to vary their username
   (the easiest field to change without losing identity). See
   Section 13.

5. **Wrong-attach (user attaches to wrong existing pair).** See
   Section 13 — different problem, different response.

6. **User maliciously impersonates another pair.** See Section 9.
   Effectively impossible by guessing; possible only with shoulder-
   surf access to the target's screen.

Trade-off: v2 is more robust for the common case (local data loss)
and no worse for the edge cases. Net win.

---

## 5. Data model — Option B + user_id locked

This section locks the data-model decisions. Two related but
independent locks:

- **Option B (locked session 10 second pass):** stable `pair_id`
  with mutable `pair_key` as a lookup, replacing the prior implicit
  "pair_key as partition key" model.
- **user_id (locked session 10 third pass):** stable per-user UUID,
  with personal data hanging off user_id rather than pair_id.

Together they form the three-identifier architecture: user_id (the
person, forever), pair_id (the current relationship), pair_key (the
recovery hash).

### Option B recap

The pair-key hash is NOT the partition key. The `pairs` table has
two columns of interest:
- `pair_id TEXT PRIMARY KEY` — stable per-pair UUID, generated at
  pair creation, never changes.
- `pair_key TEXT UNIQUE NOT NULL` — current canonical hash, updates
  on every rotation, indexed for lookup performance.

Child references use `pair_id`, never `pair_key`. Rotation is a
single-row UPDATE on `pairs.pair_key` plus a UPDATE on the rotating
user's row. No child rows are touched. Cost is O(1).

(Full rationale for Option B is preserved in the session-10
second-pass notes within this doc's git history. The decision is
behind us. This section captures the lock.)

### user_id

Every user has a stable identifier:
- `user_id TEXT PRIMARY KEY` on the `users` table — server-generated
  UUID v4, 36 chars, assigned at the moment of `users` row creation
  (which is at the end of onboarding, NOT at first app open —
  abandoned onboarding flows don't create orphan user_ids).
- Never changes for the life of the user.
- Returned to the client in the response to onboarding completion
  or recovery completion.
- Stored in `user://identity.cfg` and used as the operational
  identifier for all per-user data access.

Personal data attaches to user_id:
- `days.user_id` — every day row belongs to a specific user.
- The user's identity values (name, username, active_leader), coin
  balance, theme state, tutorial progress, subscription state, etc.
  are all on the `users` row keyed by user_id.

The `users` row also has a `pair_id` column indicating which pair
(if any) the user is currently in. `users.pair_id` is **nullable**
— a user without a pair has `NULL` here.

### Why this matters

Three things become possible (or trivially possible) that weren't
before:

1. **A user can leave a pair.** Set `users.pair_id = NULL`. The
   user's day rows, coin balance, theme state, etc. are untouched.
   The pair's `pairs` row continues to exist with one or zero
   active users (handling of orphan pairs is a pair-lifecycle
   question — Section 17).

2. **A user can re-pair.** Set `users.pair_id = new_pair_id`. The
   user's history comes with them. They can pair with someone new
   and have their existing day rows preserved.

3. **Future group/triple expansion is structurally feasible.** A
   user can in principle be in multiple pairs (or some
   group-shaped relationship). The data model doesn't force a
   single relationship per user; that's a UX constraint imposed at
   the app level. If group features land in v2, the architectural
   change is in pair semantics, not in user data.

These are made possible by user_id but their actual UX, semantics,
and edge cases are not designed yet. See Section 17.

### Schema implications

Migration_005 (already being landed in tile 1.3's bundle) now
absorbs these additions:

- `pairs.pair_id TEXT PRIMARY KEY` (Option B).
- `pairs.pair_key TEXT UNIQUE NOT NULL` (Option B).
- `users.user_id TEXT PRIMARY KEY` (user_id addition).
- `users.pair_id TEXT NULL REFERENCES pairs(pair_id)` — replaces
  the prior assumption that pair_id was non-nullable.
- `days.user_id TEXT NOT NULL REFERENCES users(user_id)` — day
  rows attach to user_id, not pair_id.
- `bug_reports.user_id TEXT NOT NULL REFERENCES users(user_id)` —
  bug reports attach to the reporting user. The pair_id at time of
  report may be captured as a snapshot field for context but isn't
  a foreign key (bug reports are write-only operational log; see
  Section 18).

The `name`, `username`, `active_leader`, `active_theme`,
`active_stickers` columns remain on the `users` row, identifying
the user's current display state and pair-key contribution.

No new migration file is required. All session-10 changes are
absorbed into migration_005.

### Cost summary

The user_id addition is essentially free at the schema level — one
column on users, one foreign key shift on days/bug_reports. At the
query level, "show me partner's days" is now: lookup partner's
user_id via pair → fetch days by user_id. One extra step. Trivial.

What user_id buys:
- Decoupling personal data from pair lifecycle.
- Cleaner conceptual model (user is a thing, pair is a relationship
  between users, not the other way round).
- Future flexibility (leaving, re-pairing, groups).

### Why the original v2 didn't have user_id

Honest reflection: the v1 model conflated person and pair because
nothing about the relationship was expected to change. v2 introduced
rotation (mutable pair-key) without introducing the deeper
separation (mutable relationship membership). user_id was the
missing piece. Caught in session 10 third pass after Morgan asked
"can we just give users a number when they first get in" — which
is exactly the right framing.

---

## 6. The three-identifier model

This is the architectural core of the doc. Every other section
depends on this distinction being clear.

There are three identifiers in the system:

### `user_id` — the person identifier

- **What it is:** a server-generated UUID (TEXT, 36 chars), assigned
  at the moment of `users` row creation.
- **What it represents:** a specific person, across all their
  relationships, forever.
- **Lifetime:** never changes. A user has exactly one user_id from
  creation onward.
- **Mutability:** none.
- **Visibility:** server-internal. Never displayed to users. Not in
  URLs that a user might see.
- **Storage:** `users.user_id` (PRIMARY KEY), `user://identity.cfg`
  on the client.
- **What attaches to it:** the user's personal data — day rows,
  coin balance, theme state, tutorial progress, subscription
  entitlements, identity values (name/username/active_leader), bug
  reports they've filed.

### `pair_id` — the relationship identifier

- **What it is:** a server-generated UUID (TEXT, 36 chars), assigned
  at pair creation.
- **What it represents:** a specific pair relationship between two
  user_ids.
- **Lifetime:** stable for the life of the relationship. If the
  pair dissolves, the pair_id may persist as a historical record
  (Section 17) but isn't reused.
- **Mutability:** none. The pair_id itself doesn't change. What
  changes is which user_ids are currently attached to it (e.g.
  if a user leaves) and what the pair_key hash is (rotation).
- **Visibility:** in URLs for operational endpoints that need to
  identify both the relationship and the acting user
  (e.g. `PUT /pair/:pair_id/users/:user_id`). Per-user-only
  operations (like fetching one user's day rows) use user_id
  directly without pair_id in the URL.
- **Storage:** `pairs.pair_id` (PRIMARY KEY), `users.pair_id`
  (nullable FK), `user://identity.cfg` on the client when paired.

### `pair_key` — the recovery credential

- **What it is:** SHA-256 hash (truncated 16 hex chars) of the
  canonical six-value string. See Section 12.
- **What it represents:** a way for a partner-mediated recovery
  flow to find a specific pair given six typed values.
- **Lifetime:** updates on every rotation (username or active_leader
  change for either user).
- **Mutability:** changes whenever the underlying six values change.
- **Visibility:** the hash itself is never shown to users. It's
  computed server-side from the values they type during recovery.
- **Storage:** `pairs.pair_key` (UNIQUE, indexed). Client stores
  the current pair_key in identity.cfg as a cached value, used
  only as a fallback if pair_id-based recovery somehow fails.
- **What it attaches to:** nothing. Child rows do not reference
  pair_key. It is purely a lookup column.

### Why the separation matters

Most of the design complexity people normally associate with
"keys that change" comes from confusing identity (who is this) with
addressing (how do I find this). The three-identifier split puts
each concept in its own slot:

- **Who is this person?** user_id.
- **Which relationship are they in?** pair_id.
- **How can a partner help them find their way back?** pair_key.

Rotation only touches pair_key (and the rotating user's row).
Personal data only touches user_id. Relationship membership only
touches pair_id on users + the existence of a pairs row. These
three concerns don't overlap operationally.

### State in `user://identity.cfg`

After successful onboarding or recovery, each client stores:

- `user_id` (own identifier, operational)
- `pair_id` (current relationship, or NULL if solo)
- `pair_key` (current recovery hash, advisory cache)
- `own_name`, `own_username`, `own_active_leader` (own three values,
  for display and for pair-key recomputation when rotating)
- `partner_user_id` (if paired — for direct partner-data queries)
- `partner_name`, `partner_username`, `partner_active_leader`
  (partner's three values as last known, for display)
- Optional: pending values during in-flight rotation (Section 11)

The `user_id` and `pair_id` are the load-bearing identifiers for
ongoing operation. Everything else is either display state or
recovery cache.

### Read paths under the three-identifier model

A few example reads to make the model concrete:

**"Show me my own calendar":**
1. Read `user_id` from identity.cfg.
2. `GET /users/:user_id/days?range=current_year`.
3. Server returns days for that user_id.

**"Show me my partner's calendar":**
1. Read `partner_user_id` and own `user_id` from identity.cfg.
2. `GET /users/:partner_user_id/days?range=current_year&caller=:user_id`.
3. Server validates that the caller user_id and the target user_id
   share a pair_id; if not, 404.
4. Server returns days for the target user_id.

**"Update my username":**
1. Read `user_id` from identity.cfg.
2. `PUT /users/:user_id` with new username in body.
3. Server reads the user's pair_id from the row, validates, runs
   rotation (Section 7), updates pair_key and the user's row in
   one transaction.
4. Returns new pair_key for the client to cache.

**"Recover from total data loss":**
1. Client has no identity.cfg. User types six values.
2. `POST /resolve` with the six values.
3. Server normalises, assembles canonical string, hashes, looks up
   in `pairs.pair_key`.
4. If found: returns user_id (own — determined by name match
   within the pair), pair_id, pair_key, partner_user_id, and the
   user's identity values + partner's identity values.
5. Client writes all of this to identity.cfg and proceeds normally.

The recovery path is the only one that uses pair_key. Everything
else operates on user_id + pair_id directly.

---

## 7. Pair-key rotation semantics

When username or active_leader changes for either user, the pair-key
hash changes. This is a **pair-key rotation**.

Under the three-identifier model, rotation is a small, well-scoped
operation that touches only two database rows:

1. Client sends `PUT /users/:user_id` with new values in the body.
2. Server opens D1 transaction.
3. Server reads the user's `users` row by `user_id` to get the
   current `pair_id` (must exist, else 404; if `pair_id` is NULL
   the user is solo, so rotation skips the pair_key update entirely
   and just runs step 9 — a solo user's username change is a
   plain UPDATE, no rotation).
4. Server reads the `pairs` row by `pair_id` (must exist, else
   500 — broken invariant) and both `users` rows for that
   `pair_id` (the rotating user plus the partner).
5. Server applies the requested change to the target user's row in
   memory.
6. Server computes the new canonical six-tuple from the resulting
   authoritative state (target user's new values + other user's
   on-disk current values).
7. Server computes the new `pair_key` hash.
8. Server `UPDATE pairs SET pair_key = new_hash WHERE pair_id =
   :pair_id`. This may fail the UNIQUE constraint if a collision
   exists (see Section 13).
9. Server `UPDATE users SET (username, active_leader) = (...)
   WHERE user_id = :user_id`.
10. Commit transaction.

**Day rows are not touched.** They're keyed by user_id, which
doesn't change during rotation. The `days` and `bug_reports` tables
are not involved in the rotation transaction at all.

**The user's identity (user_id) does not rotate.** Rotation is
purely about the pair_key recovery credential and the user's
display values. user_id is forever.

**Re-entrant by design.** The server never trusts the client's view
of the partner's state. The hash is always recomputed from the
server's own authoritative state in the same transaction as the
update. This is what makes concurrent rotations safe (Section 8).

**No rotation lock or batching window.** Considered both, rejected.
Lock = unnecessary given D1's serialization. Batching = adds
artificial UX latency to a snappy interaction without solving
anything serialization doesn't.

**Atomic guarantee.** The D1 transaction wraps each rotation. All
statements succeed or all roll back. No half-rotated states possible.

**The morning sequence is the natural rotation window** (impacts
tile 4.6). The day's biggest writes (sealing yesterday, initialising
today, coin payouts, streak updates) all fire during the morning
sequence's animated coin payout. The UI is non-interactive for the
duration. Under the user_id model this property matters less for
rotation itself (single-row UPDATE on pairs is fast and doesn't
touch day rows at all) but still matters for the *sealing* logic
that accompanies it. Tile 4.6 implementation should preserve the
non-interactive window.

See `four_tasks_morning_sequence_design_notes.md` "Migration window
property" for the hard time budget. That cross-reference uses the
older "migration" terminology; will be synced to "rotation" at next
housekeeping pass.

---

## 8. Concurrency analysis

This section explains why the system behaves correctly under
concurrent writes. It replaces what the original doc called the
"convergence proof" — under the three-identifier model, most
concurrency concerns evaporate because writes target different rows
keyed by different identifiers.

### D1's transaction semantics

D1 is SQLite under the hood. SQLite serializes transactions: any
two transactions that both want to write the database are ordered,
not interleaved. A transaction either commits in full or rolls back
in full. No reader ever sees a partial transaction.

This is the foundation. Most "what if A and B both X simultaneously"
worries dissolve once you accept that the database imposes an order
on every pair of write transactions.

### Case 1: A and B rotate simultaneously

Both clients send rotation requests at the same wall-clock time. D1
orders them — say A's lands first.

**A's transaction:**
- Reads pair by `pair_id`. Finds it.
- Reads both users for `pair_id`. Reads (A_old, B_old).
- Applies A's requested change: target_state = (A_new, B_old).
- Computes hash from target_state.
- UPDATEs `pairs.pair_key` and A's user row (by user_id).
- Commits.

**B's transaction (starts after A's commits):**
- Reads pair by `pair_id`. Finds it (still under same pair_id).
- Reads both users for `pair_id`. Reads (A_new, B_old) — A's
  change is visible because A committed.
- Applies B's requested change: target_state = (A_new, B_new).
- Computes hash from target_state.
- UPDATEs `pairs.pair_key` and B's user row (by user_id).
- Commits.

**Final state:** `pairs.pair_key` = hash(A_new, B_new), users rows
hold A_new and B_new. Both intended changes reflected. No data lost.
No 404s. No recovery flow triggered.

The key property: the server reads authoritative state inside the
transaction *immediately before* computing the hash. So whichever
rotation lands second sees the other's committed values and computes
the correct joint hash.

### Case 2: Day-tick during a rotation

B writes a day-tick (`PUT /users/:user_id/days/:date`) while A's
rotation is in flight.

Under the user_id model, these two writes target *different rows
keyed by different identifiers*. B's day-tick writes to
`days` keyed by `(user_id, date)`. A's rotation writes to `pairs`
(by pair_id) and A's `users` row (by user_id). There's no row
overlap at all.

D1 orders them. Either order is fine — the writes don't interact.
B's day row is written referencing B's user_id, completely independent
of A's pair_key change.

This is a substantial simplification over Option A (where day rows
would have been pair_key-keyed, making them potentially affected by
rotation). Under the user_id model, day writes and rotation are
fully orthogonal.

### Case 3: Leaving the pair during a partner's write

Edge case: A is mid-rotation. Simultaneously, B sets `users.pair_id
= NULL` (leaves the pair).

D1 orders them. If A's rotation commits first, B's leave just nulls
out B's pair_id afterward — A's rotation is preserved on the pair
side, and B's day rows continue to belong to B. If B's leave commits
first, A's rotation transaction reads both users for pair_id and
finds only A's row (B's pair_id is NULL, so B is no longer attached).
At this point the rotation is on a one-user pair, which is a
pair-lifecycle state Section 17 deals with — server returns an
appropriate error and A's rotation aborts cleanly.

The exact handling of one-user pairs is part of the pair-lifecycle
design conversation. For now: this case is handled correctly in the
sense that no data is lost or corrupted; the UX response is part of
that future design pass.

### Case 4: Recovery after total local data loss

B wiped their device, reinstalled, no identity.cfg.

B types six values: own three (from memory — name immutable, B knows
it; username and leader they chose themselves and presumably know) +
A's three (from memory of what A's values currently are).

Server hashes the six values. Looks up in `pairs.pair_key`. Two
sub-cases:

**Sub-case 4a: B's memory of A's values matches A's current values.**
Hash lookup finds pair. Server determines which of the two users is
"B" by matching the typed `name` against the `users.name` for the
pair (name is immutable, so this is unambiguous). Returns B's
user_id, the pair_id, the current pair_key, and full state including
A's current values. B's client writes all this to identity.cfg and
resumes normal operation.

**Sub-case 4b: A has rotated since B last knew A's values.**
B's typed values for A are stale. Hash lookup returns no row. Server
returns 404 with error code `not_found`.

Client UX in 4b: "We can't find your pair. Has your partner changed
their username or avatar since you last opened the app? Ask them
what their current display name and avatar are, and try again."

Recovery requires out-of-band contact with the partner. The partner
reads their current values off their screen; B types those plus B's
own three; recovery succeeds.

This is the legitimate recovery problem, and the only one. It does
not require any server-side endpoint beyond `/resolve` taking six
values.

### What we explicitly do NOT do

- **No three-value-search recovery endpoint.** Such an endpoint
  would be an enumeration oracle — it would let an attacker probe
  whether any given (name, username, leader) tuple corresponds to a
  real user. The threat model (Section 9) explicitly forbids this.
  Recovery requires all six values; if the partner's three are
  stale, the user must contact the partner.

- **No partner-mirror-based recovery.** Same reason. The partner-app
  is a source of truth for the partner's *current* values when the
  partner is contactable. It is not a server endpoint.

- **No "pending rotation" state visible to other clients.** Each
  rotation is atomic and either committed or not. There is no
  in-flight visible state to worry about.

### Why this is so much simpler than under Option A

The original concurrency story was trying to prove convergence for
a model where:
- The partition key (pair_key) was mutable.
- Clients used pair_key for ongoing work.
- A client could send a write to a now-defunct pair_key.
- The server had to detect this and the client had to recover.

Under the three-identifier model:
- Clients use user_id and pair_id for ongoing work, both stable.
- The pair_key is a recovery credential, never used in operational
  URLs.
- Day-tick writes go straight to user_id and never touch pair_key.
- The only place pair_key staleness matters is in the recovery
  flow, handled by the existing `/resolve` six-value lookup.

The original problem mostly evaporates.

---

## 9. Threat model

### Stakes framing

Before the scenarios: the data being protected is a two-person
habit tracker with no external integrations, no financial data,
no PII beyond a display name and username, and a subscription
worth roughly $1/week (refundable via store policy). The highest
realistic external-actor outcome is vandalism of someone's
calendar, not exfiltration of anything valuable. The doc takes
the analysis seriously below for completeness, but the appropriate
defence budget is small — none of the scenarios below justify
adding friction to legitimate user flows.

The genuinely interesting threat surface is social: a former
partner with shoulder-surf-level knowledge, a roommate who has
casually seen the other user's screen. Section 9.1 covers this,
and Section 17's change-username follow-up is the user-facing
lever for it.

External-actor scenarios (9.2 brute-force enumeration) are
covered by edge rate limiting and a search space large enough
to defeat random attack. We don't build defences beyond that.

### 9.1 Shoulder-surfing

**Attacker setup:** sees the target's screen for some period of
time. Observes the target's three values (name, username,
active_leader) AND the target's partner's three values (which are
also visible on the target's screen — that's the whole point of
partner-app-mediated recovery).

**Attack steps:**
1. Attacker reads six values from target's screen.
2. Attacker installs the app on their own device.
3. Attacker enters the recovery flow, types all six values.
4. Server hashes, finds pair, returns the matching user's `user_id`,
   the `pair_id`, and full state.
5. Attacker now has access to the pair, authenticated as one of the
   two users.

**What this gives the attacker:**
- Read access to the pair's data — that user's day rows and the
  partner's day rows via the polling path.
- Write access via the user_id — they can act as that user.
- Ability to trigger rotation, locking the legitimate user out
  until they perform their own recovery using current values.

**What this does NOT give the attacker:**
- Any other pair's data.
- The legitimate user's user_id on a separate authenticated session
  (the attacker is now the only one holding that user_id; the
  legitimate user is in the same position the attacker was in
  before the attack — they'd need to recover via the partner-
  mediated flow to get back in).
- Any financial or sensitive identity information (there is none).
- Any external services or accounts (there are none linked).

**Defence:** none. This is explicitly the threat model boundary.
The trade-off was deliberate at v2 design: partner-app-mediated
recovery requires that the partner's three values be visible on
each user's screen, which means anyone who can see the screen can
construct the recovery credential.

**Mitigations available to users:**
- A user who suspects shoulder-surf can rotate their username and/or
  active_leader. This invalidates the captured credential. (The
  attacker can then re-shoulder-surf if they have access, but the
  friction is real.)
- The privacy policy will be updated (Section 9.4) to be explicit
  that anyone who has seen the app's UI has seen the recovery
  credential.

**Why this is OK:** the data being protected is a two-person habit
tracker with no external integrations. The cost of defending against
shoulder-surfing (some kind of out-of-band confirmation channel
between partners, or a separate-credential layer) is enormous in
UX complexity. The benefit is minimal — an attacker willing to
shoulder-surf for tens of seconds of a calendar app is a vanishingly
rare scenario, and the harm is bounded (vandalism, not exfiltration
of valuable data).

### 9.2 Brute-force enumeration

**Attacker setup:** wants to attach to a specific target's pair.
Doesn't have shoulder-surf access. Knows the target's first name
through some other channel (workplace, public information, etc).

**Attack steps:**
1. Attacker enumerates possible (target_username, target_leader,
   own_three_values) combinations against the `/resolve` endpoint.
2. For each guess, server hashes and looks up; returns 404 (no
   match) or 200 (match).

**Search space:**
- Target's `name` is given (assumed known).
- Target's `username`: free text up to 40 chars. Search space is
  enormous.
- Target's `active_leader`: drawn from the sticker catalogue. Search
  space is small (the catalogue is finite).
- Attacker's own three values: free choice, but must hash with the
  target's three to a valid pair-key. So either the attacker guesses
  their own values randomly (~zero hit rate) or the attacker knows
  the partner's identity too (in which case the attack reduces to
  9.1).

**Probability of success per attempt:** approximately zero in
practice. The username space alone defeats random guessing.

**Defence:**
- Rate limiting at the Cloudflare edge (see
  `four_tasks_rate_limiting_design_notes.md`). Generous limits for
  real users (real recovery may need a handful of attempts), but
  enough to defeat brute-force enumeration.
- `/resolve` returns the same 404 response whether the six-value
  tuple is plausible or not. No information leak about partial
  matches.

**Why this is OK:** the search space is too large to brute-force.
Rate limiting closes the enumeration window. Genuine recovery (with
known six values) is unaffected because it succeeds on the first
attempt or with a single contact-the-partner retry.

### 9.3 Accidental wrong-attach

**Not an attack, but a real failure mode:** the joiner types six
values that happen to hash to an existing pair, but it's not the
pair they intended. They recover-into the wrong pair.

**Mechanism:** the joiner enters the recovery flow with their own
three values + their partner's three values. The combined six-tuple
happens to canonical-match another real pair (different intended
partners, same canonical string after normalisation).

At realistic scale, this requires a coincidence: another pair
exists where the canonical six-tuple matches exactly. With name
case-sensitive for hashing, NFC normalisation, and a sticker
catalogue of meaningful size, this is possible-but-improbable.

**Detection at attach time:** none. The server cannot tell "this
joiner intends to attach to pair X but is hashing to pair Y" because
the server has no concept of intent. The server only knows whether
a hash matches a row.

**Detection in practice:** organic. The wrong-attached user
immediately sees unfamiliar partner activity ("who is this person
who's been ticking off 'go to the gym'?") and unfamiliar history
on their own user_id (they get back day rows belonging to a real
user with the same name, but those day rows have content they don't
recognise). They uninstall and retry with corrected partner values.

**Mitigation:**
- Clear UX on attach: after recovery succeeds, the joiner sees the
  inviter's three values clearly displayed: "Welcome back. Your
  partner is [name], shown as [username] [active_leader]. Does this
  match what you remember?" If no, they uninstall and retry.
- The user_id model adds an important property here: the wrong-attach
  doesn't *transfer* the joiner's previous data into the wrong pair.
  The joiner gets back the user_id of the user in the matched pair
  whose name matches their typed name — they're now operating as
  that user, with that user's day rows visible. Their own previous
  data (if any) belonged to a different user_id and isn't affected
  by this lookup. So wrong-attach is a *visibility* problem (they
  see another pair's data), not a *data-loss* problem.

**Why this is acceptable (with caveat):** the failure surfaces
clearly to the wrong-attached user within minutes of usage (they see
calendar history and partner-panel content that's not their
partner's). At that point they uninstall and retry. The legitimate
pair on the other side may see anomalous activity during that brief
window — a write or two against their pair from the wrong-attacher,
visible as if from their own partner. They'd see this as their
partner behaving oddly for a moment. Resolution: the legitimate pair
can rotate their identity values, which invalidates the
wrong-attacher's authentication.

The frequency is low enough to accept this rough edge: it requires
the joiner's three values to coincidentally match a real other
user's three values exactly (same canonical form after
normalisation). At a username space of millions and a leader space
of dozens, this is rare in practice.

Caveat: if frequency turns out higher than expected in production
(e.g. because real users converge on a small subset of common
usernames), the mitigation is to add a second factor at recovery
time — most likely a short verification code the inviter generates
and shares out-of-band, that the joiner must also enter. This is
deferred to post-launch monitoring; not part of v1.0 scope.

### 9.4 Privacy policy alignment

The privacy policy currently says (paraphrasing) that the system
collects no personally identifying information and that the pair-key
identity model has no recovery vector outside the partner app. The
threat model treatment above is honest about the shoulder-surf
vulnerability, which the privacy policy should mention rather than
elide.

Update the privacy policy at next housekeeping pass to:
- Acknowledge that partner-app-mediated recovery means partner-
  visible values are recovery credentials.
- Note that physical access to a partner's screen constitutes
  access to the pair's recovery credential.
- Recommend rotating username/active_leader if unauthorised access
  is suspected.
- Note that the same applies to anyone you let scroll through your
  calendar.

This is for completeness, not correction — the current text doesn't
contradict the threat model, it just isn't fully explicit.

---

## 10. Picker UX (impacts tile 4.14b)

The identity picker for changing `active_leader` follows a
**commit-on-close** pattern:

- **Long-press on a sticker in the picker** → opens the context
  menu showing element-icons for every slot that sticker can fill.
- **Tap the LEADER icon in the context menu** → marks the new
  active_leader as *pending* on the client. No server call yet. The
  app may show a preview of how the new leader would look on the
  user's panel.
- **Picker closes (user dismisses with confirmed pending leader
  change)** → single server call, triggers the rotation cycle.

This collapses the request-window from "duration of user's fiddling
in the picker" to "single round-trip on commit." Combined with the
serialized rotation semantics from Section 8, simultaneous identity
changes by both users land safely.

**Note on other theme slots:** changes to non-leader theme slots
(palette, background, cell treatment, effects, etc.) do NOT follow
this commit-on-close pattern. They apply INSTANTLY — tap the icon,
slot updates, single server write fires immediately, no rotation.
Per the theme doc session 7 rewrite, the heavier commit pattern is
reserved for identity-affecting changes only.

**Implementation hooks for tile 4.14b:**
- `ThemeManager.preview_leader(sticker_id)` → local preview, no
  Backend call.
- `ThemeManager.commit_leader(sticker_id)` → calls
  `Backend.update_user_leader()`, triggers the rotation server-side.
- `ThemeManager.set_slot(slot_name, sticker_id)` → for non-leader
  slots, applies instantly and writes immediately. No preview/commit
  split.

---

## 11. Client-side defensive write requirements

Most of the defensive-write complexity from the original doc
disappears under the three-identifier model. The client's ongoing
operation uses stable identifiers (user_id and pair_id), which can't
become stale. What remains:

**Requirement 1 — store identifiers on successful pairing or
recovery.** On a successful response from `/resolve`, `/onboard`, or
any endpoint that establishes pairing, the client writes `user_id`,
`pair_id`, and current `pair_key` to `user://identity.cfg` and uses
user_id + pair_id for all subsequent requests.

**Requirement 2 — persist pending rotation values before sending.**
Before sending a rotation request (PUT changing username or
active_leader), the client writes the new values to a "pending" slot
in `user://identity.cfg`. Survives app crashes / device reboots
mid-request. On a successful response, the pending values commit to
canonical state and the slot is cleared. On a failed response, the
slot is also cleared (the user can retry).

**Requirement 3 — handle a successful response that arrives during
retry.** If the client's request response is lost (network failure
mid-flight), the client may retry the same rotation. The server's
behaviour is idempotent at the level of authoritative state:
re-running the same rotation produces the same final state because
the hash is recomputed from authoritative state. The first request's
UPDATE and the retry's UPDATE both target the same `user_id` and
both set the same final values — D1 serializes them, the second is
effectively a no-op. No special handling required.

**Requirement 4 — fall back to `/resolve` on identifier failure.**
If a request returns 404 because `pair_id` or `user_id` lookup
failed (extremely rare — implies identity.cfg corruption or a
server-side data mismatch), the client falls back to recovery:
prompt user to enter six values, call `/resolve`, get new
identifiers, retry.

That's the whole set. The "stale pair-key" recovery dance from the
original doc doesn't appear because under the three-identifier
model there is no stale-key write problem in ongoing operation
(Section 8).

The four requirements above ARE the implementation-level spec —
no separate write-rules doc.

---

## 12. Pair-key format and canonical string spec

### Hash specification

- **Algorithm:** SHA-256 of the canonical six-value string, truncated
  to 16 hex chars (64 bits of entropy). Collision probability at our
  scale: astronomically low. At 1 million pairs, the birthday-bound
  collision probability is ~2.7 × 10⁻⁸.
- **Output:** lowercase hex, no separators, fixed 16-character width.
- **Stored in:** `pairs.pair_key TEXT UNIQUE NOT NULL`. Indexed.

### Canonical string assembly

The canonical string is:

```
<name_A>|<username_A>|<active_leader_A>__<name_B>|<username_B>|<active_leader_B>
```

Where:
- `__` is a double underscore (the inter-user separator).
- `|` is a vertical bar (the intra-user separator).
- `A` and `B` are determined by lexicographic sort on the normalised
  `name` (after normalisation, see below). `A` is the smaller name.
- **Tiebreaker:** if both users' normalised names are identical
  (e.g. both genuinely named "Sandra" with identical case after
  normalisation), break the tie by sorting on `username`. If still
  tied, break on `active_leader`. If all three are tied, the two
  users are canonically indistinguishable — onboarding rejects this
  case with HTTP 400 error code `users_indistinguishable` to
  surface it loudly.

### Normalisation (applied to each value before assembly)

1. **Trim leading and trailing whitespace.**
2. **Collapse internal whitespace to single spaces** (tabs, multiple
   spaces, etc.).
3. **NFC Unicode normalisation.** Ensures decomposed and composed
   forms of the same characters produce identical hashes.

**No casefold.** Names and usernames are case-sensitive in the
canonical string. "Sandra" and "sandra" produce different hashes
and are considered different identities.

Rationale: the partner-app-mediated recovery property means a user
recovering from data loss doesn't need to *remember* exact case —
their partner can read the current spelling off their own screen.
Casefolding would lose information unnecessarily and create a
mismatch between displayed-case and hash-case. Case-sensitive is
simpler and matches user expectation: the name you typed IS the
name, character for character.

### Validation rules (enforced server-side)

The canonical form uses `|` and `__` as separators. Allowing them in
user input would let two distinct pairs produce identical canonical
strings. So:

- **Reject `|` (single pipe) in any of the six values.** HTTP 400
  with error code `invalid_character`.
- **Reject `__` (double underscore) anywhere in any of the six
  values.** HTTP 400 with `invalid_character`.
- **Reject empty strings** after normalisation. HTTP 400 with
  `empty_value`.

These rules apply at every endpoint that accepts identity values:
- `/onboard`, `/resolve`, `/join_by_values`
- Rotation-triggering writes (PUT username, PUT active_leader)
- Anywhere else a value might end up in the canonical string.

Client-side onboarding enforces the same rules with friendly error
messaging. Server-side validation is defence-in-depth.

### Length caps

- `name`: max 49 characters after normalisation.
- `username`: max 40 characters (UX likely tighter, ~20).
- `active_leader`: max 50 characters (sticker IDs are short — `frog_01`
  is 7 chars).

Length violations return HTTP 400 with error code `value_too_long`
and the offending field name.

---

## 13. Hash collision and wrong-attach

These are two different failure modes. The original doc treated
them as one thing called "ambiguous match," which was confused.

### Hash collision

**Definition:** two distinct sets of six canonical values that hash
to the same `pair_key`.

**Probability:** at our truncation (64 bits, ~2.7×10⁻⁸ at 1M
pairs). This is far below the rate at which any user will encounter
it organically. But it is not zero.

**When detected:** at write time. If a `/onboard` or rotation
UPDATE attempts to create a `pair_key` that already exists in the
`pairs` table, the UNIQUE constraint fires and the transaction
rolls back.

**Server response:** HTTP 409 with error code `pair_key_collision`.

**Resolution flow:**

For *onboarding* (a new pair being created):
1. Onboarding client sees 409 collision.
2. Client surfaces popup: "These values match an existing pair.
   Try a different username or sticker."
3. User changes one value, retries onboarding.

For *rotation* (an existing pair changing values):
1. Rotation client sees 409 collision.
2. Client surfaces popup: "This username/sticker combination is in
   use. Please choose a different one."
3. User picks something else, retries rotation.

For *recovery via `/resolve`* (a returning user typing six values):
- The joiner's hash doesn't *create* a new row, it *looks up* an
  existing one. If two pairs happened to share a hash (which they
  can't — UNIQUE constraint), only one would exist. So this case
  doesn't manifest as a 409. It manifests as a wrong-attach
  (Section 13.2).

### Wrong-attach

**Definition:** the recovering user types six values that hash to
an existing pair, but it's not the pair they intended. They
recover-into the wrong pair. This is Section 9.3's failure mode.

**Detection at attach time:** none. The server has no concept of
intent.

**Mitigation:** post-recovery UX showing the inviter's three
values to the joiner. If the joiner doesn't recognise them, they
uninstall and retry. See Section 9.3 for full treatment.

**Why this isn't a 409:** the server doesn't know two intended
pairs are competing for the same canonical string. From the server's
perspective, the lookup found a pair and authenticated the user as
the one whose name matches. The mismatch surfaces only through
human recognition.

**user_id property worth noting:** the wrong-attached user is
authenticated as the *legitimate* user of the matched pair (the
one whose typed name matches). Their own previous data — if any —
belonged to a different user_id and is not affected by this
lookup. So wrong-attach is a *visibility* problem (they see another
pair's data) and a *vandalism* risk (they can write to it), not a
*data-loss* problem for the wrong-attacher.

### Relationship between the two

A hash collision and a wrong-attach can look similar from the
user's perspective ("the system did the wrong thing"), but they're
mechanically distinct:

- **Collision** is server-detectable, returns 409, prevents the bad
  write.
- **Wrong-attach** is not server-detectable, requires post-attach
  human verification.

Treating them as separate problems with separate flows is correct.
The original doc's "ambiguous match" framing conflated them and
proposed a single resolution that wouldn't have worked for either.

---

## 14. Post-pair username revert

The Section 9.3 / Section 13 mitigations sometimes have a user
temporarily change their username to disambiguate. After
disambiguation succeeds, the user wants to revert. This section is
explicit that revert is supported and trivial.

### The flow

After ambiguous-attach resolution (or any other reason a user
temporarily varied their username), the user can revert by simply
rotating their username back to the original value via the settings
screen.

### Why the revert works

The revert is just another rotation. From the system's perspective,
"reverting" and "changing" are the same operation — a PUT with new
values. The server computes the new hash from current authoritative
state, UPDATEs `pairs.pair_key`, UPDATEs the user's row by user_id,
commits.

The post-revert pair-key is the joint hash of both users' current
values. As long as that joint hash is unique in the `pairs` table
(which it almost certainly will be — the post-attachment hash space
includes both users' identities, much sparser than the pre-attach
space that triggered the original collision), the revert succeeds.

### Edge cases

- **What if the revert collides with another pair?** Same UNIQUE
  constraint, same 409. The user picks a different revert target
  or gives up and keeps the disambiguated name. Realistic frequency:
  vanishingly low.
- **What if the partner doesn't actually revert their disambiguation
  changes?** Their problem. The pair functions correctly with the
  new values indefinitely. "Temporary" is the user's intention, not
  the system's behaviour.

### Why this section exists

Without it, ambiguous-attach resolution looks like the inviter is
permanently stuck with a temporary username. They're not. Making
this explicit removes a real UX concern.

---

## 15. What this changes about the schema

Key additions vs the original v1 sketch (landing as part of
migration_005 in tile 1.3's schema delta):

**`pairs` table:**
- `pair_id TEXT PRIMARY KEY` — stable per-pair UUID generated at
  first pair creation. Never changes. (Option B addition, session
  10 second pass.)
- `pair_key TEXT UNIQUE NOT NULL` — current canonical hash. Updates
  on every rotation. Indexed.

**`users` table:**
- `user_id TEXT PRIMARY KEY` — stable per-user UUID generated at
  `users` row creation. Never changes. (Session 10 third pass
  addition. THIS IS THE BIG ONE.)
- `pair_id TEXT NULL REFERENCES pairs(pair_id)` — which pair this
  user is currently in. NULL when solo.
- `name TEXT NOT NULL` — immutable display name.
- `username TEXT NOT NULL` — mutable display name.
- `active_leader TEXT` — sticker ID, identity-hashed.
- `active_theme TEXT` (JSON) — all non-identity theme slots.
- `active_stickers TEXT` (JSON) — sticker pool.
- `pending_unpair_notice TEXT NULL` — partner's last-known
  username at the moment of un-pair, for the dismissable banner
  on B's partner panel. NULL when no banner to show. (Session 11
  addition.)
- Plus the various per-user state columns: coin balance, lifetime
  stats, theme state, tutorial progress, subscription state,
  morning_payout_due_for, timezone (per migrations 003 + 004).

**`days` table:**
- `user_id TEXT NOT NULL REFERENCES users(user_id)` — day belongs
  to a specific user, not a specific pair. This is the structural
  shift that enables leaving / re-pairing without moving data.
- `date TEXT NOT NULL` — local date in user's timezone.
- Plus the day-content columns: tasks_done, diary, stamp,
  day_theme_state (per migration 005), and any motd/reaction columns.

**`bug_reports` table:**
- `user_id TEXT NOT NULL REFERENCES users(user_id)` — reports
  attach to the reporting user.
- `pair_id_snapshot TEXT` — the pair_id at the time of report,
  captured as a snapshot field (NOT a foreign key — pair lifecycle
  doesn't move reports around). NULL if reporter was solo.
- Plus the report content fields.

**Other locks:**
- Re-entrant rotation semantics enforced server-side.
- Picker UX pattern: commit-on-close for leader/username, INSTANT
  for all other theme slots.

Full schema deltas and migration_005 bundling captured in tile 1.3's
SCHEMA BUNDLE block in `four_tasks_godot_todo.txt`. All session-10
and session-11 changes are absorbed into migration_005 — no new
migration file required.

---

## 16. Outstanding questions (not blocking)

- **Recovery copy.** When a user lands on a fresh install and needs
  to reclaim, the onboarding flow needs a clear path for "I'm
  recovering an existing pair, here are six values." UX writing job
  — captured in onboarding doc (session 9 rewrite).
- **Recovery edge case: partner is offline / hasn't opened app
  recently.** User can type values from memory. If they don't
  remember partner's current values, they wait until partner is
  reachable or accept the failure tail.
- **Onboarding copy for the immutable name decision.** Confirmation
  step ("are you sure? you typed: Sandra") before locking. Captured
  in onboarding doc.
  - **Note on case display:** the confirmation modal should render
    the user's typed value EXACTLY as typed, not in any visually-
    emphasised case treatment. Names are case-sensitive in the
    hash; displaying "you typed: SANDRA" when the user typed
    "Sandra" would imply normalisation that doesn't happen.
- **Collision-popup UX.** Real-world collision rate is near-zero,
  but the popup copy still matters. Captured in onboarding doc.
- **Masked-hint recovery aid.** When a user fails a recovery
  attempt and the partner-app source-of-truth route isn't
  available, the server *could* return a masked hint based on the
  closest failed match. Open questions:
  - What "closest match" means.
  - Rate-limit the hint endpoint (~3 attempts per hour per IP,
    exponential backoff).
  - Which fields are eligible for hinting.
  - Show the hint after N failed attempts (e.g. 2), not on every
    attempt.
  Concept noted but not locked. **This would be an enumeration
  oracle if not carefully constrained** — same concern as Section
  8's rejected three-value-search endpoint. Design + implementation
  conversation deferred to Phase 3.
- **Privacy policy alignment.** See Section 9.4. Update at next
  housekeeping pass.
- **Cross-doc terminology sync.** "Migration" still appears in
  other docs referring to pair-key rotation (especially morning
  sequence doc's "migration window property"). Update at next
  housekeeping pass.
- **identity.cfg backup hook.** Now that user_id exists as a stable
  identifier, it could be backed up transparently via iOS Keychain
  / Android Keystore / iCloud Keychain / Google Drive backup. This
  would dramatically improve the recovery story for the common case
  (lost device, restored from backup). Not v1.0-blocking because
  the partner-mediated recovery story still works without it, but
  worth a focused conversation in Phase 4/5.

---

## 17. Pair lifecycle

**Status (session 11 close):**
- **Un-pairing: LOCKED** as symmetric-break model with full UX
  detail pass complete. Architecture from session 10 fourth-pass
  retained; UX detail filled in below. Implementation in tile 4.21.
- **Group / triple expansion: NOT designed.** Architecturally
  feasible under user_id but the UX, social texture, and
  monetisation hook need a dedicated design conversation. NOT
  v1.0 scope.

**Terminology note:** the action is called **un-pair** throughout
the UI. Internally and in backend code it remains "leave pair" /
"pair break" — same operation, more procedural name. User-facing
copy never says "leave" or "break."

### Symmetric-break model (LOCKED)

When either user initiates an un-pair, **both users go solo
together**. There's no asymmetric "I un-paired, you're stuck."

#### Entry point

The un-pair action lives in the **contextual long-press menu on
the partner panel header**. Same gesture grammar as the picker
context menu (tile 4.14b). No settings-screen entry, no surfaced
button in main UI.

Discoverability is via the in-app help menu only. The week-1
user who needs to un-pair due to a typo'd onboarding is better
served by deleting and reinstalling; the discoverability cost
of long-press is intentional friction against accidental taps.

#### Flow

1. User A long-presses the partner panel header → contextual
   menu opens → taps `Un-pair`.
2. Confirmation popup:
   ```
   Un-pair?

   [ Cancel ]  [ Un-pair ]
   ```
   No body copy. Title is the question, buttons are the answer.
   Confirm button has mild red tint. Cancel is the default/safe
   action.
3. On confirm: client calls `POST /users/:user_id/unpair`.
4. Server opens transaction:
   - Read the caller's row by `user_id` to get `pair_id` (must
     not be NULL, else 404 — caller isn't paired).
   - Read both users' rows for that `pair_id` (caller plus
     partner). Capture each user's current username for the
     notice flag.
   - `UPDATE users SET pair_id = NULL, pending_unpair_notice =
     :partner_username WHERE user_id = :caller_user_id` (sets
     caller's banner flag to *partner's* username).
   - `UPDATE users SET pair_id = NULL, pending_unpair_notice =
     :caller_username WHERE user_id = :partner_user_id` (sets
     partner's banner flag to *caller's* username).
   - `DELETE FROM pairs WHERE pair_id = :pair_id`.
   - Commit.
5. Server returns 200.
6. A's app reverts to solo UI. A then sees the **change-username
   follow-up popup** (see below).
7. B's next sync (foreground open or background poll) hits any
   endpoint that references `pair_id`, gets 404, client checks
   `/users/:user_id`, sees `pair_id` is NULL and
   `pending_unpair_notice` is populated, reconciles local state.
   Partner panel reverts to the standard fresh-solo empty state
   with the **dismissable banner** rendered in the panel
   whitespace (see below).

#### Change-username follow-up popup (A's side)

Immediately after the un-pair transaction commits, A sees:

```
Change your username?

[ text input, pre-filled with current username ]

[ Skip ]  [ Save ]
```

No body copy. The prompt's presence at this moment is the entire
message. A user who needs to make themselves harder for their
ex-partner to re-find via the standard `Add my partner` join
flow has the lever right here; a user who doesn't taps Skip.

Validation matches onboarding username rules (non-empty, no `|`
or `__`). Save commits via the standard username-update flow.
Either path lands A on their solo calendar with full solo UI.

This is not framed in copy as a "blocking" or "protection"
feature. The framing is implicit and speaks for itself.

The protection is real: B's join-by-values flow searches for
solo users matching `(name, username, active_leader)` exactly.
If A changes username, B's memorised values fail to match. A is
un-findable via memory. A determined adversary can still try
common variants, but the cost is raised from "type three values
you know" to "guess what they changed it to."

Name is not offered as changeable (immutable per Section 3).
Active_leader is not offered as changeable here — it lives in
the picker and can be changed any time. The username prompt is
the load-bearing one because username is the value most likely
to be remembered verbatim by the partner.

#### Partner-side dismissable banner (B's side)

When B next syncs, an inline dismissable banner appears in the
partner panel whitespace:

```
[username] un-paired.                            [×]
```

Where `[username]` is A's last-known username from B's
perspective (stored in `pending_unpair_notice` on B's user row
at the moment of un-pair). Position within the partner panel is
TBD at implementation — top, bottom, or floating treatment is a
build-time decision, not a design-time one.

Dismissal behaviour:
- One-shot. Dismissed banner does not return on app restart.
- Server-tracked, not client-tracked. Tapping `×` calls a dismiss
  endpoint (e.g. `POST /users/:user_id/dismiss_unpair_notice`)
  which sets `pending_unpair_notice = NULL`.
- The empty state (calendar styling + `Your partner's space is
  waiting` + `Add my partner` button) is identical to a
  fresh-solo user's. The banner is the *only* delta, and only
  until dismissed.

No popup. No modal. No interrupt. B can ignore the banner
indefinitely, dismiss it and re-pair, or dismiss it and continue
solo.

#### Re-pair after un-pairing

Standard `Add my partner` join-by-values flow. No reorientation
screen, no "you've recently un-paired" hint, no soft preference
for the ex-partner's values. A user who has un-paired looks
identical to a never-paired user except for the banner (if not
yet dismissed).

No cooldown. Either side can re-pair immediately after un-pairing
— either with each other (if neither changed username) or with
someone new.

The reversibility framing: un-pair *is* reversible at the
user-experience level. The backend `pair_id` is fresh on re-pair,
but the user-visible state (both back on each other's panels,
data intact, history visible per current rendering rules)
reconnects in seconds if both parties cooperate. The backend
discontinuity is invisible to the user.

#### Properties

- **Post-un-pair state equals post-onboarding solo state.** A
  user who has un-paired looks identical to a user who just
  finished onboarding solo: same `user_id`, `pair_id` NULL, no
  `pairs` row, full personal data scaffolding intact (calendar,
  MOTD, theme, coin balance). The only structural difference is
  `pending_unpair_notice` being populated until dismissed. This
  equivalence is the design target: solo is solo, regardless of
  history.
- **Symmetric outcome.** Either side initiating produces the
  same end state: both solo, both with their own data intact,
  both free to re-pair.
- **No mutual consent required.** This is intentional and
  matches how real relationships actually end — one person
  decides.
- **Personal data preserved on both sides.** Day rows, coins,
  theme state, history all hang off `user_id`, not `pair_id`.
  Pair dissolution doesn't touch any of it.
- **No data carried into a future pair.** The user keeps their
  own data. The next partner doesn't see anything from the
  previous pair. The pair-scoped `pairs` row is gone; nothing
  about the prior relationship persists in a queryable way.
- **The pair_key for the dissolved pair is gone.** Recovery via
  the old six-value hash returns 404 — the pair doesn't exist.
- **Idempotent.** If both users tap un-pair simultaneously, D1
  serializes the transactions; the second one finds the pair
  already gone, returns 200 because the desired state (solo) is
  achieved.
- **No app-level paternalism.** No cooldown, no typed-confirmation
  gate, no escalating friction. Long-press + popup is the
  entire friction budget.

#### Schema delta

One new nullable column on `users`:

```sql
pending_unpair_notice TEXT NULL
```

Stores the ex-partner's last-known username at the moment of
un-pair. Populated by the un-pair transaction on both users.
Cleared by the dismiss endpoint. NULL means no banner to show.

Lands in migration_005 alongside the other session-10 structural
changes (see Section 15). Locked as part of the session-11 close
— no architectural reason to split this column into a separate
migration.

#### Edge cases (all handled by the same flow)

- **The "death" case.** Surviving partner taps `Un-pair`
  themselves when ready. Same flow. They keep their history,
  go solo, can re-pair. The deceased partner doesn't get a
  notification because their app isn't being used. The
  `pending_unpair_notice` flag will be set on the deceased's
  row in the database; it has no effect because nothing reads it.
- **Offline partner.** B is offline when A un-pairs. B's local
  state is stale, but the next time B's app talks to the server
  it reconciles via the 404-then-resync path. Banner appears on
  next sync.
- **Simultaneous un-pairs.** D1 serializes. Idempotent outcome.
  Both users get the change-username popup; both see a banner
  on next sync (each carrying the other's last-known username).
- **A un-pairs, immediately regrets.** A and B both tap
  `Add my partner` and re-pair via standard join-by-values.
  No undo button needed — the standard flow *is* the undo.
  Whether the partner panel shows pre-break days vs only
  post-pair days is a separate UX decision (currently shows
  everything via `user_id`; deferred for v1.x).
- **Adversarial re-pair attempt.** A un-pairs and changes
  username via the follow-up popup. B types A's old values
  into `Add my partner` and fails to match. B has no in-app
  recourse; the protection works as intended.
- **B dismisses banner then later wants to remember A's
  username.** Information is gone from B's app. They'd have
  to ask A directly. This is correct behaviour — the banner is
  for confusion-resolution, not record-keeping.

#### Solo recovery implications

A solo user (post-un-pair or never-paired) has no pair_key. The
six-value partner-mediated recovery doesn't apply to them.

Solo recovery options:
- **Primary recovery for solo:** identity.cfg backup via
  iOS Keychain / Android Keystore / iCloud Keychain / Google
  Drive backup. The user_id rides along in the OS-level backup,
  device-restore brings it back, app picks up where it left
  off. This is how most apps handle device replacement.
- **No partner-mediated recovery for solo.** The pair_key
  mechanism is specifically a property of being paired. Solo
  users use the standard device-backup mechanism every other
  app on the phone uses.

This sharpens what pair_key actually IS in the system: a bonus
recovery layer that exists *because* you're paired, leveraging
your partner's screen as a second human verification vector.
When you're not paired, the bonus layer doesn't apply; you fall
back to standard device backup.

Implementation note for Phase 4/5: ensure identity.cfg is in
the OS backup scope on both iOS and Android. This may need
explicit configuration in the export settings. Treat as a tile
under Phase 5.

### Group / triple expansion (NOT designed)

Architecturally feasible under user_id because the data model
doesn't enforce "one pair per user" — that's a UX-layer
constraint. Could in principle support a user being linked to
multiple pairs simultaneously.

NOT v1.0 scope. NOT v1.x scope. If it happens, it's a v2
deliberate product expansion driven by monetisation (Morgan's
sketch: "third calendar slot, paid in coins, swipe past
partner panel to access").

Open questions if it ever ships:
- **What's a "group"?** Star graph (one user has multiple pair
  relationships, others don't see each other) or full mesh
  (group members all see each other)? Different products.
- **Pair-key extension.** The six-value hash is bilateral.
  Group recovery would need a different mechanism — possibly
  each pair in the group has its own pair_key independently.
- **UX.** The two-panel swipe breaks at three. Replaced with
  what? A switcher? A scrollable strip? Different shape.
- **Monetisation.** Coins to unlock the third slot, then the
  fourth, etc. Naturally pushes for subscription (subs earn
  coins faster). Fits the buddyware ethos (paid = more
  accountability surface) without breaking the v1.0 promise.
- **Social texture.** Two-person calendars have a specific
  intimacy. Multi-person calendars feel more like a project
  tracker or group chat. Might be a different product
  entirely, not just a Four Tasks feature.

These are real product questions, not engineering questions.
Defer to a dedicated design conversation if/when group
expansion is on the roadmap.

### What lands in implementation

For v1.0:
- Symmetric un-pair flow (tile 4.21).
- Change-username follow-up popup (part of tile 4.21).
- Dismissable banner mechanism (part of tile 4.21).
- `pending_unpair_notice` column in migration_005.
- Dismiss endpoint (part of tile 4.21).
- Solo state as a first-class state (pair_id NULL is fine).
- Solo recovery via OS-level identity.cfg backup (Phase 5).

For v1.x (post-launch):
- Re-pairing UX refinements (does the partner panel show all
  user_id history or only post-pair-creation? Deferred to
  actual usage data).
- Optional pair keepsake feature (export a record of a
  dissolved pair's shared days as an image). Low priority,
  high goodwill.

For v2 (if ever):
- Group / triple expansion.

---

## 18. Bug reports — out-of-band lifecycle

Bug reports do NOT participate in pair-key rotation (see schema
comments). They're a write-only log of "this thing was reported at
this moment by this then-current state," not referential data.
Under the user_id model, bug_reports attach to the reporting
user_id (with pair_id snapshotted as context).

Implication for downstream tooling: the DevKit bug inbox (tile
4.12) should treat bug_reports as their own thing, not as
pair-attached data. Deletion, archiving, triage all happen in the
inbox. This squares with the privacy policy: bug reports are
operational support data, retained as long as needed to investigate,
then deleted or archived per the policy's retention rules.

The user_id attachment means: if a user leaves a pair and re-pairs
later, their bug report history follows them (they reported the
bug, the report belongs to their user_id). The pair_id snapshot
preserves the context the report was filed in, but doesn't tie the
report's lifecycle to the pair's lifecycle.

---

## 19. Cross-references

- `four_tasks_write_rules_design_notes.md` — **SUPERSEDED** at
  session 11 close. The session-4 doc described the pre-session-10
  data model (pair-key as partition key, copy-then-delete
  rotations, no user_id). Its rules now live authoritatively
  across the architecture docs (this doc for rotation + validation
  + defensive writes, the feature docs for column rules, the
  project conventions for response envelope + status codes +
  caller). The eventual `/server/README.md` will be the working
  reference artifact. Kept in git history for context only.
- `four_tasks_theme_design_notes.md` — full theme slot catalogue.
  `active_leader` is one slot in the catalogue (the identity-hashed
  one); the rest live in the `active_theme` JSON.
- `four_tasks_morning_sequence_design_notes.md` — rotation window
  property and the natural protection it provides. (Currently uses
  the older "migration window" terminology.)
- `four_tasks_onboarding_design_notes.md` — recovery copy, immutable
  name confirmation, collision UX, post-attach verification UX.
  Name display case-sensitive (no caps treatment in confirmation
  modal).
- `four_tasks_timezone_and_sealing_design_notes.md` — sealing
  happens during the claim endpoint transaction.
- `four_tasks_rate_limiting_design_notes.md` — rate limits at the
  Cloudflare edge, which defend the recovery endpoint against
  enumeration.
- `four-tasks/server/schema.sql` — concrete data model.
- Roadmap tile 1.3 — server-side defensive writes (BLOCKER resolved
  by this doc's Section 5; tile 1.3 unblocked for implementation).
- Roadmap tile 4.14b — picker UX (long-press context menu).
