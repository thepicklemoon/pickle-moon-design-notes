# Stamp Tier — Design Notes

Last edit: 2026-05-15 21:45 AWST

**Status:** LOCKED for v1.0 (session 6). Structural design closed.
Content authoring (the actual message strings in each pool) is a
separate later task that happens against this spec.

**Implementation timing:** consumed by the morning-payout claim
endpoint (designed in `four_tasks_morning_sequence_design_notes.md`,
implementation pending alongside tile 1.3) and by tile 4.7 (stamp
slap animation). The pool files need to exist before either tile
can ship usefully; stub pools (1-2 messages per tier) are enough
for early implementation testing.

**References:**
- `four_tasks_morning_sequence_design_notes.md` — defines when and
  how stamps appear in the morning sequence, including the rest-day
  variant. This doc only covers the *content* side.
- `four_tasks_timezone_and_sealing_design_notes.md` — sealing model.
  Day sealing happens inside the claim endpoint transaction (lazy
  seal-on-open), not via a nightly cron. Stamp selection happens
  in the same transaction.
- `four_tasks_write_rules_design_notes.md` — `days.stamp` is a
  server-only column; users never write stamps directly. The claim
  endpoint picks the message and writes the column.
- `four_tasks_architectural_preference.md` — clarity-over-cleverness;
  applied here as "simple random pick from server-held pools, no
  anti-repeat tracking, no streak-banded sub-pools."

---

## What a stamp is

When the user claims their morning payout (claim endpoint, designed
in morning sequence doc), the server seals the previous day inside
the claim transaction (lazy seal-on-open per the timezone+sealing
doc — session 6 reframe of tile 1.4 from nightly cron) and picks a
*stamp message* from a tier-appropriate pool, writing it to
`days.stamp` for that date. The message + tier together form the
visual + textual content of the day's "fingerprint" on the calendar.

The tier is derived from the day's completion state:

| Tier   | Trigger                              | Visual colour |
|--------|--------------------------------------|---------------|
| Red    | 1 task completed                     | Red           |
| Orange | 2 tasks completed                    | Orange        |
| Yellow | 3 tasks completed                    | Yellow        |
| Green  | 4 tasks completed                    | Green         |
| Purple | Day marked as a rest day             | Purple        |

A 0-task day does not get a stamp — no message, no colour, the cell
sits visually empty on the calendar. The morning sequence still
plays (see Q2 in morning sequence doc) but no stamp lands.

The stamp message is a short string (typically 1-4 words) drawn from
the tier's pool. It appears in the morning sequence as the stamp
"slaps down," and persists on the calendar cell as part of the day's
permanent fingerprint.

---

## Pool structure

Pools live as **server-side TypeScript constants**, mirroring the
existing MOTD wordlist pattern. Probable file location:
`server/src/stamp_messages.ts` (final filename to match project
conventions at implementation time).

Shape:

```typescript
export const STAMP_MESSAGES = {
  red:    ["...", "...", ...],   // 5-10 entries
  orange: ["...", "...", ...],   // 5-10 entries
  yellow: ["...", "...", ...],   // 5-10 entries
  green:  ["...", "...", ...],   // 40+ entries
  purple: ["...", "...", ...],   // 20 entries
};
```

### Why server-side constants, not a D1 table

The D1 table option would allow live editing without a Worker
redeploy. At solo-dev scale that editability isn't a real benefit —
adding new messages requires a deploy anyway (Workers + clients need
to be in sync if anything ever reads the pool client-side, which is
likely for animation preview purposes). The D1 table just adds
schema complexity for no concrete benefit.

Edit the file, deploy, done. Matches the MOTD wordlist pattern
already established.

### Why not mirror pools to the client

`days.stamp` stores the *selected message string* directly, not a
pool index. The client receives the chosen message in the claim
endpoint response and in subsequent GET /pair/:key payloads. The
client never needs to know what *other* messages were in the pool.

This avoids the two-sources-of-truth problem flagged in the write
rules doc (re. MOTD wordlist mirroring). Stamp pools live only on
the server.

---

## Pool size targets

These are authoring targets, not hard rules. Hit these as content
is written; ship with fewer if needed for early launch.

| Tier   | Target size |
|--------|-------------|
| Red    | 5-10        |
| Orange | 5-10        |
| Yellow | 5-10        |
| Green  | 40+         |
| Purple | 20          |

### Why the size asymmetry

Pool sizes scale with *how often the tier is seen*, not with
authoring effort budget.

- **Green** is the target tier — a user who's doing well sees a
  green stamp most days. Visible repetition would be tiresome. 40+
  messages gives variety even for users with long green streaks.
- **Purple** (rest day) is seen at the user's chosen cadence —
  occasional but recurring. 20 messages is enough variety for
  realistic rest-day frequency.
- **Red / Orange / Yellow** are the "off day" tiers. A user
  seeing a red stamp twice in a year would experience the repeat
  as part of the rhythm, not as a bug. 5-10 messages is enough.

### Why no anti-repeat logic

Pure random pick from the pool. No tracking of last-N-shown, no
server-side single-retry filter, no per-user message history.

