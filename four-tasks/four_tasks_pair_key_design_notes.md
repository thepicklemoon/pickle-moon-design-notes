# Pair-Key Identity — Design Notes

**Status:** LOCKED. This is the canonical identity model for Four Tasks
v1.0+. Implementation lands in tile 1.3 (server-side defensive write
rules) and tile 4.14b (theme manager / picker UX).

This doc supersedes the v1 two-name model. The v1 doc
(`four_tasks_pair_key_identity_design_notes.md`) is referenced for
historical context only and is no longer authoritative.

Session 7 terminology update: throughout this doc, the third identity
value was previously called `emoji` or `icon`. It is now consistently
called `active_leader` to match the theme system's slot catalogue
language (session 7 theme doc rewrite). The semantics are identical —
it is still the user's chosen visual identity sticker — but the
column name and references have been normalised.

---

## What changed from v1

**v1 (locked in session 13):** pair-key = `{min(name,partner)}__{max(name,partner)}`.
Two values per user. Recovery = remember both names.

**v2 (this doc):** pair-key = hash of normalised six-tuple
`(name_A, username_A, active_leader_A, name_B, username_B,
active_leader_B)`, where A/B are sorted by name lexicographically.

The shift: identity is now anchored in **three values per user**, not
one, and recovery is **partner-app-mediated** — five of six values are
visible on the partner's screen at all times.

## Why the change

The v1 model was driven by collision avoidance — "two real Morgans +
two real Ayshis won't collide because the *pair* is the unit." Correct,
but it collapsed the wrong concern. The actual concern was **recovery**
in the case of local data loss (which has happened to mum and to Ayshi
on the web build).

v1 recovery requires the user to remember what they typed at onboarding.
That's fragile across long timelines: mum recovering 8 months later may
not remember whether she typed `Sandra` or `sandra` or `mum`.

v2 recovery doesn't depend on memory. The credential **is** the current
visible state on the partner's app. As long as one partner has working
data, the other can always rebuild from it.

## The six values

| Field           | Mutability | Migration trigger? | Visible to partner? |
|-----------------|------------|--------------------|---------------------|
| name            | immutable  | n/a                | yes (label on panel)|
| username        | mutable    | yes                | yes (display name)  |
| active_leader   | mutable    | yes                | yes (theme avatar)  |

Same row, both users. So six values total per pair.

**Why name is immutable:** name change in v1 was the gnarliest case,
deferred to v1.x. Locking name as immutable in v2 keeps that complexity
off the table permanently. Onboarding copy enforces the choice:

> *"Choose carefully — your name can't be changed after this.*
> *Spell it the way your partner knows you."*

Username covers all the "I want to change how I show up" cases that
name would have covered. Username is freely mutable. No regret risk.

**Why username and active_leader are mutable:** they're meant to be
expressive. Username is what you display; active_leader drives the
chosen visual identity sticker. Locking them would kill the
personalisation features. They mutate, and the pair-key mutates with
them.

**Note on `active_leader`:** this is the sticker ID string for the
user's chosen visual identity sticker (e.g. `"frog_01"`). It is ONE of
many theme slots, but it is the only theme slot that participates in
the pair-key identity hash. All other theme slots (palette source,
background, cell treatments, effects, audio, etc.) live in the
`users.active_theme` JSON column and are non-identity writes — they
freely change without migration. See the theme doc for the full
slot catalogue.

## The recovery argument

Failure scenarios for v2 recovery:

1. **User loses data; partner still has working app.** ✅ Partner's
   screen shows current username and active_leader for both users.
   User reads off five values, remembers own name (sixth), types all
   six. Server hashes, finds pair, hands back data.

2. **Both users lose data simultaneously.** ❌ Same failure case as v1.
   No mitigation. This is the "freak accident" tail.

3. **User changes active_leader, then loses data before partner notices.**
   ✅ Partner's screen shows *current* active_leader (because partner's
   app is the source of truth for that user's current state — the
   server pushes updates, partner sees them within polling interval).

