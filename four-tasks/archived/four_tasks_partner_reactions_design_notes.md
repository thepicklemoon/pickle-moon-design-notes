# Partner Reactions — Design Notes

Last edit: 2026-05-21 AWST (deferred header added)

**Status:** DEFERRED v1.x at session 12. Eliminated cross-user write
complexity from tile 1.3. Schema columns retained in v1.0 schema
(four columns on `days` + one on `users`) so the feature can ship
in a future point release without a migration. The picker UI,
endpoint surface, and immutability gate are all designed but not
implemented. Re-evaluate at the start of v1.1 planning.

Original status line preserved below for reference; the rest of
the document is the locked design as of session 7.

---

**Status (original):** LOCKED for v1.x. Architecture baked into schema and tile 1.3
defensive write rules from session 2 so the feature can ship later
without retrofitting. Surfacing model superseded session 5 (now time-
agnostic visual indicators, not push). Cross-ref to renamed pair-key
doc refreshed session 7.

**Implementation timing:** post-v1.0. Schema reserves the columns;
endpoints (tile 1.2/1.3) handle reads; UI build is v1.x.

---

## The idea

A pair of users using Four Tasks should be able to *react* to each
other's days — small expressive markers dropped on what your partner
wrote or how their day went. Hearts on a good MOTD, an angry-but-loving
face on a 1-task slog day, celebration on a green four-tasker.

This is the first feature where one user legitimately writes data
attached to the other user's row. Every other write so far has been
"user writes to their own slot." Reactions break that symmetry and
introduce **field-level write permissions** as a concept.

Capturing the architecture now (session 2) means tile 1.3 (defensive
write rules) can be built with field-level rules in mind from day one
rather than retrofitted after launch.

## Locked product decisions

**Two reaction targets per day:**
- **MOTD reaction** — reacting to the *content* of partner's diary text.
- **Task tray reaction** — reacting to the *performance* of partner's day.

These serve different emotional functions:
- MOTD reaction = reacting to what they said.
- Tray reaction = reacting to how the day went.

iMessage doesn't separate these; pairing them in Four Tasks is part of
what makes the app's emotional vocabulary richer than a generic chat app.

**Post-seal only, immutable once placed:**
- Reactions cannot be dropped on a live day. Picker is hidden / disabled
  on the current day's UI.
- Once a day is sealed (lazy seal-on-open during the claim endpoint —
  see timezone doc), reactions become possible. Partner browses back
  via past-day-tray and drops reactions on sealed days.
- Each reaction column accepts exactly ONE write per day per target.
  Server enforces: if `motd_reaction` is non-NULL, reject further writes
  to that column with 409 `reaction_already_set`. Same for
  `tray_reaction`.
- This means the partner gets ONE shot per target per day. Two reactions
  total per day (MOTD + tray), both permanent once placed.

**Picker UX with confirmation gate:**
- Tap MOTD area (or tray area) on partner's sealed day → reaction picker opens
- Long-press / hover emojis to **preview locally** (no server write, local UI tint only)
- Tap one to select → preview locks in
- Tap confirm or close picker → **confirmation popup interrupts**:
    "react with ❤️ on this day? you can't change this later"
    [Yes] [No] [ ] don't show this again
- "Yes" → single server write, reaction permanent
- "No" → return to picker
- Each target (MOTD, tray) shows its own popup first time
- "Don't show again" — single dismissal kills the popup for BOTH MOTD and
  tray reactions across all future days. Harsh but low blast radius —
  the lesson learned on one reaction applies to all of them.

The popup is client-side user education. The server is the authoritative
gate (it'll reject the second write to a non-NULL column regardless of
whether the popup fired).

**Note on UX pattern inheritance:**
The reaction picker follows a *commit-on-close* pattern (preview locally,
write on confirmation). This matches the identity picker pattern from
the pair-key v2 doc (commit-on-close for migration-triggering changes)
but is DISTINCT from the theme system's per-slot context menu (per the
session 7 theme doc rewrite), which applies slot changes INSTANTLY with
no commit step. The distinction is intentional: identity changes and
reactions are both immutable/expensive operations that warrant a
deliberate commit; theme slot changes are reversible and cheap, so they
benefit from instant feedback to encourage exploration.

**Surfacing reactions to the receiver:**

