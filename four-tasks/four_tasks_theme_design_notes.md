# Four Tasks — Theme & Sticker System Design Notes

Status: FOUNDATION LOCKED (session 5)
        STICKER CANVAS SIZE LOCKED (session 6 mobile)
        TIER 0 ANIMATED PACKS ADDED (session 6 mobile)
        FEATURE-CATALOGUE MODEL LOCKED (session 7 mobile) —
          supersedes the four-tier coverage gradient. Each sticker
          ships whatever subset of features it ships, no tier
          label. Per-slot selection from the user's accessible
          library at runtime.
        CELL TREATMENT OVERLAY + MASK ARCHITECTURE LOCKED
          (session 7 mobile) — supersedes the implied "full
          replacement" cell PNG model. Themed cells are stock
          cells decorated by an overlay layer on top + optionally
          a mask cutting the silhouette. No underlay. Z-order
          pipeline documented under the CELL TREATMENT slot.

Schema: NEW columns required at theme-system implementation —
        see "Schema implications" section.
Implementation tiles: 4.14a (basic picker, pre-fork — tap-to-toggle
pool membership only), 4.14b (post-fork — long-press per-element
context menu + per-slot theme manager, FOUR TASKS ONLY), 4.D
(sticker art).
Tile 4.G (central palette table) is SUPERSEDED — see
"Schema implications" below.

NOTE ON THE APPTRIOC FORK (locked session 5):
  The full theme system (per-slot selection via long-press context
  menu) is FOUR TASKS EXCLUSIVE. Tile 4.14b lands post-fork;
  APPtrioc snapshots at tile 4.14a and never gets the context
  menu. APPtrioc users see the picker but long-press is inert
  (themed-sticker markers visible to hint at what Four Tasks
  unlocks). The pool toggle (tap) IS available in APPtrioc. See
  top of four_tasks_godot_todo.txt "APPTRIOC FORK ARCHITECTURE"
  section.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PROBLEM
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The web prototype's theme system is the app's selling point in
embryonic form. Long-press the top-left emoji, the picker opens,
long-press an emoji inside the picker, that emoji becomes the user's
"header" — its dominant colours are extracted via off-screen canvas
pixel sampling and applied to the app background as a gradient. The
streak bar gets a tinted accent. The loading screen gets a single
colour. Everything else (day cells, task list, popups, util bar,
calendar grid) stays hardcoded slate/zinc neutrals.

It's a clever proof of concept that demonstrates the principle:
change one emoji, the app reskins. But the actual reskin is small —
only two UI surfaces participate — and the mechanism is fragile
(relies on browser emoji font rendering, requires a setTimeout to
wait for paint, depends on canvas pixel readback semantics that have
no Godot equivalent).

The Godot port replaces this with a fully hand-crafted system. Every
sticker is hand-painted. Every theme element is hand-painted.
Colours are tagged at art time, not extracted at runtime. The
picker becomes the primary interface for personalising the app. The
reskin is no longer cosmetic — it touches calendar cells, grid
tiles, decorative flourishes, the whole chromatic identity of the
app, and optionally extends to ambient atmosphere and audio.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE MODEL — PER-SLOT SELECTION FROM AVAILABLE LIBRARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The theme system has two layers:

  LAYER 1 — STICKER POOL.
    Which stickers are eligible to be randomly slapped onto days
    where the user completes all four tasks. Floor of 1 — every
    completed day needs something to celebrate with. Same
    behaviour as prototype's activeEmojis. Toggled via TAP on a
    sticker in the picker grid. ALSO feeds the MOTD's emoji slot
    (session 5 Q1 lock — see morning sequence doc). Stored
    server-side as `users.active_stickers` (JSON array).

  LAYER 2 — ACTIVE THEME SLOTS.
    A set of named slots, each of which holds zero or one
    sticker's contribution. The user fills slots from their
    accessible library by long-pressing a sticker in the picker
    and tapping the relevant element-icon in the context menu
    that appears.

Each sticker advertises whichever features it ships. The slots
that sticker can fill are exactly the elements it owns. Stickers
are not categorised into tiers; they are inventories of features.
A frog sticker might ship seven element files. A chilli sticker
might ship one. Both are valid stickers — their identity is
defined by what they contribute, not by which tier-bucket they
fall into.

