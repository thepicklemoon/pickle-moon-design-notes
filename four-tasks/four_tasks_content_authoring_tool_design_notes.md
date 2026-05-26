# Four Tasks — Content Authoring Tool Design Notes

Status: LOCKED v1 (session 17).
Owner: Morgan / THE PICKLE MOON.

## Purpose

A standalone web tool for authoring, queuing, and publishing content
drops (sticker packs, patches, announcements, personal communications)
to the live Four Tasks app. Replaces ad-hoc database edits and SQL
seeding with a structured authoring flow.

Audience: Morgan only for v1.0. Designed to be handoff-friendly for
future collaborators / contractors, but no collaboration features in
v1.0 — single-user, single-password.

## Non-goals

  - Not part of the Four Tasks app. Separate codebase, separate deploy,
    separate worker, separate URL.
  - Not user-facing. Password-gated, never linked from the app.
  - Not part of DevKit. DevKit is for testing feature beats against
    seeded state; the authoring tool writes real content into the real
    production database.
  - No collaboration / multi-user / approval workflow in v1.0.

## Stack

  - **Frontend**: Single static HTML page with vanilla JS, deployed to
    Cloudflare Pages. No framework, no build step.
  - **Backend**: Separate Cloudflare Worker (`four-tasks-admin-api`),
    bound to production D1 (`four-tasks`).
  - **Asset storage**: Cloudflare R2 for uploaded PNG art (sticker
    pack assets, hero images).
  - **Auth**: Password-gated (single shared admin password, set via
    Cloudflare Worker environment variable). Session token returned
    on login, included in subsequent requests.

URL convention: `admin.thepicklemoon.com` (CNAME to Pages project) or
`four-tasks-admin.thepicklemoon.workers.dev` (Pages default). Final
choice at implementation.

## Worker separation rationale

Authoring endpoints write to a table that, once published, affects
every user's app. The protection model is fundamentally different from
user-facing endpoints (which only allow self-writes on owned data).
Mixing the two on a single worker risks:

  - Accidental exposure of admin endpoints to unauthenticated users
    via routing mistakes.
  - Production worker bundle carrying admin-only code paths that have
    no business being in user-facing builds.
  - Harder reasoning about which endpoints have which auth model.

Separate worker isolates admin code, admin auth, and admin failure
modes entirely. Same D1 binding — the authoring tool writes to the
same `content_drops` table the user-facing worker reads from.

## Content drop shapes

The authoring tool handles three content shapes, each with its own
form and preview:

### Shape 1 — Sticker + UI drop

  - Pack name, pack description.
  - Optional subscriber-only flag.
  - One or more sticker items, each with: art upload (PNG to R2),
    item name, item description, in-app price (coins).
  - Optional pack-level art (banner / hero) uploaded to R2.

Queue cadence: weekly. Cron picks the next queued sticker drop on
Thursday UTC midnight and marks it live.

### Shape 2 — Patch / bug fix / feature announcement

  - Title, body copy.
  - Optional hero image (PNG to R2).
  - Publish mode: queued (next Thursday) OR urgent (immediate).

Queue cadence: weekly when queued. Urgent mode bypasses queue,
goes live immediately, fires on every user's next morning sequence.

### Shape 3 — Single-fire personal communication

  - Templates for: founders flag granted, lifetime free granted,
    promo code redemption success.
  - Per-template: title, body copy, optional hero image.
  - Not queued. Authoring tool edits the templates. Server fires the
    relevant template per-user when the triggering event occurs
    (founders flag set, promo code redeemed, etc.).

Shape 3 is structurally different from shapes 1 and 2: it's template
editing, not queue management.

## Authoring flow (shapes 1 + 2)

Every drop authored follows the same four-step flow:

  1. **Type picker.** User selects which shape to author.
  2. **Form fill.** Type-appropriate fields, uploads pushed to R2 on
     submit.
  3. **Preview 1 — in-app render.** Shows how the drop appears in its
     final in-app location (sticker in picker, patch in news view,
     etc.).
  4. **Preview 2 — morning sequence popup.** Shows the popup as it
     would appear in a user's morning sequence, layered over the
     in-app render.
  5. **Confirm + queue.** Posts the drop to `content_drops` with
     `status='queued'` (or `status='live'` if urgent mode on shape 2).

Both previews must render before "Publish" is enabled. Confirmation
arms the publish button only after both previews have been viewed.

## Authoring flow (shape 3)

Templates are edited in place. There's no queue, no Thursday cadence.
Save = template updated, takes effect for the next triggering event.

## Queue management

The authoring tool exposes a queue view showing:

  - All queued drops in release order.
  - Current queue length ("X weeks of runway").
  - Live drops (most recent N).
  - Ability to reorder, edit, or delete queued drops.

Editing a live drop is not supported. Once live, a drop is immutable
in the database — fixes require a follow-up patch announcement, not a
silent edit.

