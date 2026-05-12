# Pair-Key Identity v2 — Refinement Notes (Session 2)

**Context:** these notes refine, but do NOT replace, the locked design in
`four_tasks_pair_key_identity_design_notes.md` (session 13) and port plan 2.3.
They capture decisions made during the schema design for tile 1.1, where the
v1 two-name model was found to be under-specified for the recovery story
Morgan actually wanted.

Status: **LOCKED**. Implementation lands in tile 1.3 (server-side defensive
write rules) and tile 4.14 (theme manager / emoji picker UX).

---

## What changed from v1

**v1 (locked in session 13):** pair-key = `{min(name,partner)}__{max(name,partner)}`.
Two values per user. Recovery = remember both names.

**v2 (this doc):** pair-key = hash of normalised six-tuple
`(name_A, username_A, icon_A, name_B, username_B, icon_B)`, where A/B are
sorted by name lexicographically.

The shift: identity is now anchored in **three values per user**, not one,
and recovery is **partner-app-mediated** — five of six values are visible on
the partner's screen at all times.

## Why the change

The v1 model was driven by collision avoidance — "two real Morgans + two
real Ayshis won't collide because the *pair* is the unit." Correct, but it
collapsed the wrong concern. The actual concern was **recovery** in the case
of local data loss (which has happened to mum and to Ayshi on the web build).

v1 recovery requires the user to remember what they typed at onboarding.
That's fragile across long timelines: mum recovering 8 months later may not
remember whether she typed `Sandra` or `sandra` or `mum`.

v2 recovery doesn't depend on memory. The credential **is** the current
visible state on the partner's app. As long as one partner has working
data, the other can always rebuild from it.

## The six values

| Field      | Mutability | Migration trigger? | Visible to partner? |
|------------|------------|--------------------|---------------------|
| name       | immutable  | n/a                | yes (label on panel)|
| username   | mutable    | yes                | yes (display name)  |
| icon       | mutable    | yes                | yes (theme avatar)  |

Same row, both users. So six values total per pair.

**Why name is immutable:** name change in v1 was the gnarliest case, deferred
to v1.x. Locking name as immutable in v2 keeps that complexity off the table
permanently. Onboarding copy enforces the choice:

> *"Choose carefully — your name can't be changed after this.*
> *Spell it the way your partner knows you."*

Username covers all the "I want to change how I show up" cases that name
would have covered. Username is freely mutable. No regret risk.

**Why username and icon are mutable:** they're meant to be expressive.
Username is what you display; icon drives the entire theme. Locking them
would kill the personalisation features. They mutate, and the pair-key
mutates with them.

## The recovery argument

Failure scenarios for v2 recovery:

1. **User loses data; partner still has working app.** ✅ Partner's screen
   shows current username and icon for both users. User reads off five
   values, remembers own name (sixth), types all six. Server hashes,
   finds pair, hands back data.

2. **Both users lose data simultaneously.** ❌ Same failure case as v1.
   No mitigation. This is the "freak accident" tail.

3. **User changes icon, then loses data before partner notices.** ✅
   Partner's screen shows *current* icon (because partner's app is the
   source of truth for that user's current state — the server pushes
   updates, partner sees them within polling interval).

4. **Pair-key collision (two real pairs hash to same key).** ✅ Onboarding
   second pair gets prompted to vary username (the easiest field to
   change without losing identity).

5. **User maliciously impersonates another pair.** ❌ Requires guessing
   six values including the partner's three. Effectively impossible by
   guessing; possible if attacker has shoulder-surfed the entire pair UI.
   Out of threat model — the data being protected is a habit tracker,
   not financial or sensitive.

Trade-off: v2 is more robust for the common case (local data loss) and
no worse for the edge cases. Net win.

## Migration semantics

When username or icon changes, the pair-key changes. Migration is a
transactional rewrite:

1. Compute new pair-key from new authoritative state
2. Open D1 transaction
3. INSERT new `pairs` row with new key
4. UPDATE both `users` rows to point to new key
5. UPDATE all `days` rows for both users to point to new key
6. (bug_reports do NOT migrate — see schema comments)
7. DELETE old `pairs` row
8. Commit transaction

**Re-entrant by design.** The server never trusts a client's cached
pair-key. Every migration re-derives the key from current state.
Two simultaneous migrations converge:

- A and B both change emoji at near-same time
- A's request lands first: migrates to key1 (A_new + B_old)
- B's request lands second: server reads pair (now at key1), sees B's
  state-on-disk is still B_old, but the *request* says B_new. Server
  re-derives correct final key: (A_new + B_new) = key2. Migrates again.
- End state: pair at key2, both users' emojis correctly reflected.

The second migration is "wasted" only in the sense that key1 existed
briefly. Final state is always correct.

**Why no migration lock or batching window:** considered both, rejected.
Lock = unnecessary if migrations are idempotent in outcome. Batching =
adds artificial UX latency to a snappy interaction without solving
anything re-entrancy doesn't.

**Atomic guarantee:** the D1 transaction wraps each individual migration.
Within a single migration request, all 6 SQL statements succeed or all
roll back. No half-migrated states possible.

