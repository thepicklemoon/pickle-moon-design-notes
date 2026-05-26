# Four Tasks — DevKit Design Notes

Status: LOCKED (session 17, revised session 18 to reflect per-scenario
mutation model; supersedes tile 0.4 skeleton).
Owner: Morgan / THE PICKLE MOON.

## Purpose

DevKit replaces the "seed-and-boot" pattern from the PWA prototype's
debug screen. The native DevKit is the live app with an always-visible
debug button overlaid in the main view. Tapping it opens a scenario
menu; selecting a scenario mutates whatever state gates the thing being
tested, then returns control to the app so the feature plays out in
situ.

Goals:
  - Test every staged feature without waiting real days or running curls.
  - Behave as close to the real app as possible while a scenario is live.
  - Stay lean — DevKit is not a feature, it's scaffolding.

## Non-goals

  - No time-of-day or calendar-date manipulation. Features trigger off
    state, not clocks. "User is on day 21" is achieved by rewriting
    `users.created_at`, not by overriding `today_iso()`.
  - No production-side override hooks. DevKit endpoints live on a
    separate worker against a separate D1. Production worker never
    sees DevKit code.
  - Not user-facing. DevKit is stripped from store builds. The PWA-era
    user-facing reboot button does not carry forward.

## Mental model — scenarios are targeted mutations

The earlier "reset-to-default, then seed" model was overbuilt. Each
scenario clears or rewrites only the specific fields that gate the
thing being tested. Nothing else is touched.

  - Morning sequence + payout → clear `morning_payout_due_for`, leave
    `created_at` and `days` alone.
  - Day-21 subscription disclosure → rewrite `created_at` to 21 days
    ago, leave everything else.
  - Friday content drop → clear the "seen this drop" flag on both
    users, no other mutation.
  - Fresh install → client wipes `identity.cfg`, server resets the two
    test user rows to their day-zero shape (no days, no streak, no
    coins).

Scenarios are not additive. Each one is its own self-contained
mutation. Calendar state from a previous scenario is irrelevant —
either the new scenario rewrites the fields it cares about, or it
doesn't care about them.

The "Default" scenario is the only one that lays down meaningful
calendar data. It's there to give the app something to render after a
fresh dev D1 — see "Default scenario" below.

## Test users

  - User A: name `TestUser`, username + active_leader TBD at
    implementation.
  - User B: name `TestPartner`, username + active_leader TBD at
    implementation.
  - Both paired. `pair_id` stable across scenarios.
  - `user_id`s stable across scenarios.

Stable IDs across scenarios matter because the Godot `identity.cfg` on
the dev machine doesn't re-write per scenario — the same identity boots
the app every time, the server-side state mutates around it.

Names are ASCII-only by deliberate choice — both render cleanly in
Departure Mono + JetBrains Mono. (Note: server-side name validation
currently NFC-normalises but does not restrict to ASCII. Non-ASCII
input from a real user would render as boxes in v1.0 fonts. That's a
Phase 3 onboarding decision, not a DevKit concern.)

## Default scenario

The default scenario exists to give a freshly-installed DevKit build
something visually interesting on the calendar. It is NOT a baseline
that every other scenario resets back to.

Seed contents:
  - TestUser + TestPartner rows, paired.
  - Day rows on both users covering one of each cell colour: red,
    orange, yellow, green, and a rest day. ~6 days total per user.
  - No mid-streak ceremony, no 30-day spread.
  - No pending popups, no first-run beats.

Tapping "Default" in the scenario menu re-runs this seed against the
dev D1, overwriting whatever's there. Useful when a previous scenario
left the calendar in a weird shape and you want to get back to a clean
look.

## The debug button

  - One button, parented inside PanelYou's calendar area, below the
    tasks tray in scene order. Visible when the tray is retracted,
    obscured when the tray slides out. Partner panel has no button.
  - Not gated, not hidden, not gesture-triggered. It's just there.
  - Tapping opens the scenario menu as a full-screen overlay.
  - Hidden in any view that isn't the main calendar (onboarding flow,
    morning sequence, popups, etc.). It reappears when the user lands
    back on the main calendar.

The button is a build-flag artifact — present in DevKit builds, absent
in store builds. Implemented by checking `OS.has_feature("editor") or
OS.has_feature("devkit")` at `_ready()`; the `editor` feature covers
editor + editor-run builds, the `devkit` custom feature is added to
the DevKit export preset for in-pocket testing.

## Scenario menu structure

The menu is a flat list of scenarios grouped by category. Selecting
one runs its mutation, closes the menu, and lands the app at the
appropriate entry point.

### Categories and scenarios

**Default**
  - Default state — re-runs the seed described above.

