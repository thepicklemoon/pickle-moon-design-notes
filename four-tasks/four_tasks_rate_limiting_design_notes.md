# Rate Limiting — Design Notes

**Status:** LOCKED for v1.0 (session 6, partial — config pending). This
document specifies the rate limiting strategy for the Four Tasks API:
threat model, tier structure, per-endpoint limits, response semantics,
and the 429 status code exception to the otherwise-locked status code
list.

**Implementation timing:** Cloudflare configuration lands next home-
office session. No Worker code changes — rate limiting runs at the
edge, intercepting requests before the router dispatches.

**References:**
- `four_tasks_write_rules_design_notes.md` — endpoint surface, write
  rules, the rate-limiting parked item this doc closes.
- `four_tasks_architectural_preference.md` — clarity-over-cleverness
  rule that informs the edge-not-Worker decision.
- `four_tasks_pair_key_v2_design_notes.md` — pair_key derivation,
  which underpins the pair-keyed rate limit tier.

---

## Threat model

Rate limiting on Four Tasks is not a load-shedding concern. The honest
write load per legitimate pair is ~6 writes per user per day. Even at
10,000 users that's 60,000 writes per day spread across 24 hours —
trivial for Cloudflare's infrastructure.

Rate limits exist to defend against three things, in order of
likelihood:

1. **Buggy client retrying in a tight loop.** Most realistic threat.
   Own client gone wrong — a `while` loop pumping PUT day on a stale
   409 because rollback logic broke, an animation loop with a write
   side-effect, etc.
2. **Single bad actor with a paired account griefing.** They have a
   pair-key, they spam writes. Annoying but contained — they can only
   write to their own pair.
3. **Unpaired abuse** of POST /resolve. The only public-attackable
   surface that doesn't require a pair-key. Bug_report is paired-only
   so it's covered by pair-level limits.

**Non-threat:** classic DDoS. Cloudflare's edge eats that for free at
this traffic level; not a design consideration.

**Important property — user-level rate limiting is already inherent
to the UX.** The app's interaction model naturally throttles writes:
- Emoji/icon picker uses preview-on-press + commit-on-exit. The
  fiddle stage doesn't write.
- The morning sequence is unreplicatable per day and animation-bound;
  the claim endpoint fires once at the start.
- PUT day writes are gated to actual task ticks (4 per day max), MOTD
  edits (1 per day), rest day toggles (1 per day).

The realistic peak from a user genuinely *trying* to spam writes
through the app's own UI is single-digit requests per minute. Any
attack that exceeds the rate limits below requires bypassing the
client and calling the API directly. That's the threat we're sizing
against.

---

## Where rate limiting runs

**Cloudflare's edge rate limiting rules**, configured per-route in
the dashboard (or via wrangler config / Terraform once tooling
matures). Not Worker code.

Reasoning:

1. **Architectural preference** says prefer correct/legible/robust
   over clever. Edge rules are the boring, robust option — Cloudflare
   maintains the counter store, the algorithm, the resets, the key
   derivation. Zero code to write, zero code to maintain.
2. **Free at our scale.** Cloudflare's free tier covers the rule
   counts and request volumes we need.
3. **Blocked requests never hit the Worker.** No D1 reads, no D1
   writes, no Worker CPU spent on attack traffic. The attack converts
   from "scaling threat" to "free traffic Cloudflare absorbs."
4. **No counter-store invention.** In-Worker rate limiting would need
   D1/KV/Durable Objects as the counter store, with increment logic,
   expiry, key derivation. Every line of that is code that can break.

The clever option (in-Worker token bucket with D1 counters) is
reserved for cases where edge rules can't express the limit we need.
For Four Tasks v1.0, they can.

---

## Two-tier structure

Rate limits are tiered by URL shape, matching the auth model
(caller-in-URL for paired, IP-implicit for unpaired):

### Paired tier — keyed by pair_key

Anything under `/pair/:key/...`. The pair_key in the URL is the rate
limit key. Both users in the pair share the budget — symmetric with
how the pair shares everything else.

