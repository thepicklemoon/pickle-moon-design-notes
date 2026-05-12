# Pickle Moon Design Notes

Private design history, devlogs, and architectural notes for THE PICKLE MOON's projects.

## Structure

- `four-tasks/` — design docs, devlog, todo, privacy policy drafts for the Four Tasks Godot port
- `studio/` — cross-cutting notes (reserved; empty at session 2)

## Convention

Devlog and todo are the working documents updated session-to-session.
Design `.md` files are written once when an architectural decision is locked, then referenced thereafter.

Filenames are project-prefixed (`four_tasks_*.md`) so when files move around or get linked from elsewhere, their scope stays obvious.

## Related repos

- `four-tasks` — Godot project + Cloudflare Worker (private)
- `pickle-moon-public` — published privacy policy, future ToS (private, public at store submission)
- `pickle-moon-ledger` — expense records (private)