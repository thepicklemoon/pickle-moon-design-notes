# Four Tasks — Session 19 Capture & Propagation File

**Status: EPHEMERAL.** This is a temporary capture artifact from the session-19
design/drift interview. It is NOT a permanent design doc. Its job is to hold
every decision from this session in one place so the changes can be propagated
across the real docs later — slowly, at a laptop, one doc at a time, with no
risk of a half-applied edit. **Delete this file once every item below is marked
PROPAGATED.** A capture file that outlives its propagation is just a second
source of truth waiting to drift.

Each item is structured: **Finding → Decision → Target doc(s) → Exact change.**
Propagate by working top to bottom; tick each as it lands.

---

## PROPAGATION PROTOCOL (read before applying anything)

This file was written in a chat session against READ-ONLY copies of the repo.
Nothing here has been applied to the live `pickle-moon-design-notes` repo yet.
Apply it at a LAPTOP, against the live repo, in this order:

1. **Append the session 19 devlog block first** (the one written on mobile but
   not yet committed). Item 7 must be reconciled against it BEFORE writing any
   devlog entry — the existing 19 block may already cover the system-map
   authoring + focus-in poll quirk. Do not duplicate it.
2. **Bring the 19 block back to Claude** (paste or fetch via raw URL) so Item 7
   can be finalised with real context.
3. **Propagate one doc at a time, completely.** Claude outputs each full
   corrected file; commit each before moving to the next. No partial diffs.
   - **System-map items (1, 3, 6, 10) go in as ONE edited system map**, not four
     passes.
   - **Item 8 (`_dict_equal`) is a CODE edit to `State.gd` — apply + TEST on
     device** (watch poll logs go quiet on idle). Do not apply blind.
   - New docs to author: Item 2 (`four_tasks_motd_design_notes.md`).
   - New todo lines: Items 4, 14. Todo note: Item 11.
   - Running-doc edits: Item 13 (marketing notes).
   - Pair-key doc: Item 9. Terminology audit: Item 5.
4. **Session logging:** append a short SESSION 19 block (solo map-authoring) AND
   a SESSION 20 block (this drift audit + alignment interview — 14 items
   captured, alignment locked: Version-A-as-strategy-for-B, scope line =
   current features, art scoping deferred to 4.14b). Two clean blocks, two dates.
5. **Delete this file ONLY when every item is confirmed propagated** — not as you
   go. Until the live repo has the changes, this file is the sole record of the
   session. It comes out at the end, all at once.

Tick items as PROPAGATED in place as you apply them, so a resumed session knows
what's left.

---

## Item 1 — Guard machinery is inert under last-write-wins

**Finding.** `State.gd` carries `_writes_in_flight` and `_pending_snapshot`
plus a per-field merge in `_on_day_written`. Under v1.0's last-write-wins model
this machinery is mostly inert. The three user-controlled fields
(`tasks_done`, `motd`, `rest_day`) are kept-local on a successful write for
THREE DIFFERENT reasons, but the code comment only explains the first:
  - `tasks_done` — safe because last-write-wins (rapid taps, no rollback).
  - `motd` — safe because it's single-writer, write-on-exit (spinner only
    writes when the menu closes; no concurrent writer to race).
  - `rest_day` — safe because it's refundable and not time-critical; no
    in-flight guard needed for correctness.
The `_writes_in_flight` / `_pending_snapshot` scaffolding only does real work
on the FAILURE-rollback path, not on success serialization.

**Decision.** Leave the code as-is for v1.0 (do NOT rip it out remotely; it's a
clean known-safe deletion for a laptop session if ever wanted). Document that
it's inert and why, so it reads as a deliberate decision rather than active
concurrency code. Cost of leaving it is purely future-comprehension tax; this
note neutralises that.

**Target docs.** System map §11 (tech debt). Optionally the comment block in
`State.gd` `_on_day_written` (laptop-side code edit, not now).

