# THE PICKLE MOON — Repo Index & Doc-Fetch Guide

Canonical file map for the studio's repos, and the operating guide for how
Claude pulls design docs on demand. This file supersedes the old
`pickle_moon_repo_index.md` (same name, kept so existing references resolve).

Kept in the Project's files so it's always in front of Claude. Morgan
refreshes the map manually (see REFRESHING, bottom). Last synced from
`git ls-files`: **2026-06-14 (session 37)**.

═══════════════════════════════════════════════════════════════
FOR CLAUDE — READ THIS FIRST
═══════════════════════════════════════════════════════════════

Design docs are **not** uploaded to the Project or pasted per chat. They
live in the public `pickle-moon-design-notes` repo and you fetch the ones
you need yourself, via `bash` + `curl`, from raw GitHub. Don't ask Morgan
to paste a design doc you can fetch — fetch it.

The MAP below tells you what exists and the exact path of each file. When a
doc is referenced — by name, by topic, or because a task touches its area —
pull it before answering on its content.

═══════════════════════════════════════════════════════════════
FETCH — HOW
═══════════════════════════════════════════════════════════════

Base URL (note the `refs/heads/main/` form — the short `main/` form is
flaky):

  https://raw.githubusercontent.com/thepicklemoon/pickle-moon-design-notes/refs/heads/main/

Fetch a doc = base + its MAP path:

  B="https://raw.githubusercontent.com/thepicklemoon/pickle-moon-design-notes/refs/heads/main"
  curl -sS "$B/four-tasks/four_tasks_week_mode_design_notes.md" -o /tmp/wk.md
  # then `view /tmp/wk.md` (or head/grep it)

Gotchas (all proven this session):
  - Use **bash `curl`**, not `web_fetch` — `web_fetch` is gated; the raw
    GitHub host IS in the bash network allowlist.
  - **Spaces in a path** must be URL-encoded: `a quiet week/` →
    `a%20quiet%20week/`.
  - **CDN propagation:** a doc pushed in the last ~1-3 min may 404 or serve
    a stale copy. If a fetch looks wrong right after Morgan says he pushed,
    wait and retry.
  - **Do NOT rely on the GitHub API to list the repo.** The tree endpoint
    (`api.github.com/repos/.../git/trees/main?recursive=1`) works in
    principle but this environment's egress IP is shared and hits GitHub's
    unauthenticated 60/hr limit — it will often fail. The MAP in this file
    is the authoritative list; raw per-file fetches are the reliable path.

═══════════════════════════════════════════════════════════════
FETCH — WHEN
═══════════════════════════════════════════════════════════════

  - **Session start (`prep`/`sync`):** read the devlog + todo first
    (they may already be in the Project's files — check; if not, fetch
    them), follow the devlog's AI Reading Guide, then pull whatever docs
    Morgan flags as recently changed.
  - **Mid-chat:** any doc named or implicated by the work — fetch it before
    reasoning about its locked decisions. The devlog/todo are lean
    pointers; the design docs carry the actual authority and detail.
  - **Naming is regular:** `four-tasks/four_tasks_<topic>_design_notes.md`.
    If something's referenced that isn't in the MAP, try constructing the
    path and fetching anyway — a 404 means it doesn't exist, a 200 means
    the MAP is stale and should be refreshed.

═══════════════════════════════════════════════════════════════
CANNOT FETCH — PASTE INSTEAD
═══════════════════════════════════════════════════════════════

The `four-tasks` repo is **private** — its source is NOT on raw GitHub:
Godot client (`.gd`, `.tscn`), the Worker (`server/index.ts`), `schema.sql`,
`wrangler.toml`, `stamp_messages.ts`, etc.

In a Project chat these code files are usually mirrored read-only in the
Project's files (`/mnt/project/`) — check there first. If a file is absent
or looks stale, ask Morgan to paste it. The Project mirror can lag the live
repo; for code, `/mnt/project/` or a paste is the source of truth, never a
fetch.

`pickle-moon-public` and `pickle-moon-ledger` aren't relevant to build work
and aren't fetched here.