Why pair-keyed rather than IP-keyed: catches a buggy client looping
on PUT day within seconds, without affecting other pairs on the same
IP (corporate networks, family wifi). Also stronger against
determined attackers — an attacker who somehow obtained a pair-key
is locked to that pair's budget regardless of how many IPs they
rotate through.

### Unpaired tier — keyed by IP

POST /resolve only. No pair-key exists yet at this stage of the
flow, so IP is the natural fallback key. Tighter limits because the
realistic legitimate usage is "twice ever per pair" (onboarding +
recovery if it ever happens).

GET /health is unlimited — flat-cost, no DB hop, useful for
monitoring. Not worth rule budget.

---

## The sizing rule

Limits are derived from realistic peak usage with a margin:

- **Pair-keyed limits:** ceiling = (realistic peak per pair) × 1.2
- **IP-keyed limits:** ceiling = (realistic peak per pair) × 2 × 1.2

The ×2 on IP-keyed is for shared-household cases — two pairs on the
same home wifi, or one user across phone + tablet. Doesn't apply to
pair-keyed limits because the pair_key already represents both users
collectively.

The ×1.2 is universal headroom — covers brief retry storms after
network blips, app foreground/background churn, two devices polling
simultaneously within a pair.

Generosity bias: real users hitting catastrophic bugs may legitimately
generate a flurry. Limits should catch *runaway* clients (sustained
hundreds-per-minute), not annoy real usage.

---

## Per-endpoint limits

### Paired tier (keyed by pair_key)

| Endpoint                                        | Limit       |
|-------------------------------------------------|-------------|
| `GET /pair/:key`                                | 40/min      |
| `PUT /pair/:key/users/:target/days/:date`       | 15/min      |
| `PUT /pair/:key/users/:target`                  | 15/min      |
| `POST /pair/:key/bug_report`                    | 15/min      |

The three write endpoints share a 15/min budget per pair. Real-world
peak is single-digit per *hour* — 15/min catches runaway loops within
seconds without ever bothering a real user. Grouping the writes under
one budget is simpler to configure and reason about than three
separate caps.

GET /pair/:key gets its own 40/min because polling is the heavy
endpoint — 30s polling alone is 2/min, and 40/min covers two devices
in a pair polling simultaneously with app foreground re-poll churn.

### Unpaired tier (keyed by IP)

| Endpoint            | Limit    |
|---------------------|----------|
| `POST /resolve`     | 10/min   |
| `GET /health`       | unlimited |

POST /resolve at 10/min lets a confused user retry a typo a few times
during onboarding while blocking enumeration attempts cold.

### Future deferred limits

Per-day caps (separate from per-minute, to catch slow-drip abuse)
are **deferred to v1.x**. Per-minute limits are sufficient for v1.0
threat model. Revisit if real-world data shows slow-drip abuse
patterns the per-minute layer misses.

If added later, sensible starting points:
- PUT day → ~200/day (real ceiling ~20)
- POST bug_report → ~20/day (real ceiling ~3 even for a genuinely
  buggy build)

---

## Response semantics — the 429 exception

Blocked requests at the edge return **HTTP 429 Too Many Requests**
with whatever default body Cloudflare provides.

This is an explicit exception to the status code list locked in
session 2 (200/400/404/409/500 only).

### Why the exception is justified

The session 2 status code list was locked without explicit reasoning
recorded for each exclusion. 429 was on the excluded list, but the
context appears to have been "we don't need this for our endpoint
semantics" — made before rate limiting was on the table as a real
concern.

This is the kind of gap the locked-decisions-vs-design-principles
tension can produce: the lock was under-documented; the principles
weren't fully articulated when the lock was made. The architectural
preference (clarity over cleverness) and the edge-not-Worker decision
arrived together in session 6, surfacing the 429 question.

The cost of allowing 429 is essentially zero:

- No envelope shape change. The response is from Cloudflare's edge,
  not from the Worker — different layer, different contract.
- No client logic cost. Clients can treat 429 as "back off and retry
  later" generically, without endpoint-specific handling.