THE ACTIVE THEME IS A PER-SLOT SELECTION OVER THE LIBRARY:
  The user's live app at any moment is composed of one selected
  sticker per slot, picked independently. Each slot pulls from
  whichever stickers in the library ship that element.

  Example active state:
    leader        = frog
    palette       = chilli
    background    = vampire
    cell treatment = (empty — none of the user's stickers ship one)
    active effect = chilli
    ambient effect = frog
    music         = vampire
    label treatment = (empty)
    dead-cell treatment = (empty)

  The user assembled this by long-pressing each contributing
  sticker and tapping that sticker's relevant element-icon. Each
  tap immediately applies to the live app — no confirmation. The
  user is free to swap any slot at any time, building up a
  personal aesthetic from the available pieces.

NO STACKING. NO COLLISIONS.
  A slot holds ONE element. Picking a new sticker's contribution
  for a slot evicts whatever was previously in that slot. Tap
  frog's background icon then chilli's background icon → the
  background is now chilli, frog is no longer contributing to
  background.

  This means there's no "frog water + chilli sizzle splashing
  together" problem to design around. Each slot has a clean,
  single source. If the user wants the splash, they pick frog
  for active effect. If they want sizzle, they pick chilli for
  active effect. They can't have both because there's only one
  active-effect slot, period.

THE LIBRARY IS A MENU PER SLOT.
  Owning many stickers means having many *options* per slot —
  each slot lists which stickers in the library can fill it. Tap
  to swap. The user's library is a pantry; the active theme is
  a meal assembled from it.

INSTANT FEEDBACK, NO CONFIRMATION.
  Tap an element-icon in the context menu → the slot updates
  immediately and the app re-renders. No "are you sure" modal.
  No commit-on-close pattern from the v2 identity picker — that
  pattern exists because identity changes trigger pair-key
  migrations. Theme changes don't. They're per-user, mutable,
  cheap.

  This is intentional. The point of the system is *exploration*.
  You should be able to lose 20 minutes in the picker trying
  combinations, seeing what resonates, swapping back and forth.
  The instant-feedback loop is the joy.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE SLOT CATALOGUE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The full set of named slots the theme system supports. Each is
filled independently from the user's accessible library. Each
expects a specific element file in a contributing sticker's folder.

LEADER (sticker.png required — every sticker has this slot)
  The sticker chosen as the visual identity of the calendar.
  Renders in the avatar slot, contributes to in-app branding.
  Every sticker can fill this slot because every sticker ships
  sticker.png.

PALETTE SOURCE (palette.tres required — every sticker has this slot)
  The sticker whose tagged palette tints the themed UI surfaces.
  Three named roles: primary, secondary, tertiary. The renderer
  consumes these roles, substituting reference-palette pixels in
  rendered art with the active palette source's tagged colours.
  Every sticker can fill this slot because every sticker ships
  palette.tres.

BACKGROUND (background.png — optional)
  Full-grid background layer rendered behind the calendar area.
  This is the largest visual contribution a sticker can make.
  Frog's swamp water overlapping day tiles, vampire's crumbling
  castle bricks, cooking pack's hearth glow. Big surface, big
  impact when shipped. Most stickers will NOT ship this — it's
  reserved for stickers with genuine visual ambition.

CELL TREATMENT (overlay + mask architecture — locked session 7)
  Per-state day cell decoration. Each cell state can ship two
  optional files:

    cell_overlay_<state>.png  — drawn ON TOP of the stock cell.
                                Decoration, linework, vines,
                                bricks-as-outline, frost creeping
                                in from corners, etc. The stock
                                cell renders normally underneath
                                (date number, partial pips, stamp).
    cell_mask_<state>.png     — alpha mask CUTTING the stock cell.
                                Burn-from-bottom, claw-marks,
                                irregular edges. Black pixels in
                                the mask cut holes in the stock
                                cell silhouette. Used sparingly
                                for shape effects that addition
                                alone can't achieve.

  <state> is one of: unmarked, completed, partial, rest. A sticker
  ships any subset of overlays and any subset of masks. Most
  stickers won't ship either — both are decorative flourishes, not
  requirements. Stickers that DO ship treatments are the visually
  ambitious ones.

  NO UNDERLAY SLOT. Tinted backdrops behind cells are achieved
  through the palette source slot (which retints the stock cell
  fill). A separate underlay layer would be overkill for the
  decorative effect overlays already provide.

  Z-ORDER PIPELINE (locked session 7):
    1. Stock cell fill (with mask applied if present) + border
    2. Stock cell content — date number, partial-task pips
    3. Stamp art (when claimed)
    4. Theme overlay (cell_overlay_<state>.png if shipped)
    5. Completion sticker (when present, randomly slapped)

  Overlay sits above stamp, below the completion sticker. Sticker
  stays the loudest visual element on completed days; overlay is
  atmospheric decoration.

DEAD-CELL TREATMENT (cell_overlay_dead.png + cell_mask_dead.png —
                     optional, rarely shipped)
  The visual treatment for days outside the current month view (the
  April end-dates visible when looking at May, or pre-install days).
  Same overlay + mask architecture as the in-month cell treatments.
  Rose vines growing over the dead cell, jail bars, faded wash with
  shape-cutting masks, dried leaves — whatever the theme calls for.
  Most stickers won't ship this. When shipped, it's the kind of
  detail that turns casual users into gratitude posters.
  Effort:profit is low but signal value is high. Optional by
  design — the catalogue model lets it be a sparingly-used flourish
  rather than a per-pack requirement.

GRID TILE (grid_tile.png — optional)
  Repeating background tile for the calendar grid itself.
  Distinct from BACKGROUND — grid_tile is bound to the calendar
  area's structure, background is the layer behind it. Frog
  might ship swampy-water-ripples as grid_tile and a wider swamp
  vista as background.

FLOURISH NAME (flourish_name.png — optional)
  Decorative element placed near the user's name in the UI.
  Frog's fly hovering above the name, vampire's bat silhouette,
  fairy's sparkle trail. Small surface, easy to ship, signals
  attention to detail.

LABEL TREATMENT (label_background.png — optional)
  A themed surface behind each task label. Medieval parchment
  scroll, blood spatter, lily-pad outline, chalkboard texture.
  Resolves the "raw CSS popping out of a castle calendar"
  problem flagged in session 7 conversation. Subtle but
  unifying. Doesn't touch font (font is locked global — see
  below).