═══════════════════════════════════════════════════════════════
MAP — four-tasks/  ·  STATE & WORKING FILES
═══════════════════════════════════════════════════════════════

  four-tasks/four_tasks_godot_devlog.txt
      THE devlog (v3). Read first at session start — AI Reading Guide +
      summary block at top, sessions append chronologically.
  four-tasks/four_tasks_godot_todo.txt
      THE todo. Phase-structured tile list, test ledger (top block),
      ACTIVE/PARKED/DEFERRED tiers, running hour total (bottom).
  four-tasks/four_tasks_system_map.md
      Live schema + core invariants (re-derive-never-remember, the
      uniform envelope, sealing model). Update at every schema pass.
  four-tasks/four_tasks_test_matrix.md
      Device/wire test matrix.
  four-tasks/wire_regression_pack.md
      Re-runnable server wire tests (PowerShell/curl), added s32.
  four-tasks/stamp_streak_pools_working_draft.md
      Stamp + streak message pools (working draft; launch pools locked
      into stamp_messages.ts in the private repo).
  four-tasks/four_tasks_sticker_pack_brainstorm.md
      Sticker concept brainstorm (not locked).
  four-tasks/terms_of_service_draft.md
      ToS draft (s29). Awaits Morgan review + hosting in pickle-moon-public.

═══════════════════════════════════════════════════════════════
MAP — four-tasks/  ·  ACTIVE DESIGN DOCS (AUTHORITY)
═══════════════════════════════════════════════════════════════

  four_tasks_architectural_preference.md
      Clarity-over-cleverness meta-rule. Read when weighing a design call.
  four_tasks_pair_key_design_notes.md
      Three-identifier model (user_id / pair_id / pair_key), pair-key
      rotation, re-entrant transaction-wrapped writes.
  four_tasks_economy_redesign_notes.md
      Coin / streak / rescue AUTHORITY. Any number or multiplier reads it.
  four_tasks_morning_sequence_design_notes.md
      12-beat ceremony + post-sequence presentation queue (beat-collision
      resolution).
  four_tasks_staggered_disclosure_design_notes.md
      Day-2 reveals, gesture dismissal, tutorial_progress semantics.
  four_tasks_timezone_and_sealing_design_notes.md
      Local-clock weekday-of-today; sealed days immutable (constrains all
      propagation).
  four_tasks_week_mode_design_notes.md
      Per-slot per-weekday label overrides. DESIGN LOCKED (Model B, popup
      detail s34). Build = tile 4.D2. See four_tasks_week_mode_primer.md.
  four_tasks_task_label_editing_design_notes.md
      days.sealed_labels snapshot + standard-four editor (tile 4.SL, s37).
  four_tasks_calendar_cells_design_notes.md
      Dead-cell treatment + off-month nav (tile 4.24).
  four_tasks_month_stats_design_notes.md
      Month colour + stats popup; month-stamp CUT (tile 4.10).
  four_tasks_hero_beat_design_notes.md
      Seal-day hero beat (4th-tick sticker moment).
  four_tasks_motd_design_notes.md
      MOTD message + pool design (distinct from stamp messages).
  four_tasks_stamp_tier_design_notes.md
      Five stamp tiers (grey latent); tier→colour→message mapping.
  four_tasks_notifications_design_notes.md
      Local scheduled reminders, v1.0 (tile 4.26); server push deferred.
  four_tasks_onboarding_design_notes.md
      8-screen solo-as-default flow; validation rules.
  four_tasks_recovery_design_notes.md
      Two-tier recovery flow.
  four_tasks_rate_limiting_design_notes.md
      Per-IP edge rate limiting; the sole 429 path.
  four_tasks_theme_design_notes.md
      Theme system v2 (THE authority): 4-role weighted colour system, v1.0
      7-slot catalogue, pre-bake retint, R2 runtime delivery. Gates 4.14b.
  four_tasks_tracking_design_notes.md
      Per-user stat tracking; week mode framed as the anti-scope-creep
      counterexample.
  four_tasks_content_authoring_tool_design_notes.md
      Content-drops authoring/delivery (tile 4.22 receive-side built).
  four_tasks_promo_codes_design_notes.md
      Founders promo codes; flag-vs-rate distinction.
  four_tasks_devkit_design_notes.md
      DevKit overlay + scenarios.
  four_tasks_monetisation_position.md
      Monetisation stance (subscription posture, buddyware framing).
  four_tasks_marketing_notes.md
      Go-to-market, APPtrioc launch sequence.

  four_tasks_week_mode_primer.md   ← may not be pushed yet (created s37)
      Handoff primer: 4.SL recap + the 4.D2 week-mode build, with the
      seal-snapshot retrofit called out. Fetch when starting 4.D2.

