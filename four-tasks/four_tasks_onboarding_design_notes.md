# Four Tasks — Onboarding Design Notes

Status: LOCKED (session 6, mobile drafting; terminology refreshed
        session 7 for feature-catalogue theme model)
Implementation tiles: 3.1 through 3.6+ (Phase 3)
Required reading before any Phase 3 tile lands.

Cross-references:
  - four_tasks_pair_key_design_notes.md (identity, collision, recovery)
  - four_tasks_theme_design_notes.md (slot catalogue, picker model)
  - four_tasks_staggered_disclosure_design_notes.md (what NOT to show now)
  - four_tasks_morning_sequence_design_notes.md (MOTD timing)
  - four_tasks_timezone_and_sealing_design_notes.md (silent tz capture)
  - four_tasks_write_rules_design_notes.md (resolve + migration endpoints)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PROBLEM
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Four Tasks is a two-user app by premise. Pair-key v2 requires all six
identity values (name + username + active_leader × 2 users) to exist
before a real pair-key can compute. The naive interpretation — both
users must complete onboarding together — creates an adoption-killing
friction point: a user who hears about the app but can't reach their
partner right now bounces.

Onboarding must work for three real-world install patterns:

  1. Both users available together. Standard happy path.
  2. User installs solo, recruits partner later via invite link.
  3. User installs to snoop around, may or may not ever pair.

All three paths land at the same end state — a functioning pair — but
the timing varies. The architecture must support solo mode as a first-
class state, not as a degraded fallback.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DECISION SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. SOLO MODE EXISTS. A user can install, complete onboarding with
   only their own three values, and use the full app. The partner
   panel is empty but functional as a recruitment surface.

2. INVITE LINK IS THE PRIMARY PAIRING PATH. Solo user shares a link
   carrying their three values + their solo UUID. Recipient taps,
   onboarding prefills, enters their own three values, pair resolves.

3. MANUAL PAIRING IS A FALLBACK. Recipient types all six values
   manually. Server tolerates this 99%+ of the time; falls back to
   "use the link instead" on the rare three-value collision.

4. SOLO DATA MIGRATES INTO THE PAIR. When solo user A finally pairs
   with B, A's calendar history, coins, streaks all carry over.
   Re-uses pair-key migration machinery.

5. MOTD DOES NOT FIRE AT ONBOARDING. The first MOTD lands during
   the first morning sequence, with full ceremony. Onboarding ends
   on sticker pick.

6. PARTNER PANEL IS THE RECRUITMENT SURFACE. No modal nags. The
   always-visible empty partner panel is itself the prompt. Single
   contextual nudge at moments where partner involvement would have
   mattered (first streak, etc.), then silent.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE ONBOARDING FLOW (HAPPY PATH, PRIMARY INSTALL)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Standard installer with no invite link. Lands on screen 1.

SCREEN 1 — Welcome.
  Frog (or chosen mascot), heading, three lines of body copy. One
  CTA: "Get started." One small text link below it: "Joining an
  existing pair? Recover here." Recovery path branches off here
  to tile 3.4 territory.

SCREEN 2 — Your name.
  Single text input. Below it, an inline note:
    "Choose carefully — your name can't be changed after this.
     Spell it the way your partner knows you."
  Validation: non-empty, no '|' or '__' characters (server rules).
  Continue button greys out until valid.

