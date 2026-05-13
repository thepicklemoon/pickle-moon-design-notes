# PICKLE MOON Repo Index

Purpose: canonical list of files across THE PICKLE MOON's repos, with one-line descriptions. Reference doc for Claude sessions — "fetch the X doc" resolves to a specific raw URL from this list.

Last updated: 2026-05-13

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
- `four_tasks_pair_key_v2_design_notes.md` — locked v2 identity model. Six-value pair-key, partner-app-mediated recovery, migration semantics.
- `four_tasks_partner_reactions_design_notes.md` — locked partner reactions design. Sealed-day immutable reactions on MOTD + tray.
- `four_tasks_coin_name_design_notes.md` — locked coin name generator. Server-side affix rules, paid rerolls, censorship layer.
- `four_tasks_staggered_disclosure_design_notes.md` — locked meta-principle: features reveal over days/weeks, not at onboarding.
- `four_tasks_write_rules_design_notes.md` — locked write rules for tile 1.3. Field-level permissions, validation, rejection codes for all three write endpoints.
- `four_tasks_morning_sequence_design_notes.md` — locked morning sequence beat-by-beat. Coin payout, stamp tier system, MOTD reveal, user-locked-out ceremony.
- `four_tasks_theme_design_notes.md` — per-sticker palette system. palette.tres + folder-per-sticker assets, retint at render time.
- `four_tasks_architectural_preference.md` — locked meta-rule: clarity over cleverness for low-throughput load profile.
- `four_tasks_monetisation_position.md` — subscription model + tone position. $1/week or $4/month. Founders model for prototype users.
- `four_tasks_marketing_notes.md` — APPtrioc launch strategy, subculture targeting, streamer outreach, journal section for ongoing thoughts.

Background docs (reference only, superseded or pre-v2):
- `four_tasks_pair_key_identity_design_notes.md` — v1 identity sketch from prototype era. Background only; superseded by v2 on recovery.

Subfolders:
- `privacy/` — drafts and reference docs related to the privacy policy. Published version lives in pickle-moon-public.

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