The architectural preference doc applies here directly: simple
random is correct, legible, and robust. Anti-repeat tracking adds
schema or query complexity for an aesthetic benefit that doesn't
matter at the pool sizes specified. With 40 green messages, the
math says repeats are rare; with 20 purple messages, repeats are
still infrequent at realistic rest-day cadence.

If real-world usage shows the lack of anti-repeat actually annoys
users, anti-repeat is a backward-compatible change to add later.
v1.0 ships simple random.

---

## Tone targets

The structural rules above don't tell the author what the messages
should *feel* like. These are the authoring targets:

### Red (1 task)

Gentle, no shame, slightly self-deprecating. The user showed up,
ticked one box, and we're acknowledging that without making them
feel worse than they already might.

Examples of the tone (not actual pool content):
- "ouch"
- "rough one"
- "we'll get 'em next time"
- "showed up at least"

Avoid: anything that reads as scolding, sarcastic, or
performatively-encouraging in a way that lands as fake.

### Orange (2 tasks)

Neutral, "got through it" energy. Halfway is halfway; we're not
celebrating, we're not commiserating, we're noting it.

Examples of the tone:
- "decent enough"
- "more than nothing"
- "you showed up"
- "alright"

Avoid: anything that frames orange as failure (it isn't) or as
success (it isn't quite). The honest middle.

### Yellow (3 tasks)

Light affirmation. The user got most of it done. Worth a nod.

Examples of the tone:
- "nice"
- "solid effort"
- "almost there"
- "good day"

Avoid: anything that emphasises the missing fourth task (no "so
close" or "next time get all four"). The yellow stamp is its own
result, not a near-miss of green.

### Green (4 tasks)

Celebration, but not over-the-top. The tone the existing MOTD
wordlists hit — playful, varied, slightly absurd, recognisably
"the app voice." Variety needs to be high because this is the
most-frequently-seen tier; messages should feel like discoveries
rather than templates.

This is the tier where authoring intent gets to flex. 40+ messages
with real variety is a content-writing task in its own right.

Avoid: anything that feels like the same message written 40 times
with different words. Each entry should have its own small
character.

### Purple (rest day)

Affirmation of the *choice to rest*. Permission-giving rather than
congratulatory. The user actively chose to mark this day as rest;
the stamp acknowledges the choice as valid and complete.

Examples of the tone:
- "well rested"
- "took the day"
- "good call"
- "earned it"

Avoid: anything that reads as evaluating the rest day ("good
choice!" lands as approval-seeking; "earned" without context lands
as moralising). Lean toward neutral acknowledgement of the
decision.

---

## What gets locked vs what stays open

### Locked (this doc)

- Tier-to-task-count mapping.
- Pool structure (server-side TypeScript constants).
- Random-pick selection mechanism.
- Pool size targets per tier.
- Tone targets per tier.

### Deferred — not blocking

**Streak-aware stamping.** Considered and explicitly deferred. The
concept: tier messages that read better at low streaks vs high
streaks. Three structural options were on the table:

1. Single pool per tier (this doc — current decision).
2. Streak-banded sub-pools (e.g. green-early / green-mid /
   green-long). Triples authoring cost. Adds routing logic.
3. Single pool, but author messages with streak-context awareness
   so the random mix naturally surfaces context-appropriate
   messages over time.

Option 3 was the recommended middle ground, but on review the
"adds subtle flavour" benefit didn't justify even the soft
authoring overhead. Locked decision: single pool per tier with no
streak awareness in authoring intent.

If real-world usage at scale shows users would benefit from
streak-aware messaging, revisit then. Backward-compatible — pool
authoring can move toward option 3 without any schema or code
change; option 2 needs schema or constant-structure changes.

**Anti-repeat tracking.** Also deferred. Simple random is enough
at the specified pool sizes. Revisit if usage shows real annoyance.

**Localisation / translation of pools.** Out of v1.0 scope. The
pool structure (one array per tier) is the simplest possible
shape, which makes future localisation straightforward — swap the
constants module for a locale-aware version that returns the same
shape. Not a v1.0 concern.

---

## Implementation notes

When the morning-payout claim endpoint implementation lands:

1. Import `STAMP_MESSAGES` from the pool module.
2. Determine the tier from `days.tasks_done` count and
   `days.rest_day`. Tier mapping is the table at the top of this
   doc.
3. `Math.random()` into the appropriate pool array; pull one
   message string.
4. Write that string to `days.stamp` as part of the atomic claim
   transaction.
5. Include the chosen string + the tier (or its colour) in the
   claim endpoint response so the client can play the stamp slap
   animation against real data.

The tier is implicit from `tasks_done` + `rest_day` and doesn't
need its own column in `days`. The morning sequence animation
(tile 4.7) reads the tier from the same derivation.

### Stub pools for early implementation

Until real content is authored, stub each pool with 1-2 placeholder
strings (e.g. `"[red stamp message]"`). This lets the claim
endpoint and morning sequence animation ship and be tested without
blocking on content authoring. Stub messages get replaced as real
authoring happens; no code changes required.

---

## Closing note

The design here is deliberately small. The work that matters most
isn't structural — it's the 80+ message strings across five tiers
that will define how the daily payoff *feels* over months of use.
This doc locks the scaffolding so that authoring work, when it
happens, fits cleanly without re-litigating the structure.