LOCKED SESSION 5 (supersedes earlier push-based draft):
Reactions are TIME-AGNOSTIC and surface via benign visual
indicators on past-day cells, in BOTH directions. No push
notifications, no animation, no urgent badges.

  Direction 1 — your view of partner's panel:
    Partner's sealed days that you have NOT YET reacted to get
    a small ambient marker. Soft, not urgent. (Visual style
    TBD at tile 4.16 implementation — a small lit corner,
    subtle glow, tiny dot in soft colour. NOT a red dot, NOT
    a notification-spam "1" badge.) Marker means "you can do
    something here if you want." Marker clears when you react
    or explicitly skip.

  Direction 2 — your view of your own panel (reactions you've
    received from partner):
    Your past-day cells that have RECEIVED a reaction get a
    marker drawing your eye to them. Marker clears after
    you've viewed it. The reaction itself stays visually on
    the day forever.

Both directions use the same indicator pattern. Symmetric
vocabulary. Neither direction gets ceremonial moment treatment.

Possible dual-indicator state: a cell can simultaneously have
"you haven't reacted yet" AND "partner reacted to this." UI
needs to read cleanly when both are present. Detail for tile
4.16 implementation.

WHY THIS MODEL (and not push):
  Reactions don't have a 24-hour engagement window. Partner can
  react now, in an hour, in three days, or never. The feature is
  a backlog, not a stream. Push notifications would force the
  wrong tempo onto a feature that's deliberately relaxed. The
  benign indicator matches the actual rhythm — you see it when
  you happen to look at the partner panel, you engage if you
  want to, you don't if you don't.

DISCOVERY:
  Tutorial reveal at day 4 (preferred) or day 6 (backup) — see
  four_tasks_staggered_disclosure_design_notes.md and
  four_tasks_morning_sequence_design_notes.md "Partner reaction
  surfacing (Q7)" section for the full rationale.

The morning_sequence doc captures this in more depth — when
reading either doc, the surfacing model is the same.

**The day's two phases:**
- **Live phase**: each user lives their own day. Tick tasks, write MOTD,
  set rest. Self-focused. No noise from partner.
- **Post-seal phase**: day becomes history for both. Reactions are the
  language of "I saw your day, here's how I felt about it."

The temporal separation is part of what makes the gesture meaningful —
you're appreciating something complete, not interrupting something in progress.

## Picker UX

See "Picker UX with confirmation gate" above under "Locked product
decisions" for the full flow. Key points:

- Mirrors the identity-picker pattern from
  `four_tasks_pair_key_design_notes.md` for the preview-on-press /
  commit-on-exit shape. (NOT to be confused with the theme system's
  per-slot context menu, which is instant-commit — see above.)