**Exact change.** Add a §11 bullet:
> The optimistic-write guarding in `State.gd` (`_writes_in_flight`,
> `_pending_snapshot`, the per-field merge in `_on_day_written`) is inert under
> v1.0's last-write-wins model. On the success path, user-controlled fields are
> always kept local — safe for `tasks_done` (LWW), `motd` (single-writer
> write-on-exit), and `rest_day` (refundable, not time-critical), each for a
> different reason. The guard machinery does real work only on the
> failure-rollback path. It is safe to delete; kept only against a future
> endpoint that genuinely rejects user writes. The `State.gd` comment currently
> explains only the rapid-tap (task) reason; the MOTD and rest_day reasons are
> undocumented in-code.

---

## Item 2 — MOTD design note (NEW DOC)

**Finding.** The MOTD widget is one widget with three trigger contexts, but only
two are documented anywhere. The third (anytime long-press reroll) lives nowhere
on disk, and its cost rule + disclosure timing + write-target were never pinned.

**Decision.** Write a focused MOTD design note. The model:
  - **One widget, three triggers:**
    1. *Onboarding* — full widget handed to the user with effectively infinite
       (cost-suppressed) tutorial coins; reroll until satisfied. Learn-by-play.
       Tutorial coins discarded on transition (per tile 3.6).
    2. *Morning sequence* — yesterday's message is branded onto yesterday's cell
       at seal, then the same widget spins up today's message. Runs as a beat,
       then disappears. First reroll of the sequence is free (per tile 4.5).
    3. *Anytime long-press* — long-press the MOTD subtitle on the panel to
       reopen the same widget outside any flow. Rerolls cost flat ~90-110 coins
       (per tile 4.5, "vent not sink"). This trigger is staggered-disclosure
       TAUGHT, not available/advertised day-one.
  - **Display model: carry-forward-until-seal.** The subtitle shows the current
    message and keeps showing it — same day or next day — UNTIL the morning
    sequence runs, brands the old message onto yesterday's cell, and rotates in
    a fresh one. The message persists forward; morning-seal is what swaps it.
  - **Write target:** a long-press reroll ALWAYS composes TODAY's message,
    regardless of what's currently displayed. Yesterday's branded message is
    sealed at morning payout and is never editable. (Lines up with the
    immutable-past monetisation invariant — consistency check passes.)

**Target docs.** New file: `four_tasks_motd_design_notes.md` (design-notes
`four-tasks/` subfolder). Plus repo index entry. Plus devlog "DOCS LOCKED" when
the session block is written.

**Exact change.** Author the note per the model above. Cross-reference tiles
3.6 / 4.4 / 4.5 / 4.6 and the staggered-disclosure doc. Mark which trigger each
tile builds.

---

## Item 3 — Doc-vs-code drift: panel_heading fallback vs carry-forward

**Finding.** Built `panel_heading.gd` (tile 2.A) renders today's `motd` with a
TWO-ROW FALLBACK lookup ("today empty → show yesterday"). The locked design
(Item 2) is a CARRY-FORWARD value rotated by morning-seal. Same pixels on
screen most of the time; different mechanisms; they diverge at the
pre-morning-sequence window and on gap days.

**Decision.** Current fallback implementation is PROVISIONAL — it looks
identical on screen, so it stays as-is for now and gets reconciled when tile 4.6
(morning sequence) makes the rotation real. The intended display rule is "most
recent message at or before today" (see Item 6 — same walkback SHAPE as claim,
different predicate: `motd != ''`, sealed-or-not). NOT the literal claim query.

**Target docs.** System map §7.7 (panel_heading description) + §11 (tech debt).

**Exact change.**
  - §7.7: correct the description to note the built code uses a two-row fallback
    lookup, but the locked design is carry-forward-until-seal; reconciliation
    deferred to tile 4.6.
  - §11: add tech-debt bullet — "panel_heading subtitle uses a one-row-back
    fallback; intended rule is most-recent-non-empty-motd ≤ today (gap-day
    safe). Reconcile when 4.6 lands."

