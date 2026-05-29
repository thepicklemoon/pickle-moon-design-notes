# PICKLE MOON Repo Index

Last edit: 2026-05-28 AWST

Purpose: canonical list of files across THE PICKLE MOON's repos, with one-line descriptions. Reference doc for Claude sessions — "fetch the X doc" resolves to a specific file from this list.

Last updated: 2026-05-28 (session 19 — doc de-stale pass: all design docs reconciled to shipped v1.0 schema + system map; system map + test matrix added to repo; devkit + content-authoring docs indexed; deferred/ and archive/ folders represented).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
REPO: pickle-moon-design-notes (PUBLIC)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Raw URL base:
https://raw.githubusercontent.com/thepicklemoon/pickle-moon-design-notes/main/

## Root level

- `pickle_moon_repo_index.md` — this file. The directory.
- `README.md` — repo intro.
- `.gitignore` — repo ignore rules.
- `a quiet week/design_doc.md` — other Pickle Moon project (not Four Tasks). Kept in-repo deliberately.
- `envelope/envelope-doc.md` — other Pickle Moon project (not Four Tasks). Kept in-repo deliberately.

## four-tasks/ — Four Tasks Godot port

Raw URL prefix for this subfolder:
https://raw.githubusercontent.com/thepicklemoon/pickle-moon-design-notes/main/four-tasks/

Working documents (high churn):
- `four_tasks_godot_devlog.txt` — ACTIVE devlog (current). Summary block of sessions 1-9 at top, then sessions 10+. Primary context doc. AI reading guide at top.
- `four_tasks_godot_devlog_session_1-9.txt` — archived early devlog (sessions 1-9 in full prose). Reference only — for reconstructing how a decision was reached, not the decision itself. Authority for any locked decision lives in the relevant design doc + the active devlog's summary block.
- `four_tasks_godot_todo.txt` — active tile list, phases, parked items.

Live system state (the "what shipped" reference — authoritative for current build):
- `four_tasks_system_map.md` — live operational view of the shipped system: schema, endpoint contracts, autoloads, scenes, seal invariants. The ground-truth doc the design notes are reconciled against. Created session 19.
- `four_tasks_test_matrix.md` — test coverage matrix: endpoints, client autoloads, devkit, bug-report path. Created session 19.

Locked design docs (architecture / meta-rules):
- `four_tasks_architectural_preference.md` — meta-rule: clarity over cleverness for low-throughput load profile. Examples + when the clever option is permitted.
- `four_tasks_staggered_disclosure_design_notes.md` — meta-principle: features reveal over days/weeks, not at onboarding. Day-4 partner-reactions slot deferred with the feature. Coordinator + reveal content lands Phase 4.
- `four_tasks_pair_key_design_notes.md` — identity model. Three-identifier system: user_id (stable per-user UUID, forever), pair_id (stable per-pair UUID, current relationship), pair_key (recovery hash of six values, lookup-only). Name immutable; username + active_leader mutable, trigger pair-key rotation. Case-sensitive name matching. Schema shipped wholesale session 12 (header note clarifies the migration_005 mentions are historical).

Locked design docs (server / backend):
- `four_tasks_rate_limiting_design_notes.md` — Cloudflare edge rate limits. Two-tier: user_id-keyed (identified endpoints) + IP-keyed (resolve / user creation). 429 status code exception to the otherwise-locked set. Config lands Phase 5. Re-keyed off the shipped /users/:user_id surface session 19.
- `four_tasks_timezone_and_sealing_design_notes.md` — IANA timezone per user, boot-only sync. Lazy seal-on-open absorbed into claim endpoint transaction at tile 1.3 (session 12). No cron.

Locked design docs (gameplay / UX):
- `four_tasks_morning_sequence_design_notes.md` — 12-beat morning ritual. Rest-day variant, partner panel visibility, crash-resistance via walkback/sealed_at gate, shipped claim endpoint (POST /users/:user_id/claim), MOTD storage (days.motd, frozen at seal; header vs tray surfaces), MOTD reroll flat-cost (90-110, supersedes doubling). Reconciled to shipped session 19.
- `four_tasks_onboarding_design_notes.md` — onboarding flow. Solo-as-default (no fork screen), 8-screen flow with copy locked, pairing-by-typed-values (POST /users/:user_id/join_by_values, shipped shape), recovery via POST /resolve, theme+MOTD in day-zero onboarding, library sharing soft-plant. Case-sensitive name confirmation. Reconciled to shipped session 19.
- `four_tasks_stamp_tier_design_notes.md` — stamp tier mapping (red/orange/yellow/green/purple), server-side message pools, tone targets per tier. Random pick, no anti-repeat for v1.0.
- `four_tasks_week_mode_design_notes.md` — per-weekday opt-in task templating. Long-press day-name to toggle. Divergence on first edit. Personal not shared. v1.0 scope (promoted session 9). Schema shipped: week_mode_weekdays bitmask + user_weekday_overrides (user_id, weekday) PK + task_labels JSON. Corrected session 19.
- `four_tasks_theme_design_notes.md` — theme + sticker system. Feature-catalogue model. Pixel-frequency palette derivation from sticker.png (session 8 — supersedes palette.tres). Asset naming `<id>_<role>.png`. Per-slot variant rotation by stable date-hash. Cell shows sticker (stamp is on the tray). MOTD + UI fonts locked global. Cell-overlay-bucket vs runtime-tier open question flagged (4.14b).
- `four_tasks_devkit_design_notes.md` — DevKit scenario menu (tile 2.9). In-build tooling to drive states for testing (seed days, force seals, jump dates, dismiss reveals). Replaced the killed tile-0.4 devkit skeleton. Session 17.
- `four_tasks_content_authoring_tool_design_notes.md` — tool for authoring the content pools (stamp messages, MOTD wordlists, rest-day labels, task placeholders) outside hand-edited source. Session 17.