ACTIVE EFFECT (effects_active.tres — optional)
  Event-triggered animation that plays on user interaction.
  Water splash when tapping a cell (frog), sizzle when pressing
  a button (chilli), drip when claiming the morning sequence
  (vampire), sparkle trail when selecting a sticker (fairy).
  Single playback per trigger, fire-and-forget, low performance
  cost. Each active effect declares which UI events it responds
  to — cell tap, button press, sticker select, claim, etc.

AMBIENT EFFECT (effects_ambient.tres — optional)
  Continuous, atmospheric animation. Lily pads bobbing and
  drifting, candle flames flickering, dust motes drifting from
  castle bricks, bees visiting flowers, embers floating up from
  a fire. Always-on while the relevant scenes are visible.
  Performance-budget-significant — see "Performance rules" below.

THEME AUDIO (theme_audio.tres + .ogg files — optional, rarely shipped)
  Audio contributions. Can include:
    - Ambient loop (e.g. distant frog croaks for the swamp)
    - Stinger sounds at specific moments (e.g. fire crackle on
      morning sequence claim for cooking pack)
    - UI sound replacements (button press sounds, sticker
      placement sounds)
  Theme audio is the heaviest content axis — see "Audio
  considerations" below.

OPEN-EXTENSION:
  The slot list is not closed. As future stickers prove out new
  element ideas (a stamp variant per sticker, a custom morning-
  sequence transition, a custom 4T-complete celebration), new
  slots can be added. The folder convention naturally supports
  this — drop a new optional filename, the system discovers it,
  the picker grows a new icon. No central manifest to update.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FONT — GLOBAL, NOT THEMED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

MOTD font: LOCKED GLOBAL. The MOTD is the most-read text surface
in the app and the strongest expression of the app's voice.
Per-theme MOTD typography would dilute brand identity in service
of per-theme atmosphere. The MOTD content already changes per
theme via the stinger/vocabulary pool; changing the typography on
top of that reduces the app's identity to nothing but a skin.

The locked font is part of the global aesthetic — pix-art-adjacent
but not too blocky, intended to gel with the painted sticker
aesthetic. This font choice is itself a cohesive force binding all
themes together regardless of how visually different they are.

Decision is reversible in v1.x if per-theme typography demonstrates
clear engagement lift, but locked global for v1.0.

ALL OTHER UI FONTS: also global. Task labels keep the global font
even when LABEL TREATMENT is themed. The label's *background*
changes per theme, the label's *typography* does not. This
maintains readability across themes and avoids glyph coverage
risks (user-typed task content may include accented characters,
emoji, unusual punctuation).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ASSET ARCHITECTURE — ONE FILE PER FEATURE, FOLDER PER STICKER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Each sticker lives in its own folder under `res://assets/stickers/<id>/`.
The folder may contain any subset of:

  REQUIRED:
    sticker.png        — the sticker itself, 32×32 RGBA
    palette.tres       — tagged colour roles

  OPTIONAL (any subset):
    background.png         — full-grid background layer
    grid_tile.png          — calendar grid background tile
    cell_overlay_unmarked.png   — overlay drawn on unmarked cells
    cell_overlay_completed.png  — overlay drawn on completed cells
    cell_overlay_partial.png    — overlay drawn on partial cells
    cell_overlay_rest.png       — overlay drawn on rest day cells
    cell_overlay_dead.png       — overlay drawn on out-of-month cells
    cell_mask_unmarked.png      — alpha mask cutting unmarked cells
    cell_mask_completed.png     — alpha mask cutting completed cells
    cell_mask_partial.png       — alpha mask cutting partial cells
    cell_mask_rest.png          — alpha mask cutting rest day cells
    cell_mask_dead.png          — alpha mask cutting dead cells
    flourish_name.png      — decoration near user's name
    label_background.png   — surface behind task labels
    effects_active.tres    — tap/event-triggered animation spec
    effects_ambient.tres   — continuous ambient animation spec
    theme_audio.tres       — audio pack spec
    *.ogg                  — audio files referenced from theme_audio.tres
    effects_custom.gd      — custom GDScript for effects (escape hatch)

A minimal sticker ships sticker.png + palette.tres. A maximal
sticker ships everything. Most ship somewhere in between, choosing
what makes sense for their identity. Sparsity is its own design
axis — a minimal sticker that ships only an exquisite tap-sizzle
can be more compelling than a maximal one that ships everything.
The catalogue is enriched by variety in sticker *shape*, not just
sticker *count*.

