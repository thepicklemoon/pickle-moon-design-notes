# Four Tasks — Completion Seal & Sticker Pool Design Notes

Status: **LOCKED (this session)** for the seal model + pool-feed contract.
Shop/selector UI is **DESIGNED HERE, built incrementally** — the data spine
(this note's §1–§4) is the dependency; the panels (§5–§6) hang off it.

Authority for: the fourth-tick completion seal, the completion-sticker freeze,
the owned-vs-selected pool model, and how a day's permanent sticker is chosen
and rendered. Plugs into the theme system (`four_tasks_theme_design_notes.md`
is the authority for slots, palette, R2 delivery, picker UX — this note does
NOT redesign any of that). Consumes the placement helper already shipped
(`CalendarMath.sticker_placement`, tile 4.FT).

═══════════════════════════════════════════════════════════════
§0 — THE TWO-SEAL MODEL (the crux; everything follows from this)
═══════════════════════════════════════════════════════════════

"Seal" was one event; it is now two. This split is the whole design.

  COMPLETION SEAL — at the FOURTH TICK, tonight.
    The day's CONTENT locks. No further task edits. A sticker is chosen
    from the active pool and FROZEN onto the day. Immutable from this
    instant: survives app-close overnight, survives unpair, survives
    unsubscribe. This is the moment the hero beat fires.

  ACCOUNTING SEAL — at the MORNING CLAIM, next open.
    The day is ACCOUNTED: payout runs, streak walks, stamp lands,
    `sealed_at` is set, `day_theme_state` is frozen. This is the existing
    server seal model, UNCHANGED. The morning ceremony is the second
    felt moment of the day.

WHY SPLIT, NOT COLLAPSE: the server's entire claim/span/rescue machinery
rests on the invariant **today never seals** (index.ts buildSpan: span upper
bound is `date < localDate`, strictly — relaxing to `<=` "would seal today
the instant it opens and then 409 every further task edit"). `sealed_at`
means *accounted*, and accounting is necessarily a backward-looking morning
event (streak/rescue/span only make sense over completed days). Making the
fourth tick set `sealed_at` would require teaching the whole walkback to
distinguish sealed-tonight-by-completion from sealed-by-accounting from
sealed-grey-gap — a rewrite of the most load-bearing, most fragile server
logic. The completion seal is therefore a SEPARATE, LIGHTER mechanism that
locks content + freezes the sticker WITHOUT touching `sealed_at`.

The hero card copy is already correct under this model: the day is *finished*
tonight ("four for four"), but "sealed in the morning — your coins arrive with
tomorrow's open." Finished ≠ accounted.

═══════════════════════════════════════════════════════════════
§1 — SCHEMA
═══════════════════════════════════════════════════════════════

Three schema changes, applied in ONE reset pass (reset-not-migrate regime —
no prod pairing to preserve):

───────────────────────────────────────────────────────────────
§1a — THE STICKERS MASTER CATALOGUE (new table — the backbone)
───────────────────────────────────────────────────────────────

CORRECTION TO EARLIER ASSUMPTION: the buyable catalogue is NOT
`content_drops.payload`. There is a baseline pool of stickers that exist from
launch (~100, Morgan's call on volume), and THAT is the catalogue. Content
drops are the POST-LAUNCH delivery pipe that PUBLISHES NEW ROWS into this
table — they are not the catalogue itself. The catalogue is a master table
the shop, selector, and pick logic all read from.

Until now "a sticker" was an opaque string in JSON arrays (`active_stickers`,
`owned_stickers`) with art bytes in `content_drop_assets` — nothing defined
what sticker ids EXIST, their names, or prices. This table is that source of
truth.

  CREATE TABLE stickers (
    sticker_id       TEXT PRIMARY KEY,          -- the id used in the arrays
    display_name     TEXT NOT NULL,
    price_coins      INTEGER NOT NULL DEFAULT 1, -- 1 for now (pipeline stub,
                                                 -- months from launch; real
                                                 -- prices = economy doc later)
    subscriber_only  INTEGER NOT NULL DEFAULT 0, -- free baseline vs sub-gated
    ships_extra      INTEGER NOT NULL DEFAULT 0, -- 1 = ships more than leader
                                                 -- (the themed-marker hint —
                                                 -- theme doc); refine when art
                                                 -- slot metadata is needed
    status           TEXT NOT NULL DEFAULT 'live', -- 'queued' | 'live'; a drop
                                                   -- seeds 'queued', flips live
    released_at      INTEGER
  );

  - SHOP READS: SELECT * FROM stickers WHERE status='live' AND
    (subscriber_only=0 OR <pair subscribed>) AND sticker_id NOT IN (owned).
    All-minus-owned. Tap a row → cost + buy/preview (UI configured later).
  - SELECTOR READS: names/art for OWNED ids (the menu list).
  - PICK LOGIC: unchanged — it only needs ids from active_stickers; doesn't
    touch this table.
  - DROPS CONNECT HERE: a post-launch 'sticker' content_drop publishes its
    items as new `stickers` rows (queued → live on release). content_drops
    stays the announcement/delivery record; this table is the catalogue. The
    art bytes still live in content_drop_assets (R2) — this table is metadata,
    not pixels.
  - BASELINE SEED: the ~100 launch stickers are seeded rows (status='live',
    price 1 for now). SQL seed file, manually authored, same as launch packs
    were always going to be (schema comment, content_drops).

  ART-ECONOMY NOTE: `ships_extra` is a coarse v1.0 flag, not the full slot
  manifest. The theme doc owns which slots a sticker fills; that detail can be
  a JSON column here later or derived from which `<id>_<slot>.png` files exist
  in the bundle. v1.0 doesn't need the full manifest in SQL — the folder scan
  already discovers slots at load. Keep this table lean; expand only when a
  query genuinely needs slot data.

───────────────────────────────────────────────────────────────
§1b — completion_sticker column (on `days`)
───────────────────────────────────────────────────────────────

  completion_sticker  TEXT NOT NULL DEFAULT ''   -- frozen at 4th tick;
                                                  -- '' = day not completed.

Rationale for a dedicated column (not writing `day_theme_state` early):
  - `day_theme_state` is frozen at the ACCOUNTING seal and is the full
    screen-level slot map. The completion sticker is known one event
    earlier (fourth tick). A dedicated field holds it in the gap between
    completion-tonight and accounting-tomorrow, with no ambiguity about
    when `day_theme_state` is authoritative.
  - It is the immutability anchor: once non-empty, it never changes.
    Mirrors `motd_sticker` (already on `days`, frozen at seal) — same
    shape, same render path (`Stickers.texture_for` / the bundled+R2
    folder scan), generous identifier cap.

───────────────────────────────────────────────────────────────
§1c — owned_stickers column (on `users`) — Q1 RESOLVED
───────────────────────────────────────────────────────────────

  owned_stickers   TEXT NOT NULL DEFAULT '[]'   -- the full OWNED set.

`active_stickers` (existing) becomes strictly the SELECTED subset (the pool),
and must be ⊆ `owned_stickers`. Two arrays on the users row, owned is the
superset, no join. (Decision A from the earlier open question — confirmed:
buying adds to `owned_stickers` AND `active_stickers` (selected by default,
Morgan), so a fresh purchase immediately enters the completion lottery.)
Leader (`active_leader`, own column) is independent — being leader doesn't
force pool membership (§6, default).

reset.sql + schema.sql regime applies — add all three changes, reset, reseed
(including the §1a baseline sticker catalogue seed). No migration.

═══════════════════════════════════════════════════════════════
§2 — WHICH STICKER: the pick contract
═══════════════════════════════════════════════════════════════

The pool can hold many stickers (owned AND selected — see §4). At the
fourth tick exactly one is chosen and frozen. The choice is:

  DERIVED, then FROZEN. The server picks deterministically from the
  CURRENT pool at the moment of the fourth tick, using the same stable
  hash family as the placement helper, then writes the result into
  `completion_sticker`. After that write it never re-derives — the frozen
  id is the record.

  Pick function (server-side, mirrors CalendarMath._stable_hash so client
  and server agree if the client ever needs to preview pre-write):
    index = stable_hash(date) % pool.length
    chosen = pool[index]

  Pool source at pick time = the user's `active_stickers` (the owned AND
  selected set; §4). Empty pool is impossible in practice (floor of 1 —
  onboarding commits leader + companions; selector enforces ≥1 stays in
  pool). Defensive: empty pool → `completion_sticker` stays '' → cell
  renders the stock placeholder, no crash.

WHY DERIVE-THEN-FREEZE rather than pure re-derive (the rejected option):
  Pure re-derivation (hash the date every render, never store) breaks
  immutable-past the instant the pool changes — buy a new pack, and every
  past day's sticker reshuffles because `pool[index]` maps differently.
  Freezing at the fourth tick is what makes "this day wears THIS sticker
  forever" true. Re-derivation is correct ONLY for the placement (where
  the spot is a pure function of the immutable date); the sticker IDENTITY
  must freeze because its input (the pool) is mutable.

NO PRE-SEAL WINDOW: the sticker is locked at the fourth tick, full stop.
There is no "live until morning" mutability for the sticker — Morgan's
call, categorical. Changing your pool after a day completes does NOT
change that day's frozen sticker. (This differs from screen-level theme
slots, which DO stay live until the accounting seal per the theme doc's
completed-month chrome rule — but the per-cell completion sticker is
frozen earlier, at completion, because completion is its freeze event.)