- Adds a one-time-per-user confirmation popup ("you can't change this
  later") because reactions are immutable and the user needs to know
  before locking in.
- Popup is dismissible forever via a "don't show again" checkbox.
- Once dismissed, the picker write goes straight through.

## Schema (locked)

**Four columns on `days`** (the reactions themselves):

```sql
motd_reaction          TEXT,        -- emoji char, NULL = no reaction
motd_reaction_at       INTEGER,     -- unix ts of placement
tray_reaction          TEXT,        -- emoji char, NULL = no reaction
tray_reaction_at       INTEGER      -- unix ts of placement
```

All four nullable. All four are PARTNER-writable, OWNER-readable.
Each column accepts exactly ONE write — server rejects subsequent writes
if the column is already non-NULL.

**One column on `users`** (the popup-dismissal preference):

```sql
reaction_confirm_dismissed INTEGER NOT NULL DEFAULT 0
```

Owner-writable (each user manages their own preference). Not a migration
trigger. 0 = show the confirmation popup, 1 = user has dismissed it
forever.

## Field-level write rules (impacts tile 1.3)

The defensive write rules for `days` get a layer of field-level
granularity, additionally gated by sealed-state. See
`four_tasks_write_rules_design_notes.md` for the full table; the
reactions-specific rules are summarised here.

### Live day (sealed_at IS NULL)

| Field            | Owner can write | Partner can write |
|------------------|-----------------|-------------------|
| tasks_done       | yes             | no                |
| diary            | yes             | no                |
| stamp            | server-only     | no                |
| rest_day         | yes             | no                |
| sealed_at        | server-only     | no                |
| day_theme_state  | server-only     | no                |
| motd_reaction    | no              | **no (sealed-only)** |
| motd_reaction_at | no              | **no (sealed-only)** |
| tray_reaction    | no              | **no (sealed-only)** |
| tray_reaction_at | no              | **no (sealed-only)** |

### Sealed day (sealed_at IS NOT NULL)

| Field            | Owner can write | Partner can write             |
|------------------|-----------------|-------------------------------|
| tasks_done       | no              | no                            |
| diary            | no              | no                            |
| stamp            | no              | no                            |
| rest_day         | no              | no                            |
| sealed_at        | no              | no                            |
| day_theme_state  | no              | no                            |
| motd_reaction    | no              | **yes, only if currently NULL** |
| motd_reaction_at | no              | **yes, only if motd_reaction is NULL** |
| tray_reaction    | no              | **yes, only if currently NULL** |
| tray_reaction_at | no              | **yes, only if tray_reaction is NULL** |

Implementation pattern for tile 1.3:
- The PUT endpoint for a day takes a JSON body with only the fields
  being changed.
- Cross-user writes use `?caller=` query param to identify the writer.
  Self-writes have no `?caller` — caller is implicit.
- Server inspects which fields the body is trying to write.
- Server computes:
  - the role of the caller (owner or partner)
  - the sealed state of the day
- For each field in the body: check the lookup table above. Reject the
  *entire* request with 404 if any field is unauthorised (per the auth
  conversation — no 403s, "permission denied" folds into "not found").
- For reaction writes: additionally check that the column is currently
  NULL. Reject with 409 `reaction_already_set` if non-NULL (immutability
  enforcement — 409 because the column IS writable in principle but
  the current state forbids it; 404 reserved for "this combination of
  caller + column isn't a writable thing").
- Once authorised, apply the writes in a single SQL statement.

The server is the authoritative immutability gate. The client-side
confirmation popup is user education on top of that.

### `users.reaction_confirm_dismissed`

Separate writability rule on the users table:
- Owner can write to their own row's `reaction_confirm_dismissed`
- Not a migration trigger (changing it doesn't rewrite pair-key)
- Always 0 (default) or 1 — toggleable in principle but the popup never
  re-appears once dismissed, so the toggle is effectively one-way

## Stamp vs tray reaction — coexistence

The day's `stamp` column and the `tray_reaction` are similar in spirit
(both express "how did this day go") but serve different roles:

- **Stamp** = mechanical, auto-set based on number of tasks completed.
  Visual outcome marker. Owned by the row's user.
- **Tray reaction** = emotional, deliberate, set by the partner.
  Affectionate or sympathetic response to the outcome.

They render together but are not in tension. Stamp says "you did 3 of 4
today"; reaction says "and that's still beautiful." Or "wuss." Up to the
partner.

Tile 4.7 (stamp slap animation) and the not-yet-numbered tray-reaction
tile both render onto the same physical area of the calendar cell.
Layout will need to position them as complementary, not competing.

## Open work (for the v1.x implementation tile)

- New tile in Phase 4 area (call it 4.16): "Reaction picker UI and
  Backend hookup."
- Endpoint signatures (per `four_tasks_write_rules_design_notes.md`):
    - `PUT /pair/:key/users/:target/days/:date?caller=:caller_name`
      where `:target` is the row being reacted *to* (the partner whose
      day is sealed) and `?caller` is the user dropping the reaction.
    - Body: `{"motd_reaction": "❤️"}` or `{"tray_reaction": "🔥"}` or both.
    - Once placed, reactions are IMMUTABLE. The server rejects any
      write to a non-NULL reaction column with 409
      `reaction_already_set`. (The original "setting to null = removing
      the reaction" framing from an early draft is superseded —
      reactions cannot be removed once placed, per the locked
      immutability rule above.)
- Tile 4.16 NOT in v1.0 ship scope. UI work + endpoint, both v1.x.
  Schema columns and write rules ARE in v1.0 (locked in tile 1.3).

## Cross-references

- `four_tasks_pair_key_design_notes.md` — identity picker UX pattern
  this feature inherits (commit-on-close for immutable / migration-
  triggering operations).
- `four_tasks_theme_design_notes.md` — contrast point: the theme
  system's per-slot context menu uses INSTANT commit, not the
  preview-on-press / commit-on-exit pattern. Different feature, different
  UX rules.
- `four_tasks_write_rules_design_notes.md` — full schema + field-level
  rules + reaction endpoint shape lives there.
- `four_tasks_morning_sequence_design_notes.md` — Q7 surfacing
  model fully detailed there.
- `four_tasks_staggered_disclosure_design_notes.md` — day-4 scheduled
  reveal lives there.
- `four_tasks_timezone_and_sealing_design_notes.md` — sealing is lazy-
  on-open during claim endpoint, no nightly cron.
- `four-tasks/server/schema.sql` — concrete columns.
- Roadmap tile 1.3 — field-level defensive writes need to ship with v1
  even though the *feature* is v1.x.
- Future roadmap tile 4.16 (TBD) — reaction picker UI build.