FILE EXISTENCE IS DECLARATION:
  No central manifest declaring which sticker has which features.
  The engine scans the sticker folder at load and builds the
  context-menu element-icon list from what's present. Adding a
  new feature to a sticker is dropping a new file in its folder.
  Removing a feature is deleting the file. Zero metadata
  bookkeeping.

WHY FOLDER-PER-STICKER, NOT SPRITE-SHEET-PER-STICKER:
  Sprite sheets force a fixed slot layout up front. The catalogue
  model explicitly rejects fixed slot layouts. Folder-per-sticker
  lets each sticker bring exactly what it brings. Aseprite
  workflow matches: one Aseprite project per asset, exports go
  into the sticker's folder side-by-side.

WHY OPTIONAL EVERYTHING:
  The catalogue model rejects the idea of a "full theme" bar that
  every featured sticker must clear. Demanding completeness either
  caps the catalogue size at "however many full themes can be
  painted" or forces shipping low-effort filler to fill slots.
  Both kill content cadence. The optional model lets every new
  sticker contribute whatever it can, sustainably, indefinitely.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PICKER UX
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The picker is the single surface where the entire theme system is
managed. No dedicated "atmosphere mixer" screen.

GESTURES:
  - TAP on a sticker → toggles pool membership (Layer 1 — the
    active_stickers pool). Floor of 1, identical to prototype and
    APPtrioc.
  - LONG-PRESS on a sticker (with vibration feedback) → opens the
    sticker's CONTEXT MENU.

THE CONTEXT MENU:
  A small popover anchored to the long-pressed sticker showing
  element-icons for every feature that sticker ships. Visual
  language only — icons, no text — for screen economy and style.

  If the sticker is LOCKED (not owned, or partner-owned and the
  user's subscription has lapsed):
    Single PREVIEW button. Tap → temporarily applies ALL of the
    sticker's elements to the live app so the user can see what
    they're missing. Reverts when the menu closes (or after a
    timeout).

  If the sticker is UNLOCKED:
    One icon per element the sticker ships. Tap an icon →
    immediately applies that element to the relevant slot in the
    live app. No confirmation. No commit-on-close. Just send it.

    Plus an APPLY ALL button → fills every slot this sticker can
    fill, evicting whoever was previously contributing to each.

ICON VOCABULARY (placeholder — deferred to tile 4.14b implementation):
  Real iconography decisions belong at implementation time once
  the visual style is concrete. Placeholders for design discussion:

    LEADER             — sticker thumbnail itself or a flag icon
    PALETTE SOURCE     — artist's palette icon
    BACKGROUND         — rectangle / picture-frame icon
    CELL TREATMENT     — grid / 3-cell icon
    DEAD-CELL TREATMENT — faded cell icon
    GRID TILE          — small repeating tile pattern
    FLOURISH NAME      — decoration / ribbon icon
    LABEL TREATMENT    — text-with-underline icon
    ACTIVE EFFECT      — sparkle icon
    AMBIENT EFFECT     — drifting-particles icon
    THEME AUDIO        — quarter-note icon

  These are placeholders. The final icons should be visually
  consistent with the app's painted aesthetic and immediately
  legible at icon-size. Defer the actual icon design to a focused
  visual-design pass when tile 4.14b is ready.

THEMED-STICKER MARKING IN THE GRID:
  Stickers that ship more than the bare minimum (sticker.png +
  palette.tres) are visually marked in the picker grid — they
  have more to offer than palette-only stickers. The marking
  hints at richness without spoiling what specifically the
  sticker ships. Discovery preserved.

  Exact marking is an art decision (glint, corner badge, subtle
  border, small animation). Defer to tile 4.14b implementation.

  Note: the marking does NOT differentiate between locked and
  unlocked stickers in the user's view. The user always sees
  what's in their accessible library marked the same way; locked
  status is revealed only on long-press (when the context menu
  shows preview-only).

STAGGERED DISCLOSURE OF THE LONG-PRESS GESTURE:
  Long-press in the picker is a slow-reveal feature per the
  staggered disclosure doc. Day-1 onboarding teaches TAP only
  (pool toggle). Long-press is revealed at day 7+, surfaced
  explicitly: "you've collected a few stickers — did you know
  you can long-press to combine their elements?" Until then,
  long-press is functional but unsurfaced.

  Four Tasks only. APPtrioc never reveals long-press — for
  APPtrioc users, long-press is inert.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE PALETTE-AGNOSTIC ART RULE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

All theme art (including the sticker itself) is painted in a
canonical reference palette. At runtime, the renderer substitutes
the reference palette for the active palette source's tagged
colours.