4. **Pair-key collision (two real pairs hash to same key).** ✅
   Onboarding second pair gets prompted to vary username (the easiest
   field to change without losing identity).

5. **User maliciously impersonates another pair.** ❌ Requires guessing
   six values including the partner's three. Effectively impossible by
   guessing; possible if attacker has shoulder-surfed the entire pair UI.
   Out of threat model — the data being protected is a habit tracker,
   not financial or sensitive.

Trade-off: v2 is more robust for the common case (local data loss) and
no worse for the edge cases. Net win.

## Migration semantics

When username or active_leader changes, the pair-key changes. Migration
is a transactional rewrite:

1. Compute new pair-key from new authoritative state
2. Open D1 transaction
3. INSERT new `pairs` row with new key
4. INSERT both `users` rows under new key (target user with new
   username/active_leader values; partner copied unchanged)
5. INSERT all `days` rows for both users under new key
6. DELETE old `days` rows
7. DELETE old `users` rows
8. (bug_reports do NOT migrate — see schema comments)
9. DELETE old `pairs` row
10. Commit transaction

**Re-entrant by design.** The server never trusts a client's cached
pair-key. Every migration re-derives the key from current state.
Two simultaneous migrations converge:

- A and B both change active_leader at near-same time
- A's request lands first: migrates to key1 (A_new + B_old)
- B's request lands second: server reads pair (now at key1), sees B's
  state-on-disk is still B_old, but the *request* says B_new. Server
  re-derives correct final key: (A_new + B_new) = key2. Migrates again.
- End state: pair at key2, both users' active_leaders correctly reflected.

The second migration is "wasted" only in the sense that key1 existed
briefly. Final state is always correct.

**Why no migration lock or batching window:** considered both, rejected.
Lock = unnecessary if migrations are idempotent in outcome. Batching =
adds artificial UX latency to a snappy interaction without solving
anything re-entrancy doesn't.

**Atomic guarantee:** the D1 transaction wraps each individual migration.
Within a single migration request, all SQL statements succeed or all
roll back. No half-migrated states possible.

**Stale-key write handling.** Even with re-entrant migrations and atomic
transactions, there's a cross-request race: a client sends a write to
the old pair-key while a migration is mid-flight, the migration commits
first, the write arrives at a non-existent key. Server response in that
case: HTTP 404 with a `{"error":"not_found"}` body (per the unified
write rules envelope). Client falls into the recovery path: re-derives
the key from current state and retries the original write. Should be a
rare path in practice — the morning sequence handles the highest-volume
case naturally (see below) and intra-day collisions require both users
tapping near-simultaneously.

**The morning sequence is the natural migration window** (impacts
tile 4.6). The day's biggest writes (sealing yesterday, initialising
today, coin payouts, streak updates) all fire during the morning
sequence's animated coin payout. The UI is non-interactive for the
duration — user is watching the animation, can't tap anything, can't
fire other writes. That ~15-30s window is a built-in protection for the
once-per-day high-stakes writes: the server has predictable,
uninterruptible time to complete any migration or sealing logic before
the user regains control. This means mid-day intra-pair writes
(active_leader swap, day-tick) are the only remaining places stale-key
conflicts can happen, and those are rare by definition.

We did NOT design the morning sequence FOR this — it exists for the
joy of the coin payout. But it incidentally solves the most dangerous
write-collision case for free, and tile 4.6's implementation should
preserve that property (no user-tappable controls until coins settle).

See `four_tasks_morning_sequence_design_notes.md` "Migration window
property" for the hard time budget.

## Picker UX (impacts tile 4.14b)

The identity picker for changing `active_leader` follows a
**commit-on-close** pattern:

- **Long-press on a sticker in the picker** → opens the context menu
  showing element-icons for every slot that sticker can fill.
