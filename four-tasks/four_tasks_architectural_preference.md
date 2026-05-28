# Architectural preference — clarity over cleverness

Status: LOCKED (session 2). Foundational meta-rule. Refreshed session 7
with additional examples of the rule in action.

This is a meta-rule that sits above the individual backend / identity /
theme decisions. It governs how to resolve trade-offs in any design
conversation.

---

## The rule

Four Tasks is a low-throughput calendar app. A typical user produces
roughly six writes per day (four task ticks, possibly an MOTD edit,
possibly sticker-pool fiddling). At 10,000 users that's 60,000 writes
daily, spread across 24 hours — load Cloudflare's infrastructure handles
without effort.

The architectural preference, locked in session 2:

**When a design choice is between (a) the more correct, legible, or
robust option and (b) the more performant or clever option, prefer (a).
The clever option is reserved for cases where performance is genuinely
constrained.**

This rule pre-empts a class of premature-optimisation debates. It does
not mean "never optimise" — it means optimisation needs a real concrete
reason (measured latency, measured cost, measured pain) rather than a
theoretical performance gain.

---

## Decisions already in this codebase that follow from this rule

- **Uniform JSON envelope** (`{ok, data}` / `{ok, error}` on every
  endpoint). Slightly more bytes on the wire than raw JSON; kills the
  class of bug that lost mum's and Ayshi's data on the web build.
- **JSON-in-a-column for `task_labels` and `tasks_done`.** Not
  normalised into separate per-slot rows. Always read together,
  never queried individually.
- **Hash-based pair-key.** Fixed-width primary key for cleaner
  indexes; readable alternative kept in `users` rows for debugging.
- **Re-entrant pair-key rotations.** Sometimes performs a "wasted"
  duplicate rewrite when two simultaneous rotations land. Correct,
  legible, no lock or batching window needed.
- **Polling for partner data every 30s.** Not WebSocket realtime.
  Polling is correct *because* it's simpler, not in spite of it.
- **Lazy seal-on-open instead of nightly cron** (session 6 part 1).
  Sealing logic lives in the claim endpoint transaction rather than a
  scheduled job. Removes an entire class of "what if the cron failed"
  reasoning. Per-user, on-demand, atomic.
- **Palette derived from the painted sticker.png instead of a sidecar
  file or central table** (session 8 theme doc, supersedes both the
  earlier per-sticker palette.tres model and the central-table model).
  The renderer reads the painted pixels by frequency; the art file is
  the only source of truth. Adding a sticker is dropping a folder; no
  registry, no sidecar to keep in sync. File-existence is declaration.
- **Folder-per-sticker with optional feature files** (session 7 theme
  doc rewrite, supersedes earlier tier-based model). Each sticker
  ships whatever subset of theme files makes sense. No central
  manifest. Sparsity is a design axis. Adding a new theme slot later
  is dropping a new optional file, no schema change needed.
- **JSON column for per-day theme state snapshot** (session 7 theme
  doc rewrite, supersedes earlier two-column sketch). Single
  `days.day_theme_state` column captures the full slot map. Forward-
  compatible with future slot additions without further schema
  changes.
- **Per-slot context menu with INSTANT application** (session 7 theme
  doc rewrite). No commit-on-close pattern for theme slot changes —
  theme state is per-user, mutable, cheap. Identity changes
  (active_leader, username) keep the heavier commit pattern because
  they trigger pair-key rotation.
- **Server-side coin name generator instead of client-side**
  (coin name doc). One source of truth, tamper-proof, content
  iteration without client deploys.
- **Trust-the-client + length cap on MOTD content.** Server doesn't
  validate MOTD wordlists. Duplicating ~300 words across client and
  server would create two sources of truth that will drift.
- **Single coin pricing ramp for all stickers** (session 7
  monetisation update). Each pack costs the same regardless of how
  feature-rich it is, because "value" is what the user perceives.
  Trade-off between accuracy of pricing and clarity of the price
  curve resolved in favour of clarity.

---

## When to invoke the rule

Most reasonably during design conversation, when someone (Morgan or
Claude) catches themselves about to suggest the clever option for a
non-load-bearing reason. The rule asks: *why is the clever option
necessary?* If the answer is "it would be faster / more efficient /
more elegant in the abstract," the rule says no. If the answer is
"the obvious option fails at the load profile we expect," the rule
allows the clever option.

Future decisions should be checked against this rule before any
performance argument is taken seriously.