## Cron-driven release

  - Cloudflare cron trigger fires Thursday 00:00 UTC.
  - Trigger handler runs:
    ```
    SELECT * FROM content_drops
    WHERE status='queued' AND shape IN ('sticker','patch')
    ORDER BY queue_position ASC LIMIT 1
    ```
  - If a row is returned: UPDATE status='live', released_at=NOW().
  - If queue is empty: no-op. User just doesn't get a drop popup this
    week. (Acceptable — communicate buffer status honestly in marketing
    if needed.)

The cron is a release mechanism, not a delivery mechanism. Delivery
to individual users happens at morning sequence time per the rules
below.

## Per-user delivery rules

Once a drop is `live`, individual users see it as a morning sequence
popup according to these rules:

  - **Forward-only.** A user only sees drops that became live on or
    after their `users.created_at` date. Drops live before the user
    joined are not pushed to them as popups.
  - **Per-user local time.** A drop becomes eligible for a user when
    the user's local date (per `users.timezone`) has crossed into the
    drop's release day. A Perth user sees Thursday's drop Thursday
    morning their time; a US user sees it later when their local
    Thursday rolls over.
  - **Once only.** A drop is delivered to each user at most once. Per-
    user delivery is tracked in a separate table (`content_drop_deliveries`,
    or equivalent — exact schema deferred to morning sequence work).
  - **Urgent drops bypass the local-time check.** Urgent mode (shape 2
    only) makes the drop eligible to all users immediately, regardless
    of timezone or release day.

## Decoupling: content live vs popup delivery

Critical principle: a drop being `live` and a user seeing the popup
are independent. The popup is a notification, not a gate.

  - When a drop is live, its content (stickers, prices, etc.) is
    immediately available to all users — purchasable in the store,
    visible in the picker, etc.
  - The popup is a separate beat in the user's morning sequence.
  - If the popup fails to render (bug, dismissed, network error mid-
    load), the content is still accessible. User just didn't see the
    notification.
  - There is no in-app "news history" in v1.0. Users who missed a
    popup or want to review old drops go to thepicklemoon.com (see
    "News button" below).

## News button — deferred

A v1.0 app does NOT include an in-app news button or history view.
The decision: post-launch, thepicklemoon.com hosts a news page that
reads from the same `content_drops` table via a public read endpoint.
The website's news page is the authoritative archive.

If user feedback indicates a strong desire for in-app news, revisit
post-launch. Schema supports it — `content_drops` stores every
published drop permanently — but no UI work in v1.0.

## Beat collision — hard dependency on morning sequence

A user's morning sequence may owe them: yesterday's payout + a queued
drop + a staggered disclosure beat + an urgent comm + a shape-3
personal trigger. Stacking five interruptions is not acceptable UX.

Resolution is **out of scope for this doc**. Must be designed in
`four_tasks_morning_sequence_design_notes.md` before content drops
ship. Likely needed:

  - Priority hierarchy across all popup beats.
  - Per-morning popup cap (max 2-3, others defer to subsequent
    mornings).
  - Per-user, per-beat "shown" flag.

Flagging this as a blocker on shipping any content drop in production.
Authoring tool can be built and tested against DevKit before the
morning sequence collision rules are designed; deployment to prod
blocks on the morning sequence work.

## Schema (sketch — final form locked at implementation)

Tables added by this work:

  - `content_drops` — one row per authored drop. Columns: id, shape,
    status (queued/live), queue_position, subscriber_only, payload
    (JSON of shape-specific fields), released_at, created_at.
  - `content_drop_assets` — R2 references for art uploads attached to
    drops. Columns: id, drop_id, role (sticker/banner/hero), r2_key.
  - `content_drop_deliveries` — per-user delivery tracking. Columns:
    user_id, drop_id, delivered_at.
  - `comm_templates` — shape-3 templates. Columns: id, template_key
    (founders/lifetime_free/promo_success), payload, updated_at.

Exact schema designed at implementation time. Migration follows the
"v1.0 schema is single source of truth" pattern — schema.sql edited
in place, no migration files until post-launch.

## Authoring tool UI surface

Aesthetic minimum: clean, monochromatic, readable. Doesn't have to
look like a chrome-plated SaaS dashboard, but does have to not look
like a smashed crab. Reference style: the Pickle Moon brand's existing
visual language (loader screen, app chrome).

Tabs / sections:

  - Login (password gate).
  - New drop (type picker → form → previews → publish).
  - Queue (list of queued drops, reorderable, editable, deletable).
  - Live (list of recent live drops, read-only).
  - Templates (shape-3 communication templates, editable).
  - Logout.

No analytics, no metrics, no engagement dashboards. Out of scope.

## v1.0 build order

The authoring tool is not v1.0-blocking for app launch. Estimated
sequence:

  1. App ships with morning sequence beat collision rules designed
     and implemented (Phase 3-4 work).
  2. Authoring tool built post-launch as content pipeline goes from
     "Morgan SQL-edits directly" to "Morgan authors via tool."
  3. First sticker pack drop fires through the authoring tool.

Pre-launch, the app must support reading from `content_drops` and
firing the morning sequence beats — the schema, the read endpoints,
the in-app popup rendering. Authoring side can ship later.

This doc captures the design for the full pipeline. Implementation
order follows the launch sequence.

---

End of doc.
