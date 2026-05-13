# Four Tasks — Theme & Sticker System Design Notes

Status: FOUNDATION LOCKED (session 5)
Schema: no immediate impact — see "Schema implications" below
Implementation tiles: 4.14a (basic picker, pre-fork — tap-to-toggle
pool membership only), 4.14b (post-fork — long-press context menu +
theme manager + retint, FOUR TASKS ONLY), 4.D (sticker art).
Tile 4.G (central palette table) is SUPERSEDED by this doc — see
"Schema implications" below.

This document supersedes the previous `four_tasks_theme_design_notes.md`
referenced in the AI Reading Guide (which existed but was lost). If the
original surfaces, cross-reference rather than merge — this is the canonical
post-session-5 statement.

NOTE ON THE APPTRIOC FORK (locked session 5):
  The full theme system (Lever 2 + Lever 3 — calendar leader and
  palette source via long-press context menu) is FOUR TASKS EXCLUSIVE.
  Tile 4.14b lands post-fork; APPtrioc snapshots at tile 4.14a and
  never gets the context menu. APPtrioc users see the picker but
  long-press is inert (themed-sticker markers visible to hint at
  what Four Tasks unlocks). The pool (Lever 1) IS available in
  APPtrioc. See top of four_tasks_godot_todo.txt "APPTRIOC FORK
  ARCHITECTURE" section.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PROBLEM
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The web prototype's theme system is the app's selling point in embryonic
form. Long-press the top-left emoji, the picker opens, long-press an emoji
inside the picker, that emoji becomes the user's "header" — its dominant
colours are extracted via off-screen canvas pixel sampling and applied to
the app background as a gradient. The streak bar gets a tinted accent. The
loading screen gets a single colour. Everything else (day cells, task list,
popups, util bar, calendar grid) stays hardcoded slate/zinc neutrals.

It's a clever proof of concept that demonstrates the principle: change one
emoji, the app reskins. But the actual reskin is small — only two UI
surfaces participate — and the mechanism is fragile (relies on browser
emoji font rendering, requires a setTimeout to wait for paint, depends on
canvas pixel readback semantics that have no Godot equivalent).

The Godot port replaces this with a fully hand-crafted system. Every
sticker is hand-painted. Every theme is hand-painted. Colours are tagged
at art time, not extracted at runtime. The picker becomes the primary
interface for personalising the app. The reskin is no longer cosmetic —
it touches calendar cells, grid tiles, decorative flourishes, the whole
chromatic identity of the app.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE THREE LEVERS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Each user has three independent settings, all controlled from the same
sticker picker grid. None of the three constrains the others. A sticker
can be all three, two, one, or none.

LEVER 1 — Pool membership.
  Which stickers are eligible to be randomly slapped onto days where the
  user completes all four tasks. Same behaviour as the prototype's
  `activeEmojis`. Floor of 1 (the user cannot deselect down to zero —
  every completed day needs *something* to celebrate with).

  Locked session 5 (Q1): the Lever 1 pool ALSO feeds the MOTD's emoji
  slot. The MOTD is four spinners (adverb + verb + noun + emoji), and
  the emoji slot draws from the same pool the user has actively
  toggled in the picker. Change your pool, change both your completed-
  day stickers AND your MOTD emoji vocabulary. Single source of
  truth. See four_tasks_morning_sequence_design_notes.md "The MOTD
  structure (Q1)" section.

LEVER 2 — Calendar leader.
  The sticker chosen as the "avatar" of the user's calendar. When set,
  triggers a hand-drawn theme art set tied to that sticker. The frog as
  leader turns unmarked-day cells into lily pads, calendar grid tiles
  swampy, places a fly above the user's name. Each sticker that ships
  with a theme art set has its own bespoke supporting pieces. Stickers
  *without* a theme art set still work as leader — they fall back to
  a default theme (see "Theme coverage gradient" below).

