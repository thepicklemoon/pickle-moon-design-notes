# Four Tasks — Content Authoring Tool Design Notes

Status: LOCKED v1 (session 17). UPDATED v2 (session 21, 29 May 2026) —
authoring front-end reframed onto DevKit; pipeline split into a local
authoring stage and a server publishing stage; Aseprite-wrapper flow,
corkboard preview, boundary validation, and immutable bundle added.
Owner: Morgan / THE PICKLE MOON.

## Purpose

A pipeline for authoring, queuing, and publishing content drops
(sticker packs, patches, announcements, personal communications) to
the live Four Tasks app. Replaces ad-hoc database edits and SQL
seeding with a structured authoring flow.

Audience: Morgan only for v1.0. Designed to be handoff-friendly for
future collaborators / contractors, but no collaboration features in
v1.0 — single-user, single-password. The authoring stage is built so
that a co-creator could produce a valid pack without learning the repo
conventions; that's the longer-term justification for investing in it,
and it is post-launch work (see build order).

## Pipeline shape — two stages

The pipeline splits into two stages with one clean handoff between
them. This split is the load-bearing v2 change.

  - **Authoring stage** (local): paint the art in Aseprite, see it
    composed, assemble the elements, validate, and produce a single
    self-contained bundle. Runs on Morgan's laptop, inside DevKit.
    Writes nothing to production.
  - **Publishing stage** (server): take a validated bundle, upload art
    to R2, write the drop to `content_drops`, manage the queue, run the
    cron release, handle per-user delivery. Owns every production write.
    Web-based so the queue is manageable from a phone on a roster.

The **bundle** is the handoff artifact: a validated, self-contained,
immutable package of art + manifest + message. The authoring stage's
only output is a bundle; the publishing stage's only content input is
a bundle.

This split is *why* the original "keep authoring out of DevKit" lock
holds rather than breaks: the dangerous part (production writes) stays
isolated in the admin worker, and the authoring stage — which produces
only a local bundle — is safe to host in DevKit.

OPEN SEAM (next decision, deferred): how the bundle crosses from
authoring to publishing — DevKit POSTs the bundle directly to the
admin worker, or Morgan uploads the bundle file through the web tool.

## Non-goals

  - The publishing stage is not part of the Four Tasks app. Separate
    codebase, separate deploy, separate worker, separate URL.
  - Not user-facing. Password-gated, never linked from the app.
  - The authoring stage lives in DevKit but writes nothing to
    production — it only produces a local bundle. DevKit is excluded
    from release exports (tile 5.2), which is exactly where a dev-only
    authoring tool belongs.
  - No collaboration / multi-user / approval workflow in v1.0.
  - No AI in the render loop or the art pipeline (see "AI — rejected").

---

# AUTHORING STAGE (local — DevKit + Aseprite)

## DevKit menu placement

DevKit opens to a tool picker. Content authoring sits alongside the
other dev tools (bug reports, user dashboard, test matrix). Selecting
it enters the authoring flow for a sticker / UI pack.

## Aseprite is the paint surface — wrapped, not embedded

Aseprite cannot be embedded: it is a standalone app with its own UI
toolkit, no SDK, no canvas widget. Its source is public but
source-available under a EULA (no code reuse in a shipped product),
and the only open fork (LibreSprite) is GPL, which would force Four
Tasks itself to go GPL. Building a homegrown pixel editor was also
rejected: the visible features (draw, line, fill, layers, palettes)
are the tip; the engineering that makes a pixel editor usable —
pixel-perfect line drawing, correct layer compositing, and above all
a robust undo stack — plus 25 years of tuned *feel* are what make
Aseprite the industry standard. A worse homegrown editor would slow
the exact bottleneck (pack production volume) it was meant to fix.

So the relationship is inverted: the workflow **wraps** Aseprite using
Morgan's own license. Integration is via an Aseprite Lua extension
plus a watched export folder.

### Aseprite extension (canvas + flow)

The extension knows the slot list for the pack being authored. For
each slot it:

  - Creates a new sprite at the correct canvas size for that slot.
  - Presents the slot with **Submit / Skip** (the first slot, the
    sticker, has Submit only — a pack must contain at least its hero).
  - On Submit, exports the PNG to the watched folder with the correct
    derived name (`<id>_<role>.png`) — Morgan never types a filename.