- **Tap the LEADER icon in the context menu** → marks the new
  active_leader as *pending* on the client. No server call yet. The
  app may show a preview of how the new leader would look on the
  user's panel.
- **Picker closes (user dismisses with confirmed pending leader change)**
  → single server call, triggers the migration cycle.

This collapses the race-condition window from "duration of user's
fiddling in the picker" to "single round-trip on commit." Combined
with re-entrant migrations, simultaneous identity changes by both users
land safely.

**Note on other theme slots:** changes to non-leader theme slots
(palette, background, cell treatment, effects, etc.) do NOT follow
this commit-on-close pattern. They apply INSTANTLY — tap the icon,
slot updates, single server write fires immediately, no migration.
Per the theme doc session 7 rewrite, the heavier commit pattern is
reserved for identity-affecting changes only.

**Implementation hooks for tile 4.14b:**
- `ThemeManager.preview_leader(sticker_id)` → local preview, no
  Backend call.
- `ThemeManager.commit_leader(sticker_id)` → calls Backend.update_user_leader(),
  triggers the migration server-side.
- `ThemeManager.set_slot(slot_name, sticker_id)` → for non-leader
  slots, applies instantly and writes immediately. No preview/commit
  split.

## Client-side defensive write requirements

The server-side rules guarantee state integrity under any sequence of
failures. The client must hold up its half of the contract to avoid
lockout scenarios. These requirements apply to tile 1.7 (Godot identity
layer) and tiles 4.14b / 4.15 (active_leader and username edit flows):

**Requirement 1 — persist pending change before sending request.**
Before sending a migration-triggering PUT, client writes the new
username/active_leader to a "pending" slot in `user://identity.cfg`.
Persists across crashes, app kills, OS terminations, device reboots.

**Requirement 2 — recover from pending state on 404.**
If a subsequent GET /pair/:key returns 404 and a pending change is in
the slot, the client uses the pending values (not the canonical old
values) when deriving the recovery key. Handles the case where the
migration request was processed server-side but the response never
reached the client.

**Requirement 3 — clear pending slot on confirmed success.**
When the migration response confirms `migrated: true` and returns the
new pair-key, the client commits the pending values to canonical state,
updates the stored pair-key, and clears the pending slot.

**Requirement 4 — concurrent migrations are handled by retry.**
If two clients in the same pair migrate simultaneously, one will hit
404 because the other's migration committed first. The 404 recovery
path discovers the new state via /resolve using pending values, then
retries the original PUT against the new key. Re-entrant by design.

See `four_tasks_write_rules_design_notes.md` "Client-side defensive
write requirements" for the implementation-level spec.

## Pair-key format

Pair-key is a **hash**, not a human-readable string.

- Algorithm: SHA-256 of the canonical six-value string, truncated to
  16 hex chars. (Collision probability at our scale: astronomically low.)
- Canonical form:
  `name_A|username_A|active_leader_A__name_B|username_B|active_leader_B`
  where A/B are sorted by `name` lexicographically. Pipes and double-
  underscore are separators; chosen because they're rare in natural
  user input but no longer a security concern (forgery is out of
  threat model — see "user maliciously impersonates" above).
- Stored as TEXT in `pairs.key` and referenced by all child rows.

**Why hash, not concatenation:** fixed-width keys are slightly faster
as primary keys (especially for the high-volume `days` table indexed
by pair_key + user_name + date). Human values are always available by
looking up the `users` rows for that pair — pair-key doesn't need to
be readable.

## What this changes about the schema

