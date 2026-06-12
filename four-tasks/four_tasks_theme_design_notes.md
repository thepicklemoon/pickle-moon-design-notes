# Four Tasks — Theme & Sticker System Design Notes (v2)

Last edit: 2026-06-11 AWST (session 32, mobile — the THEME PREP
design conversation)

Status: v1.0 CATALOGUE LOCKED (session 32) — seven slots ship at
        launch; everything else moved to the DEFERRED CATALOGUE
        with its locked architecture preserved. COLOUR SYSTEM
        LOCKED (session 32) — up to four pixel-weighted roles plus
        a free neutral ramp; supersedes the strict three-flat-roles
        rule. RETINT MECHANISM LOCKED (session 32) — pre-bake at
        slot change. PALETTE TRANSFORM STACK reserved (session 32,
        v1.x feature, v1.0 architecture hook).

This is theme doc v2. It restructures and supersedes v1 (the
2026-05-15 edition). Everything still load-bearing from v1 is
carried here; v1's superseded models are recorded as one-line
history, not full sections. Supersession chain for the record:
  - Four-tier coverage gradient → FEATURE CATALOGUE (s7).
  - Full-replacement cell PNGs → overlay + mask (s7; now DEFERRED).
  - palette.tres sidecar / central palette table (4.G) →
    pixel-frequency derivation from sticker.png (s8).
  - Three flat roles, greys-deferred → four weighted roles + the
    canonical neutral ramp (s32, this edition).
  - Cell treatments in v1.0 → v1.x incremental (s32).

Implementation tiles: 4.14a (basic picker, pre-fork — tap-to-toggle
pool only; APPtrioc inherits), 4.14b (post-fork — long-press
context menu + per-slot theme manager, FOUR TASKS ONLY), 4.20
(palette derivation devtool), 4.D (sticker art, Phase 4b).

APPTRIOC FORK NOTE (locked s5, unchanged): the full theme system
is Four Tasks exclusive. APPtrioc snapshots at 4.14a — pool toggle
works, long-press is inert, themed-sticker markers visible as the
conversion hint.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE MODEL — PER-SLOT SELECTION FROM AVAILABLE LIBRARY
(locked s7, unchanged)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Two layers:

  LAYER 1 — STICKER POOL. Which stickers are eligible to be
    randomly slapped onto completed days (and feed the MOTD emoji
    slot — morning-seq doc Q1). Floor of 1. Toggled via TAP in the
    picker. Stored as `users.active_stickers`.

  LAYER 2 — ACTIVE THEME SLOTS. Named slots, each holding zero or
    one sticker's contribution, filled independently from the
    user's accessible library via long-press → context menu →
    element icon.

Stickers are INVENTORIES OF FEATURES, not tiers. A sticker ships
whatever subset of elements it ships; the slots it can fill are
exactly the elements it owns. The active theme is a per-slot
selection over the library: background from frog, palette from
chilli, popup skin from vampire — assembled piece by piece.

NO STACKING, NO COLLISIONS: a slot holds ONE element; picking a
new contribution evicts the old one.

INSTANT FEEDBACK, NO CONFIRMATION: every element tap applies
immediately. Theme changes are per-user, mutable, cheap (no
pair-key consequence except the leader). Exploration is the joy —
losing twenty minutes in the picker is the intended behaviour.
APPLY ALL fills every slot the sticker can fill in one tap.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE v1.0 SLOT CATALOGUE (locked session 32)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Seven slots ship at launch. The trimming principle (Morgan, s32):
the day cell stays SACRED — stock calendar look, tier colour ramp,
completion sticker on the hero moment, nothing else. The theme
speaks through everything AROUND the cells. Richness comes from
combinatorics (N stickers × 7 slots × any palette source × per-slot
mixing), not from per-cell art.

1. LEADER (`<id>_sticker.png` — required on every sticker)
   The identity sticker. Avatar slot, in-app branding, recovery
   anchor. Participates in the pair-key hash; changing it triggers
   rotation. Every sticker fills this slot.