**Identity flows**
  - Fresh install (client wipes identity.cfg, server resets both test
    users to day-zero shape, app boots to onboarding screen 1).
  - Recovery flow (boot to recovery screen with primed six-tuple).
  - Unpair received (banner pending, partner's last activity present).

**Daily rituals**
  - Morning sequence + payout (clear morning_payout_due_for so sequence
    fires on next boot).
  - Rest-day morning variant.
  - MOTD entry prompt due.

**Staggered disclosure**
  - Day-2 philosophy reveal trigger primed.
  - Gesture dismissal pattern primed.
  - Picker hint trigger primed.
  - Week mode cascade primed.

**Content drops**
  - Patch announcement queued.
  - Sticker pack drop available (free).
  - Sticker pack drop available (subscriber).

**Monetisation**
  - Founders rate copy primed.
  - Promo code redemption primed.
  - Day-21 subscription disclosure primed (sets `created_at` to 21
    days ago).

## How a scenario runs

Every scenario is a single endpoint call from the client to the dev
worker, plus a UI route on the client. Three logical steps:

  1. **Server mutation.** The dev worker performs the SQL needed to
     prime the scenario. Usually one or two `UPDATE`s, occasionally a
     small batch. Targeted, not blanket.
  2. **Local cleanup (if needed).** Some scenarios need client-side
     state cleared too — most obviously "Fresh install" wipes
     `identity.cfg`. Most scenarios don't need this step.
  3. **Land at entry point.** Push the app onto the screen where the
     scenario is observed. Usually main calendar; sometimes onboarding,
     sometimes opens directly into morning sequence beat 1.

## Infrastructure separation

DevKit runs against a **separate worker and D1 database** from
production. Two reasons:

  - Production users' calendar data can never be touched by DevKit
    code paths, accidental or otherwise.
  - Production worker URL is distinct from dev. Misdirected traffic
    physically cannot reach the wrong database.

Deployment surface:

  - Production: `four-tasks-api.thepicklemoon.workers.dev`, D1
    database `four-tasks`.
  - DevKit: `four-tasks-api-dev.thepicklemoon.workers.dev`, D1
    database `four-tasks-dev`.

Both are managed in one `wrangler.toml` via `[env.dev]` / `[env.production]`
sections. Deploy with `wrangler deploy --env dev` or `wrangler deploy
--env production`. Both free-tier on Cloudflare.

The dev D1 is a fresh empty database. Production data is not migrated
in — there is nothing to migrate. The schema is applied via
`schema.sql` (the same file production uses) and the default seed via
`seed_devkit.sql` (new, lives alongside the existing `seed_partner.sql`).

### Source file: shared, route-gated

Both environments deploy the same `index.ts` source. The `/devkit/*`
routes are gated at the top of the dispatcher by `env.IS_DEV === "true"`.
The dev wrangler.toml's `[env.dev.vars]` sets `IS_DEV = "true"`;
production omits the var, so production worker 404s on devkit paths.

This is a pragmatic relaxation of the original "production bundle
contains zero DevKit code" goal. The practical security boundary is
identical (routes unreachable in prod) and the engineering cost of
the relaxation is one env-var check. If strict source separation is
ever needed (audit, app-store concern), this becomes a build-step
strip in a later session.

## DevKit endpoints

Each scenario gets its own endpoint on the dev worker. They live under
`/devkit/scenario/<name>` and are only deployed to the dev worker —
production `index.ts` doesn't reference them.

Initial set for tile 2.9:

  - `POST /devkit/scenario/default` — re-runs the seed.
  - `POST /devkit/scenario/fresh_install` — wipes TestUser + TestPartner
    day rows, resets their mutable fields to day-zero shape, leaves
    user_ids and pair_id intact. Client wipes identity.cfg separately.
  - `POST /devkit/scenario/recovery_flow` — same server-side teardown
    as fresh_install; client behaviour differs.

All other scenarios stub out as menu entries with no endpoint wired
yet — server returns 404, button shows a small grey "(stub)" label.
Scenarios get implemented one at a time as their feature lands.

## Ad-hoc data nudges

For test cases that don't justify a named scenario, wrangler from the
terminal hits the dev D1 directly:

```
wrangler d1 execute four-tasks-dev --remote --command "UPDATE users SET coins = 5000 WHERE user_id = '...';"
```

Or a one-off SQL file run via `--file=...`. Useful for testing edge
cases that only need to be reproduced once. No need to formalise these
into named scenarios unless they get re-used.

## Chicken-and-egg handling

Scenarios for features that haven't been built yet show in the menu
with a "(stub)" label. Tapping them does nothing meaningful — either
the endpoint 404s, or it runs but no client-side feature is there to
display the result. As each feature lands in its tile, the scenario
wires up against it.

This inverts the usual order — DevKit defines the observable surface
ahead of time, features implement against that contract.

## Implementation notes

  - DevKit is one tile (renumbered 2.9, replacing the user-facing
    reboot button which is killed entirely).
  - Scenes: `devkit_menu.tscn` (the overlay), `devkit_button.tscn`
    (the corner button parented inside PanelYou).
  - Script: `DevKit.gd` autoload. Always registered in `project.godot`;
    `_ready()` checks the build flag (see below) and self-deletes if
    absent. Cost-free in store builds.
  - Loader prints `[Loader] editor=<bool> devkit=<bool>` at boot as a
    sanity check on the current build context.
  - **Build flag — two checks combined:**
      - `OS.has_feature("editor")` is true in the Godot editor and
        editor-run builds. Editor runs are treated as DevKit by default
        — only Morgan uses the editor, and he always wants the dev
        worker there.
      - `OS.has_feature("devkit")` is a custom feature set in DevKit
        export presets (Project → Export → preset → Resources → Custom
        Features). Store presets leave it unset.
      - Combined check: `OS.has_feature("editor") or OS.has_feature("devkit")`.
        Used identically in `Backend._base_url()`, `DevKit._ready()`,
        and `devkit_button.gd._ready()`.
  - `Backend.gd` `BASE_URL` constant becomes build-conditional via the
    combined check — points at the dev worker in DevKit builds (editor
    + dev export), production worker in store builds.
  - Schema for dev D1: `schema.sql` (same file as production).
  - Default seed for dev D1: `seed_devkit.sql` (new file).

## Out of scope for v1.0

  - Additive scenario composition.
  - Recording scenario playback for screenshot capture.
  - Per-scenario expected-state assertions (i.e. DevKit doesn't verify
    that the feature behaves correctly — that's manual eyeballing).
  - Time/date manipulation. Stays off the table permanently.

---

End of doc.