Key additions vs the original v1 sketch (now landing as part of
migration_005 in tile 1.3's schema delta):

- `users.username` column (was conflated with `name` in v1)
- `users.active_leader` column — replaces the prototype-era `emoji` /
  `icon` column. Identity-hashed, participates in pair-key.
- `users.active_theme` JSON column — all non-identity theme slots
- `users.active_stickers` JSON column — sticker pool
- Documentation that `name` is immutable, `username` +
  `active_leader` are migration triggers
- Re-entrant migration semantics enforced server-side
- Picker UX pattern: commit-on-close for leader/username, INSTANT for
  all other theme slots

See `four_tasks_write_rules_design_notes.md` for the full schema deltas
and migration bundling.

## Outstanding questions (not blocking)

- **Recovery copy.** When a user lands on a fresh install and needs to
  reclaim, the onboarding flow needs a "recover existing pair" path with
  clear instructions to ask the partner for their five visible values.
  UX writing job — landed in the onboarding doc (session 6).
- **Recovery edge case: partner is offline / hasn't opened app recently.**
  User can still type the values from memory. If they don't remember
  them, they can wait until partner is online or accept data loss. No
  technical fix — just accept this as the failure tail.
- **Onboarding copy for the immutable name decision.** Needs a confirmation
  step ("are you sure? you typed: SANDRA") before locking. Captured in
  onboarding doc.
- **Collision-popup UX.** When a pair-key collides with an existing
  pair, onboarding shows a popup asking the user to vary their
  username. Real-world collision rate will be near-zero, but the
  popup copy still matters — users are universally familiar with
  "this username is taken" patterns from social media / gaming /
  every signup flow, so we can lean on that mental model. Copy job,
  not a design problem. Captured in onboarding doc.
- **Masked-hint recovery aid.** When a user fails a recovery attempt
  and the partner-app source-of-truth route isn't available (or didn't
  help), the server can return a masked hint based on the *closest*
  failed-match attempt — e.g. user typed "Morgan Ryan" and the closest
  stored entry is "morgan ryan", server responds with hint
  `M***** r***`. Lets the user iterate without the server ever
  revealing a stored value in plaintext to an unauthenticated session.
  Open design questions:
    • What "closest match" means — fuzzy match on own name only, or
      on the full six-tuple?
    • Rate-limit the hint endpoint to prevent enumeration. Probably
      3 attempts per hour per IP, with exponential backoff.
    • Which fields are eligible for hinting — name only (highest-value
      for legitimate users, lowest-risk because attacker would also
      need username + active_leader)? Or all three?
    • Show the hint after N failed attempts (e.g. 2), not on every
      attempt — keeps the recovery flow from leaking on a single
      typo and protects against casual enumeration.
  Concept locked. Design + implementation = Phase 3.

## Bug reports — out-of-band lifecycle

Bug reports do NOT migrate with pair-key changes (see schema comments).
This is deliberate — they're a write-only log of "this thing was reported
at this moment by this then-current pair-key", not referential data.

Implication for downstream tooling: the DevKit bug inbox (tile 4.12 +
its admin-side counterpart in DevKit) should treat bug_reports as their
own thing, not as user-attached data. Deletion, archiving, triage all
happen in the inbox, not via the pair. This squares with the privacy
policy: bug reports are operational support data, retained as long as
needed to investigate, then deleted or archived per the policy's
retention rules.

## Cross-references

- `four_tasks_write_rules_design_notes.md` — server-side defensive
  writes, where migration transactions live. Schema deltas for
  migrations 003 + 004 + 005 bundled here.
- `four_tasks_theme_design_notes.md` — full theme slot catalogue.
  `active_leader` is one slot in the catalogue (the identity-hashed
  one); the rest live in the `active_theme` JSON.
- `four_tasks_morning_sequence_design_notes.md` — migration window
  property and the natural protection it provides.
- `four_tasks_onboarding_design_notes.md` — recovery copy, immutable
  name confirmation, collision UX.
- `four_tasks_timezone_and_sealing_design_notes.md` — sealing happens
  during the claim endpoint transaction, no nightly cron.
- `four-tasks/server/schema.sql` — concrete data model implementing
  this design.
- Roadmap tile 1.3 — server-side defensive writes.
- Roadmap tile 4.14b — picker UX (long-press context menu).