2. PALETTE SOURCE (`<id>_sticker.png` — every sticker)
   The sticker whose painted colours tint all themed surfaces.
   Roles derived by pixel frequency — see THE COLOUR SYSTEM.
   Every sticker fills this slot.

3. BACKGROUND (`<id>_background.png` — optional)
   Full-screen layer behind everything. The largest visual
   contribution a sticker can make. Reserved for stickers with
   genuine visual ambition.

4. BORDER (`<id>_border_corner.png` OR `<id>_border_tl/tr/bl/br.png`
   — optional; NEW s32)
   Corner flourishes that gather the screen together — the
   app currently "just ends" at the panel margin; the border gives
   it an edge. Z-ORDER: above the BACKGROUND slot, below every
   card and all content. It never costs the calendar width; it may
   tuck behind the cards' edges.
   ART-ECONOMY CONVENTION: ship ONE `border_corner.png` and the
   engine mirrors it to all four corners; ship any subset of the
   four explicit `_tl/_tr/_bl/_br` files for organic asymmetry
   (explicit files win over the mirrored single where both exist).
   Aesthetic sibling of the popup skin — author them as a pair;
   the system keeps them as separate slots (mixing freedom is the
   model; cohesion is an authoring choice, not a mechanical one).

5. CALENDAR CARD (`<id>_calendar_card.png` — 9-slice; NEW s32)
   The card behind the calendar grid (currently the flat white
   panel). 9-slice in role colours + neutrals.

6. TRAY CARD (`<id>_tray_card.png` — 9-slice; NEW s32)
   The tasks-tray card surface. 9-slice, same format family as
   the calendar card — the expectation is most stickers shipping
   one ship both (clone-and-tweak is the intended economy).
   CONSTRAINTS, renderer-side not art-side:
     - The 9-slice must survive the ceremony choreography — lift,
       wobble, stamp landing, subtitle slot — all of which render
       ON the card, none of which the card art may obstruct.
     - TRAY BUTTONS STAY STOCK. [LOCK AT FIRST ART — Morgan, s32:
       "probably the card can but the buttons can't; lock once I
       see examples." Prominence of the tray's interactive
       elements wins over theming until proven otherwise.]

7. POPUP SKIN (`<id>_popup.png` — 9-slice; NEW s32, closes the
   long-flagged catalogue gap)
   One 9-slice skinning EVERY dialog/popup card surface: rest
   confirm, rescue confirm, pair dialog, the MOTD picker card,
   and the future help menu / month-stats / milestone popups as
   they land. One slot, one file, all dialogs — popups are a
   family and skin as one.

EMPTY SLOTS RENDER STOCK. There is no "default theme" sticker;
the unthemed app is the stock UI exactly as it exists today (white
cards, flat panel background, no border). The onboarding starter
pack makes the first themed look a day-zero possibility, not a
requirement.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE COLOUR SYSTEM (locked session 32 — supersedes the
three-flat-roles rule and resolves the deferred grey question)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ROLES — up to four, pixel-weighted:
  A sticker is painted with UP TO FOUR non-neutral colours. At
  load, the engine counts pixels per unique colour (neutrals and
  transparency excluded), sorts descending, and assigns:
    role-primary    = most-painted colour
    role-secondary  = next
    role-tertiary   = next
    role-accent     = fourth, IF the sticker has a fourth
  The painter expresses palette intent through pixel area: frog's
  dominant green becomes the app's dominant colour, the beige
  belly the supporting colour, the red highlight the accent.
  Painting decisions ARE palette decisions — no sidecar, no
  authoring step (s8 rule, carried).

  WEIGHTING IS THE PAINTER'S, TWICE OVER: derivation decides the
  MAPPING ORDER only; how much primary vs accent a themed surface
  shows is decided by how that surface's art is painted. A
  background that is 80% role-primary pixels renders 80% in the
  palette sticker's dominant colour.

  ROLE-ACCENT FALLBACK (s32): non-sticker theme art may use
  roles 1-3 freely; role-accent is an OPTIONAL fourth reference
  that FALLS BACK TO ROLE-TERTIARY when the active palette source
  yields only three colours. Art never breaks on a 3-colour
  sticker; "3 or 4 colours, doesn't matter" stays true for the
  painter on both sides.