SCREEN 3 — Confirm your name.
  Modal-style screen showing the entered name in large type:
    "You typed: SANDRA
     Is this how your partner knows you?"
  [Yes, lock it in] [Back to edit]
  Name is committed as immutable from this point. No going back
  via app UI. (Reboot button in settings would wipe and start
  over — that's a separate path.)

SCREEN 4 — Your username.
  Single text input. Inline note:
    "Your username. You can change this any time."
  Validation as above. No confirmation step — mutable, low stakes.

SCREEN 5 — Your sticker.
  Reduced picker grid showing the LAUNCH STARTER SET — a curated
  small selection of feature-rich stickers from the catalogue
  (those that ship the most slot contributions, the visual
  showpieces). Tap to select. Selected sticker shows the chosen
  state. The app background previews the chosen theme in real
  time so the user sees what they're picking.

  No themed-marker badges in this reduced picker. Every option is
  feature-rich, so the badge is meaningless here.

  Continue button enabled once a sticker is selected.

  Silent effects of this choice:
    - active_leader = selected sticker (identity-hashed, participates
      in pair-key)
    - active_theme.palette = selected sticker (defaults to match
      leader)
    - active_theme also fills any other slots this sticker can
      contribute to (background, cell treatments, etc.) — the
      onboarding pack acts as an APPLY-ALL on the first sticker
    - active_stickers pool = [selected sticker] + a small set of
      neutral default stickers (3-4 — chosen at art time)

SCREEN 6 — Do you have your partner with you?
  Two CTAs:
    [Yes, they're here] — proceeds to screens 7-9 (partner entry)
    [Not yet — I'll invite them later] — proceeds to screen 10
                                          (solo confirm)

  This is the fork between "both-together" and "solo install."
  Below the buttons, one line of soft copy:
    "Four Tasks works best as a pair, but you can start solo
     and invite them later."

  Snoopers exit here too — they pick "Not yet" and explore.

SCREEN 7 — Partner's name.
  As screen 2, but for partner. Same immutability copy:
    "Choose carefully — your partner's name can't be changed
     after this. Spell it the way they know themselves."

SCREEN 8 — Confirm partner's name.
  As screen 3, partner version.

SCREEN 9 — Partner's username.
  As screen 4, partner version. Mutable, no confirmation.

  After screen 9: server resolves pair-key. Either:
    - New pair created → proceed to screen 11 (tasks).
    - Existing pair matched → user is rejoining via recovery path.
      Handled per tile 3.4. Probably exits onboarding entirely
      and loads the existing pair data.
    - Collision (same hash as a DIFFERENT existing pair) →
      collision popup per tile 3.4. User varies username and
      retries.

SCREEN 10 — Solo confirm (only if "Not yet" chosen at screen 6).
  One screen explaining what solo mode is:
    "You're starting solo. Your partner's space will be empty
     until they join. When you're ready, tap the empty partner
     calendar to send them an invite."
  [Continue]
  Server creates a solo pair-key (UUID-based, see Server section).

SCREEN 11 — Your four tasks.
  Four text inputs, vertically stacked. Each input has a
  placeholder pulled from a curated pool (e.g. "drink some water",
  "stretch for five minutes", "text your mum back"). Placeholders
  randomise per install — variety + creative priming.

  Above the inputs, one line of context:
    "Four things you'd like to remember to do each day. Keep them
     small. You can change them any time."

  Validation: all four non-empty. Continue button gates on this.

SCREEN 12 — Calendar tour.
  Onboarding ends here. App loads the main calendar. A single
  dismissable tutorial overlay appears with three or four arrow-
  annotations pointing at key UI elements:
    - "This is today. Tap each task as you do it."
    - "Swipe right to see your partner's calendar." (or "to see
      your partner's space" in solo mode)
    - "This is your coin balance and streak — these grow as you
      go."
  One [Got it] button dismisses. Overlay never shows again.

  MOTD does NOT fire here. First MOTD waits for tomorrow morning's
  ceremony.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE ONBOARDING FLOW (INVITE LINK PATH)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

User B taps a link of the form:
  https://thepicklemoon.com/pair?u=<solo_uuid>&n=<name>&un=<username>&l=<active_leader>

If app is not installed: link routes through the app store via
universal link / app link configuration. Link parameters survive
the install (handled by platform install-referrer mechanisms — see
"Platform configuration" below).

If app is installed: link opens app directly into a special
onboarding entry point.

SCREEN A1 — Joining your partner.
  Shows the inviter's name, username, and leader sticker prominently:
    "You're joining MORGAN (morgan91, 🐸)."
  [Continue] [This isn't right]

  "This isn't right" exits to standard fresh onboarding (screen 1).

SCREEN A2 — Your name.
  As primary screen 2. Same immutability copy.

SCREEN A3 — Confirm your name.
  As primary screen 3.

SCREEN A4 — Your username.
  As primary screen 4.

SCREEN A5 — Your sticker.
  As primary screen 5.

  After screen A5: client sends solo_uuid + B's three values to
  server's join-by-link endpoint. Server validates, computes pair-
  key, migrates A's solo row to real pair, attaches B. Returns
  success. Client proceeds to screen A6.

SCREEN A6 — Your four tasks.
  As primary screen 11.

SCREEN A7 — Calendar tour.
  As primary screen 12. Single difference: partner panel is now
  populated with A's existing solo data, so the tour text reads
  naturally ("Swipe right to see your partner's calendar.").

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE ONBOARDING FLOW (MANUAL JOIN — FALLBACK)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

User B installs fresh, no link. Goes through primary flow,
arrives at screen 7 (partner's name) and onward, typing all
three of A's values manually (read from screenshot, dictated
over phone, etc.).

After screen 9 (server resolve):
  - If A is a solo user uniquely matching the typed values:
    server migrates A's solo row, creates the real pair, returns
    success. B's app proceeds to screen 11.
  - If A is solo but multiple solo users match the typed values:
    server returns "ambiguous_match" error. Client shows a
    dedicated popup:
      "We found more than one person with these details. Please
       ask your partner to send you the invite link directly —
       tap the empty partner calendar on their app."
    [OK] returns to screen 9 with the partner username field
    cleared (most common disambiguator).
  - If no solo user matches and no real pair matches: server
    creates a new real pair with B's six values, leaves A's
    actual solo pair untouched. B and A are now NOT connected
    (B's "partner" data is a phantom user that doesn't exist).
    This is the misspelling case. B will notice the partner
    panel is permanently empty/incorrect. Recovery: reboot B's
    install, retype carefully.

  The misspelling case is a design accept. The user typed wrong
  values; the app cannot read minds. Future tile: a "is this
  your partner?" sanity check screen after pair resolve that
  shows partner's name + active_leader for confirmation before fully
  committing — defer to v1.x polish.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SOLO MODE — APP BEHAVIOUR
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

What works in solo mode:
  - Full task list, tick, MOTD, calendar.
  - Stamp animation on completion.
  - Coin economy at base rate (no 1.5x multiplier — that requires
    a partner).
  - Morning sequence — fires normally. Partner panel section of
    the sequence is replaced with a single recruitment beat (see
    "Recruitment surface" below).
  - Theme system (active_stickers pool, active_leader, active_theme
    slots) all functional. Post-fork features (per-slot context menu
    on long-press) still gated by staggered disclosure day 7+.
  - Streaks, rest days, milestones — all work.

What does not work:
  - Partner reactions. No partner exists. Tile 4.16 surface hidden.
  - 1.5x multiplier on daily payout. Single-user rate only.
  - Partner panel content. Empty calendar + recruitment CTA.

What appears in the partner panel (recruitment surface):
  - A muted, gently styled empty calendar — visibly NOT broken,
    just empty.
  - Centred copy:
      "Your partner's space is waiting."
      "Send them an invite — they'll see your tasks, you'll see
       theirs, and you'll cheer each other on."
  - One primary CTA: [Send invite].
  - One small secondary text link: "They've already started solo?
    Combine accounts." (Defer to v1.x — pairing two solo users is
    a separate flow, see "Open work" below.)

[Send invite] opens the native share sheet with a pre-composed
message:
  "Hey! I'm using Four Tasks — it's a small habit tracker for
   pairs. Tap to join me: <invite_url>"
User edits or sends as-is.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SERVER — SOLO IDENTITY MODEL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

A solo user has:
  - A solo_uuid (server-generated, UUIDv4, unguessable by construction).
  - A pairs row keyed by `solo:<uuid>` (not by the six-value hash —
    because there are no six values yet).
  - A single users row attached to that pair-key. Partner half is
    absent.
  - A days row each day they use the app, attached to the solo pair-key.

The solo pair-key is a string with a distinguishable prefix
(`solo:`) so server logic can branch on solo-vs-real cheaply
without an extra column. Format:

  pairs.key = "solo:" + uuid_v4_hex

Real pair-keys remain the SHA-256-truncated-16-hex format from
pair-key v2. The two formats are non-overlapping by prefix.

The solo state is otherwise identical to a real pair from the
data model's perspective — same days table, same users table,
same coin/streak/MOTD fields. Only the pair-key shape differs
and the partner user row is absent.

SCHEMA:
  No new columns required. Existing pair-key column handles solo
  via prefix convention.

  Existing `users` table: solo user is just a single row pointing
  at the solo pair-key, with the partner row absent.

  Migration_005 (new): no schema changes; just documentation in
  schema comments that pair-keys may be `solo:<uuid>` and that
  the users table may have one or two rows per pair-key.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SERVER — JOIN ENDPOINTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Two endpoints for joining a solo pair:

POST /join_by_link
  Body: {
    solo_uuid: string,
    inviter_values: { name, username, active_leader },  // from the link, for validation
    joiner_values:  { name, username, active_leader }
  }
  Logic:
    1. Look up pairs row by solo:<solo_uuid>. 404 if not found.
    2. Look up users row attached. Validate stored values match
       inviter_values exactly (case-sensitive, whitespace-normalised).
       404 with "stale_invite" if mismatch — the inviter changed
       their identity after generating the link.
    3. Compute new real pair-key from all six values.
    4. Open transaction.
    5. INSERT new pairs row with real pair-key.
    6. UPDATE existing users row to point to new pair-key.
    7. INSERT new users row for joiner, pointing to new pair-key.
    8. UPDATE all days rows from solo pair-key to new pair-key.
    9. DELETE old solo pairs row.
    10. Commit.
    11. Return new pair-key + full pair state to joiner.

POST /join_by_values
  Body: {
    inviter_values: { name, username, active_leader },
    joiner_values:  { name, username, active_leader }
  }
  Logic:
    1. Search solo pairs (pair-key starts with `solo:`) whose
       attached user matches inviter_values exactly.
    2. If exactly one match: proceed as join_by_link from step 3.
    3. If zero matches: search REAL pairs whose A or B user matches
       inviter_values AND whose other user matches joiner_values.
       This handles the recovery case (existing real pair, B is
       reinstalling). If found, return pair state to joiner without
       creating anything new. If not found, create a new real pair
       with both users (the no-such-partner case — partner is a
       phantom until they install).
    4. If more than one solo match: return 409 "ambiguous_match".
       Client surfaces the "use the link" message.

Both endpoints are part of tile 1.3's write rules scope, alongside
the existing resolve/put endpoints. Field-level rules and rejection
codes per the write rules design doc.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
INVITE LINK FORMAT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

URL shape:
  https://thepicklemoon.com/pair?u=<solo_uuid>&n=<name>&un=<username>&l=<active_leader>

All four parameters URL-encoded. The pickle-moon-public repo hosts
the landing page at thepicklemoon.com/pair which:
  - On a phone with the app installed: deep-links into the app via
    universal links (iOS) / app links (Android).
  - On a phone without the app: redirects to the appropriate app
    store, preserving the parameters via the platform's install-
    referrer mechanism.
  - On desktop: shows a simple "open this on your phone" message
    with a QR code containing the same URL.

Values in the URL are plaintext. They are not secrets — they are
visible on the inviter's app screen to anyone they show their
phone to. The pair-key v2 doc's threat model already accepts that
shoulder-surfing all six values is out of scope. The URL only
exposes three of six.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PLATFORM CONFIGURATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Universal Links (iOS):
  - Apple App Site Association (AASA) file hosted at
    thepicklemoon.com/.well-known/apple-app-site-association
  - Lives in pickle-moon-public repo, deployed alongside the
    privacy policy at Phase 5.

App Links (Android):
  - Digital Asset Links file at
    thepicklemoon.com/.well-known/assetlinks.json
  - Also in pickle-moon-public.

Install-referrer survival:
  - iOS: Apple's deferred deep linking (via Branch.io or
    SKAdNetwork-equivalent) — third-party SDK decision deferred
    to Phase 5.
  - Android: Google Play Install Referrer API — built-in, no SDK.

  The third-party SDK question (Branch, AppsFlyer, etc.) for iOS
  install-referrer is its own decision — defer to Phase 5. For
  v1.0, accept that iOS users without the app installed lose the
  invite parameters across the App Store roundtrip. They land on
  fresh onboarding and have to type values manually. Sub-optimal
  but ships. Android users get the full experience day one.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SCREEN COUNT SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Primary flow (paired):       12 screens (1-12).
Primary flow (solo):         9 screens  (1-6, 10-12, skipping 7-9).
Invite link flow:            7 screens  (A1-A7).
Manual fallback (paired):    12 screens (same as primary).

Confirmation modals on top of screen counts:
  - Name confirmation: own name (always), partner's name (paired flow only).
  - Tutorial overlay (screen 12 / A7): one dismissable overlay.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STAGGERED DISCLOSURE INTERSECTION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The staggered disclosure principle says onboarding teaches the
bare minimum. This doc holds the line on that:

  Onboarding teaches:
    - Your name, your username, your sticker.
    - Your partner's three values (or that you can invite later).
    - Your four tasks.
    - Where today is, where your partner's panel is, what coins
      and streaks are (one-shot tour overlay).

  Onboarding does NOT teach:
    - Rest days (revealed day 3).
    - Coin economy depth — what to spend on (revealed day 2).
    - MOTD reroll (revealed when first useful).
    - Theme picker context menu (revealed day 7+, post-fork).
    - Partner reactions (revealed day 4).
    - Streak milestones (revealed when first hit).
    - Coin name reroll (revealed when first affordable).
    - Subscription (revealed day 21).
    - Bug catcher / help menu (always accessible, not surfaced).

The tutorial overlay at screen 12 is the absolute floor of what
must be communicated. Everything else is the staggered-disclosure
system's job over the following weeks.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RECOVERY PATH (RECAP)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Triggered from screen 1's "Joining an existing pair? Recover here"
text link. Owned by tile 3.4 design.

Flow:
  - User enters all six values from memory (or from their partner's
    visible app state).
  - Server resolves to existing pair-key.
  - If match: data loads, user lands in main app, skips onboarding.
  - If no match: collision popup with masked-hint mechanism (see
    pair-key v2 doc's "Masked-hint recovery aid" section).

Recovery does NOT touch solo pairs — only real pairs. A user who
was previously solo and lost their data has the same recovery
options as any other user, but their solo pair-key (`solo:<uuid>`)
was server-generated and not memorable. They can't recover a solo
state. They can re-install fresh as solo and re-invite their
(future or never-existent) partner.

This is a deliberate accept. Solo users have less to lose than
paired users — no streak with another person, no shared history.
Recovery prioritises paired data.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TODO TILE UPDATES REQUIRED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Phase 3 tiles need rewording against this doc. Currently:

  3.1 [CODE] Onboarding scene state machine.
  3.2 [CODE] Welcome step.
  3.3 [CODE] Name + partner name steps.
  3.3a [CODE] Username step.
  3.4 [CODE] Pair-key resolve + collision/recovery popup.
  3.5 [CODE] Task setup step.
  3.6 [CODE] Slot intro + diary popup steps.

After this doc, tile structure should become roughly:

  3.1 — State machine + entry routing (link vs fresh vs recovery).
  3.2 — Welcome screen + recovery branch entry point.
  3.3 — Own name + confirmation modal.
  3.3a — Own username.
  3.4 — Own sticker (reduced launch-starter-set picker).
  3.4a — "Do you have your partner with you?" fork screen.
  3.5 — Partner name + confirmation, partner username.
  3.5a — Solo confirm screen.
  3.6 — Server resolve + collision/recovery popup (existing tile,
        expanded to include solo + invite endpoints).
  3.7 — Four tasks input with placeholder pool.
  3.8 — Calendar tour overlay.
  3.9 — Invite link onboarding entry path (A1-A7 flow).

The old tile 3.6 ("slot intro + diary popup") is dropped — MOTD
no longer appears at onboarding.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OPEN WORK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Defer to implementation or future design passes:

  - Exact copy on every screen — UX writing pass. This doc captures
    the message intent; the prose is iteration territory.
  - Placeholder task pool — needs a curated list of ~50 examples.
    Content authoring tile.
  - Two-solo-users pair-merge flow. If both A and B installed solo
    independently and later want to combine, neither has the other's
    invite link and manual join sees ambiguous match. Defer to v1.x.
    Workaround at launch: one user reboots their install and joins
    the other via link.
  - iOS install-referrer SDK decision (Branch.io vs alternatives vs
    accept Phase-1 friction). Defer to Phase 5.
  - "Is this your partner?" sanity-check screen after manual pair
    resolve, showing partner name + active_leader sticker for
    confirmation. Defer to v1.x polish.
  - Solo recruitment surface visual design — the empty partner
    panel's exact look. Art + UX tile.
  - Confirmation modal exact UX — single button to lock immutable
    name vs full-screen modal vs inline accordion. Implementation
    decision.
  - Recovery flow detail — masked hint mechanism, rate limiting,
    fields eligible for hinting. Captured in pair-key design doc,
    lands at tile 3.6 implementation.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED ARCHITECTURAL CALLS THIS SESSION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  1. Solo mode is a first-class state, not a degraded fallback.
     The architecture supports it via prefix-based pair-key
     convention with zero schema cost.

  2. The invite link's UUID is what makes link-path joining safe
     against three-value collisions. Manual-path joining handles
     the rare collision case via explicit "use the link" error.

  3. Solo data migrates into the pair on join. Re-uses pair-key
     migration machinery. No new code path; the existing migration
     transaction handles the case.

  4. MOTD ceremony lives in the morning sequence, not onboarding.
     This preserves the first morning's emotional weight.

  5. The partner panel itself is the recruitment surface. No modal
     nags. Trust the always-visible empty state.

  6. Name remains immutable even in solo mode. A solo user who later
     pairs cannot change their own name to better match their
     partner's expectations. Be deliberate at onboarding.