═══════════════════════════════════════════════════════════════
MAP — four-tasks/deferred/  ·  v1.x OR KILLED
═══════════════════════════════════════════════════════════════

  four_tasks_partner_reactions_design_notes.md   field write rules (v1.x)
  four_tasks_leaderboard_design_notes.md         KILLED for v1.0
  four_tasks_coin_name_design_notes.md           coin-name generator (v1.x)
  four_tasks_achievements_brainstorm.md          speculative, may never ship
  four_tasks_godot_devlog_session_1-9.txt        devlog v1 archive (s1-9)
  four_tasks_godot_devlog_session_10-25.txt      devlog v2 archive (s10-25)
      NOTE: the v3 devlog header refers to these as `_v1_sessions_1-9` /
      `_v2_sessions_10-25` — the real filenames are the above, in
      deferred/. Fetch by the real path.

═══════════════════════════════════════════════════════════════
MAP — four-tasks/archive/  ·  SUPERSEDED
═══════════════════════════════════════════════════════════════

  four_tasks_write_rules_design_notes.md
      Superseded field-write-rules doc; background only.

═══════════════════════════════════════════════════════════════
MAP — four-tasks/privacy/
═══════════════════════════════════════════════════════════════

  privacy_policy_v7.md          current privacy policy
  privacy_policy_findings.md    findings / rationale
  privacy_policy_v1..v6.md, privacy_policy_v4_annotated.txt, README.txt
      version history — ignore unless tracing a decision.
  (The hosted policy lives in the separate pickle-moon-public repo.)

═══════════════════════════════════════════════════════════════
MAP — OTHER PROJECTS (NOT Four Tasks — out of scope for FT work)
═══════════════════════════════════════════════════════════════

  a quiet week/design_doc.md        separate project (stay-at-home-dad
                                    roguelike). Path needs %20 encoding.
  envelope/envelope-design-doc.md   separate project ("Envelope").

═══════════════════════════════════════════════════════════════
MAP — REPO ROOT
═══════════════════════════════════════════════════════════════

  pickle_moon_repo_index.md   THIS file.
  README.md, .gitignore       repo housekeeping.

═══════════════════════════════════════════════════════════════
THE FOUR REPOS (on @thepicklemoon)
═══════════════════════════════════════════════════════════════

  pickle-moon-design-notes  PUBLIC  — devlogs, todos, design docs, planning.
      Everything in the MAP above. Raw-fetchable. Local: C:\dev\
      pickle-moon-design-notes\.
  four-tasks                PRIVATE — Godot project + Cloudflare Worker (TS).
      NOT fetchable; code mirrored in the Project's /mnt/project/ or pasted.
      Local: C:\dev\four-tasks\.
  pickle-moon-public        PRIVATE — privacy policy + future ToS; goes
      public at store submission (Phase 5).
  pickle-moon-ledger        PRIVATE — expense CSV + receipts.

Machine-independent paths: both Morgan's machines (work laptop user `steez`,
home PC user `Necruccio`) use the same `C:\dev\` layout. Repos live OUTSIDE
OneDrive deliberately.

═══════════════════════════════════════════════════════════════
REFRESHING THIS FILE (Morgan)
═══════════════════════════════════════════════════════════════

When docs are added/removed/renamed:

  cd C:\dev\pickle-moon-design-notes
  git ls-files

Paste the output to Claude and have it reconcile the MAP — add new lines,
drop deleted ones, fix renames — and bump the "Last synced" date at the top.
Then keep both copies in step: commit this file to the repo root AND update
the copy in the Project's files (the Project copy is the one Claude reads by
default; the repo copy is the fetchable backup).