LEVER 3 — Palette source.
  The sticker whose colours flood the app. Three named colour roles
  (primary, secondary, tertiary) pre-tagged on every sticker at art time.
  The app consumes those roles, not raw RGB. Set the chilli pepper as
  palette source, the app reads chilli's tagged palette (primary = hot
  red, secondary = pepper-flesh cream, tertiary = stem-green or near-
  black) and applies them to every UI surface that maps to a themed
  colour role.

INDEPENDENCE EXAMPLES:
  - Frog as leader + frog as palette source + frog in pool.
    Most common case. Everything frog. Lily pad cells in frog-green.
  - Frog as leader + chilli as palette source + frog not in pool.
    The calendar is shaped like frog (lily pads, swamp grid, fly) but
    coloured like chilli (red lily pads, red swamp tiles, red fly).
    Frog stickers never appear as completion stickers.
  - Frog in pool + chilli as leader + crow as palette source.
    Calendar is chilli-themed (whatever chilli's theme art set is),
    coloured in crow's palette, but completed days may show frog
    stickers (and any other pool members).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ASSET ARCHITECTURE — ONE FILE PER PIECE, FOLDER PER STICKER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Each sticker lives in its own folder under `res://assets/stickers/<id>/`.
The folder contains:

  sticker.png        — the sticker itself (the little drawing).
  palette.tres       — Godot Resource with the sticker's tagged palette.
  cell_unmarked.png  — (optional) custom unmarked-day cell variant.
  grid_tile.png      — (optional) custom calendar grid tile.
  flourish_name.png  — (optional) decorative element near the user's name.
  ... etc, as theme pieces are conceived.

A sticker with no theme art ships only `sticker.png` + `palette.tres`.
A sticker with a partial theme ships some optional files but not others.
A sticker with a full theme ships all of them. The engine discovers
sticker capability by which files exist in the folder — no separate
manifest declaring "this sticker has a theme." File existence IS the
declaration.

WHY FOLDER-PER-STICKER, NOT SPRITE-SHEET-PER-STICKER:
  Sprite sheets force a fixed slot layout up front. The moment one
  sticker wants a flourish another doesn't, the sheet either grows
  empty cells or breaks atomicity. Folder-per-sticker lets each theme
  bring exactly what it brings, no more no less. Adding a new theme
  piece concept ("ooh, a stamp variant per sticker") is dropping a
  new optional filename into the convention, not a sheet re-layout.
  Aseprite workflow also matches: one Aseprite project file per
  sticker, exports go into the sticker's folder side-by-side.

WHY OPTIONAL FILES, NOT REQUIRED:
  The theme art set is a quality lever, not a structural requirement.
  Demanding every sticker ship every theme piece would either (a)
  cap the catalogue size at "however many full themes Morgan can
  paint" or (b) force shipping low-effort theme art to fill the
  required slots. Both bad. The optional model lets the catalogue
  grow on two axes independently: more stickers + more themed
  stickers among them.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE PALETTE-AGNOSTIC ART RULE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

All theme art (including the sticker itself) is painted in a canonical
*reference palette*. At runtime, the renderer substitutes the reference
palette for the active palette source's tagged colours.

This means:
  - The frog sticker is not painted in literal frog-green. It is painted
    in (reference-primary, reference-secondary, reference-tertiary) —
    three flat reference colours. The palette.tres file says
    "for this sticker, reference-primary = mossy-green, reference-
    secondary = wet-belly cream, reference-tertiary = dark-water."
  - When frog is the palette source, the renderer loads frog's palette
    and substitutes. The sticker renders in its natural colours.
  - When chilli is the palette source instead, the renderer loads
    chilli's palette and substitutes — the frog sticker renders in
    chilli's colours. Red frog, cream belly, dark stem-green eyes.

THIS APPLIES TO ALL RENDERED THEME ART:
  - The sticker itself when it appears as a completion sticker on a
    day cell, when shown in the picker grid, anywhere.
  - The unmarked-cell variant.
  - The grid tile.
  - The name flourish.
  - Any other theme piece.

THE ONE EXCEPTION:
  When a sticker is rendered standalone — i.e., NOT as part of an
  app-wide theme application — it renders in its OWN palette key.
  The frog always looks like frog in the picker grid (because the
  picker grid renders each sticker in that sticker's own palette,
  not the active palette source's). This preserves the visual
  identity of each sticker as a recognisable choice.

  Rephrased: the rule is "render with active palette source IF the
  sticker is being shown as part of the active theme; render with
  the sticker's own palette OTHERWISE." In practice this means: the
  picker always shows authentic-coloured stickers; the calendar /
  cells / flourishes / completion stickers all retint to the active
  palette source.

WHY THIS MATTERS:
  Eliminates the cross-theme clash that would otherwise occur. If
  frog is leader and vampire is palette source, the lily pads must
  not stay bright green — they would clash against a black-and-red
  app. The retint rule makes the whole composition cohere
  regardless of which sticker is leader vs which is palette source.

  Also: one source of truth per sticker. No "sticker frog" + "theme
  frog" double-maintenance. The sticker art and theme art are the
  same art, rendered through whichever palette is currently active.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE PALETTE KEY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Every sticker ships with palette.tres. Approximate shape (Godot Resource
syntax — schema to confirm at implementation time):

  extends Resource
  class_name StickerPalette

  @export var primary:   Color   # the dominant brand-style colour
  @export var secondary: Color   # supporting / accent colour
  @export var tertiary:  Color   # contrast / darkest colour

The reference colours used in the sticker's PNG art must be a fixed,
known set so the renderer can perform palette substitution. Candidate
reference values to lock at implementation time:

  reference-primary:   pure red    #FF0000
  reference-secondary: pure green  #00FF00
  reference-tertiary:  pure blue   #0000FF

These are deliberately ugly values — they will never appear in finished
art because they always get substituted. The ugliness is the point: if
a stray reference-pure-red pixel ever survives substitution because the
asset wasn't fully painted in reference colours, it will be obvious
during testing and easy to find.

(Open: do we want a fourth "accent" or "highlight" role for finer
control? Three may be enough — the prototype managed with three —
but worth checking against the first full theme art set to see if
something feels missing. Defer to first painted theme.)

SACRED SEMANTIC COLOURS:
  Some colours are not themed and never change regardless of palette
  source:
    - Completion green (stamp tier 4 / four-of-four done)
    - Rest day purple
    - Error red
    - Warning orange
    - Stamp tier 1/2/3 partial colours
    - Body text near-black on light backgrounds
    - Border / outline near-white on dark
  These are hardcoded in the engine, not in any palette.tres file.
  Primary/secondary/tertiary are the THEMED colour roles. Sacred
  semantics are SEPARATE and unaffected by palette source.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PICKER UX
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Inherits and extends the prototype's gesture model.

THE SAME GRID, THE SAME GESTURES, A NEW CONTEXT MENU:
  - TAP toggles a sticker in/out of the pool (Lever 1).
    Floor of 1, identical to prototype.
  - LONG-PRESS (with vibration feedback) opens a small context menu
    over the pressed sticker, with two icons:
      [📅] calendar icon  — set this sticker as calendar leader (Lever 2)
      [🎨] artist palette — set this sticker as palette source  (Lever 3)
    Both icons are user-visual-language: calendar is universally "this
    is for the calendar," artist palette is universally "this is for
    the colours." The picker explains itself.

  The context menu replaces the prototype's "long-press = set as
  header" behaviour, which conflated leader and palette into one
  action. The new menu introduces the decision point that separates
  the two levers.

THEMED-STICKER MARKING:
  Stickers that ship with a theme art set are visually marked in the
  grid — they have *more to offer* than non-themed stickers. Exact
  marking style is an art decision (a glint, a corner badge, a subtle
  border treatment, a small animation). The principle: the marking
  should HINT at specialness without spoiling what the theme is.
  Discovery preserved.

THE STAGGERED-DISCLOSURE OVERLAY:
  Themed stickers are themselves a slow-reveal feature per the
  staggered disclosure design doc. Not all users need to learn about
  themes on day 1. A user can use Four Tasks as a habit tracker for
  weeks before discovering that long-pressing a sticker offers two
  options instead of one. When the moment is right (tutorial reveal
  triggered, see four_tasks_staggered_disclosure_design_notes.md),
  the app surfaces the theme system explicitly.

PREVIEW-ON-PRESS, COMMIT-ON-CLOSE:
  Inherits from the v2 identity model's emoji picker UX (see
  four_tasks_pair_key_v2_design_notes.md "Emoji picker UX"). When
  the user holds and previews a sticker via context-menu selection,
  the app palette-swaps locally so they can see the effect. Only on
  picker close does a single server write commit the change. This
  also folds into the pair-key migration story — palette source
  changes trigger a pair-key migration server-side (since the
  sticker is part of the user's identity tuple).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THEME COVERAGE GRADIENT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Not every sticker ships with a theme art set. The catalogue is a
gradient:

  TIER 1 — Full theme.
    Sticker + palette + all optional theme files. Setting this as
    leader transforms the calendar comprehensively. These are the
    "selling point" stickers.

  TIER 2 — Partial theme.
    Sticker + palette + some optional theme files (e.g., custom cell
    variant but no flourish). Setting as leader transforms some
    surfaces, default theme elsewhere.

  TIER 3 — Palette only.
    Sticker + palette. Setting as leader uses the default theme art
    set; the only effect of being leader is "this sticker appears in
    the avatar slot." Setting as palette source still works fully.

The catalogue at launch will skew TIER 3 with a small set of TIER 1
stickers as the showpiece. As Morgan paints more themes over time,
TIER 3 stickers get promoted to TIER 1. The gradient is a sustainable
content cadence — each new themed sticker is a content drop, a
reason for users to revisit the picker, a marketing beat.

DEFAULT THEME ART SET:
  Stickers without custom theme files use a default set. The default
  is itself painted in reference colours so it retints to the active
  palette source. The default is plain and pleasant — it's a fallback,
  not a fight for attention.

  Open: does the default theme set get its own palette.tres (so it has
  an "identity" when no themed sticker is leader/palette source) or is
  it always rendered against the active palette source? Decide at
  first-paint time.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ASSET DIMENSIONS — DEFERRED TO IMPLEMENTATION SESSION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

This doc captures the system. Dimensions are an art-tile concern and
need to be locked before any sticker is painted (re-exporting 30
stickers at a different size is a tax to avoid). To resolve next
session:

  - Source art resolution (paint at 2x or 4x, export at 1x?)
  - Calendar cell sticker size (the small render on a day cell)
  - Picker grid sticker size (the larger render in the picker)
  - Unmarked-cell variant size (replaces the full day cell)
  - Grid tile size (the background of the calendar grid)
  - Name flourish bounding box
  - Safe-area / padding rules within each export

The 1080×2400 target resolution + Godot's `texture_filter = Nearest`
pixel-art setting (tile 0.2) constrain the answer space. Decide
once, document once, paint to spec thereafter.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RETINT MECHANISM — DEFERRED TO IMPLEMENTATION SESSION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The "palette-agnostic art rule" requires the engine to substitute the
reference palette for the active palette at render time. Three
candidate approaches:

  A. Shader-based substitution at draw time.
     Fragment shader checks each pixel against the three reference
     colours and substitutes from a uniform. Cheap per-frame cost,
     no asset duplication. Requires writing one shader and applying
     it consistently across every themed surface.

  B. Pre-bake at theme-change time.
     When palette source changes, the engine generates retinted
     copies of all theme art in memory (or on disk cache). Renders
     read the cached copies. Theme change is a one-time cost; per-
     frame cost is normal sprite rendering. Memory cost scales with
     theme art count.

  C. Indexed-palette PNG with runtime palette swap.
     Theme art shipped as indexed-colour PNGs. The PNG palette
     itself is rewritten at theme-change time. Standard texture
     samplers handle the rest. Workflow constraint: Aseprite must
     export indexed.

Decision blocked on: knowing the actual count of theme art pieces
(memory cost of B), prototyping a shader for A to see if it fits
the pixel-art aesthetic without artefacts, and confirming
Aseprite's indexed-export workflow for C. All three are
implementable in Godot; differences are workflow + perf + asset
size.

Defer to first implementation pass (tile 4.14b). Whichever path is
picked, the AUTHORING workflow stays the same: paint in reference
colours, ship as palette-agnostic art + palette.tres.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SCHEMA IMPLICATIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

NO IMMEDIATE SCHEMA CHANGE.

The existing schema already stores the user's icon (which has been
the de facto "leader + palette source combined" field). The new
system splits that into two server-side identity fields:

  - calendar_leader  — sticker ID currently set as leader
  - palette_source   — sticker ID currently set as palette source

Both nullable initially, default to whatever the user picked at
onboarding. The pool (Lever 1) lives in active_stickers as a JSON
array column or similar, mirroring the prototype's activeEmojis.

Migration concern: the current schema (post-migration_002) treats
icon as a single value participating in the pair-key tuple. The new
model has TWO sticker-id fields. The pair-key tuple must still be
stable, so only ONE of them should participate in the hash. The
calendar leader is the user-facing "identity sticker" so that's the
candidate. Palette source becomes a non-identity field, freely
changeable without pair-key migration.

OR: the existing `icon` field is repurposed as `calendar_leader` (the
identity participant) and a new `palette_source` field is added
alongside (non-identity). This is a tile-1.3-style migration —
schema rename + new column. Belongs alongside migration_003 or as
migration_004.

The full schema diff and pair-key tuple implications need a
dedicated design pass against four_tasks_pair_key_v2_design_notes.md
to confirm the identity-participation split. Defer to implementation
session.

THE PALETTE_TABLE.TRES TILE (4.G) IS SUPERSEDED.
  The old roadmap tile 4.G described "Curated palette table — 86
  emojis × 3 colours saved into res://data/palette_table.tres." The
  new system distributes the palette to per-sticker palette.tres
  files (one per folder), so no central table exists. Tile 4.G
  becomes redundant. Update the todo to reflect this.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHAT THIS DOC LEAVES OPEN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Captured for the implementation session, not now:

  - Asset dimensions (resolution, cell size, picker size, padding).
  - Retint mechanism (shader, pre-bake, or indexed-palette).
  - Reference palette exact values (placeholder #FF0000/#00FF00/#0000FF).
  - Default theme art set design and palette identity.
  - Themed-sticker marker visual style (glint, badge, border, animation).
  - Context-menu visual design (size, position, dismissal behaviour).
  - Whether to add a fourth colour role (accent / highlight).
  - Schema rename: icon → calendar_leader + new palette_source column
    (migration sequencing against migration_003).
  - Pair-key tuple split — which of calendar_leader vs palette_source
    participates in the identity hash. Almost certainly calendar_leader
    (the user-facing identity sticker) but needs a dedicated pass
    against the v2 design notes.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED DOCS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  - four_tasks_pair_key_v2_design_notes.md
    Identity model. Picker UX pattern (preview-on-press, commit-on-
    close), pair-key migration triggered by identity changes.

  - four_tasks_staggered_disclosure_design_notes.md
    Themed stickers as a slow-reveal feature. The "long-press has
    two options now" tutorial reveal lives in the disclosure
    schedule (day 7+ in current draft schedule, Four Tasks only —
    APPtrioc never reveals this).

  - four_tasks_architectural_preference.md
    Clarity over cleverness — informed the choice to tag palettes
    at art time rather than extract at runtime. Hand-painted is the
    medium; algorithmic colour extraction is performance-coded
    cleverness for a problem we don't have.

  - four_tasks_write_rules_design_notes.md
    Pair-key migration semantics that fire when calendar_leader
    changes (Lever 2 commit). Palette source changes will need to
    decide their own write rule once the schema split is locked.

  - four_tasks_morning_sequence_design_notes.md
    Q1 — MOTD's emoji slot reads from the Lever 1 pool. The pool
    is dual-purpose by design.