---

## Item 4 — DevKit mockable clock (NEW TODO)

**Finding.** `State.today_iso()` / `install_iso()` use a hardcoded +8h (AWST,
no DST). Date-dependent bugs (gap-day blank banners, midnight rollover) can't be
reproduced on Morgan's hardware because he can't move the clock. Real IANA
support is the proper fix but is a named PRE-LAUNCH blocker (do not pull
forward — see Item 5 reasoning).

**Decision.** Add a cheap DevKit affordance to MOCK the local clock
(force "pretend it's 11pm" / a different offset) so date-boundary bugs are
reproducible NOW, without building real timezone support. Also gives tile 4.6
(morning sequence) a way to test the rollover beat.

**Target docs.** Todo (new ACTIVE or DevKit item). DevKit design notes if it
warrants a scenario entry.

**Exact change.** Add todo line:
> [ ] DevKit mockable clock — force `today_iso()`/`install_iso()` to a chosen
> date/time so gap-day and midnight-rollover bugs are reproducible without real
> IANA support. Unblocks date-boundary testing for MOTD subtitle + morning
> sequence (4.6).

---

## Item 5 — Terminology: timezone gap is NOT atomicity

**Finding.** In conversation the hardcoded-+8 timezone limitation was referred
to as an "atomicity" issue. It is not. Atomicity = all-or-nothing transactions
(`env.DB.batch()` in claim/rotation — those are fine). The timezone gap is a
date-computation-correctness problem (what calendar date is it for this user,
given IANA zone + DST; Godot 4 has no built-in tz database).

**Decision.** Make sure "atomicity" never gets written into a doc to describe
the timezone gap, or a future reader hunts for a transaction bug that doesn't
exist.

**Target docs.** None to add — this is a guard against a mistaken edit. If the
IANA todo line / timezone-and-sealing doc anywhere conflates the two, fix the
wording to "date-computation correctness."

**Exact change.** Audit the IANA todo entry + `four_tasks_timezone_and_sealing_
design_notes.md` for any "atomicity" misuse re: timezone; correct if present.

---

## Item 6 — Day-state taxonomy (NEW DOC SECTION)

**Finding.** "Empty day" is not one concept — it's per-consumer, and each
feature invents its own inline predicate. They're individually correct today but
nobody wrote down the taxonomy, so every new day-history walker re-derives it
and risks drift. We hit this from three directions this session (claim walkback,
subtitle fallback, day-row-existence invariant).

The states (currently collapsed into ~2 by the schema):
  1. **Never touched** — no row in `days`.
  2. **Row exists, no content** — row created but all-false tasks, empty motd,
     not rest. Indistinguishable from (1) to every query by design (§10.4).
  3. **Row with content, unsealed** — real activity, pre-seal. What claim hunts.
  4. **Row with content, sealed** — immutable history. What the subtitle wants
     to display as the carried-forward message.
  (A 5th will arrive with week mode: an opted-out weekday that doesn't count —
  interacts with streak, currently unresolved in todo.)

Per-consumer predicates (all correct, all currently inline & undocumented):
  - **Calendar `_classify`**: row empty OR absent → UNMARKED. Treats 1+2 same.
  - **Claim walkback**: `sealed_at IS NULL AND (rest_day=1 OR tasks_done !=
    all-false)`. Hunts state 3.
  - **Subtitle (intended)**: most recent `motd != ''`, sealed-or-not. Hunts 3|4.

**Decision.** Document the taxonomy ONCE — four (soon five) states + the
canonical predicate each consumer should use + why. NO schema change, NO code
change (do not add an `is_empty` column, a `status` enum, or create-rows-on-open
— all are the clever-option premature-structure move the meta-rule refuses).
Pre-empts re-derivation across the next three day-walking tiles (4.6, 4.8,
4.D2) and the week-mode/streak interaction.

