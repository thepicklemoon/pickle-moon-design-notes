# Four Tasks — Recovery Design Notes

**Status:** LOCKED (session 25, 4 June 2026 AWST) — server contracts and
identity model locked; recovery-screen UI shape flagged OPEN for build time.

**Tile:** 3.10 part B (recovery). Hard pre-store-launch blocker — the store
transition forces a client identity break (package-name and/or
debug→release keystore change), so every existing user (paired *and* solo)
must be able to re-attach their server `user_id` from a fresh install.

**Authority cross-refs:**
- `four_tasks_pair_key_design_notes.md` — three-identifier model, pair-key
  derivation, rotation semantics. This doc inherits all of it.
- `four_tasks_architectural_preference.md` — clarity over cleverness; the
  separate-route decision below is an instance of it.

---

## 1. Why recovery is the way it is

Recovery is deliberately a **two-person job** for paired users. This was
baked into the identity model on purpose: there is no email, no password,
no account server, and therefore **no customer-support surface**. A pair
recovers itself by both partners re-entering the six values they already
know. Nobody at THE PICKLE MOON can (or needs to) reset anyone's account.

This is the load-bearing reason the identity model looks the way it does.
Recovery design does not get to weaken it for convenience.

Solo recovery exists, but it is the **degraded mode** — intentionally thin.
A solo user has no second party to corroborate identity, so solo recovery
is necessarily weaker (single-triple match, theoretical ambiguity accepted).
That weakness is acceptable precisely because it self-heals the moment the
user pairs (see §4). Solo is not gold-plated; pairing is the upgrade path.

---

## 2. Two tiers

| Tier   | Who | Server route        | Match key            | Lookup target            |
|--------|-----|---------------------|----------------------|--------------------------|
| Paired | `pair_id` set | `POST /resolve`        | pair_key (two triples) | `pairs` table            |
| Solo   | `pair_id IS NULL` | `POST /resolve_solo`   | self-triple (one triple) | `users` table, solo only |

Two separate routes, not one route with a `mode` flag. The two flows take
genuinely different inputs (two triples vs one) and hit different tables;
folding them would put a branch inside one endpoint to serve two concerns.
Clarity-over-cleverness: keep them apart.

`/resolve` already exists and is unchanged. `/resolve_solo` is new.

---

## 3. `POST /resolve_solo` contract

**Purpose:** a logged-out solo user re-attaches their `user_id` by
re-entering their own three identity values.

**Request body:**
```json
{
  "self": {
    "name": "Morgan Ryan",
    "username": "nec",
    "active_leader": "chilli"
  }
}
```

Single `self` triple. No `partner` (that is the whole point of solo).

**Validation:** reuse `validateIdentityFields(self, "self")` — same
non-empty / no-`|` / string-type rules as everywhere else. 400 `bad_input`
on failure.

**Lookup:** the solo set is exactly `pair_id IS NULL`. This is the *same
predicate* the partner-finder in `join_by_values` already uses to locate an
available partner — "solo / available to pair" and "solo / recoverable" are
the same population, expressed by the null pair_id. No new flag, no new
column.

```sql
SELECT user_id
FROM users
WHERE name = ?
  AND username = ?
  AND active_leader = ?
  AND pair_id IS NULL
```

(Bind normalised values via `normaliseField`, matching how the row was
stored at onboard.)

**Outcomes** (mirroring `join_by_values` collision philosophy exactly):

| Rows | Status | Envelope |
|------|--------|----------|
| 1    | 200 | `{ok:true, data:{user_id}}` |
| 0    | 404 | `{ok:false, error:"user_not_found", message:"No solo user matches those values."}` |
| >1   | 409 | `{ok:false, error:"ambiguous_match", message:"Multiple users match those values. ..."}` |

The `>1` case is theoretically possible (two solo users sharing all three
values) but vanishingly rare for the friends-and-family cohort. We accept
it rather than gold-plate. The implicit remedy is the same one
`join_by_values` already gives for partner ambiguity: change a username, or
pair up — at which point recovery becomes robust (§4). Solo is the degraded
mode; do not engineer the ambiguity away.

