# Coin Name Generator — Design Notes

**Status:** LOCKED architecturally for v1.x. Schema reserved session 2.
Generator implementation + reroll UI ship v1.x — full design landed
before any code so the rule iteration can be the joyful part later.

**Implementation timing:** post-v1.0. Schema is in place; rule table
and generator function build later. Reroll UI is even further out
(coin economy integration).

---

## The idea

Coins are currently flavoured as `[username]-coins` — flat, generic,
forgettable. The replacement is a fun, naming-rule-driven affix system
that *transforms* the username into a coin name. Each pair's coin
economy gets its own invented vocabulary.

Examples (illustrative only — final affix pool to be tuned):
- `lara` → `larinos`
- `tim` → `timmets`
- `morgan` → `morginks`
- `ayshi` → `ayshiles`

The point isn't accuracy or consistency — it's *flavour*. A tiny piece
of personality layered on top of the username that makes the coin
economy feel uniquely the user's.

---

## Locked decisions

### 1. Server-side authoritative

Coin name is stored on the `users` row, server-side. Same logic as
every other identity-adjacent field: server is the rule of truth,
client renders what server says.

Two new columns on `users`:

```sql
coin_name              TEXT    NOT NULL DEFAULT '',  -- derived from username, mutable via reroll
coin_name_reroll_count INTEGER NOT NULL DEFAULT 0   -- how many rerolls user has burned
```

`coin_name` is initially derived at user creation (onboarding step
between username and icon, probably) by running the username through
the generator. Reroll generates a new variant from the same rule table
and overwrites `coin_name`, incrementing `coin_name_reroll_count`.

### 2. Rich rule set, iterable

The rule set lives in `res://data/coin_name_rules.tres` (a Godot
Resource file the generator reads). Server logic mirrors the structure
in TypeScript so the same rules apply server-side at creation/reroll
time.

Wait — actually, the rules belong on the *server* as the authority.
The client gets the result, not the rules. The server holds a JSON
rule table (in the Worker code as a constant, or in D1 as a config
table if it gets big). Client never runs the generator; client just
displays the `coin_name` field it gets back.

