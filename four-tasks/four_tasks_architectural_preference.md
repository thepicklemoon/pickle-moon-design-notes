# Port plan insert — architectural preference

Paste this into `four_tasks_godot_port_plan.md` under section 2 (the
architectural decisions block), as section 2.0 or wherever it fits the
existing numbering. It's a meta-rule that sits above the individual
backend/identity/theme decisions.

---

### 2.0 Architectural preference — clarity over cleverness

Four Tasks is a low-throughput calendar app. A typical user produces
roughly six writes per day (four task ticks, possibly a diary edit,
possibly emoji fiddling). At 10,000 users that's 60,000 writes daily,
spread across 24 hours — load Cloudflare's infrastructure handles
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

Decisions already in this codebase that follow from this rule:

- **Uniform JSON envelope** (`{ok, data}` / `{ok, error}` on every
  endpoint). Slightly more bytes on the wire than raw JSON; kills the
  class of bug that lost mum's and Ayshi's data on the web build.
- **JSON-in-a-column for `task_labels` and `tasks_done`.** Not
  normalised into separate per-slot rows. Always read together,
  never queried individually.
- **Hash-based pair-key.** Fixed-width primary key for cleaner
  indexes; readable alternative kept in `users` rows for debugging.
- **Re-entrant pair-key migrations.** Sometimes performs a "wasted"
  duplicate rewrite when two simultaneous migrations land. Correct,
  legible, no lock or batching window needed.
- **Polling for partner data every 30s.** Not WebSocket realtime.
  Polling is correct *because* it's simpler, not in spite of it.

Future decisions should be checked against this rule before any
performance argument is taken seriously.
