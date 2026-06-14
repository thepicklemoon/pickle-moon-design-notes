FOUR TASKS — TASK-LABEL EDITING + SEALED-LABEL SNAPSHOT
═══════════════════════════════════════════════════════

Authority for: editing the standard-four task labels after onboarding, and the
per-day label freeze that makes sealed days immutable.

STATUS: design locked 2026-06-14, UNBUILT. Supplies the storage that the
week-mode doc's render rule 1 ("Day sealed → frozen seal state") already assumes
exists.
CROSS-REF: four_tasks_week_mode_design_notes.md (render rule, override editing),
four_tasks_timezone_and_sealing_design_notes.md (the seal walk).


THE GAP THIS CLOSES
───────────────────
Labels render live from `users.task_labels` for every date (task_list.gd). They
are set once at onboarding and never edited → frozen-render and live-render have
been byte-identical for 36 sessions, so the missing freeze was invisible. Any
label-edit surface (this doc's editor, or 4.D2 overrides) makes labels mutable;
with no per-day snapshot, an edit rewrites sealed history. The seal already
snapshots `active_theme` → `day_theme_state` for exactly this reason — labels get
the identical treatment.


INVARIANT
─────────
- A sealed day renders the labels resolved for it AT SEAL. Immutable thereafter.
- Editing labels (standard four OR overrides) affects unsealed days only. Never
  back-propagates to a sealed day.


SCHEMA — migration 001 (first numbered migration)
──────────────────────────────────────────────────
  ALTER TABLE days ADD COLUMN sealed_labels TEXT NOT NULL DEFAULT '[]';

- JSON array of 4 strings: the resolved labels as rendered for that day, frozen
  at seal.
- '[]' = no snapshot → render falls back to live (covers all pre-migration
  sealed rows).
- schema.sql (canonical) updated to include the column so fresh DBs match.
- Applied --remote to dev AND prod. Dev `days` was hand-patched; prod is the
  clean schema-of-record — confirm both end IDENTICAL after the ALTER.


RESOLVED FOUR
─────────────
The four labels as actually shown for a given day:
- v1.0 now (pre-week-mode): `users.task_labels`. No overrides exist.
- Post-4.D2: per week-mode render rule — override slot text where its bit is
  active that weekday, else standard-four[i].
The snapshot stores whichever applied that day. Mid-history edits self-resolve:
each seal freezes the THEN-current resolved four (a day sealed before a rename
keeps the old names; a day sealed after keeps the new ones).


SEAL-WRITE — sealSpan (index.ts)
────────────────────────────────
For every day sealed in the walk, write `sealed_labels` = JSON of the resolved
four, alongside stamp / sealed_at / day_theme_state. BOTH paths:
- never-opened INSERT (sealed grey): snapshot the resolved four for that date
  too — those rows still show four labels when opened.
- existing-row UPDATE: same.
Day-row serialization (GET payloads) must include `sealed_labels` so the client
has it on load.


RENDER RULE — client, task_list.gd `_resolve_task_labels`
─────────────────────────────────────────────────────────
- Day sealed AND `sealed_labels` non-empty (≠ []) → use `sealed_labels`.
- Else → live (`users.task_labels`; + overrides post-4.D2).
Supersedes the current always-live read. This is the storage week-mode render
rule 1 resolves against.


EDIT SURFACE — STANDARD FOUR  (the genuinely untiled piece)
───────────────────────────────────────────────────────────
- Entry: the cal-icon menu (per week-mode doc — "the standard four … maintained
  via the cal-icon menu"). Not a new surface.
- UI: four FIXED slots (LineEdit), pre-filled with current `users.task_labels`.
  No reorder, no add/remove. `tasks_done` is positional (index 0-3); reordering
  scrambles which historical tick maps to which label. Rename-in-place only.
- Validation (client, mirrors server): 4 entries, each non-empty after trim,
  ≤49 chars, no '|'.
- Save: whole-array PUT via `Backend.update_user(user_id, {task_labels:[...]})`.
  Endpoint built + validated (PUT /users/:id). Optimistic local update, rollback
  on server reject (house pattern).
- Propagation: FREE. Unsealed days render live → pick up the edit immediately;
  sealed days hold their snapshot. No propagation code to write.
- Disclosure: availability per staggered-disclosure doc. Should be reachable
  early (users set their real tasks fast). Exact gate at implementation.


MIGRATIONS DISCIPLINE  (starts here)
────────────────────────────────────
- New folder `server/migrations/`. Forward-only numbered SQL:
  001_add_days_sealed_labels.sql.
- Apply order: dev --remote → verify → prod --remote → verify. Record applied
  state.
- Reconcile dev ↔ schema.sql ↔ prod at this pass (dev `days` drifted from hand
  ALTERs; prod clean). End all three identical.
- Drop-and-reapply is DEAD from here — prod holds real data.


EDGE CASES
──────────
- Pre-migration sealed rows: `sealed_labels` '[]' → live fallback. Acceptable —
  those labels never changed. Known soft-spot, not a bug.
- Mid-history standard-four edit: each sealed day keeps the labels current at
  ITS seal; later seals freeze the new set. Correct by construction.
- Partner: labels are per-user; a partner's sealed days render their OWN snapshot
  (their payload carries it). No pair interaction.
- Empty / whitespace save: client blocks; server normalises (normaliseField)
  belt-and-braces.


BUILD ORDER  (each device-verified before tick)
───────────────────────────────────────────────
1. Migration 001: schema.sql + dev/prod ALTER + reconcile.
2. Server: sealSpan snapshots `sealed_labels` (both paths) + day serialization
   includes it.
3. Client: render rule reads the snapshot on sealed days.
4. Client: standard-four editor in the cal-icon menu + update_user wiring.
4.D2 (later): override resolution feeds the resolved four — no change to this
   doc's mechanism.
