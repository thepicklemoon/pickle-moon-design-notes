# PICKLE MOON Repo Index

Last edit: 2026-05-16 22:36 AWST

Purpose: canonical list of files across THE PICKLE MOON's repos, with one-line descriptions. Reference doc for Claude sessions — "fetch the X doc" resolves to a specific raw URL from this list.

Last updated: 2026-05-16

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
- `four_tasks_pair_key_design_notes.md` — identity model. Six-value pair-key (name+username+icon × 2), partner-app-mediated recovery, re-entrant transactional migrations. UNLOCKED at session 9 pending Section 4 decision (Option A pair-key as partition key vs Option B stable pair_id + lookup pair_key). BLOCKS tile 1.3 implementation.

Locked design docs (server / backend):
- `four_tasks_write_rules_design_notes.md` — write rules for tile 1.3. Field-level permissions, validation, state preconditions, rejection codes for POST bug_report / PUT day / PUT user.
- `four_tasks_rate_limiting_design_notes.md` — Cloudflare edge rate limits. Two-tier (pair-keyed + IP-keyed). 429 status code exception to the otherwise-locked set. Config lands next home-office session.
- `four_tasks_timezone_and_sealing_design_notes.md` — IANA timezone per user, lazy seal-on-open inside claim endpoint transaction. Replaces tile 1.4's earlier "nightly cron" framing.

Locked design docs (gameplay / UX):
- `four_tasks_morning_sequence_design_notes.md` — 12-beat morning ritual. Rest-day variant, partner panel visibility, crash-resistance, claim endpoint architecture, MOTD reroll flat-cost (90-110, supersedes doubling).
- `four_tasks_onboarding_design_notes.md` — onboarding flow. Solo mode as first-class state, invite link primary pairing path with manual join fallback, solo data migrates into pair on join.
- `four_tasks_partner_reactions_design_notes.md` — sealed-day immutable partner reactions on MOTD + tray. First feature with field-level write rules (partner writes other user's row). Schema reserved v1.0, feature ships v1.x.
- `four_tasks_stamp_tier_design_notes.md` — stamp tier mapping (red/orange/yellow/green/purple), server-side message pools, tone targets per tier. Random pick, no anti-repeat for v1.0.
- `four_tasks_week_mode_design_notes.md` — per-weekday opt-in task templating. Long-press day-name to toggle. Divergence on first edit. Cal-icon stays universal standard-four editor. Today-only edit rule. Personal not shared. Promoted to v1.0 scope at session 9.
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
- (None on disk. Earlier index entries referenced a v1 pair-key sketch but the only pair-key file in the repo is the active design doc above.)

Subfolders:
- `privacy/` — drafts and reference docs related to the privacy policy. Published version lives in pickle-moon-public.
- `deferred/` — design docs for features explicitly off the v1.0 roadmap. Captured fully so future-Morgan can revisit without re-deriving; not active development.

## four-tasks/deferred/ — features deferred from v1.0

Raw URL prefix for this subfolder:
https://raw.githubusercontent.com/thepicklemoon/pickle-moon-design-notes/main/four-tasks/deferred/

- `four_tasks_leaderboard_design_notes.md` — single global leaderboard ranked by lifetime_coins, streak as secondary. DEFERRED (session 8) — possibly not shipping at launch. Re-evaluate after v1.0 with real usage data. Full design + rejected alternatives preserved for if/when revisited.
- `four_tasks_coin_name_design_notes.md` — server-side affix rule system, transforms username into flavoured coin economy display name. Schema reserved v1.0, feature ships v1.x or later. Moved to deferred at session 9 housekeeping.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
REPO: four-tasks (PRIVATE)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Godot project + Cloudflare Worker source. NOT fetchable by Claude — paste relevant files into chat when needed.

Notable paths Claude should know exist:
- `server/src/index.ts` — Worker source.
- `server/schema.sql` — D1 schema with migration history.
- `server/wrangler.toml` — Worker config.
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
- Morgan refers to files by short name ("the devlog", "write rules doc", "marketing notes"). Claude maps to filenames via this index.
- When Morgan says "the docs have changed," ask which ones and fetch fresh. Don't assume staleness.
