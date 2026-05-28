# Four Tasks — Rate Limiting Design Notes

**Status:** LOCKED for v1.0 (config pending). Cloudflare edge rate limiting. The 429 status code is the sole exception to the otherwise-locked status set (200/400/404/409/500).

**Implementation:** Cloudflare dashboard config, a Phase 5 prep task. No Worker code — rate limiting runs at the edge, before the router.

**Endpoint surface this doc targets:** the shipped `/users/:user_id/...` API (session 12), not the old `/pair/:key/...` URLs. Pair_key is recovery-only and never appears in a routing URL, so it is **not** a rate-limit key. See `four_tasks_system_map.md` §5 for the live endpoint list.

---

## Why edge, not Worker

Cloudflare edge rules, configured per-route. Not Worker code. Because:

- Blocked requests never reach the Worker — no D1 hit, no CPU on attack traffic. The attack becomes free traffic Cloudflare absorbs.
- No counter store to invent (in-Worker limiting would need D1/KV/Durable Objects + increment/expiry logic — all code that can break).
- Free at our scale, and it's the boring/robust option per the architectural-preference rule.

The clever option (in-Worker token bucket) is reserved for limits edge rules can't express. v1.0 doesn't need it.

---

## The threat being sized against

Write load is ~6 writes per user per day — trivial. Rate limits are **not** load-shedding; they defend against, in order of likelihood:

1. A buggy client retrying in a tight loop (most realistic — our own client gone wrong).
2. A single bad actor with a real account spamming writes (contained — they can only write to their own `user_id`).
3. Enumeration of `POST /resolve` (the only public, pre-identity surface).

Classic DDoS is a non-threat — Cloudflare's platform protection eats it free at this traffic level. The app's UX already throttles writes naturally (4 task ticks + 1 MOTD + 1 rest-toggle per day; picker previews don't write; claim fires once). So limits should catch *runaway* clients (sustained hundreds/min), not annoy real bursts.

---

## Key derivation

Two tiers, keyed by what identifier the URL carries:

- **Identified tier — keyed by `user_id`** from the URL path. Everything under `/users/:user_id/...` plus `GET /users/:user_id`. A buggy or malicious client is locked to one user's budget regardless of IP rotation; other users on the same IP (family wifi, corporate NAT) are unaffected.
- **Unidentified tier — keyed by IP.** `POST /resolve` and `POST /users` — no `user_id` exists in the URL yet (resolve is pre-recovery; create is pre-identity). IP is the only available key.

`GET /health` is unlimited — flat-cost, no DB hop, useful for monitoring.

---

## Per-endpoint limits

Numbers are realistic-peak × headroom. Real per-user write peak is single-digit per *hour*; per-minute caps catch runaway loops within seconds without touching real use.

### Identified tier (keyed by `user_id`)

| Endpoint | Limit |
|---|---|
| `GET /users/:user_id` | 40/min |
| `PUT /users/:user_id/days/:date` | 15/min |
| `PUT /users/:user_id` | 15/min |
| `POST /users/:user_id/claim` | 15/min |
| `POST /users/:user_id/bug_report` | 15/min |
| `POST /users/:user_id/unpair` | 15/min |
| `POST /users/:user_id/dismiss_unpair_notice` | 15/min |
| `POST /users/:user_id/join_by_values` | 15/min |

The writes can share one 15/min budget per `user_id` — simpler to configure than per-endpoint caps, and 15/min is miles above any legitimate pattern. `GET /users/:user_id` gets its own 40/min because polling is the heavy path (30s polling = 2/min; 40/min covers two devices in a pair polling with foreground re-poll churn).

`join_by_values` sits in this tier because the caller is a real solo `user_id`; the enumeration concern is naturally bounded (you must be a created solo user to call it, and ambiguous/no-match just returns an error without leaking which triples exist).

### Unidentified tier (keyed by IP)

| Endpoint | Limit |
|---|---|
| `POST /resolve` | 10/min |
| `POST /users` | 10/min |
| `GET /health` | unlimited |

`POST /resolve` at 10/min lets a confused user retry a recovery typo a few times while blocking enumeration cold. `POST /users` at 10/min covers legitimate onboarding (once per install) with headroom for retries after a network blip.

### Deferred to v1.x

Per-day caps (to catch slow-drip abuse the per-minute layer misses). Not needed for the v1.0 threat model. If added: PUT day ~200/day (real ceiling ~20), bug_report ~20/day (real ceiling ~3).

---

## The 429 exception

Edge-blocked requests return **HTTP 429**, the only allowed exception to the locked status set. Justification:

- The 429 comes from Cloudflare's edge, not the Worker — different layer, different contract. No envelope change.
- The Worker's own response surface stays purely semantic (200/400/404/409/500 — each about request *meaning*). 429 is the one standard code about traffic *shape*, not meaning, so confining it to the edge keeps that separation clean.
- No client cost — 429 is handled generically as "back off and retry."

**Narrow scope:** 429 is allowed *only* for edge rate-limiting responses. The Worker never returns 429. If the Worker ever had to express "too fast" itself, the convention-correct code would be 409 `rate_limited` (state-conflict semantics) — but v1.0 doesn't need it.

Effective status set: 200/400/404/409/500 from the Worker, 429 from the Cloudflare edge only.

---

## Configuration plan (Phase 5)

Verify against the current Cloudflare UI at config time. Rough steps:

1. Dashboard → four-tasks Worker/route → Security → Rate Limiting Rules.
2. Per rule: match expression (URL pattern + method), characteristic (the key — IP, or `user_id` from the URL path), rate (requests/period), action (block → 429).
3. Test each rule by exceeding it from a known IP / `user_id` and confirming 429.

Prefer wrangler-config / IaC over dashboard if available at the time, for repeatability.

Applies to the production Worker and, when it exists, the APPtrioc Worker (its own separate config).

---

## Out of scope

- App-logic constraints (one stamp/day, one reroll/coin-name) — enforced in Worker code, not rate limits.
- In-Worker rate limiting — deferred; revisit only if edge rules prove insufficient.
- Per-rule observability/alerting — Cloudflare default analytics suffice for v1.0.

---

## Cross-references

- `four_tasks_system_map.md` — live endpoint list (§5) and status-code policy.
- `four_tasks_pair_key_design_notes.md` — why pair_key is recovery-only and not a routing/rate key.
- `four_tasks_architectural_preference.md` — edge-not-Worker follows from clarity-over-cleverness.