THE NEUTRAL RAMP — free, structural, never retinted:
  BLACK, WHITE, LIGHT GREY, DARK GREY are usable without limit in
  BOTH stickers and non-sticker theme art: outlines, shadows,
  highlights, anti-aliasing, advanced pixel-art technique. They
  are excluded from palette derivation and never substituted.
  Canonical authoring values (locked s32; chosen as true neutrals
  near the luminances of the stock UI's slate ramp so themed art
  and stock chrome read as one family):
    BLACK        #000000
    DARK GREY    #5A5A5A
    LIGHT GREY   #B4B4B4
    WHITE        #FFFFFF

  THE EXACT "GREY" RULE (closes the v1 deferred question):
    DERIVATION-EXCLUDED = any pixel where R == G == B exactly,
    plus any pixel with alpha < 1.0. Crisp, tolerance-free,
    devtool-matchable. Painters should prefer the four canonical
    values, but ANY true neutral is legal and structural.
  ANTI-ALIASING DISCIPLINE: AA between a role colour and anything
  produces intermediate non-neutral pixels that pollute the
  derivation count and (in non-sticker art) dodge substitution.
  Rule: AA neutral-to-neutral freely; role colours stay hard-edged
  flats. The devtool's full colour list is where violations show.

REFERENCE ROLE VALUES (locked s32 — replaces the placeholders):
  Non-sticker theme art is painted in these four reference values,
  substituted exactly at bake time:
    role-primary    #FF0000
    role-secondary  #00FF00
    role-tertiary   #0000FF
    role-accent     #FF00FF
  Maximally distinct, never plausible as final art, trivially
  exact-matchable. Plus the neutral ramp, painted as-is.

STICKER.PNG IS THE EXCEPTION (carried): the sticker itself is
painted in its REAL colours and is never retinted — it is the
source the palette is derived FROM; retinting it would be
circular. It always looks like itself, in the picker and on cells.

PAINTER'S CONSTRAINT (amended for four roles):
  Up to four non-neutral colours, with CLEAR PROPORTIONAL
  DIFFERENCE between adjacent ranks — no 49/51 splits. If two
  ranks are close, adjust pixels until intent is unambiguous.
  More than four non-neutrals = graceful failure: the top four
  propagate, the rest render in the sticker only. Discipline, not
  load-time validation; the devtool catches mistakes.

SACRED SEMANTIC COLOURS (carried, unchanged): the cell tier ramp
(grey/red/orange/yellow/green), rest-day purple, error red,
warning orange, body-text near-black — hardcoded, never themed,
unaffected by palette source. The day cell's voice stays the
app's voice.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RETINT MECHANISM — LOCKED: PRE-BAKE AT SLOT CHANGE (s32)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

When the palette source (or, v1.x, the transform stack) changes,
the engine generates retinted copies of every loaded non-sticker
theme texture: per pixel, exact-match the four reference values →
substitute the derived roles (accent falling back to tertiary);
neutrals and everything else pass through. Cache the baked
ImageTextures; renderers consume the cache. Re-bake on change;
per-frame cost is ordinary sprite rendering.

WHY PRE-BAKE over the v1 candidates:
  - Clarity over cleverness: it is "make a recoloured copy when
    the theme changes" — fully legible GDScript (Image.get_pixel /
    set_pixel loops), debuggable by eye in the 4.20 devtool, no
    shader knowledge required.
  - The v1.0 art volume is small (per active theme: one
    background, two card 9-slices, one popup 9-slice, ≤4 border
    corners — a handful of small textures). Bake time and memory
    are trivial.
  - Transform stack composes for free (functions applied to the
    derived roles BEFORE baking).
  - Locked-sticker PREVIEW = bake into a separate preview cache,
    swap pointers, revert on close/timeout. No special path.
