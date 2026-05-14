# Four Tasks — Sticker Pack Brainstorm

Status: RUNNING DOC. Add to this as inspiration hits.
Rewritten session 7 to match the feature-catalogue theme model.

Purpose: catalogue ideas for sticker packs — what each pack ships in
terms of theme slots, what its visual vibe is. Implementation-agnostic
— this is a content brainstorm, not architecture. The "how" lives in
four_tasks_theme_design_notes.md.

CONVENTION PER ENTRY (revised session 7):

Each pack entry lists:
  - Leader sticker name + silhouette/palette direction.
  - SLOTS SHIPPED — which named slots this pack contributes art to,
    using the slot catalogue from the theme doc. Each pack can ship
    any subset; sparsity is its own design axis. The list is the
    pack's identity, not a checklist to clear.
  - Companion stickers — other stickers in the pack (each with their
    own optional slot contributions; can be palette-only or
    feature-rich in their own right).
  - Notes — free-form thoughts on palette direction, tone, ambition,
    flavour.

Slot vocabulary (from theme doc):
  leader, palette, background, cell_overlay_<state>, cell_mask_<state>,
  dead_cell overlay/mask, grid_tile, flourish_name, label_background,
  active_effect, ambient_effect, theme_audio.

A pack that ships only palette is a valid pack. A pack that ships
everything is also valid. The shape of what each pack contributes
is the personality.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PACKS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