This means:
  - The frog sticker is not painted in literal frog-green. It is
    painted in (reference-primary, reference-secondary, reference-
    tertiary) — three flat reference colours. The palette.tres
    file says "for this sticker, reference-primary = mossy-green,
    reference-secondary = wet-belly cream, reference-tertiary =
    dark-water."
  - When frog fills the palette source slot, the renderer loads
    frog's palette and substitutes. The sticker renders in its
    natural colours.
  - When chilli fills the palette source slot instead, the
    renderer loads chilli's palette and substitutes — the frog
    sticker renders in chilli's colours. Red frog, cream belly,
    dark stem-green eyes.

THIS APPLIES TO ALL RENDERED THEME ART:
  - The sticker itself when it appears as a completion sticker
    on a day cell.
  - All cell treatment variants.
  - Background and grid tile.
  - Flourishes.
  - Label backgrounds.
  - Active and ambient effect art.

THE ONE EXCEPTION:
  When a sticker is rendered standalone — i.e., as a thumbnail in
  the picker grid, not as part of the active theme application —
  it renders in its OWN palette key. The frog ALWAYS looks like
  frog in the picker grid (because the picker renders each sticker
  in that sticker's own palette, not the active palette source's).
  This preserves the visual identity of each sticker as a
  recognisable choice.

  Rephrased: the rule is "render with active palette source IF
  the sticker is being shown as part of the active theme; render
  with the sticker's own palette OTHERWISE." In practice: the
  picker always shows authentic-coloured stickers; the calendar /
  cells / flourishes / completion stickers all retint to the
  active palette source.

WHY THIS MATTERS:
  Eliminates the cross-theme clash that would otherwise occur. If
  frog fills leader and vampire fills palette source, the lily
  pads must not stay bright green — they would clash against a
  black-and-red app. The retint rule makes the whole composition
  cohere regardless of which sticker fills which slot.

  Also: one source of truth per sticker. No "sticker frog" +
  "theme frog" double-maintenance. The sticker art and theme art
  are the same art, rendered through whichever palette is
  currently active.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE PALETTE KEY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Every sticker ships with palette.tres. Approximate shape (Godot
Resource syntax — schema to confirm at implementation time):

  extends Resource
  class_name StickerPalette

  @export var primary:   Color   # dominant brand-style colour
  @export var secondary: Color   # supporting / accent colour
  @export var tertiary:  Color   # contrast / darkest colour

The reference colours used in the sticker's PNG art must be a
fixed, known set so the renderer can perform palette substitution.
Candidate reference values to lock at implementation time:

  reference-primary:   pure red    #FF0000
  reference-secondary: pure green  #00FF00
  reference-tertiary:  pure blue   #0000FF

These are deliberately ugly values — they will never appear in
finished art because they always get substituted. The ugliness is
the point: a stray reference-pure-red pixel that survives
substitution is obvious during testing.

(Open: do we want a fourth "accent" or "highlight" role for finer
control? Three may be enough — the prototype managed with three —
but worth checking against the first full theme art set to see if
something feels missing. Defer to first painted theme.)

SACRED SEMANTIC COLOURS:
  Some colours are not themed and never change regardless of
  palette source:
    - Completion green (stamp tier 4 / four-of-four done)
    - Rest day purple
    - Error red
    - Warning orange
    - Stamp tier 1/2/3 partial colours
    - Body text near-black on light backgrounds
    - Border / outline near-white on dark
  These are hardcoded in the engine, not in any palette.tres
  file. Primary/secondary/tertiary are the THEMED colour roles.
  Sacred semantics are SEPARATE and unaffected by palette source.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EFFECTS — ACTIVE AND AMBIENT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Two distinct effect slots, each conceptually different.

ACTIVE EFFECTS (effects_active.tres) — event-triggered, one-shot.
  Fires once per relevant UI event:
    - Cell tap → splash/sizzle/drip/sparkle on the tapped cell.
    - Button press → small flash or particle on the button.
    - Sticker selected → sparkle around the picker tile.
    - Morning sequence claim → themed celebration accent.
    - Stamp slap → themed reinforcement of the stamp animation.
  Cheap per-event cost. Fire-and-forget. Performance budget
  concern is negligible.

  effects_active.tres declares which UI events the sticker
  responds to and what parameters (tween curves, particle counts,
  audio sting if paired with theme_audio.tres). The active effects
  coordinator reads the .tres and routes events.

AMBIENT EFFECTS (effects_ambient.tres) — continuous, atmospheric.
  Always-on while the relevant scene is visible:
    - Lily pads bob and drift on the day cells
    - Candle flames flicker on castle bricks
    - Dust motes drift down from a stone background
    - Bees occasionally visit flowers on a garden background
    - Embers float up from a fire-themed background
  Performance-budget-significant. Subject to global rules below.

PERFORMANCE RULES (apply to all effects):
  - Ambient effects only run on visible viewport. Off-screen
    scenes pause their effects.
  - All effects pause when the app is backgrounded.
  - Each effects.tres declares a particle / element count. The
    coordinator caps the GLOBAL active count across all running
    effects — if a sticker's ambient effect alone uses too many
    particles, the coordinator silently degrades quality. No
    user-visible failure mode.
  - Reduce-motion accessibility (iOS/Android system preference):
    when enabled, ambient effects pause entirely; active effects
    play a simpler fallback (static flash instead of particle
    burst).