═══════════════════════════════════════════════════════════════
§3 — THE CONTENT LOCK (fourth tick is terminal)
═══════════════════════════════════════════════════════════════

Once a day has four ticks, it is DONE. The task-write path
(`PUT /days/:date`, writeDay) must REJECT further task edits on a day whose
stored `tasks_done` already has four true (or which already carries a
non-empty `completion_sticker`).

  - Rejection code: **409** (conflict — the day is already complete),
    uniform envelope `{ok:false, error:"day_complete", message:...}`.
    409 is the existing "already sealed / immutable" status in this API;
    completion is the same class of immutability, earlier.
  - The lock is server-enforced (source of truth). The client ALSO stops
    offering ticks on a complete day (the tray reflects locked state), but
    the server is the wall — a stale client cannot un-complete a day.
  - REST INTERACTION: a completed day cannot be toggled to rest (rest is a
    not-worked designation; a four-tick day is worked). The rest endpoint
    rejects on a complete day, same 409.
  - WHAT STILL WRITES post-completion: nothing owner-facing on that day.
    The morning accounting seal still processes it (sets sealed_at, stamp,
    promotes the sticker — §4). accounted_for is already 1. The completion
    lock blocks USER edits, not the system's accounting pass.

CURRENT STATE NOTE: task mutability you see in testing today (untick/retick
freely) is TEST-ONLY and does NOT reflect launch. This lock is the launch
behaviour. DevKit scenarios that need to re-test completion reseed the day
unsealed + uncompleted (completion_sticker '' , tasks_done reset) via the
existing scenario reset.