Locked design docs (commercial):
- `four_tasks_monetisation_position.md` — v2.0 (session 6/7). Subscription delivers bilateral library access + small stacking coin bonus. All packs bought with coins. Fortnightly alternating drop cadence (subscriber-only / free-thank-you). Founders pricing tier for launch cohort. day_theme_state JSON column shipped v1.0.

Pending design docs (server / backend — NOT YET WRITTEN):
- `four_tasks_promo_codes_design_notes.md` — promo-codes tables + user columns shipped inert in v1.0 schema (session 12); the /redeem endpoint and this doc are pending. Formalise before subreddit outreach. Single-code-per-user enforcement, code shape, founders_rate permanence, redemption_attempts log.

Running documents (not locked, edit freely):
- `four_tasks_marketing_notes.md` — APPtrioc launch play, subculture targeting, streamer outreach, promo code + subculture launch model, journal section. Partner reactions noted v1.x, not launch copy.
- `four_tasks_sticker_pack_brainstorm.md` — content brainstorm. Pack ideas with leader + slots-shipped + companions + notes. Feature-catalogue model.
- `four_tasks_achievements_brainstorm.md` — hidden Easter-egg achievements rewarding global-first unlockers with lifetime subscriptions. NOT locked. Prompt-as-document.

Discussion / reference (captured thinking, deferred from implementation):
- `four_tasks_tracking_design_notes.md` — per-user stat tracking discussion. DEFERRED. Two trackers with committed UX (lifetime_coins, longest_streak) are the entire scope. Goodhart audit + event-logging alternative captured for future revisit.

Session captures (handoff artefacts — under Morgan's control, not load-bearing reference):
- `four_tasks_session_19_capture.md` — session 19 capture.
- `s19primer.md` — session 19 primer.

Subfolders:
- `privacy/` — drafts and reference docs for the privacy policy. Published version lives in pickle-moon-public.
- `deferred/` — design docs for features explicitly off the v1.0 roadmap. Captured fully; not active development.
- `archive/` — superseded docs retained for history; not authoritative.

## four-tasks/deferred/ — features deferred from v1.0

Raw URL prefix:
https://raw.githubusercontent.com/thepicklemoon/pickle-moon-design-notes/main/four-tasks/deferred/

- `four_tasks_leaderboard_design_notes.md` — single global leaderboard ranked by lifetime_coins, streak secondary. DEFERRED (session 8). Re-evaluate after v1.0 with real usage data. Full design + rejected alternatives preserved.
- `four_tasks_coin_name_design_notes.md` — server-side affix rule system transforming username into a flavoured coin display name. Schema reserved v1.0, feature ships v1.x+. Moved to deferred session 9.
- `four_tasks_partner_reactions_design_notes.md` — sealed-day immutable partner reactions on MOTD + tray. DEFERRED at session 12 — eliminated cross-user write complexity from tile 1.3. Schema columns retained in v1.0 schema, feature ships v1.x. Surfacing principle (time-agnostic, no push) lives in the morning sequence doc Q7.

## four-tasks/archive/ — superseded, retained for history

Raw URL prefix:
https://raw.githubusercontent.com/thepicklemoon/pickle-moon-design-notes/main/four-tasks/archive/

- `four_tasks_write_rules_design_notes.md` — SUPERSEDED at session 11. Every section had a canonical home elsewhere (project conventions, pair-key Sections 7/11/12/13, feature docs). Not authoritative. Tile 1.3 implemented against architecture docs directly.

## four-tasks/privacy/ — privacy policy drafts + reference

Raw URL prefix:
https://raw.githubusercontent.com/thepicklemoon/pickle-moon-design-notes/main/four-tasks/privacy/

- `README.txt` — folder intro / status.
- `privacy_policy_findings.md` — research + requirements notes feeding the policy.
- `privacy_policy_v1.md` … `privacy_policy_v7.md` — versioned drafts. v7 is the latest. Published version lives in pickle-moon-public.
- `privacy_policy_v4_annotated.txt` — annotated v4 (review pass).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
REPO: four-tasks (PRIVATE)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Godot project + Cloudflare Worker source. NOT fetchable by Claude — paste relevant files into chat when needed.

Notable paths Claude should know exist:
- `server/src/index.ts` — Worker source. Endpoints live at https://four-tasks-api.thepicklemoon.workers.dev (11 as of session 12).
- `server/src/stamp_messages.ts` — stamp message pools by tier (stubs as of session 12; real authoring parked).
- `server/schema.sql` — D1 schema, v1.0 lock. Single source of truth (migration_001/002/003.sql DELETED session 12; remote D1 wiped + reapplied, UUID d92e5792-61a1-42cf-969c-9b33b8cc1475).
- `server/wrangler.toml` — Worker config.
- `server/tsconfig.json` — TypeScript config (session 12).
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

- The pickle-moon-design-notes repo is public; raw URLs are fetchable when a file isn't already in the Project knowledge.
- The Project knowledge holds the current design docs; prefer it. Fetch a fresh raw URL only when Morgan flags a file as newer than the Project copy.
- For "what shipped" questions, the system map + test matrix are authoritative over the design docs.
- Morgan refers to files by short name ("the devlog", "the morning sequence doc", "marketing notes"). Map to filenames via this index.
- When Morgan says "the docs have changed," ask which ones rather than assuming staleness.
