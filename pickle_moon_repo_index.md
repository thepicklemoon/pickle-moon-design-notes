# PICKLE MOON Repo Index

Purpose: canonical list of files across THE PICKLE MOON's repos, with one-line descriptions. Reference doc for Claude sessions — "fetch the X doc" resolves to a specific raw URL from this list.

Last updated: 2026-05-15 (session 7 cross-reference sweep — all design docs aligned to feature-catalogue theme model, active_leader naming, JSON-column migration_005, fortnightly cadence, cell overlay/mask architecture)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
REPO: pickle-moon-design-notes (PUBLIC)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Raw URL base:
https://raw.githubusercontent.com/thepicklemoon/pickle-moon-design-notes/main/

## Root level

- `pickle_moon_repo_index.md` — this file. The directory.

## four-tasks/ — Four Tasks Godot port

Working documents (high churn):
- `four_tasks_godot_devlog.txt` — session-by-session devlog. Primary context doc. AI reading guide at top.
- `four_tasks_godot_todo.txt` — active tile list, phases, parked items.

Design docs (locked, reference only):
- `four_tasks_pair_key_design_notes.md` — locked identity model (renamed session 7 from pair_key_v2). Six-value pair-key, partner-app-mediated recovery, migration semantics. active_leader is the migration-triggering identity field (renamed from prototype-era `emoji`/`icon`). Picker UX is commit-on-close for leader/username changes; non-leader theme slots use instant-commit per theme doc.
- `four_tasks_partner_reactions_design_notes.md` — locked partner reactions design. Sealed-day immutable reactions on MOTD + tray. Surfacing model is time-agnostic visual indicators (no push) per session 5 update. Reaction picker UX inherits commit-on-close pattern from identity picker, NOT instant pattern from theme picker.
- `four_tasks_coin_name_design_notes.md` — locked coin name generator. Server-side affix rules, paid rerolls with doubling cost (contrast with MOTD reroll flat cost). Architecturally locked, implementation post-v1.0.
- `four_tasks_staggered_disclosure_design_notes.md` — locked meta-principle: features reveal over days/weeks, not at onboarding. Cross-refs aligned to renamed pair_key doc and feature-catalogue theme model. Achievements brainstorm explicitly noted as counter-example (no scheduled reveal).
- `four_tasks_write_rules_design_notes.md` — locked write rules for tile 1.3. Field-level permissions, validation, rejection codes for all three write endpoints. Schema delta bundles migrations 003 + 004 + 005 (session 7 update — now includes full active_leader / active_theme JSON / active_stickers / day_theme_state shape). Tile 1.4 nightly cron reframed to lazy seal-on-open.
- `four_tasks_morning_sequence_design_notes.md` — locked morning sequence beat-by-beat. Coin payout, stamp tier system, MOTD reveal, user-locked-out ceremony. Claim endpoint architecture integrated with seal-on-open transaction. Session 7 terminology refresh ("Lever 1" → "active_stickers pool").
- `four_tasks_theme_design_notes.md` — locked theme system (session 5 foundation, session 6 sticker canvas + TIER 0, session 7 rewrite to feature-catalogue model + cell overlay/mask architecture). Per-slot selection from accessible library, long-press context menu with per-element icons, sparsity as a design axis, MOTD font + UI fonts locked global. Cell treatments are overlay + optional mask layers ON TOP of stock cells (NOT full replacements). Schema is JSON-based (active_theme map, day_theme_state on days). Required reading before tile 4.14a/b and tile 4.D.
- `four_tasks_architectural_preference.md` — locked meta-rule: clarity over cleverness for low-throughput load profile. Refreshed session 7 with examples covering lazy seal-on-open, feature-catalogue theme model, JSON day_theme_state, instant slot application, single pricing ramp.
- `four_tasks_monetisation_position.md` — locked v2.0 (session 6 part 2, schema + cadence updates session 7). Subscription delivers bilateral library access + small stacking coin bonus. All packs bought with coins, permanently owned. Past days immutable. Founders pricing tier distinct from founders flag. Fortnightly alternating drop cadence (subscriber-only / free-thank-you weeks alternating). Single pricing ramp 300-600 cap regardless of pack feature-richness. Year-2 standard pricing +50% above founders. Required reading for Phase 5.
- `four_tasks_marketing_notes.md` — APPtrioc launch strategy, 9 subculture targets, streamer outreach, long-tail community plan, journal section. Section 6 (added session 7): full promo code + subculture-targeted launch model — 3-month free trials, soft landing into founders pricing, capped codes with wave-based retargeting, no waitlist, library sharing included in trial.
- `four_tasks_timezone_and_sealing_design_notes.md` — locked session 6 part 1. Timezone localisation + lazy-seal-on-open architecture. Each user has stored IANA timezone; sealing happens in claim endpoint transaction when user opens app on a later local date. Replaces tile 1.4's "nightly cron sweep" framing. Required reading before tile 1.3 implementation.
- `four_tasks_onboarding_design_notes.md` — locked session 6 part 1 (terminology refreshed session 7). Full onboarding flow design. Solo mode as first-class state (snoopers, partner-not-available installers). Invite link primary pairing path with solo_uuid + three values; manual join fallback. Solo data migrates into pair on join. MOTD removed from onboarding (first MOTD lands in first morning sequence). Required reading before any Phase 3 tile.

Running brainstorm docs (high churn, never "done"):
- `four_tasks_sticker_pack_brainstorm.md` — content brainstorm. Pack ideas with explicit slot listings per pack (rewritten session 7 to feature-catalogue model). Six entries (frog, chilli, vampire, wizard, car, fairy), each listing which named slots that pack ships and at what ambition level. Parked ideas section. Reference for tile 4.D and ongoing fortnightly drop pipeline.
- `four_tasks_achievements_brainstorm.md` — speculative future work, NOT LOCKED. Hidden Easter-egg achievements rewarding global-first unlockers with lifetime subscriptions. Ten starting candidates plus seed ideas. Adjudication architecture sketched. May never ship — documented to prevent the idea going undocumented.

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