The slot identity travels *forward* through the flow; the filename is
a derived output, not a source of truth. This is the fix for the
"filename convention backfires" failure: a convention can't backfire
if no human ever authors a filename, and the DevKit refuses to ingest
any file whose name doesn't match an expected slot.

## Watched folder + corkboard preview

DevKit watches the export folder. On each file change it reloads that
texture and re-renders the preview. The loop is **save-driven, not
per-stroke**: paint, save, see it update a beat later. Per-stroke
live rendering would need either embedding (impossible) or a socket
pumping Aseprite's canvas every frame (large plumbing for a marginal
nicety) — save-driven gives ~95% of the feel for a fraction of the
cost.

The preview is a **corkboard**, not the real app: static rects and
nine-patch nodes holding the watched PNGs, arranged in representative
compositions. **No State, no scenarios, no data dicts.** The corkboard
exists to catch art failures, and each stage catches a different class
of failure — it is, in effect, a visual test matrix for art (which is
why it sits next to the Test Matrix tool):

  - **9-slice popup** — catches scaling seams. A nine-patch pins its
    corners and stretches its edges/centre, so a border that looks
    right at one size can seam or bleed at another. This stage must
    render at two extents (a small confirm popup and a tall help menu)
    or carry a size slider. The other stages can be single fixed
    snapshots; this one cannot.
  - **Cell grid** — catches "does this read at true size among
    neighbours." Cells render at true app size in a representative
    grid, with the element being painted shown next to others. A
    sticker that is gorgeous at preview-zoom and mud at real cell size
    fails here.
  - **Background under UI** — catches legibility. The background slot
    rendered with representative UI chrome and text on top, at real
    opacity, so a busy background that kills streak-bar readability is
    caught in situ.
  - **Task tray** — catches the tray's own composition (tray surface +
    stamp slot; stamp lives on the tray, not the cell).

View modes: **all at once** (dashboard) or **focused** (just the
surface being painted).

CRITICAL: sizes must be real. The corkboard's only value is "do these
read together at correct size," which is lost if the geometry is
eyeballed. Rect dimensions, grid spacing, and nine-patch extents come
from the same `Layout` constants the live app uses. Art comes from the
watched files; geometry comes from real constants; nothing else is in
the system. (A corkboard that hardcodes its own geometry will silently
drift from the app and start lying — pull from `Layout`.)

## Assembly + edit-back

After painting through the slots, a final assembly screen lets Morgan:

  - Toggle each element on / off and view them in concert.
  - Step back to any individual slot to repaint, or back to the start.

Sparsity is allowed: a pack ships whatever slot-subset its art fills.
A pack is not required to fill every slot.

## Boundary validation

The "corruption in the queue" failure is really "something invalid got
into the queue." The fix is to make invalid state un-enqueueable by
validating hard at the boundary — at bundle time, before anything is
queued. Validation checks:

  - Completeness: every *declared* slot has art (declared ≠ all).
  - Dimensions: each PNG matches its slot's expected size.
  - Palette constraint: ≤3 non-grey colours where the theme model
    requires it (sticker.png is the colour exception per the theme
    doc).
  - ID uniqueness: the sticker / pack ID is unique against the live
    catalogue *and* against every other queued bundle.

Only art that passes every check can be bundled. The queue therefore
only ever holds already-valid things.

## The bundle (handoff artifact)

On a passing validation, the tool produces a single self-contained,
immutable bundle:

  - Art is **copied into** the bundle, not referenced in place — so
    editing a working PNG next week cannot reach back and corrupt a
    queued drop.
  - A manifest declares the slots, IDs, dimensions, prices, and
    pack-level metadata.
  - Morgan's drop message is attached.
  - A queue-position intent is attached (where in the current queue
    this drop should land).
  - The bundle is checksummed/versioned so corruption is detectable at
    push time.

"Edit a queued bundle" = pull it out, repaint/reassemble, re-validate,
produce a new bundle, re-queue. Never mutate a bundle in place. This is
the immutable-past principle applied to content — same shape as the
rest of the architecture.