**Target docs.** System map — new section (e.g. §10.5 or a dedicated "Day-state
taxonomy" block near §10.4). Could alternatively be its own tiny note, but
system map is the natural home since §10.4 already lives there.

**Exact change.** Add the four/five-state table + per-consumer predicate list +
the "do not add schema for this" guard.

---

## Item 7 — Session 19 has no devlog block

**Finding.** System map change log (§13) records two session-19 entries
(2026-05-27): initial draft + §10.4/focus-in-poll additions. The §11 focus-in
note also says "(session 19, observed)." But the devlog stops at session 18.
A substantive session (new architectural doc authored, a behavioural poll quirk
observed/flagged) left no devlog entry — against the devlog discipline.

**Decision.** Session 19 needs either a proper devlog block (system map authored,
focus-in poll quirk observed + flagged in §11) OR, if it was pure doc-authoring,
a deliberate note that it's commit-message-only. Given a new architectural doc
+ an observed bug, it's substantive enough to warrant a real block.

**Target docs.** Devlog (`four_tasks_godot_devlog.txt`).

**Exact change.** Add a SESSION 19 block: system map (`four_tasks_system_map.md`)
authored as a new observational architecture doc; focus-in deferred poll
suspected non-functional in editor (NOTIFICATION_APPLICATION_FOCUS_IN/OUT may
not propagate in Godot editor run window) — flagged §11, verify on real device.
Time-log it. Update running total. Repo index entry for the new system map doc
if not already present.

---

## Item 8 — `_dict_equal` is key-order-sensitive (CODE FIX, laptop-side)

**Finding.** `State._dict_equal` (line ~516) tests dictionary equality via
`JSON.stringify(a) == JSON.stringify(b)`. `JSON.stringify` preserves key
INSERTION order and does not sort, so two dictionaries with identical content
but different key order compare as UNEQUAL. A dictionary has no meaningful key
order — `{name, coins}` and `{coins, name}` are the same data — so comparing
them by an order-dependent method is simply wrong, regardless of whether it
currently shows.

This function gates poll re-renders: every 30s the poll handler uses it to ask
"did this self/partner/pair/day actually change?" A false "not equal" fires a
spurious `*_changed` signal and re-renders listeners for nothing.

Concrete trigger path: a day row created locally via `_new_day_row` has Godot
insertion order (`user_id, date, tasks_done, motd, rest_day, stamp, sealed_at,
day_theme_state`); the server's version of the same row (via `parseDayRow`,
spread of D1 column order) may differ in key order. If a locally-shaped row
survives in `State.days` until a poll compares it against the server's row, the
mismatch fires `day_changed` on EVERY poll forever for that row. Open timing
question (does a local-shape row ever survive across a poll boundary, or is it
always replaced by a server row first?) — Morgan to confirm timing at laptop;
likely "can survive" (tap right after a poll, slow write response, next poll
lands first).

**Why dormant:** Phase 2 cells are flat colours — re-rendering an identical flat
cell is invisible. Cost becomes visible once cells carry sticker art + theme
overlays (Phase 4): wasted redraw every 30s, possible flicker.

**Decision.** Replace `_dict_equal` with an ORDER-INDEPENDENT deep compare
(recursive structural equality; arrays stay order-sensitive — that's correct —
dictionaries become order-insensitive). This is more correct AND more legible
than the JSON trick (the JSON trick reads as clever; deep-equal reads as
obvious), so the meta-rule points toward changing it. Lesson to retain: *a
dictionary has no meaningful order — never compare dictionaries by a method that
depends on order.*

**APPLY AT LAPTOP, NOT REMOTELY.** Bug is dormant/invisible in Phase 2; cost of
waiting is zero; cost of a remote untested edit to a core autoload is a possible
poll-gating regression. When at the machine: Claude outputs the complete
`State.gd` with the deep-equal fix; Morgan runs it, watches poll logs go quiet
on idle (no `day_changed` spam on no-op polls); THEN locked. Fix before Phase 4
cell richness.