═══════════════════════════════════════════════════════════════
§4 — MORNING PROMOTION + THE OWNED-vs-SELECTED POOL
═══════════════════════════════════════════════════════════════

POOL MODEL (two sets, per the theme doc Layer 1 + Morgan's description):
  - OWNED: every sticker the user has (onboarding starter set + coin
    purchases + partner's owned set while subscribed). Entitlement. NOT a
    single column yet — onboarding writes the starter set into
    `active_stickers`; purchases extend ownership. (Ownership storage detail:
    see §4a — this needs one decision.)
  - SELECTED (the POOL): the subset the user has switched ON in the
    selector — these are the stickers eligible to be slapped on a completed
    day. Stored as `active_stickers` (the theme doc's Layer 1 array).
  - THE FEED: §2's pick reads `active_stickers` = the SELECTED set. Owned
    but unselected stickers are usable as leader / theme slots but do NOT
    enter the completion-sticker lottery.

MORNING PROMOTION: when the accounting seal runs (sealSpan, index.ts
~1601/1615 where `day_theme_state` is written), it folds the day's
`completion_sticker` into the frozen `day_theme_state` as the completion-
sticker entry. After promotion, the permanent render reads the sticker from
the frozen `day_theme_state` (consistent with every other per-day frozen
slot); before promotion (the just-completed, not-yet-accounted day) the
render reads `completion_sticker` directly. Both resolve to the same id —
promotion is a move, not a change.

PARTNER / UNSUB IMMUTABILITY (Morgan, explicit): if the frozen sticker is
one from the PARTNER's pool (you were subscribed, had access) and you later
unsubscribe or unpair — that day KEEPS that sticker forever. The id is
frozen into the row; entitlement gates USE (can you newly select it), not
SIGHT (rendering a frozen id). This is the theme doc's existing
partner-render rule (asset pixels aren't the secret; fetch-by-id covers
assets the viewer doesn't own). No new rule — the frozen id + partner-render
rule already deliver this.

───────────────────────────────────────────────────────────────
§4a — ownership storage: RESOLVED (was open)
───────────────────────────────────────────────────────────────

Decision A taken: `owned_stickers` column holds the owned set; `active_stickers`
is the selected subset (⊆ owned). Both on the users row, no join. See §1c. A
purchases/entitlement table (option B) is NOT used for v1.0 — no per-item
metadata needed; the `stickers` catalogue (§1a) holds price/status, and
ownership is just an id list. Revisit only if purchase history needs auditing
(monetisation-ledger territory, post-launch).

═══════════════════════════════════════════════════════════════
§5 — STICKER SHOP (panel left of calendarYOU)
═══════════════════════════════════════════════════════════════

A third panel in the horizontal swipe: SHOP ← calendarYOU ↔ calendarPARTNER.
(Confirm swipe topology: shop is a new leftmost panel, so the container grows
to 3 panels wide and current_view gains a state. App.gd's two-panel swipe
becomes three-panel — a real change to the swipe state machine, costed at
build.)

PURPOSE: browse buyable stickers, purchase with COINS. On purchase:
  - Server: verify coin balance ≥ price, deduct, add sticker to OWNED set
    (§4a), uniform envelope. 409 if already owned; 400 if unaffordable
    (or a dedicated insufficient_coins code — match the rest-day fee
    precedent in index.ts).
  - Coins are the economy's existing currency (economy doc is the price
    authority — DO NOT invent prices here; the shop reads a price per
    sticker defined server-side).
  - Purchase is server-authoritative; client shows optimistic "owned" with
    rollback on reject, per the house pattern.

UI IS LATER (Morgan): the panel's visual design, sticker card layout, price
display, purchase confirmation affordance — all deferred. This note specifies
the DATA + the buy transaction; stub the panel (a list of buyable ids +
price + a buy button is enough to wire and test the transaction).

PARTNER STICKERS IN SHOP: partner's owned stickers are ACCESSIBLE (usable)
while subscribed but are NOT re-purchased — they appear in the SELECTOR
(§6), not as shop buys. The shop sells stickers neither of you owns.

═══════════════════════════════════════════════════════════════
§6 — STICKER SELECTOR (slide-down menu)
═══════════════════════════════════════════════════════════════

A menu that slides DOWN, containing every sticker you AND your partner own.
Each sticker is:
  - SELECTABLE AS LEADER (sets `active_leader` — pair-key participant,
    triggers rotation per the pair-key design notes; the ONE sticker choice
    with a pair-key consequence).
  - TOGGLEABLE IN/OUT OF POOL (adds/removes from `active_stickers` — the
    selected set §4; no pair-key consequence, instant, cheap, per the theme
    doc's "exploration is the joy / instant feedback no confirmation").

  - POOL FLOOR: at least one sticker must remain in the pool (the lottery
    can't be empty). The selector prevents removing the last one.
  - LEADER IS ALSO POOL-ELIGIBLE: the leader can be in or out of the pool
    independently of being leader (being your identity sticker doesn't force
    it into the completion lottery). Confirm if Morgan wants leader forced
    into pool — default: independent.

UI IS LATER (Morgan): the slide-down animation, the per-sticker row layout,
the leader/pool toggle affordances, the per-sticker CONTEXT MENU (the theme
doc's long-press → slot-assignment, Four Tasks only). STUB the context menu
for now — a sticker row with [set leader] [in pool ✓/✗] is enough to drive
the data and test the feed. The richer slot-customisation UI is tile 4.14b
scope and reads the theme doc.

═══════════════════════════════════════════════════════════════
§7 — CLIENT READ PATH (how the cell gets the real sticker)
═══════════════════════════════════════════════════════════════

CORRECTED (build session): the sticker lives in `days.completion_sticker`
for ALL TIME — one source, read it always. Earlier drafts of this section
had a two-source split (completion_sticker for live days, day_theme_state for
accounted days) requiring a "morning promotion" to copy the sticker across.
That promotion was REDUNDANT and is NOT built: `completion_sticker` is a
permanent column, immutable once the content lock sets it (§3), and the
morning accounting seal (sealSpan) NEVER touches the column — verified by
absence, so a frozen sticker survives seal unchanged. Storing the same fact
in day_theme_state too would duplicate it for no benefit (violates
clarity-over-cleverness). day_theme_state stays purely the screen-level theme
snapshot it already is; the per-cell completion sticker is its own column.

Replaces the hardcoded placeholder star in day_cell.gd / hero_beat.gd
(tile 4.FT) with a real lookup:

  - Read `completion_sticker` off the day row (State.days[user][date]).
    Non-empty → that's the frozen sticker id; resolve to a texture and
    render it (hand-placed via CalendarMath.sticker_placement, unchanged).
  - Empty ('') → day not completed → no sticker (stock cell, as now for
    non-GREEN). Note: completion_sticker non-empty is now the TRUE
    "completed" signal — more precise than the GREEN tier classification,
    which fires at four ticks regardless. The cell can drive the sticker
    off completion_sticker directly.

The id resolves to a texture via the bundled+R2 folder scan (theme doc
runtime delivery: res:// first, then user://stickers/). v1.0 launch packs
are BUNDLED, so the resolve is synchronous off disk — no await. Partner days
/ post-reinstall historical days may reference an id not on disk → degrade to
stock render + retry on next poll (theme doc partner-render rule). The
placeholder star becomes the explicit stock fallback for "completed but no
sticker resolvable."

PLACEMENT IS UNCHANGED: CalendarMath.sticker_placement(date) still gives the
hand-placed spot + tilt; only the TEXTURE source changes from the procedural
star to the resolved pool sticker. The hero beat and the permanent cell both
swap their texture source identically, preserving the seamless hand-off
already verified in 4.FT.

═══════════════════════════════════════════════════════════════
§8 — BUILD ORDER (server-first, per the house rule)
═══════════════════════════════════════════════════════════════

  1. SCHEMA (one reset pass): create `stickers` catalogue (§1a); add
     `completion_sticker` to days (§1b); add `owned_stickers` to users
     (§1c). Seed the baseline ~100 stickers (status='live', price 1).
     Update onboarding seed so the starter set's ids exist as catalogue
     rows AND land in both owned_stickers + active_stickers. reset + reseed.
     [DONE — dev DB, 11 baseline rows verified.]
  2. SERVER — completion seal: writeDay picks + freezes completion_sticker
     at the fourth tick; content-lock rejects further edits (409); rest
     endpoint rejects on complete day.
     [DONE — logic-verified; wire test pending (step 5b). NOTE: the rest-
     endpoint 409 is NOT yet added — see below.]
  3. (CANCELLED) — morning promotion. Was: fold completion_sticker into
     day_theme_state at accounting. REDUNDANT: completion_sticker is a
     permanent column the seal never touches; the sticker already lives in
     its forever home. See §7. No work.
  4. SERVER — shop buy + selector writes: purchase reads `stickers` for
     price, verifies coins, deducts, adds id to owned_stickers AND
     active_stickers (selected by default); selector writes active_leader
     (rotation) + toggles active_stickers (pool). Catalogue read endpoint
     for the shop (all live, minus owned). Wire-test all via PowerShell
     regression pack BEFORE any client.
  5. CLIENT — read path (§7): swap placeholder star for the resolver
     (read completion_sticker). Verify the cell/hero show the real frozen
     sticker.
  6. CLIENT — selector: leader slot top-right of calendarYOU; tap → context
     menu (stub; one option = make-leader+palette); long-press a sticker in
     the owned list → toggle pool (die-cut-corner square = in-pool). Vary
     leader + pool, watch the feed respond. Verify pool changes affect
     FUTURE completions, never past.
  7. CLIENT — shop: THIRD swipe panel (SHOP ← calendarYOU ↔ calendarPARTNER).
     App.gd's two-panel swipe becomes three-panel — container grows to 3-wide,
     current_view gains a third state, commit thresholds get a third target
     (a real swipe-state-machine change, not free — costed here). Populate
     from the catalogue read (all live minus owned). Tap a sticker → cost +
     buy. Preview window DEFERRED (needs the palette/retint pipeline, tile
     4.14b/4.20 — NOT built; the shop ships buy-without-preview until then).
  8. UI polish (panel visuals, slide animations, context-menu icon vocab,
     preview window): LATER, separate tiles.

  OUTSTANDING in step 2: the REST endpoint (setRestDay) must also reject a
  completed day (409) — §3 says a four-tick day can't be toggled to rest. The
  writeDay lock is in; the setRestDay lock is NOT yet written. Add before the
  step-5b wire test.

Each server step wire-tested before the next; each client step
device/editor-verified before ticking. No new client code while the test
ledger has open items (house rule).

═══════════════════════════════════════════════════════════════
§9 — DECISIONS STILL OPEN (gate the build)
═══════════════════════════════════════════════════════════════

  SERVER TILE COMPLETE + WIRE-PROVEN (this session, on dev D1):
    - Schema (catalogue + completion_sticker + owned_stickers), seeded, live.
    - Completion seal: 4th tick freezes a deterministic sticker (verified:
      today → "chilli" from the 4-sticker pool). sealed_at stays null at
      completion — two-seal model confirmed in the wire response.
    - Content locks: writeDay AND setRestDay both 409 "day_complete" on a
      completed day. Verified.
    - Shop: GET /users/:id/shop (catalogue minus owned) + POST .../shop/buy
      (coins → owned+selected, 409 already_owned / insufficient_coins).
      Verified: bought green_glowbug, coins 4250→4249, dropped off buyable
      list, re-buy rejected.
    - Selector writes: NO new endpoints — PUT /users/:id already handles
      active_leader (with pair-key rotation) + active_stickers.
    - parseUserRow now parses owned_stickers (was returning raw string).
    - createUser + both DevKit seeds now populate owned_stickers = the
      starter active_stickers (the starter set is owned, not just selected).

  ONBOARDING ↔ ECOSYSTEM (Morgan's flag — RESOLVED, flow unchanged):
    Onboarding's experience is UNCHANGED (screen 5b still picks leader +
    companions as now). The concern was that granted stickers must live inside
    the catalogue/owned/pool ecosystem, not as free-floating ids. Fixed by the
    createUser owned_stickers write: onboarded starter stickers are now
    first-class catalogue members (owned + selected + shop-aware), identical in
    kind to bought stickers. The only standing requirement is the COUPLING:
    the client CATALOGUE const (onboarding_companions.gd) and the server
    `stickers` table must list the same ids — a sticker grantable at onboarding
    must exist in the catalogue, or it lands outside the ecosystem again. Both
    currently 11, matched. Maintain in lockstep (documented in seed_stickers
    .sql header).

  STILL OPEN:
    Q3 (§6) — leader forced into pool, or independent? DEFAULT TAKEN:
              independent. Flag if Morgan wants leader forced in.
    Q4 (§5) — insufficient-coins: DECIDED — reused 409 insufficient_coins
              (matches rest-day fee precedent). Closed.
    Q6 (NEW, deferred) — onboarding's free allocation vs the paid shop at
              CATALOGUE SCALE. Today onboarding picks ~5 free from the full
              catalogue (fine at 11 stickers). At the planned ~100, "pick 5
              free from 100" makes cheap shop stickers pointless and the
              onboarding screen unwieldy. Likely wants a `starter_eligible`
              flag on `stickers` (onboarding picks from the starter subset;
              everything else is shop-bought) + a fixed small free allocation.
              NOT a v1.0-now problem — defer until catalogue grows. Flow stays
              as-is until then. Pure economy/onboarding design, no blocker on
              the client read path.

Everything else in §0–§8 is LOCKED.