FALLBACK RECORDED: if slot-change ever visibly hitches on weak
devices as the catalogue grows, the same reference-value model
moves to a fragment shader (v1's option A) without touching any
art. Indexed-PNG (option C) is retired — workflow constraint for
no advantage over pre-bake.

THE PALETTE PIPELINE (one line to hold):
  sticker.png → derive roles (pixel count, neutrals excluded)
  → [v1.x: apply transform stack] → bake reference-painted art
  → render.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PALETTE TRANSFORM STACK — v1.x FEATURE, v1.0 HOOK (s32)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Meta stickers ship a PALETTE TRANSFORM instead of (or alongside)
art: a function over the derived role set, applied between
derivation and bake. Composable by construction — the active
transforms are an ordered list, each a pure function on four
colours. Examples (Morgan, s32):
  - jester  → permute the role mapping (primary↔tertiary scramble)
  - yin-yang → invert the palette
  - chrome  → hue-shift all roles by a fixed amount
Combining jester + yin-yang = a scrambled, inverted palette.
Toggled from the meta sticker's context menu like any element.

STATE: `active_theme.mods` — ordered JSON array of transform IDs.
The slot map stays a map; mods are the one list-shaped key (a
transform stack is inherently ordered and stacking is the point —
the no-stacking rule applies to ART slots, not to functions).

v1.0 ships the hook (the pipeline stage exists, the list is read
and applied; an empty list is a no-op). The meta stickers
themselves, their context-menu affordance, and the transform
vocabulary are v1.x content. Cost of the hook now: one function
call site. Cost of retrofitting later: re-plumbing the bake path.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ASSET DIMENSIONS (s32 — derived from Layout.gd, replacing the
v1 deferral; all render targets are the 1080×2400 design viewport)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Measured render geometry (Layout.gd constants):
  Panel content width: 1080 − 2×24 margin            = 1032
  Calendar card:       1032 wide × ~1020 high (min), r16 corners
  Cell box:            ~138 × 130 (7 cols, 6px gaps, inside the
                       card's 12px inner margin), r10 corners
  Sticker on cell:     ~100px (≈75% cell width) — 32×32 source
                       at ~3x. LOCKED s6, unchanged.
  Tray card:           1032 wide × ~935 high (min), r16 corners
  Background:          full viewport 1080×2400

AUTHORING SIZES (integer Nearest scaling; [TUNE AT FIRST ART]
markers are expected to move once real pixels exist):

  BACKGROUND          360 × 800 source @ 3x render.
                      3x keeps pixel density in the same family
                      as the ~3x sticker. [TUNE AT FIRST ART:
                      3x vs 4x (270×600) — pick whichever density
                      paints better, then lock for the catalogue.]

  CARD 9-SLICES       48 × 48 source, 16px slice margins, @ 3x
  (calendar / tray /  (corners render 48px; compatible with the
   popup)             stock r16 corner rounding). Centre patch
                      tiles or stretches — declare per file via
                      a one-line .import convention at 4.14b.
                      [TUNE AT FIRST ART: margin 12 vs 16.]

  BORDER CORNERS      96 × 96 source @ 3x (≈288px corner zone).
                      Anchored to viewport corners, may overlap
                      the background freely and tuck under cards.
                      [TUNE AT FIRST ART: size + whether edges
                      between corners ever ship — v1 is corners
                      only, per the s32 sketch.]

  RULE OF THE FAMILY: one source-pixel scale per asset class,
  integer render scale always, Nearest always. When in doubt
  match the sticker's effective ~3x so the app reads as one
  pixel-art world.

DEFERRED WITH THEIR SLOTS: cell variant sizes, grid tile size,
flourish/label bounding boxes, effect sprite sizes — specced when
those slots return (DEFERRED CATALOGUE below).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ASSET ARCHITECTURE — FOLDER PER STICKER, FILE EXISTENCE IS
DECLARATION (carried s7/s8; filename set updated s32)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Each sticker lives in `stickers/<id>/` (bundled: res://assets/
stickers/<id>/; downloaded: user://stickers/<id>/ — see RUNTIME
ASSET DELIVERY). Files use `<id>_<role>.<ext>` — prefix for
out-of-folder readability, snake_case for Godot/git/Windows
sanity.

  REQUIRED:
    frog_sticker.png          — 32×32 RGBA. Leader visual AND
                                palette source of truth.

  OPTIONAL v1.0 SET (any subset):
    frog_background.png
    frog_calendar_card.png    — 9-slice
    frog_tray_card.png        — 9-slice
    frog_popup.png            — 9-slice
    frog_border_corner.png    — single, engine-mirrored ×4
    frog_border_tl.png        — explicit corners (any subset;
    frog_border_tr.png          explicit wins over mirrored)
    frog_border_bl.png
    frog_border_br.png

  v1.x FILENAMES (recognised-and-ignored by the v1.0 engine —
  graceful forward compatibility, carried rule): the cell
  overlay/mask family, grid_tile, flourish_name,
  label_background, effects_*.tres, theme_audio.tres + .ogg,
  *_custom.gd, and the per-cell `_<N>` variant convention (stable
  date-hash selection — locked s8, parked with the cell slots).

NO MANIFEST: the engine scans the folder, strips the `<id>_`
prefix, matches role suffixes against the known catalogue, builds
the context menu from what exists. Unknown roles ignored. Adding
a feature to a sticker = dropping a file. (Carried verbatim in
spirit from v1 — this rule is the catalogue model's engine.)

A minimal sticker ships sticker.png alone. A maximal v1.0 sticker
ships seven files. Sparsity is a design axis, not a deficiency —
the catalogue is enriched by variety in sticker SHAPE.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RUNTIME ASSET DELIVERY — R2 (NEW s32; the s29-flagged gap)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The 4.22 receive side records `content_drop_assets.r2_keys` but
nothing client-side can fetch or render them — without this
design, post-launch packs deliver announcements, not art. The
design:

  - LAUNCH PACKS ARE BUNDLED in the binary (res://). R2 delivery
    is for POST-LAUNCH drops only. One code path renders both:
    the folder-scan looks in res:// first, then user://stickers/.
  - MANIFEST: the drop's `content_drop_assets` rows ARE the
    manifest — one row per file, `r2_keys` the fetch path,
    filenames following the `<id>_<role>` convention. No second
    manifest format.
  - FETCH MOMENT: on drop-popup acknowledgement (the 4.22 client
    half), the client downloads the pack's files to
    user://stickers/<id>/ with a small progress affordance, then
    acks delivered. Explicit, attended, once.
  - IMMUTABILITY: a sticker id's assets never change after
    release (immutable-past extends to art — a re-release is a
    new id). Therefore: cache forever, no versioning, no
    invalidation machinery. Integrity = byte-length check against
    the asset row; hash verification is v1.x if ever needed.
  - PARTNER-RENDER RULE: partner days render with the PARTNER's
    `day_theme_state`, so the viewer may encounter sticker ids
    they don't own and were never offered. Asset PIXELS are not
    the secret — ENTITLEMENT gates use, not sight (the locked
    preview already shows everything). The client may fetch any
    referenced sticker's assets by id (`GET .../stickers/:id/
    assets` or direct public R2 paths — endpoint shape decided at
    build). Missing-while-offline degrades to stock render +
    retry on next poll. Same rule covers the user's own
    historical days after a reinstall.
  - APPtrioc: no R2 path — its sticker set is bundled, full stop.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PICKER UX (carried s7/s8; icon vocabulary re-cut for v1.0)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The picker is the single management surface. TAP = pool toggle
(layer 1). LONG-PRESS (vibration) = context menu (layer 2,
Four Tasks only, trigger-gated reveal on first picker open per
the staggered-disclosure doc).

CONTEXT MENU: icon-per-shipped-element, no text. v1.0 vocabulary
(placeholder glyph concepts; final icon art at 4.14b):
  LEADER · PALETTE · BACKGROUND · CALENDAR CARD · TRAY CARD ·
  POPUP SKIN · BORDER — plus APPLY ALL.
LOCKED sticker → single PREVIEW button: bakes the sticker's full
contribution into the preview cache, applies live, reverts on
menu close or a 30-SECOND TIMEOUT (locked s32 from the v1
"say 30s" — generous enough to tap around and watch it breathe).
Locked stickers render ENTICING, never greyed — locked status is
revealed only on long-press (carried rule; exact treatment is
4.14b art).

THEMED-STICKER MARKING (carried): stickers shipping more than
sticker.png are marked in the grid — richness hinted, specifics
unspoiled, identical for locked/unlocked. Marker style is 4.14b
art; it is ALSO the APPtrioc conversion hint, so it must read in
the HTML5 build.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PALETTE DERIVATION DEVTOOL (tile 4.20; carried, synced to s32)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

DevKit sub-screen, READ-ONLY by design (no override hatch — wrong
ordering is fixed by re-painting; the PNG stays the only truth).
Minimum view, updated for s32:
  - Load a sticker.png (file dialog / drag-drop), show at ~6x.
  - Swatch panel: primary / secondary / tertiary / ACCENT (when
    present) — swatch + RGB + pixel count each.
  - Full non-neutral colour list, sorted, with counts.
  - AMBIGUITY BADGE when adjacent ranks are within 10% pixel
    count (starting threshold, tune against real packs).
  - NEUTRAL AUDIT (new): list any near-neutral non-neutrals
    (R≈G≈B but not exactly equal — usually AA accidents that will
    pollute derivation) so they get cleaned before shipping.
THE EXCLUSION FUNCTION IS SHARED: the devtool and the runtime
renderer call the SAME derivation function (R==G==B + alpha
exclusion, four-role sort). Write it once, validate by eye in the
tool, wire it into the bake path. Build order pinned at the tile.
Deferred enhancements carried from v1: live themed-surface
preview, side-by-side, batch report, histogram.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FONT — GLOBAL, NOT THEMED (carried s7, condensed)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

All typography is global — the MOTD font especially (the app's
voice; per-theme type would dissolve brand identity into skin).
Themed surfaces change what's BEHIND text, never the text itself.
Readability + glyph-coverage insurance. Reversible in v1.x only
on demonstrated engagement evidence.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SUBSCRIPTION INTERACTION (carried s8/v1, condensed; economy doc +
monetisation doc remain the commercial authority)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  - ACCESSIBLE LIBRARY = own purchases + partner's purchases when
    the pair holds ≥1 active subscription. Access is BINARY
    per-sticker — in the library means every element usable.
  - CATALOGUE GATE: regular monthly drops are subscriber-only
    purchasable; free users keep launch-era packs + occasional
    thank-you drops. Unsubscribing keeps what you bought, locks
    new drops.
  - IMMUTABLE PAST: `days.day_theme_state` freezes the FULL slot
    map at seal (shipped, v1.0 schema). Sealed days render their
    era's theme forever, sub state notwithstanding. The
    PARTNER-RENDER RULE (runtime delivery section) is what makes
    this hold across devices and viewers.
  - PREVIEW IS THE MARKETING: locked preview applies everything
    live; the user's own taste does the persuasion. No nag.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
COMPLETED-MONTH CHROME — THE MONTH WEARS ITS LAST OUTFIT
(locked s35; replaces the CUT month stamp — see
four_tasks_month_stats_design_notes.md for the cut rationale)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Cells freeze per-day (IMMUTABLE PAST), but the SCREEN-LEVEL slots
(background, calendar card, tray card, popup skin, border) are not
per-cell — without a rule, browsing a past month would dress it in
the CURRENT theme, silently rewriting history. The rule:

  A COMPLETED month (strictly before the current local month)
  renders its screen-level slots from the `day_theme_state` of
  the month's LAST SEALED ACCOUNTED day — the last day the user
  actually showed up, in the outfit they were last seen in.

  - ACCOUNTED (accounted_for = 1), not merely sealed: span-
    sealing backfills never-opened gap days as sealed grey with
    accounted_for = 0 and bakes the CLAIM-era theme into them; a
    month abandoned in week two must wear week two's look, not
    the look of the eventual return. Anchoring on accounted
    days makes that automatic.
  - PRE-SEAL INTERIM: the month's final accounted day freezes
    its state only when the next claim seals it. Until then the
    just-completed month renders live theme and SELF-HEALS at
    the claim — no special case, re-derive-never-remember.
  - FALLBACK: no sealed accounted day in the month (or '{}'
    states from pre-theming history) → empty slot map → stock
    render, by the existing absent-key rule. Honest, no code.
  - CURRENT MONTH: always live theme — it's still being dressed.
  - PARTNER MONTHS: identical rule against the PARTNER's day
    states (you can see what combo they were running in April).
    This is the existing PARTNER-RENDER RULE doing its job —
    entitlement gates use, not sight; asset fetch by id covers
    stickers the viewer doesn't own.
  - CELLS are untouched by this section: they keep their own
    per-day states. A mid-month theme swap patchworks the cells
    inside a chrome frozen to the month's final look — that
    patchwork is the record, not a bug.

KEEPSAKE EXPORTING (image/share artefact) is a v1.x consideration
and deliberately NOT designed here; this section is pure render
policy, no new data, no server work.

DISCLOSURE TIE-IN: a one-shot month-close note (last day of the
month or its eve — "today's look seals the month" framing, copy
TBD) is registered in the staggered-disclosure doc, trigger-gated.

BUILD: lands inside tile 4.14b's theme-manager scope — the
renderer already has to source per-day states for cells; month
chrome adds "which day's state feeds the screen-level slots" as
one selection rule. BUILD-TIME CHECK: confirm the days payload
ships `accounted_for` to the client; if not, it's a one-line
payload addition at the next index.ts unfreeze.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SCHEMA (shipped tile 1.3 — unchanged by s32)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  users.active_leader   TEXT NN            -- pair-key participant
  users.active_theme    TEXT NN DEFAULT '{}'  -- JSON slot map
  users.active_stickers TEXT NN DEFAULT '[]'  -- JSON pool array
  days.day_theme_state  TEXT NN DEFAULT '{}'  -- frozen at seal

active_theme keys at v1.0: palette, background, calendar_card,
tray_card, popup, border (+ the reserved `mods` array, v1.x).
Absent/null key = empty slot = stock render. New slots are new
JSON keys — no migration, by design. Leader stays its own column
(identity hash participant); everything else is theme state, no
rotation on change.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DEFERRED CATALOGUE — v1.x, ARCHITECTURE PRESERVED (s32)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Everything below is DESIGNED and PARKED. Ship incrementally once
the v1.0 surface is painted and lived-in; nothing here blocks
launch. The locked architecture is preserved so returning to a
slot is a build task, not a re-design.

CELL TREATMENTS (deferred s32 — Morgan: the cell is the most
daunting, smallest surface; deferring it un-obscures the whole
vision). Locked architecture preserved verbatim in spirit:
  - Overlay + mask, NO underlay (s7): `cell_overlay_<state>.png`
    draws ON TOP of the stock cell; `cell_mask_<state>.png`
    alpha-cuts its silhouette. Z-pipeline: stock fill (masked) →
    stock content (date, pips) → overlay → completion sticker.
    The cell face carries NO stamp (sticker on the cell, stamp on
    the tray — s14 correction stands).
  - State buckets: unmarked / completed / partial / rest (+ dead).
    OPEN AT RETURN: bucket-vs-nine-states split (one completed
    overlay for the whole done ramp, or completed=4/4 with
    partial=1-3). The s32 grilling's recommendation on record:
    coarse buckets — the overlay is atmosphere; the tier already
    speaks through the sacred stock colour.
  - Per-cell `_<N>` variants by stable date-hash (s8).
  - DEAD-CELL treatment defers with the family.

GRID TILE — deferred: background + calendar card own that
territory in v1.0; a third stacked layer must earn its way back.

FLOURISH NAME, LABEL TREATMENT — deferred; small surfaces, real
charm, after the big five prove the system.

ACTIVE + AMBIENT EFFECTS — deferred with their locked rules
preserved: declarative .tres + coordinator autoload (signal-
driven, event-routed); ambient only on visible viewport, pause on
background, global particle cap with silent degrade; reduce-
motion accessibility branch (ambient off, active simplified);
effect art is reference-painted and palette-baked like all theme
art; effects_custom.gd escape hatch, bias against. The .tres
schemas + coordinator design get their own pass when effects
return.

THEME AUDIO — deferred (audio is v1.1 app-wide regardless).
Locked principles preserved: ~3-4MB per audio pack argues runtime
download (the R2 path above now exists for exactly this); ambient
loops stop on background, stingers foreground-only; LICENSING —
original, documented royalty-free, or verified PD, never
ambiguous, ever; mandatory global audio toggle, opt-out beats any
theme; audio ships on SELECT packs as content events, not per
pack.

META MODS — the transform stack above; the hook is v1.0, the
stickers are v1.x.

ANTICIPATED FUTURE SLOTS (s8 footnote, carried compressed —
parking spots, not promises; the bias stays "make it work within
current slots"):
  - cell_underlay (glow/halo — the one effect overlay-above can't
    do), grid_revealed_image (cells as windows into one
    continuous image; z probably below stock fill, above
    background), position-modular cell variants (even/odd,
    weekday-aware), cell_custom.gd escape hatch.
  - REJECTED and staying rejected: image-extends-beyond-cells
    (theatre), per-frame variant randomisation (kills "this is MY
    May 15" — stable date-hash is the locked answer).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHAT THIS DOC LEAVES OPEN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Marked [TUNE AT FIRST ART] above — settle with real pixels, then
lock here:
  - Background source density (3x vs 4x).
  - 9-slice margins (16 vs 12) + centre tile-vs-stretch defaults.
  - Border corner size; whether edge strips ever join the corners.
  - Tray buttons stock-vs-skinnable [LOCK AT FIRST ART, leaning
    stock].
  - Devtool ambiguity threshold (start 10%).
Owed at 4.14b (build-time, not art-blocking):
  - Final context-menu icon art + menu visual design.
  - Themed-sticker marker style (must read in the APPtrioc HTML5
    build).
  - Exact endpoint shape for the partner-render asset fetch.
  - 9-slice centre-mode declaration convention.
Deferred with their features: everything in the DEFERRED
CATALOGUE; thank-you-drop mechanism (monetisation doc's to spec).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHAT MORGAN PAINTS, PER PACK (the practical summary)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

A maximal v1.0 pack of one sticker:
  1× sticker.png        32×32, real colours, ≤4 non-neutrals +
                        neutral ramp, proportional clarity
  1× background.png     360×800, reference values + neutrals
  1× calendar_card.png  48×48 9-slice, reference values
  1× tray_card.png      48×48 9-slice (clone-and-tweak the above)
  1× popup.png          48×48 9-slice (same family)
  1-4× border corners   96×96 each (one file = auto-mirrored)
A minimal pack: sticker.png alone. Everything between is valid.
Open Aseprite.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED DOCS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  - four_tasks_pair_key_design_notes.md — active_leader is the
    identity/rotation participant; all other slots are free
    writes.
  - four_tasks_staggered_disclosure_design_notes.md — long-press
    reveal is trigger-gated on first picker open; Four Tasks only.
  - four_tasks_architectural_preference.md — no-sidecar derivation
    (s8) and the pre-bake mechanism (s32) are both this rule
    applied.
  - four_tasks_morning_sequence_design_notes.md — MOTD emoji slot
    reads the pool; the tray-card slot must respect the ceremony
    choreography.
  - four_tasks_monetisation_position.md + economy redesign doc —
    library access, catalogue gate, immutable past, pack pricing.
  - four_tasks_sticker_pack_brainstorm.md — content-side pack
    planning under the catalogue model.
  - system map §6.7/§7 — Palette.gd sacred-vs-themed split; the
    bake cache becomes the themed side's source at 4.14b.
