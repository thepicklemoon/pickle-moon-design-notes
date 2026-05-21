# PICKLE MOON Repo Index

Last edit: 2026-05-21 AWST

Purpose: canonical list of files across THE PICKLE MOON's repos, with one-line descriptions. Reference doc for Claude sessions — "fetch the X doc" resolves to a specific raw URL from this list.

Last updated: 2026-05-21 (session 12 close — tile 1.3 landed, partner reactions deferred, write_rules superseded, schema migration files deleted)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
REPO: pickle-moon-design-notes (PUBLIC)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Raw URL base:
https://raw.githubusercontent.com/thepicklemoon/pickle-moon-design-notes/main/

## Root level

- `pickle_moon_repo_index.md` — this file. The directory.
- `README.md` — repo intro.

## four-tasks/ — Four Tasks Godot port

Raw URL prefix for this subfolder:
https://raw.githubusercontent.com/thepicklemoon/pickle-moon-design-notes/main/four-tasks/

Working documents (high churn):
- `four_tasks_godot_devlog.txt` — active devlog (v2, opened session 9 close). Summary block of sessions 1-9 at top, then sessions 10+. Primary context doc. AI reading guide at top.
- `four_tasks_godot_devlog_v1_sessions_1-9.txt` — archived v1 devlog (sessions 1-9 in full prose). Reference only — for historical reconstruction of how a decision was reached, not for the decision itself. Authority for any locked decision lives in the relevant design doc + v2 summary block.
- `four_tasks_godot_todo.txt` — active tile list, phases, parked items.

Locked design docs (architecture / meta-rules):
- `four_tasks_architectural_preference.md` — meta-rule: clarity over cleverness for low-throughput load profile. Examples + when the clever option is permitted.
- `four_tasks_staggered_disclosure_design_notes.md` — meta-principle: features reveal over days/weeks, not at onboarding. Coordinator + reveal content lands Phase 4.
- `four_tasks_pair_key_design_notes.md` — identity model. Three-identifier system: user_id (stable per-user UUID, forever), pair_id (stable per-pair UUID, current relationship), pair_key (recovery hash of six values, lookup-only). Name immutable; username + active_leader mutable, trigger pair-key rotation. Case-sensitive name matching. Personal data attaches to user_id; relationship attaches to pair_id.

Locked design docs (server / backend):
- `four_tasks_rate_limiting_design_notes.md` — Cloudflare edge rate limits. Two-tier (pair-keyed + IP-keyed). 429 status code exception to the otherwise-locked set. Config lands next home-office session.
- `four_tasks_timezone_and_sealing_design_notes.md` — IANA timezone per user, boot-only sync. Lazy seal-on-open absorbed into claim endpoint transaction at tile 1.3 (session 12). No cron.

Pending design docs (server / backend):
- `four_tasks_promo_codes_design_notes.md` — NOT YET WRITTEN. Schema landed session 12 against ad-hoc decisions (2 tables, 4 user cols). Formalise before subreddit outreach. Captures single-code-per-user enforcement, code shape, founders_rate permanence, redemption_attempts log.

Locked design docs (gameplay / UX):
- `four_tasks_morning_sequence_design_notes.md` — 12-beat morning ritual. Rest-day variant, partner panel visibility, crash-resistance, claim endpoint architecture, MOTD reroll flat-cost (90-110, supersedes doubling).
- `four_tasks_onboarding_design_notes.md` — onboarding flow. Solo-as-default (no fork screen), 8-screen flow with copy locked, pairing-by-typed-values via single /join_by_values endpoint, theme+MOTD in day-zero onboarding, library sharing soft-plant. Case-sensitive name confirmation modal.
- `four_tasks_stamp_tier_design_notes.md` — stamp tier mapping (red/orange/yellow/green/purple), server-side message pools, tone targets per tier. Random pick, no anti-repeat for v1.0.
- `four_tasks_week_mode_design_notes.md` — per-weekday opt-in task templating. Long-press day-name to toggle. Divergence on first edit. Cal-icon stays universal standard-four editor. Today-only edit rule. Personal not shared. v1.0 scope (promoted session 9).
- `four_tasks_theme_design_notes.md` — theme + sticker system. Feature-catalogue model. Pixel-frequency palette derivation from sticker.png (session 8 — supersedes palette.tres). Asset naming `<id>_<role>.png`. Per-slot variant rotation by stable date-hash. MOTD + UI fonts locked global. 32×32 sticker canvas.