- No security implication.
- No semantic confusion. 429 is the only status code in the standard
  set that exists for an operational concern (traffic shape) rather
  than a semantic one (request meaning). Allowing it specifically for
  edge rate limiting keeps the Worker's response surface entirely
  semantic — every code the Worker returns is still about meaning.

### The narrow scope of the exception

429 is allowed **only for edge rate limiting responses**. The Worker
itself never returns 429. The Worker's response surface remains
200/400/404/409/500.

If the Worker ever needed to express "you're going too fast" itself
(e.g. for in-Worker logic that can't be expressed as an edge rule),
the right code per the existing convention would be 409
`rate_limited` — state-conflict semantics, "the current state of your
traffic conflicts with my willingness to serve." But v1.0 doesn't
need this.

### Updated status code list

Effective v1.0:

| Code | Origin            | Used for                                    |
|------|-------------------|---------------------------------------------|
| 200  | Worker            | Success                                     |
| 400  | Worker            | Bad request (malformed, validation fail)    |
| 404  | Worker            | Not found / permission denied               |
| 409  | Worker            | State conflict                              |
| 429  | Cloudflare edge   | Rate limited (edge rules only)              |
| 500  | Worker            | Server error                                |

The Worker code itself still uses only 200/400/404/409/500. 429 is
purely a platform-layer response.

---

## What an attack looks like under this scheme

A bad actor attempts to hammer POST /resolve to enumerate pair-keys,
or has somehow obtained a pair-key and tries to hammer PUT day to
corrupt data:

1. Request hits Cloudflare's edge.
2. Cloudflare checks the rate limit rule keyed on IP (for /resolve)
   or pair_key (for paired endpoints).
3. First N requests in the window pass through to the Worker
   normally.
4. Once N is exceeded, every subsequent request returns 429 at the
   edge.
5. The Worker never sees the blocked requests. No D1 reads, no D1
   writes, no CPU spent.
6. The attacker can keep hammering — they're bouncing off Cloudflare.
   Costs don't rise; data is untouched.

The one caveat: a sufficiently determined attacker can rotate IPs
(botnet, residential proxies) to evade IP-keyed limits. That's a
real DDoS-grade attack and falls to Cloudflare's broader platform
protection (free at this traffic level), not these rules. The rate
limit rules defend against the realistic threat — one annoyed
person with a script.

Pair-keyed limits are *stronger* against a determined attacker than
IP limits, because the pair-key itself is hard to obtain (six-value
derivation model). An attacker who somehow holds a pair-key is
locked to that pair's budget no matter how many IPs they rotate.

---

## Configuration plan

Cloudflare dashboard configuration is the next-home-office task.
Rough steps (to be verified against current Cloudflare UI when
actually configuring):

1. Cloudflare Dashboard → the four-tasks Worker / route → Security
   → Rate Limiting Rules (or WAF → Rate Limiting Rules, depending on
   plan).
2. For each rule:
   - Match expression (URL pattern + method)
   - Characteristics (the rate limit key — IP or pair_key from URL
     path)
   - Rate (requests per period)
   - Action (block with 429)
3. Test each rule by exceeding the limit from a known IP / pair_key
   and confirming 429 returns.

Wrangler-config-based rate limiting (if available at the time) is
preferred over dashboard configuration for IaC / repeatability —
revisit at config time.

---

## Closing notes

### What this doc does not cover

- **Stamp/coin economy limits** — those are app-logic constraints
  (one stamp per day, one reroll per coin name, etc.), enforced in
  Worker code, not rate limits.
- **In-Worker rate limiting** — deferred. Not needed for v1.0;
  revisit if edge rules prove insufficient.
- **Per-day caps** — deferred to v1.x as noted above.
- **Per-rule observability / alerting** — deferred; Cloudflare's
  default analytics suffice for v1.0.

### Historical-context flag

This doc resolves the rate-limiting parked item from
`four_tasks_write_rules_design_notes.md` (session 4) and the
ACTIVE-tier todo item from session 5. The 429 exception was made
fresh against current principles because no recorded reasoning
existed for its original exclusion. Future locked decisions should
include the reasoning behind each component so that downstream
re-derivations can compare like-for-like rather than work from
the resulting list alone.