EFFECTS RENDERING IS PALETTE-AWARE:
  Effect art (particle sprites, splash sprites, drip sprites) is
  painted in reference palette colours and retinted at runtime per
  the standard palette substitution rule. The frog water splash
  in chilli palette renders as cream-and-red liquid; the vampire
  drip in frog palette renders as moss-green ooze. Cohesion
  preserved across mixed compositions.

EFFECTS COORDINATOR — ARCHITECTURE NOTE:
  Single Godot autoload, signal-driven. UI events emit signals
  (cell_tapped, button_pressed, sticker_selected, etc). The
  coordinator listens, looks up which active effect (if any) is
  bound to that event, plays it. Ambient effects are registered
  on scene entry, unregistered on exit.

  effects_custom.gd in a sticker folder is the escape hatch for
  effects that don't fit the declarative effects_active.tres /
  effects_ambient.tres spec. Avoid where possible — every custom
  script is per-pack maintenance — but available when an effect
  is genuinely unique (e.g. a multi-frame procedural animation
  that needs real logic).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
AUDIO CONSIDERATIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Theme audio is the heaviest content axis. Including it in a
sticker has more implications than including any visual element.

FILE SIZE:
  A 2-minute ambient loop is roughly 2MB at low ogg bitrates.
  Stinger sounds are smaller (~50KB each). A pack that ships an
  ambient loop + 4-5 stingers + replacement UI sounds is in the
  3-4MB range. Multiple themed audio packs add up quickly.
  Consider runtime download rather than bundling all audio in the
  app shell, especially as the catalogue grows.

PLATFORM AUDIO RULES:
  iOS background audio is strict. Theme audio that should
  continue playing while the user is briefly in another app needs
  declared entitlements and proper AVAudioSession category
  selection. Audio that should stop when backgrounded needs the
  opposite. Decide per-effect type:
    - Ambient loops: stop when backgrounded (atmospheric, not
      essential).
    - Stinger sounds: don't play in background (they're tied to
      foreground events).
  iOS doesn't make this hard, but it requires conscious setup.

LICENSING:
  Every audio file needs a clear rights story:
    - Original composition (Morgan or commissioned artist).
    - Royalty-free with documented licence.
    - Public domain (rare, must be verified).
  NO ambiguous "I found it on YouTube" audio. Ever.
  The marketing notes collaboration template (Section 4) covers
  artist-commission terms — same template applies to theme
  composers.

USER PREFERENCE:
  Mandatory global "theme audio on/off" toggle in settings.
  Some users will never want app audio. A subset will want SFX
  but not music. Defer the granularity of the toggle (master vs
  music vs SFX vs UI sounds) to settings-screen implementation,
  but the principle is: audio is opt-out at user level, no matter
  what the active theme ships.

AUDIO IS LIKELY RARE IN THE CATALOGUE:
  Painting a sticker takes hours. Painting a sticker AND
  composing its audio takes days. Most stickers will NOT ship
  audio. The ones that do are content events — anchors that
  marketing can build a drop around. Don't commit to audio per
  pack; commit to audio on select packs as content drops.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SUBSCRIPTION INTERACTION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The theme system intersects with the monetisation v2.0 model
(four_tasks_monetisation_position.md). Key intersection points:

LIBRARY ACCESS:
  The user's "accessible library" — the set of stickers whose
  context menus they can interact with — depends on subscription
  state:
    - All stickers OWNED by the user (purchased with coins): in
      library, all slots usable.
    - All stickers OWNED by the user's partner (if pair has at
      least one active subscription): in library, all slots
      usable.
    - All other stickers: VISIBLE in the picker grid (so the user
      knows they exist) but LOCKED. Long-press shows only the
      preview button, not the per-element icons.

  Library access is binary per-sticker, not per-element. If a
  sticker is in your accessible library, all of its elements are
  usable. Subscription doesn't differentiate "you can use the
  cell treatment but not the ambient effect" — that would be
  needless complexity for no monetisation gain.

CATALOGUE PURCHASE GATE:
  Most monthly pack drops are SUBSCRIBER-ONLY PURCHASABLE. A
  free user cannot spend coins on most new packs even if they
  have the coins. The catalogue extension over time is one of
  the things subscription buys.

  Free users CAN purchase:
    - Their onboarding-free starter pack (already locked).
    - Their pre-launch / launch-era packs (whatever ships at
      v1.0).
    - Occasional thank-you drops (off-schedule, infrequent, free
      to everyone, possibly review-gated — see Open work).

  Free users CANNOT purchase regular monthly drops while
  unsubscribed.

  This creates a clean subscription value:
    - Free user pays nothing. Has access to a fixed catalogue
      that grows occasionally with thank-you drops.
    - Subscriber pays subscription. Has access to the
      ever-growing catalogue + partner's library + small coin
      bonus.
    - Unsubscribing reverts the user to the free catalogue
      window — they keep what they bought, but new drops are
      locked until they re-subscribe.

