Session 19 primer — Four Tasks

Read first:
- `four_tasks_godot_devlog.txt` — full devlog, end-of-session-18.
  Follow the AI Reading Guide at top. Session 18 has two blocks
  (tile 2.9 DevKit + tile 2.A panel heading/fonts). Running total
  ~91.75h.
- `four_tasks_godot_todo.txt` — phase tile list. Phase 2 closed
  through 2.A. ACTIVE block has three items added at session 18
  close worth eyeballing before any Phase 3 code.
- `four_tasks_devkit_design_notes.md` — DevKit is live, scenarios
  fire, used to set test state.
- `four_tasks_pair_key_design_notes.md` — Section 12 specifically.
  Username display normalisation is in ACTIVE to audit against this.
- `four_tasks_morning_sequence_design_notes.md` — wants a design
  pass this session. Two ACTIVE items (beat collision resolution,
  seal-day hero beat) both target this doc.

What you're walking into:
- Phase 2 is functionally closed. Gate language says "reboot to
  onboarding" but no Onboarding scene exists — that's Phase 3's
  job. So Phase 2 closes by opening Phase 3.
- Tile 3.1 (onboarding scene state machine + entry routing) is
  the topmost open tile.
- Two ACTIVE items want resolution BEFORE 3.1 lands:
    1. Morning sequence beat collision + seal-day hero beat —
       shared design pass against the morning sequence doc.
       Affects onboarding screen 6 (first MOTD slot machine fires
       inside morning-sequence-shaped beats). Skip this and screen
       6 gets rewritten.
    2. Username display normalisation vs pair-key recovery
       (Section 12). Affects onboarding screens 2/3/4 (name +
       username entry). Five-minute audit, captures a decision.

Working environment for this session:
- FIFO laptop, intermittent mobile connection, mobile Claude lag.
- No DevKit hot-loop available — laptop probably isn't set up to
  run the Godot project. Treat this as a design / non-code session.

Goal for this session:
- Design pass on morning sequence beat collision + seal-day hero
  beat. Output: revised `four_tasks_morning_sequence_design_notes.md`.
  Capture priority hierarchy across payout / staggered disclosure /
  content drops / urgent comms / single-fire comms, per-morning
  popup cap, per-user per-beat "shown" tracking, the seal-day beat
  itself (cell flies up + enlarges + sticker pops + coin-payout-
  framed-as-tomorrow popup).
- Five-minute audit: lowercase-at-render username display behaviour
  vs pair-key Section 12. Decide whether to also normalise stored
  values, or accept the divergence. Capture decision.
- If time and connectivity allow, sketch tile 3.1 shape (state
  machine, screen-routing rules, scene structure) — but DO NOT
  write code without home-laptop access.

Read everything before responding. Confirm you've read the morning
sequence doc + pair-key Section 12 before beginning the design
conversation.