Locked design docs (commercial):
- `four_tasks_monetisation_position.md` — v2.0 (session 6/7). Subscription delivers bilateral library access + small stacking coin bonus. All packs bought with coins. Fortnightly alternating drop cadence (subscriber-only / free-thank-you). Founders pricing tier for launch cohort.

Running documents (not locked, edit freely):
- `four_tasks_marketing_notes.md` — APPtrioc launch play, subculture targeting, streamer outreach, promo code + subculture launch model (Section 6, session 7), journal section.
- `four_tasks_sticker_pack_brainstorm.md` — content brainstorm. Pack ideas with leader + slots-shipped + companions + notes. Rewritten session 7 to feature-catalogue model.
- `four_tasks_achievements_brainstorm.md` — hidden Easter-egg achievements rewarding global-first unlockers with lifetime subscriptions. NOT locked. Prompt-as-document. Session 7.

Discussion / reference (captured thinking, not committed):
- `four_tasks_tracking_design_notes.md` — per-user stat tracking discussion. DEFERRED to non-implementation. Two trackers with committed UX (lifetime_coins, longest_streak via leaderboard) are the entire scope. Goodhart audit framework + event-logging alternative path captured for future revisit.

Background / superseded:
- `four_tasks_write_rules_design_notes.md` — SUPERSEDED at session 11. Audited as derivative of other docs; every section had a canonical home elsewhere (project conventions, pair-key Sections 7/11/12/13, feature docs). File retained in git history; not authoritative. Tile 1.3 implemented against architecture docs directly.

Subfolders:
- `privacy/` — drafts and reference docs related to the privacy policy. Published version lives in pickle-moon-public.
- `deferred/` — design docs for features explicitly off the v1.0 roadmap. Captured fully so future-Morgan can revisit without re-deriving; not active development.

## four-tasks/deferred/ — features deferred from v1.0

Raw URL prefix for this subfolder:
https://raw.githubusercontent.com/thepicklemoon/pickle-moon-design-notes/main/four-tasks/deferred/

- `four_tasks_leaderboard_design_notes.md` — single global leaderboard ranked by lifetime_coins, streak as secondary. DEFERRED (session 8) — possibly not shipping at launch. Re-evaluate after v1.0 with real usage data. Full design + rejected alternatives preserved for if/when revisited.
- `four_tasks_coin_name_design_notes.md` — server-side affix rule system, transforms username into flavoured coin economy display name. Schema reserved v1.0, feature ships v1.x or later. Moved to deferred at session 9 housekeeping.
- `four_tasks_partner_reactions_design_notes.md` — sealed-day immutable partner reactions on MOTD + tray. Was first feature with field-level write rules. DEFERRED at session 12 — eliminated cross-user write complexity from tile 1.3. Schema columns retained in v1.0 schema (4 days cols), feature ships v1.x.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
REPO: four-tasks (PRIVATE)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Godot project + Cloudflare Worker source. NOT fetchable by Claude — paste relevant files into chat when needed.

Notable paths Claude should know exist:
- `server/src/index.ts` — Worker source. 11 endpoints live at https://four-tasks-api.thepicklemoon.workers.dev as of session 12.
- `server/src/stamp_messages.ts` — stamp message pools by tier (stubs as of session 12; real authoring is parked).
- `server/schema.sql` — D1 schema with v1.0 lock. Single source of truth (migration_001/002/003.sql DELETED at session 12; live remote D1 wiped + reapplied with UUID d92e5792-61a1-42cf-969c-9b33b8cc1475).
- `server/wrangler.toml` — Worker config.
- `server/tsconfig.json` — TypeScript config (added session 12).
- `scenes/` — Godot scenes.
- `scripts/` — GDScript autoloads + non-scene helpers.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
REPO: pickle-moon-public (PRIVATE → PUBLIC at Phase 5)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- `privacy_policy.md` — published privacy policy. Goes public at store submission.
- Future: `terms_of_service.md` — drafted before paid users exist.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
REPO: pickle-moon-ledger (PRIVATE)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Expense records. NOT fetchable by Claude — paste relevant rows when needed.

- `ledger.csv` — single source of truth for expenses.
- `receipts/` — supporting docs per row.
- `summary_FY*.md` — annual summaries generated at 30 June each year.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
USAGE NOTES FOR CLAUDE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- Claude can fetch any file under pickle-moon-design-notes via raw URL.
- First fetch per session needs the user to paste the full raw URL (permissions quirk). Subsequent siblings can be derived.
- Morgan refers to files by short name ("the devlog", "the morning sequence doc", "marketing notes"). Claude maps to filenames via this index.
- When Morgan says "the docs have changed," ask which ones and fetch fresh. Don't assume staleness.