FROG
  Leader: mossy green frog, cream belly. Default cosy/wholesome
          pack. Good day-1 onboarding option.
  Palette: mossy green primary, wet-belly cream secondary, dark
           water tertiary.
  Slots shipped (ambition: high — flagship animated pack):
    - palette
    - background (swamp water vista that overlaps day tiles —
      the swamp encroaches into the calendar grid; day cells
      feel half-submerged)
    - grid_tile (swampy water ripples between cells)
    - cell_overlay_completed (small lily pad bobs into corner of
      completed cells)
    - flourish_name (fly hovering near user's name)
    - active_effect (water splash on cell tap)
    - ambient_effect (lily pads drift across the calendar
      surface, occasional bubble rises from the swamp)
    - theme_audio (distant frog croaks as ambient loop, water
      splosh stinger on cell tap — pairs with active_effect)
  Companion stickers: lily pad, orchid/tulip, fly, swamp flower.
    Most are palette-only (sticker.png + palette.tres). Fly might
    ship its own active_effect (small swarming buzz on tap).
  Notes:
    - The "swamp encroaches into the calendar" framing is the
      pack's signature beat. Worth painting carefully.
    - Frog + fairy pair naturally (woodland adjacent).

CHILLI
  Leader: hot red chilli, pepper-flesh cream highlight.
  Palette: hot red primary, pepper-flesh cream secondary,
           stem-green or near-black tertiary.
  Slots shipped (ambition: medium — kitchen vibe, no audio):
    - palette
    - background (fire glow + cast iron pan surface texture)
    - flourish_name (small flame near user's name)
    - active_effect (sizzle pop on cell tap — pepper splashes
      and steam)
    - label_background (chalkboard texture, like a restaurant
      menu board)
  Companion stickers: cooking-themed stuff — herbs, salt and
    pepper shaker, bread loaf, knife, garlic bulb, lemon. Mostly
    palette-only. Knife might ship its own active_effect (chop
    motion on tap).
  Notes:
    - The sizzle active effect IS the pack's signature. Pack
      could theoretically ship just palette + active_effect and
      still feel like a complete chilli identity.
    - Pairs interestingly with frog (kitchen + swamp = farmhouse
      vibe). Pairs harshly with vampire (red palette + dark
      palette = clashy unless intentional).

VAMPIRE
  Leader: hooded vampire silhouette or fanged face — horror-cute
          rather than horror-grim, to fit the app's overall warmth.
  Palette: dark crimson primary, candle-flame cream secondary,
           pitch black tertiary.
  Slots shipped (ambition: high — gothic atmospheric pack):
    - palette
    - background (crumbling castle brick wall, possibly cloudy
      moonlit sky behind)
    - grid_tile (gothic stone tiling between cells)
    - cell_overlay_completed (single bead of blood at corner)
    - cell_mask_unmarked (irregular brick-erosion bottom edge —
      cells look weathered)
    - flourish_name (bat silhouette circling near user's name)
    - active_effect (drip stinger on cell tap)
    - ambient_effect (candle flames flicker in the background,
      occasional bat flies across the calendar)
    - theme_audio (low organ drone as ambient loop, drip plink
      as cell tap stinger)
  Companion stickers: castle gate, blood vial, fang, coffin,
    bat, candelabra. Some palette-only, some ship their own
    active effects.
  Notes:
    - Could lean Castlevania / Bloodborne pixel-art reference
      without being a direct copy.
    - The mask-cut weathered brick edges on cells could be the
      pack's standout detail — most users won't notice
      consciously but it'll elevate the feel.

WIZARD
  Leader: hooded wizard with staff, or just a glowing magic orb.
  Palette: rich purple primary, parchment cream secondary,
           star-gold tertiary.
  Slots shipped (ambition: medium — focus on flourish + effects):
    - palette
    - background (cloud tower / cosmic sky)
    - flourish_name (sparkles trailing from user's name like
      a wand effect)
    - active_effect (sparkle burst on cell tap)
    - label_background (aged parchment texture under task
      labels)
  Companion stickers: staff, tome, alchemy vials, magic orb,
    starfield, scroll. Some palette-only, some ship sparkle
    effects.
  Notes:
    - Doesn't need ambient effect to feel complete — the
      sparkle active_effect carries the magical identity.
    - Cosy spell-caster energy, not Lord-of-the-Rings grim.

CAR
  Leader: small pixel-art car silhouette in cherry red.
  Palette: saturated arcade red primary, racing-stripe white
           secondary, tarmac black tertiary.
  Slots shipped (ambition: low — palette-heavy, single signature
                 effect):
    - palette
    - background (racing finish-line checkered flag pattern)
    - active_effect (vroom + tyre marks puff on cell tap)
  Companion stickers: other cars, traffic cone, racing flag,
    fuel pump, gear shift, trophy. Most palette-only — the
    pack's joy is "many cars" not "feature-rich cars."
  Notes:
    - Probably the lowest-effort pack on this list. Aimed at
      the user who wants playful arcade energy without
      atmosphere overload.
    - Could ship a v1.x update with engine-rumble theme_audio
      if popular.

FAIRY
  Leader: small pixel-art fairy or woodland sprite.
  Palette: spring-leaf green primary, blush pink secondary,
           dawn-gold tertiary.
  Slots shipped (ambition: medium):
    - palette
    - background (deep mossy forest floor with dappled light)
    - flourish_name (sparkle trail from user's name)
    - cell_overlay_completed (small mushroom pops up in corner)
    - active_effect (pixie dust sparkle burst on cell tap)
  Companion stickers: gnomes + woodland folk, mushrooms (a few
    varieties), acorn, pinecone, butterfly, dewdrop. Mix of
    palette-only and palette+active_effect.
  Notes:
    - Pairs naturally with frog (both woodland).
    - Pairs cleanly with wizard (sparkle aesthetic).
    - Pride-themed variant could be a v1.x drop using rainbow
      palette over the base art — marketing/LGBT outreach
      surface (see marketing notes Section 2).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PARKED IDEAS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

(Half-ideas, single-sticker concepts, themes that haven't formed
companion sets yet. Stays here until they grow into pack entries.)

  -

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NOTES ON USING THIS DOC
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- This doc is brainstorm. Nothing here is locked. Cross out, rewrite,
  delete, reorganise freely.
- When a pack feels ready to paint, the slot list above becomes the
  Aseprite job list. Each named slot = one asset to paint (or in the
  case of effects/audio, one .tres + supporting files to author).
- Slot lists are aspirational, not contracts. A pack can ship fewer
  slots than listed at first drop, with more added in v1.x updates.
- Don't feel obligated to ship every slot for every pack. Sparsity is
  a design axis — a chilli pack that ships only palette + active_effect
  could be more compelling than one that ships everything.
- Pack drop cadence is fortnightly alternating (subscriber-only vs
  free-thank-you) per monetisation v2.0 — so the brainstorm here is
  the long-term pipeline, with packs queued ~3-4 deep at launch and
  refilled continuously post-launch.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED DOCS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  - four_tasks_theme_design_notes.md — slot catalogue, picker UX,
    palette retint mechanism. The architecture this brainstorm
    plugs into.
  - four_tasks_monetisation_position.md — fortnightly drop cadence,
    subscriber-only vs free-thank-you alternating model, single
    pricing ramp regardless of pack feature-richness.
  - four_tasks_marketing_notes.md — pack subcultures + targeted
    cohort starter pack offers (FIFO, ADHD, etc.) draw from this
    catalogue rather than commissioning bespoke per-cohort art.