## AI — rejected (with the one honest exception noted)

AI does not belong in the render loop. Rendering art onto a cell is
deterministic plumbing — a file watcher and a texture reload. An AI
agent there would be slower, non-deterministic, cost money per render,
and occasionally render *wrong*, to do a job a watcher does perfectly
and identically every time. It would make the pipeline *less*
foolproof, the opposite of the goal.

The one place AI could plausibly reduce volume is generating supporting
UI elements from a hand-painted hero sticker. Rejected for Four Tasks
specifically: hand-painted craft is the locked go-to-market (session
20 — "quality does the work advertising would"). AI-filled supporting
art won't reliably match a hand-painted hero, and the moment it's
noticeable it dilutes the made-by-hand identity that is the whole
pitch. Recorded so the option is known and consciously declined, not
forgotten.

---

# PUBLISHING STAGE (server — web tool + admin worker)

## Stack

  - **Frontend**: Single static HTML page with vanilla JS, deployed to
    Cloudflare Pages. No framework, no build step.
  - **Backend**: Separate Cloudflare Worker (`four-tasks-admin-api`),
    bound to production D1 (`four-tasks`).
  - **Asset storage**: Cloudflare R2 for art (uploaded at publish time,
    from the bundle).
  - **Auth**: Password-gated (single shared admin password, set via
    Cloudflare Worker environment variable). Session token returned on
    login, included in subsequent requests.

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
modes entirely. Same D1 binding — the admin worker writes to the same
`content_drops` table the user-facing worker reads from.

## Content drop shapes

The pipeline handles three content shapes, each with its own form and
preview:

### Shape 1 — Sticker + UI drop

  - Pack name, pack description.
  - Optional subscriber-only flag.
  - One or more sticker / UI items, each with: art (PNG, from the
    bundle → R2 at publish), item name, item description, in-app price
    (coins).
  - Optional pack-level art (banner / hero).

Shape 1 is the shape the authoring stage produces a bundle for. The
art, names, IDs, and message all originate in the local authoring flow;
the publishing stage uploads and queues the bundle.

Queue cadence: weekly. Cron picks the next queued sticker drop on
Thursday UTC midnight and marks it live.

### Shape 2 — Patch / bug fix / feature announcement

  - Title, body copy.
  - Optional hero image (PNG to R2).
  - Publish mode: queued (next Thursday) OR urgent (immediate).

Authored directly in the web tool (no art-assembly flow needed).
Queue cadence: weekly when queued. Urgent mode bypasses the queue,
goes live immediately, fires on every user's next morning sequence.

### Shape 3 — Single-fire personal communication

  - Templates for: founders flag granted, lifetime free granted,
    promo code redemption success.
  - Per-template: title, body copy, optional hero image.
  - Not queued. The web tool edits the templates. Server fires the
    relevant template per-user when the triggering event occurs
    (founders flag set, promo code redeemed, etc.).

Shape 3 is structurally different from shapes 1 and 2: it's template
editing, not queue management.

## Publishing flow (shape 1, from a bundle)

  1. **Receive bundle.** From the handoff seam (DevKit POST or web
     upload — see OPEN SEAM above).
  2. **Upload art to R2.** Bundle art written to R2; keys recorded.
  3. **Announcement-popup preview.** Shows the drop's morning-sequence
     announcement popup as a user would see it. (The art itself was
     already previewed on the corkboard during authoring; this preview
     is the *notification*, not the art.)
  4. **Confirm + queue.** Posts the drop to `content_drops` with
     `status='queued'` at the bundle's intended queue position.

## Publishing flow (shape 2)

  1. Form fill in the web tool.
  2. Announcement-popup preview.
  3. Confirm + queue (`status='queued'`) or urgent (`status='live'`).

Preview must render before "Publish" is enabled.

## Publishing flow (shape 3)

Templates are edited in place. No queue, no Thursday cadence. Save =
template updated, takes effect for the next triggering event.

## Queue management

The web tool exposes a queue view showing:

  - All queued drops in release order.
  - Current queue length ("X weeks of runway").
  - Live drops (most recent N).
  - Ability to reorder or delete queued drops. (Editing a queued shape-1
    drop's art means producing a new bundle in the authoring stage and
    re-queuing; the web tool reorders/deletes, it does not repaint.)