This keeps it tamper-proof (client can't generate a custom name) and
matches the architectural-preference rule from session 2 (clarity over
cleverness — single source of truth).

**Rule structure (sketch):**

```typescript
const COIN_NAME_RULES = {
  // Each category maps a phonetic pattern to a pool of possible affixes.
  // Generator picks one at pseudo-random based on username + reroll_count
  // (so reroll is deterministic per-user — same reroll count = same name).
  ends_in_vowel:       ["nos", "ths", "rels", "vies", "lons"],
  ends_in_consonant:   ["mets", "ets", "iks", "inks", "ords"],
  ends_in_y:           ["lings", "drops", "sprigs"],
  ends_in_two_vowels:  ["nights", "tides"],
  // ...etc, easily extended
};
```

The generator function reads username's ending, picks the category,
picks an affix from the category's pool using a deterministic index
based on `(username, reroll_count)`. Deterministic = same inputs always
produce same output, so reroll count N → reroll count N+1 always gives
the user a different name (rather than possibly the same one twice).

**Iterability:** adding a new category or affix is a single edit to
the rule table. No code change beyond that. Joy to extend.

### 3. Reroll mechanism — coin spend, escalating cost

Reroll progression locked to **coin spend with doubling cost**, matching
the MOTD reroll system (web build precedent, ported to tile 4.5):
- First reroll: 50 coins (numbers TBD when coin economy is tuned)
- Each subsequent reroll: doubles
- Stored cost can be checked client-side from `coin_name_reroll_count`,
  but server enforces the deduction

This matches the "you're spending coins to rename your coins" gesture
that makes the feature feel coherent.

### 4. Coin name re-derives on username change

Username is freely mutable (pair-key v2 design). When username changes:
- Server runs the generator on the NEW username, using the SAME
  `coin_name_reroll_count` value
- New `coin_name` overwrites the old one
- Reroll count is preserved (the user's "investment" carries over)

User's experience: "I changed my display name, my coin name changed
with it — but I'm still at reroll-count-3, my next reroll still costs
what it would have."

This keeps the reroll economy fair across username changes without
creating an exploit ("change username to reset reroll cost").

### 5. Visibility — partner does NOT see partner's coin name (initially)

In the prototype, partner coin names were only visible on the
leaderboard if either user made the top 10. Not on the partner calendar
UI. Decision for v1.x: **keep that behaviour. Coin names are personal
flair, not shared display.**

Open question for later: should partner coin names appear in the
partner panel UI? Pros: shared joy, "your partner now has 47 ayshiles".
Cons: privacy intrusion if user picks something personal/meaningful;
partner doesn't *need* to know the name to see the count.

Defer to v1.x product polish. Schema doesn't change either way.

### 6. Offensive output mitigations — three layers

Phonetic rules don't know about slurs. We can't fully prevent all
unfortunate combinations, but we can stack defences.

Three mitigations, ALL retained:
- **Curated affix pool:** only ship affixes that are checked safe
  across common name endings. Boring affixes ≪ offensive accidents.
  Pool curation is itself a content task; the rule table starts
  empty and gets filled with deliberate, vetted entries.
- **Server-side blocklist filter:** before settling on a generated
  name, check against a small list of substrings to avoid. If the
  name hits the blocklist, generator rerolls automatically (NOT
  charging the user). Free internal reroll until clean output.
- **User-facing reroll:** the user always has agency. If the name
  feels off, they can spend coins to reroll. Costs scale, but it's
  always available.

**Leaderboard censorship (extra layer):** if a coin name still slips
through and lands on the leaderboard, server-side censorship masks
parts of the name visible to non-partner users. The partner already
chose to share their habit data with this user; offensive coin names
in that context are a less-severe concern (they have agency to
discuss/intervene). Strangers on the leaderboard get the censored
version.

Implementation order (when this lands):
1. Curated affix pool (content task, ongoing)
2. Server-side blocklist filter (small list + iteration loop)
3. Reroll UI (user agency)
4. Leaderboard censorship layer

### 7. The generator runs server-side, only

Repeating for clarity: the rule table, the affix pool, the deterministic
selection logic — all server-side. Client never sees the rules, never
runs the generator. Client gets `coin_name` as a string and renders it.

Why: prevents tampering, matches the "server is the rule of truth"
pattern, keeps the rule table iteration as a content-only operation
that doesn't require client deploys.

---

## Schema impact (locked, applied via migration_002)

Two columns added to `users`:

```sql
ALTER TABLE users ADD COLUMN coin_name              TEXT    NOT NULL DEFAULT '';
ALTER TABLE users ADD COLUMN coin_name_reroll_count INTEGER NOT NULL DEFAULT 0;
```

Both owner-writable, not migration-triggering (changing them doesn't
rewrite pair-key — they're presentation-layer data on identity-adjacent
fields, not identity itself).

For existing pre-feature users: `coin_name` is empty string on
existing rows, regenerated lazily on next login or on the user's
profile-load endpoint hit (whichever comes first).

---

## Tile implications

Three tiles in Phase 4 (numbering TBD when this lands):

- **Coin name generator (server-side):** rule table + generator function
  + blocklist filter + auto-reroll loop. Hooks into onboarding (set
  initial name) and username-change (re-derive).
- **Reroll UI (client-side):** button in settings, cost display,
  confirmation popup ("spend X coins to reroll your coin name?"),
  server call, render new name.
- **Leaderboard censorship layer:** filter applied at the
  `GET /leaderboard` endpoint for non-self / non-partner viewers.

The first one is the bulk of the work. The second is small UX once the
backend exists. The third is one regex pass on a list output.

---

## Open questions for future-Morgan

- **Initial cost + doubling formula.** 50 coins first, doubling thereafter
  is a placeholder. Tune when the coin economy is tuned more broadly.
- **Partner visibility of partner coin name.** Defer to v1.x product polish.
- **Edge case: very short usernames.** "Ai" → ?. Affix pool needs to
  handle 2-letter names gracefully. Test cases when the rule table is
  being built.
- **Edge case: usernames with non-alphabetic characters.** Numbers,
  underscores, etc. The username-change UX should already constrain
  to alphabetic, but server should fail-safe with a default affix.

## Cross-references

- `four-tasks/server/schema.sql` — concrete columns (after migration 002)
- `four_tasks_staggered_disclosure_design_notes.md` — the principle
  that this feature, like rest-day intro and subscription nudge, is
  not surfaced day 1
- Future tile: coin name generator (server)
- Future tile: reroll UI (client, v1.x)
- Tile 4.5 — MOTD reroll system; same economy pattern