**Target docs/code.** `State.gd` (`_dict_equal` rewrite). System map §11 note
that it's fixed once done.

**Exact change.** Replace the JSON-stringify body of `_dict_equal` with a
recursive deep-equal helper (handles nested Dictionary order-independently,
Array order-dependently, scalar by value). Full file produced laptop-side.

---

## Item 9 — Identity-field character policy needs a single home

**Finding.** Character rules for identity fields (name, username,
active_leader) keep surfacing ad-hoc and live nowhere as a stated policy:
  - `|` and `__` are banned server-side in `validateIdentityFields`
    (`index.ts`) — and MUST stay banned: they're the pair-key canonical-string
    separators (`name|username|active_leader__...`). A field containing them
    would make the canonical string ambiguous and could collide two distinct
    identity sets to one hash. This is hash integrity, not cosmetics.
  - `/` and `\` discussed this session — harmless today (user-controlled values
    live in JSON bodies, never in URL paths; paths only carry server-generated
    UUIDs + validated dates), but nothing of value is lost by banning them.
  - `-` (hyphen) and single `_` proposed for banning, then REJECTED: hyphens are
    valid in real names (Jean-Luc, Mary-Kate) and `name` is immutable, so a ban
    rejects real people at onboarding for zero safety gain. Single `_` is common
    in usernames and only the DOUBLE `__` is dangerous. Banning these trades
    real usability for no additional safety — the inverse of the right trade.

Correction logged for the threat model: tightening username validation does NOT
"close a routing risk" — there was never a live routing hole (no user value
reaches a path). Ban `/`/`\` for tidiness if desired, not for routing safety.

**Decision.** Don't ban `-` or single `_`. Keep `|` and `__` banned. `/` and
`\` optional-ban (cheap, safe). The real action is to give the character policy
a SINGLE documented home with its reasoning, so it stops being re-derived each
time it comes up (same disease as Item 6's day-state taxonomy). Open question to
resolve when written: deny-list (current) vs allow-list mindset
("letters, numbers, spaces, common name punctuation: hyphen, apostrophe, single
underscore, period"). Allow-list fails closed and is cleaner, but has its own
edge cases (Unicode/non-Latin names, apostrophes like O'Brien) and touches
onboarding + pair-key spec + validation in three places — i.e. it has
architectural shadow and deserves its own small decision, not an inline fix.

**Target docs.** `four_tasks_pair_key_design_notes.md` (Section 12 canonical
string already specifies separators + validation — extend it to state the full
character policy + reasoning). Onboarding doc cross-ref. Possibly a one-liner in
project conventions.

**Exact change.** Document in pair-key Section 12: the banned set (`|`, `__`,
optionally `/` `\`) WITH the hash-integrity reason; the explicitly-ALLOWED set
(`-`, single `_`, apostrophe, period, spaces); and flag the deny-vs-allow-list
question as open. Note that current enforcement is server-side in
`validateIdentityFields`; onboarding must enforce the same client-side (UX
caps already noted in todo Phase 3 validation rules).

---

## Item 10 — `index.ts` router conventions (note only) + cosmetic dupe

**Finding.** The Worker router is ordered + prefix-matched: each branch tests
"path starts with `/users/` and ends with `/<suffix>`". Works today because all
suffixes are distinct and none is a prefix of another, and because no
user-controlled value ever lands in a path segment (paths carry only
server-generated UUIDs + validated dates). The `/days/:date` PUT is matched
before the bare `PUT /users/:user_id`; even if it weren't, the bare branch would
compute an invalid user_id and fail safe with 400. No bug.

Also: duplicated comment block in `index.ts` (~lines 167-173, "Endpoint: POST
/users" header appears twice). Pure cosmetic litter.

**Decision.** No code fix needed for routing. Capture the convention so a future
route addition doesn't unknowingly break the ordering/prefix assumption. Fix the
dupe comment on next laptop touch of the file (commit-message-only, no devlog).

**Target docs.** System map §5 (endpoints).

**Exact change.** Add a §5 note: "The router is order-dependent and
prefix-matched; suffixes must stay mutually non-prefixing, and the router
assumes no path segment contains a user-controlled value (paths carry only
server-generated UUIDs and validated dates). Adding a route with a colliding
suffix, or routing on a user-supplied value, breaks these assumptions."

---

## Item 11 — 4.14b: build the picker to accept placeholder art

**Finding.** The launch art-volume number (how many stickers/elements for the
shop to "feel alive") can't be pinned until the theme system (4.14b) is real and
art can be seen in situ on a device. Trying to scope it earlier is the source of
the painting anxiety — it's a Phase 4 decision being carried in Phase 2.

Also corrected a sizing error in Morgan's mental model worth recording: the
feature-catalogue model means a sticker ships whatever subset of slots its art
fills — sparsity is a design axis, a one-element sticker is valid and complete.
The "~2 tweakable things per sticker" is a CEILING some reach, not a FLOOR owed
to every sticker. So "40-50 stickers" does NOT mean "100+ elements" — it means
mostly-sparse stickers + a handful rich. And not all 40-50 are launch scope: a
chunk are ammunition for the post-launch content-drop cadence (which Morgan
wants stockpiled anyway). Launch needs "shop feels alive on first open," not the
whole catalogue.

**Decision.** Park the art-volume scoping until 4.14b exists. When building
4.14b, make the picker populatable with PLACEHOLDER art (even ugly boxes) so the
"how many to feel alive" number can be felt out fast with throwaway assets
before committing real painting hours.

**Target docs.** Todo — note on tile 4.14b. (Not a doc; a build consideration.)

**Exact change.** Add to tile 4.14b description: "Build the picker so it can be
populated with placeholder/throwaway art, to feel out the minimum sticker count
that reads as 'alive' before committing real painting hours. Launch art scope is
deliberately deferred to this tile — do not attempt to size it earlier."

---

## Item 12 — Launch video is the conversion aperture (Morgan-owned)

**Finding/status.** The launch video (tile 5b.10) is the single load-bearing
distribution event — every number past the few hundred the subreddit might give
routes through it converting YouTube viewers. NOT a single point of failure on
atrioc's goodwill (Morgan's read: community-goodwill creator, low bar that he'll
"go along with the joke for content"; the bet is exposure to his viewership, not
his sustained adoption — a robust bet). The real hinge is the video landing well
with a quality-filtering audience.

NOT a flagged risk to action — Morgan owns this. He has an internal doc, is
YouTube-literate (knows the style/format), has been weighing the narrative for
months (mines-or-not, APPtrioc-first-or-Four-Tasks-first, memes-as-gateway vs
laugh-past-to-the-real-thing), and rates his own delivery/charisma. No Claude
doc needed. Logged here only so it's not mistaken for unscoped later.

**Open creative questions Morgan is holding (his to resolve):** which product
the video leads with and how it hands off; whether the desired outcome is mass
APPtrioc meme-adoption as a gateway or laugh-past-to-Four-Tasks; the cold-open
hook; length.

**Decision.** No action for Claude. Leave the 20h edit budget + production line
in tile 5b.10 as-is. If Morgan ever wants a second pair of eyes on the
narrative/structure pre-5b.10, that's an open invitation, not a task.

---

## Item 13 — Post-launch operating model (hub + one channel)

**Finding.** The follow-through plan (content drops + subreddit + Discord +
localisation + street teams) is a second full-time job that starts at launch and
never ends, lands on a FIFO roster (week on/week off), competes for the same
hours as the partner the whole venture is meant to protect, and arrives just as
build-novelty dopamine drains. It is the least-designed, highest-personal-cost
part of the plan. Running channels BADLY is worse than not running them — a dead
Discord signals abandonment to the exact people you're converting.

Morgan's own answer surfaced the model: the thing he's MOST excited about is the
cosy thepicklemoon.com landing page (devlogs, community, warm place to land,
in-app link destination). Excitement is the only fuel that survives a roster, so
lead with that.

**Decision (operating model):**
  - **thepicklemoon.com = the HUB.** Owned (not rented land — survives Reddit/
    Discord deplatforming/algorithm), permanent, cosy, devlogs, in-app link
    target. Lowest marginal effort: domain registered, Cloudflare Pages known
    from APPtrioc plan, devlog content already written in public design-notes
    repo. The channel Morgan will actually tend because he wants to.
  - **ONE conversation channel at launch, not two.** Tension: Morgan LIKES
    Discord most; strategic read favours Reddit (better discovery, APPtrioc post
    lives there anyway, ASYNC so it forgives a roster — Discord demands
    real-time presence a week-on-week-off schedule can't give; founder vanishing
    fortnightly reads as abandoned). Lean Reddit at launch, Discord DEFERRED
    until community volunteers or proven traction can keep it warm. Decide
    consciously; do not drift into running both.
  - **Content buffer must survive the WORST realistic month**, not hit zero.
    "Skip-this-fortnight comms acceptable if buffer hits zero" (todo) is right in
    spirit but a buffer that reaches zero was too shallow. Paint the stockpile
    pre-launch while build-energy exists.
  - **First hire may need to come before the 10k rung.** The load from 3k→10k
    may exceed one rostered person without wheels coming off at home. Case for a
    part-time community manager funded by 3k-quit income, to make scaling
    survivable rather than as a reward for scale. (Ladder numbers are
    aspirational, not gospel — Morgan clarified.)

**Target docs.** `four_tasks_marketing_notes.md` (running doc — natural home for
the operating model + channel decision + buffer-depth principle).

**Exact change.** Add an operating-model section to marketing notes capturing
hub-and-spoke (site hub, Reddit primary async channel, Discord deferred),
buffer-survives-worst-month principle, and the earlier-hire consideration.

---

## Item 14 — Referral / buddyware revenue-share has commercial design shadow

**Finding.** Morgan's growth engine: offer adopters/referrers ~50% of a referred
subscriber's fees for a year, tracked, to weaponise word-of-mouth (no ad
budget). Strategically strong — turns every happy user into an incentivised
distributor, right approach for the launch model. BUT it is a referral-
attribution + revenue-SHARE system: real schema, real money movement, touches
the existing promo_codes/redemption_attempts tables and the
subscription/RevenueCat plumbing. Paying real people is the kind of thing where
getting money-movement wrong is worse than a wrong sticker.

**Decision.** Treat as a Phase-5 commercial-infrastructure piece needing its OWN
design pass before build — same care the pair-key got — not an ad-hoc bolt-on.
Open questions a design note must answer: attribution mechanism (how a referral
is tracked to a referrer — code? link? both?), payout mechanism + cadence + tax
implications (Morgan is a sole trader — paying out revenue share has accounting/
ledger consequences), fraud/self-referral defence, what counts as a qualifying
subscription, the 1-year window's start/stop, interaction with founders rate +
promo trials.

**Target docs.** New (future): `four_tasks_referral_design_notes.md` — NOT YET
WRITTEN, flagged for Phase 5 prep. Add a placeholder line in todo + repo index
pending-docs.

**Exact change.** Add todo line under Phase 5 prep / commercial: "[ ] Referral
revenue-share (buddyware) design pass — attribution, payout mechanism + cadence
+ tax/ledger consequences, fraud defence, qualifying-sub definition, interaction
with founders rate + promo trials. Moves real money — needs own design note
before build."

---

## Items still pending — interview not finished

The code/drift audit is now complete (Items 1-10). Remaining:
  - The broader ALIGNMENT interview Morgan asked for at the top — "what does
    success look like, what do you want from the next tiles" — has NOT started.
    This is the next thing to do.