Editing a live drop is not supported. Once live, a drop is immutable in
the database — fixes require a follow-up patch announcement, not a
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

The cron is a release mechanism, not a delivery mechanism. Delivery to
individual users happens at morning sequence time per the rules below.

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
    user delivery is tracked in a separate table
    (`content_drop_deliveries`, or equivalent — exact schema deferred
    to morning sequence work).
  - **Urgent drops bypass the local-time check.** Urgent mode (shape 2
    only) makes the drop eligible to all users immediately, regardless
    of timezone or release day.

## Decoupling: content live vs popup delivery

Critical principle: a drop being `live` and a user seeing the popup are
independent. The popup is a notification, not a gate.

  - When a drop is live, its content (stickers, prices, etc.) is
    immediately available to all users — purchasable in the store,
    visible in the picker, etc.
  - The popup is a separate beat in the user's morning sequence.
  - If the popup fails to render (bug, dismissed, network error mid-
    load), the content is still accessible. User just didn't see the
    notification.
  - There is no in-app "news history" in v1.0. Users who missed a popup
    or want to review old drops go to thepicklemoon.com (see "News
    button" below).

## News button — deferred

A v1.0 app does NOT include an in-app news button or history view. The
decision: post-launch, thepicklemoon.com hosts a news page that reads
from the same `content_drops` table via a public read endpoint. The
website's news page is the authoritative archive.

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
Both stages can be built and tested locally before the morning
sequence collision rules are designed; deployment to prod blocks on the
morning sequence work.

## Schema (sketch — final form locked at implementation)

Tables added by this work:

  - `content_drops` — one row per published drop. Columns: id, shape,
    status (queued/live), queue_position, subscriber_only, payload
    (JSON of shape-specific fields), released_at, created_at.
  - `content_drop_assets` — R2 references for art uploads attached to
    drops. Columns: id, drop_id, role (sticker/banner/hero), r2_key.
  - `content_drop_deliveries` — per-user delivery tracking. Columns:
    user_id, drop_id, delivered_at.
  - `comm_templates` — shape-3 templates. Columns: id, template_key
    (founders/lifetime_free/promo_success), payload, updated_at.

Exact schema designed at implementation time. Migration follows the
"v1.0 schema is single source of truth" pattern — schema.sql edited in
place, no migration files until post-launch.

The bundle manifest is a local artifact, not a DB table; it is consumed
at publish time to populate `content_drops` + `content_drop_assets`.
Its exact shape is locked alongside the authoring-stage build.

## Web tool UI surface

Aesthetic minimum: clean, monochromatic, readable. Doesn't have to look
like a chrome-plated SaaS dashboard, but does have to not look like a
smashed crab. Reference style: the Pickle Moon brand's existing visual
language (loader screen, app chrome).

Tabs / sections:

  - Login (password gate).
  - New patch / announcement (shape 2 form → preview → publish).
  - Incoming bundle (shape 1 — receive/upload bundle → preview →
    publish).
  - Queue (list of queued drops, reorderable, deletable).
  - Live (list of recent live drops, read-only).
  - Templates (shape-3 communication templates, editable).
  - Logout.

No analytics, no metrics, no engagement dashboards. Out of scope.

## Build order

Neither stage is v1.0-blocking for app launch. Estimated sequence:

  1. App ships with morning sequence beat collision rules designed and
     implemented (Phase 3-4 work), plus the schema, read endpoints, and
     in-app popup rendering so the app can *receive* drops.
  2. Pre-launch / early packs go through a minimal manual pipeline
     (paint in Aseprite, name via a thin script, SQL-seed or simple
     upload). The polished authoring stage is not required to ship the
     first packs.
  3. Authoring stage built post-launch, when the recurring fortnightly
     pack cost is real and felt, and framed as co-creator infrastructure
     — that's when it pays for itself.
  4. Publishing stage (web tool + admin worker) built to consume
     bundles and manage the queue.

This doc captures the design for the full pipeline. Implementation
order follows the launch sequence; do not pull the authoring stage
forward pre-launch (a polished pipeline ships zero stickers to users).

---

End of doc.