**Stale-key write handling.** Even with re-entrant migrations and atomic
transactions, there's a cross-request race: a client sends a write to
the old pair-key while a migration is mid-flight, the migration commits
first, the write arrives at a non-existent key. Server response in that
case: HTTP 409 (Conflict) with a `{"error":"stale_pair_key","current_key":
"<new_hex>"}` body. Client updates its cached key from the response and
retries the original write. Should be a rare path in practice — the
morning sequence handles the highest-volume case naturally (see below)
and intra-day collisions require both users tapping near-simultaneously.

**The morning sequence is the natural migration window** (impacts tile 4.6).
The day's biggest writes (sealing yesterday, initialising today, coin
payouts, streak updates) all fire during the morning sequence's animated
coin payout. The UI is non-interactive for the duration — user is
watching the animation, can't tap anything, can't fire other writes.
That ~5s window is a built-in protection for the once-per-day high-stakes
writes: the server has predictable, uninterruptible time to complete
any migration or sealing logic before the user regains control. This
means mid-day intra-pair writes (icon swap, day-tick) are the only
remaining places stale-key conflicts can happen, and those are rare
by definition.

We did NOT design the morning sequence FOR this — it exists for the
joy of the coin payout. But it incidentally solves the most dangerous
write-collision case for free, and tile 4.6's implementation should
preserve that property (no user-tappable controls until coins settle).

## Emoji picker UX (impacts tile 4.14)

The picker must distinguish **preview** from **commit**:

- **Long-press on an emoji in the picker** → swap palette locally for
  preview. No server call. No pair-key migration. Pure client-side theme
  swap so the user can see how the new icon would look.
- **Picker closes with a confirmed selection** → single server call,
  triggers the migration cycle.

This collapses the race-condition window from "duration of user's
fiddling in the picker" to "single ~200ms round-trip on commit." Combined
with re-entrant migrations, simultaneous emoji changes by both users
land safely.

**Implementation hooks for tile 4.14:**
- `ThemeManager.preview_emoji(emoji_id)` → local palette swap, no
  Backend call.
- `ThemeManager.commit_emoji(emoji_id)` → calls Backend.update_user_emoji(),
  triggers the migration server-side.
- Picker UI exits via either Confirm button (calls commit_emoji with
  current selection) or Cancel button (reverts to last committed emoji).

## Pair-key format

Pair-key is a **hash**, not a human-readable string.

- Algorithm: SHA-256 of the canonical six-value string, truncated to
  16 hex chars. (Collision probability at our scale: astronomically low.)
- Canonical form: `name_A|username_A|icon_A__name_B|username_B|icon_B`
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

The schema is in `four-tasks/server/schema.sql`. Comments at the top
of that file capture the locked decisions from this doc inline.

Key additions vs the original v1 sketch:
- `users.username` column (was conflated with `name` in v1)
- Documentation that `name` is immutable, `username` + `icon` are
  migration triggers
- Comments calling out re-entrant migration semantics for tile 1.3
- Comments calling out the picker UX pattern for tile 4.14

## Outstanding questions (not blocking)

- **Recovery copy.** When a user lands on a fresh install and needs to
  reclaim, the onboarding flow needs a "recover existing pair" path with
  clear instructions to ask the partner for their five visible values.
  UX writing job — defer to Phase 3 (onboarding tiles).
- **Recovery edge case: partner is offline / hasn't opened app recently.**
  User can still type the values from memory. If they don't remember
  them, they can wait until partner is online or accept data loss. No
  technical fix — just accept this as the failure tail.
- **Onboarding copy for the immutable name decision.** Needs a confirmation
  step ("are you sure? you typed: SANDRA") before locking. Phase 3 work.
- **Collision-popup UX (Phase 3, tile 3.4).** When a pair-key collides
  with an existing pair, onboarding shows a popup asking the user to
  vary their username. Real-world collision rate will be near-zero, but
  the popup copy still matters — users are universally familiar with
  "this username is taken" patterns from social media / gaming / every
  signup flow, so we can lean on that mental model. Copy job, not a
  design problem. Captured here so Phase 3 work picks it up.
- **Masked-hint recovery aid (Phase 3, tile 3.4).** When a user fails
  a recovery attempt and the partner-app source-of-truth route isn't
  available (or didn't help), the server can return a masked hint based
  on the *closest* failed-match attempt — e.g. user typed "Morgan Ryan"
  and the closest stored entry is "morgan ryan", server responds with
  hint `M***** r***`. Lets the user iterate without the server ever
  revealing a stored value in plaintext to an unauthenticated session.
  Open design questions for when Phase 3 lands:
    • What "closest match" means — fuzzy match on own name only, or
      on the full six-tuple?
    • Rate-limit the hint endpoint to prevent enumeration (an attacker
      probing for matches to known names). Probably 3 attempts per
      hour per IP, with exponential backoff.
    • Which fields are eligible for hinting — name only (highest-value
      for legitimate users, lowest-risk because attacker would also
      need username + icon)? Or all three?
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

- Port plan section 2.3 — original identity decision (v1 model)
- `four_tasks_pair_key_identity_design_notes.md` — the v1 deferred-work doc
  from session 13; useful background but superseded by this doc on
  recovery semantics
- `four-tasks/server/schema.sql` — the concrete data model implementing
  this design
- Roadmap tile 1.3 — server-side defensive writes, where migration
  transactions live
- Roadmap tile 4.14 — theme manager + emoji picker UX