Return only `{user_id}` — no `pair_id` (solo has none). This differs from
`/resolve`, which returns both. Another reason the routes are separate:
different response shapes.

---

## 4. The self-healing handoff

Solo-recoverable and pair-recoverable are mutually exclusive states keyed on
the same column:

- **Solo** (`pair_id IS NULL`): recoverable via `/resolve_solo`, single-triple,
  weak (ambiguity possible). Also visible to the partner-finder.
- **Paired** (`pair_id` set): recoverable via `/resolve`, six-value pair_key,
  robust (collision-checked, two-party). Drops out of both the solo-recovery
  query and the partner-search query.

The transition is automatic and requires no special-casing. The instant a
user pairs, `pair_id` goes non-null:
- they leave the `/resolve_solo` population,
- they leave the `join_by_values` partner-search population,
- they enter the `/resolve` population.

A user's recovery strength upgrades from weak to robust at the pairing
event, for free. This is the structural justification for solo recovery
being thin: it is the entry-level tier that everyone graduates out of by
pairing.

---

## 5. Client surface

The branch *into* recovery already exists and is correct:
- `onboarding.gd` — `PATH_CHOICE` screen emits `recover`; `_on_alt` routes
  it via `_goto_offspine(Step.RECOVERY_STUB)`.
- `RECOVERY_STUB` is an off-spine terminal placeholder. Its current copy
  explicitly states `resolve_pair is NOT called here` — it is a dead-end
  awaiting this tile.

What this tile builds behind the stub:
1. A real recovery screen (replacing the placeholder) that collects identity
   values and dispatches to the correct tier.
2. `Backend` methods for both routes: the existing paired resolve, plus a new
   `resolve_solo(self_triple)` calling `POST /resolve_solo`.
3. On success: write the returned `user_id` to `identity.cfg`, then reboot
   into the calendar via the normal `Identity` boot path
   (`boot_resolved` → `Backend.load_user` → State hydrate).

### Logged-out guard (anti-malicious)

Recovery is only reachable from a **logged-out state** — no active session
on disk. You cannot recover-over a live user. This blocks a casual malicious
attempt to swap the active account out from under whoever is signed in.

This is a pure client/State rule, not a server concern: the recovery branch
is unreachable from `PATH_CHOICE` when `identity.cfg` already holds a valid
user. (Server cannot enforce this — it has no session concept — and does not
need to. The guard is about local device access, which is exactly where the
threat is.)

---

## 6. OPEN — recovery-screen UI shape (decide at build time)

The recovery branch must serve **both** tiers, which means the screen behind
`RECOVERY_STUB` is a small flow, not a single form. Unresolved:

- **Explicit tier choice vs inferred.** Either the user picks
  "I have a partner" / "I'm solo" up front, or the screen collects the self
  triple first and offers "add my partner's details" as an optional second
  step (present → paired resolve; absent → solo resolve). The second is
  fewer decisions for the user but muddier about *which* recovery is
  happening.
- **Reuse of `pair_dialog` input widgets.** The paired path needs the same
  name+username+sticker-PICKER inputs already built for pairing. The solo
  path needs one such triple. Likely the recovery screen composes the same
  input component the pair dialog uses, once or twice.

Per project pattern (design before implementation, but do not pre-decide UI
that wants to be felt on a screen): this is left OPEN deliberately. Resolve
it at the start of the build session, on the device, not here.

The server contracts above (§3) are stable regardless of which UI shape
wins — both tiers are callable independently, so the screen can dispatch to
either without server changes.

---

## 7. Summary of what locks

- Two-tier recovery: paired (`/resolve`, existing) and solo
  (`/resolve_solo`, new). LOCKED.
- Separate routes, not a mode flag. LOCKED.
- Solo lookup = `pair_id IS NULL` self-triple match, reusing the
  partner-finder population. No new schema, no new flag. LOCKED.
- Three outcomes mirroring `join_by_values`: 200 / 404 `user_not_found` /
  409 `ambiguous_match`. LOCKED.
- Self-healing handoff at pairing; solo deliberately thin. LOCKED.
- Logged-out guard on the client recovery branch. LOCKED.
- Recovery-screen UI shape: OPEN, decide at build time (§6).