PAST DAYS PRESERVE FULL SLOT STATE:
  Immutable-past principle from monetisation v2.0 extends to ALL
  theme slots, not just leader and palette source. The full
  active-slot configuration is preserved at day-seal time. A
  subscriber's days from their subscription period render
  forever with whatever slots they had filled, even after
  unsub.

  Schema implication: `days.day_theme_state` (JSON, frozen at
  seal) captures the full slot map for that day, not just
  day_leader + day_palette as previously sketched in
  monetisation v2.0.

  This is the second iteration on monetisation v2.0's schema
  recommendation. The previous two-column approach
  (day_leader + day_palette) is superseded by the single
  json column to handle the full slot set. Update migration_005
  accordingly when tile 1.3 lands.

INSTANT FEEDBACK STILL APPLIES TO LOCKED STICKERS' PREVIEW:
  The preview button on a locked sticker temporarily applies all
  the sticker's elements live. The user sees what they're
  missing. The preview is the marketing — no nag screen needed,
  the user's own taste does the persuasion work.

  Preview reverts on context menu close OR after a generous
  timeout (say 30 seconds) so the user can interact with the
  preview state — tap things, see the active effect fire, scroll
  the calendar — before it disappears.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ASSET DIMENSIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

STICKER.PNG — LOCKED (session 6 part 2):

  32 × 32 pixels, transparent background, RGBA.

  Reasoning:
    - 1080×2400 portrait target; 7-column calendar with ~12px
      padding and 6×6px gaps gives each day cell ~134px of width.
    - Sticker renders at ~75% of cell width (prototype ratio) =
      ~100px on-screen.
    - 32×32 source → ~3x integer-ish scale on cell. Clean Nearest,
      no fractional artefacts.
    - 32×32 at picker render (~6x scale) fills picker grid
      confidently.
    - Forces silhouette discipline while leaving headroom for
      personality detail.

  Effective art area ~26×26 inside with margin for rotation
  wobble (prototype's stickerOffset applies on day cells).

DEFERRED TO IMPLEMENTATION SESSION:
  - Background.png canvas size (full-grid background — sizing
    depends on actual calendar area dimensions).
  - Grid tile size (depends on cell render size).
  - Cell treatment variants (probably ~128×128 for the cell —
    one cell width — but exact ratio to be measured against the
    rendered day cell scene).
  - Dead-cell treatment (same as cell variants).
  - Flourish_name bounding box.
  - Label_background bounding box (depends on task row height).
  - Effect art (particle sprites) — varies per effect; declared
    in effects_active.tres / effects_ambient.tres specs.
  - Reference palette exact RGB values
    (placeholder #FF0000/#00FF00/#0000FF).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RETINT MECHANISM — DEFERRED TO IMPLEMENTATION SESSION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The palette-agnostic art rule requires the engine to substitute
the reference palette for the active palette at render time. Three
candidate approaches:

  A. Shader-based substitution at draw time.
     Fragment shader checks each pixel against the three reference
     colours and substitutes from a uniform. Cheap per-frame cost,
     no asset duplication. One shader applied across every themed
     surface.

  B. Pre-bake at slot-change time.
     When the palette source slot changes, the engine generates
     retinted copies of all theme art in memory (or on disk
     cache). Renders read the cached copies. Per-change cost
     once; per-frame cost is normal sprite rendering. Memory cost
     scales with theme art count.

  C. Indexed-palette PNG with runtime palette swap.
     Theme art shipped as indexed-colour PNGs. The PNG palette
     itself is rewritten at slot-change time. Standard texture
     samplers handle the rest. Workflow constraint: Aseprite must
     export indexed.

Decision blocked on: actual count of theme art pieces (memory cost
of B), prototyping shader A for pixel-art-correctness, and
confirming Aseprite indexed-export workflow for C. All three are
implementable in Godot; differences are workflow + perf + asset
size.

Defer to first implementation pass (tile 4.14b). Whichever path is
picked, the AUTHORING workflow stays the same: paint in reference
colours at 32×32 (or whatever dimensions for other elements), ship
as palette-agnostic art + palette.tres.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SCHEMA IMPLICATIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Per-user theme state needs to be stored server-side. Recommended
schema:

  users.active_leader      TEXT     -- sticker ID (identity field)
  users.active_theme       TEXT     -- JSON map of slot → sticker ID
  users.active_stickers    TEXT     -- JSON array of sticker IDs (pool)

active_leader is its own column because it participates in the
pair-key identity hash (per pair-key v2 — the user-facing identity
sticker is the recovery anchor). All other slots live in the
JSON map.

The JSON map approach is cleaner than a column-per-slot because
adding a new slot later doesn't require a schema migration — just
a new key in the JSON. The schema column count stays stable as
the slot catalogue grows.

JSON map shape:
  {
    "palette":        "<sticker_id>",
    "background":     "<sticker_id>",
    "cell":           "<sticker_id>",
    "dead_cell":      "<sticker_id>",
    "grid_tile":      "<sticker_id>",
    "flourish":       "<sticker_id>",
    "label":          "<sticker_id>",
    "effect_active":  "<sticker_id>",
    "effect_ambient": "<sticker_id>",
    "audio":          "<sticker_id>"
  }

Any key may be absent or null (slot is empty — default renders).

PAIR-KEY PARTICIPATION:
  active_leader participates in the pair-key identity hash. All
  other slots are non-identity fields, freely changeable without
  pair-key migration. This is the schema split the previous theme
  doc deferred. LOCKED now: leader is identity, everything else
  is theme state.

MIGRATION_005 (UPDATED FROM MONETISATION V2.0):
  Previously sketched as adding `days.day_leader + days.day_palette`
  for the immutable-past mechanic. Updated to capture the FULL
  slot map at day-seal time, not just leader+palette.

  Migration_005 final shape:
    ALTER TABLE users ADD COLUMN active_leader TEXT;
    ALTER TABLE users ADD COLUMN active_theme TEXT;       -- JSON map
    ALTER TABLE users ADD COLUMN active_stickers TEXT;    -- JSON array
    ALTER TABLE days ADD COLUMN day_theme_state TEXT;     -- JSON map

  active_leader is its own column because it participates in
  pair-key hashing. The rest are JSON for flexibility.

  Existing `icon` column from prototype-era schema is repurposed —
  if it still exists, drop it (its role is now split across
  active_leader and the active_theme JSON's `palette` key).

THE PALETTE_TABLE.TRES TILE (4.G) — STILL SUPERSEDED.
  Per-sticker palette.tres files in each sticker folder remain
  the model. No central table.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHAT THIS DOC LEAVES OPEN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Captured for the implementation session, not now:

  - Asset dimensions for non-sticker theme files (background,
    cell variants, grid tile, flourishes, label backgrounds,
    effect art). Sticker locked at 32×32 session 6.
  - Retint mechanism (shader, pre-bake, or indexed-palette).
  - Reference palette exact values (placeholder
    #FF0000/#00FF00/#0000FF).
  - Default theme art set design (what does the app look like
    when MOST slots are empty? — e.g. brand new user with only
    onboarding-free sticker).
  - Themed-sticker marker visual style (glint, badge, border,
    animation).
  - Context-menu visual design (size, position, dismissal
    behaviour).
  - Final icon vocabulary for the context menu (placeholders
    listed in "Picker UX" — defer to focused visual-design pass
    at tile 4.14b).
  - Whether to add a fourth colour role (accent / highlight).
  - effects_active.tres and effects_ambient.tres schemas —
    exact fields, supported tween / particle / shader / audio
    parameters.
  - theme_audio.tres schema — what audio events does it bind to,
    how is the ogg file path declared, ambient vs stinger
    differentiation.
  - Generic effects coordinator design — single autoload,
    signal hookup, lifecycle, viewport-visibility integration,
    reduce-motion branch.
  - Audio playback architecture — single Godot AudioStreamPlayer
    per scope vs pooled players, bus routing, fade-in/out on
    slot change.
  - Locked vs unlocked picker rendering for free vs paid users —
    locked stickers should look enticing, not greyed out.
  - Preview timeout exact duration (locked sticker preview
    behaviour).
  - Thank-you drop mechanism (free-user-purchasable packs,
    possibly review-gated unlock). Surfaced in monetisation v2.0
    discussion, not yet specified. Lives in the monetisation doc
    when fleshed out, not this one.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED DOCS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  - four_tasks_pair_key_design_notes.md
    Identity model. active_leader participates in the pair-key
    hash; all other slots do not. Picker UX inheritance (the
    long-press gesture pattern) — though the commit-on-close
    model is NOT inherited here for non-leader slots; theme slot
    changes are instant.

  - four_tasks_staggered_disclosure_design_notes.md
    Long-press gesture (per-element context menu) is a slow-
    reveal at day 7+. Four Tasks only — APPtrioc never reveals
    this. The post-fork picker depth is the conversion mechanic
    from APPtrioc to Four Tasks.

  - four_tasks_architectural_preference.md
    Clarity over cleverness — informed the choice to tag palettes
    at art time rather than extract at runtime. Hand-painted is
    the medium; algorithmic colour extraction is performance-coded
    cleverness for a problem we don't have.

  - four_tasks_write_rules_design_notes.md
    active_leader changes trigger pair-key migration. Other slot
    changes are normal user-field writes, no migration.

  - four_tasks_morning_sequence_design_notes.md
    Q1 — MOTD's emoji slot reads from the active_stickers pool.
    The pool is dual-purpose by design.

  - four_tasks_monetisation_position.md
    Library access mechanic depends on subscription state.
    Catalogue purchase gate makes subscription the access path to
    new monthly drops. Immutable past extends to full slot state,
    not just leader/palette — migration_005 updated to a single
    json column for full slot capture.

  - four_tasks_sticker_pack_brainstorm.md
    Content-side brainstorm of which packs ship which elements.
    The brainstorm uses the feature-catalogue model — each pack
    entry lists its elements explicitly rather than claiming a
    tier label.
