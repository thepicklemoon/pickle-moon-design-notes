# Four Tasks — Marketing Notes

Status: ACTIVE, EVOLVING
Started: session 5 (May 2026)
Purpose: capture marketing thinking as it arrives. Structured sections
at top for the major plays; journal entries at the bottom for raw
thoughts as they land. Re-shape when critical mass justifies it.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CURRENT STATE — TOP-OF-DOC SNAPSHOT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Launch strategy is a SEQUENCED two-product drop (revised late session 5):

  1. Four Tasks — paid, native, full-featured version on Apple App
     Store and Google Play. Ships FIRST at end of Phase 5. Quiet
     launch via subculture seeding — no atrioc beat yet. Lets the
     product find its audience on its own merits.
  2. APPtrioc — free, web-only, community-gift version of Four
     Tasks. Built in Phase 5b after Four Tasks is live. Hosted on
     Cloudflare Pages at apptrioc.thepicklemoon.com (or similar).
     NOT on app stores. NOT Capacitor-wrapped. PWA "add to home
     screen" support is the install ceiling.
  3. The atrioc video + Reddit post drops AFTER both Four Tasks
     is on stores AND APPtrioc is live. Force-multiplier moment
     on an already-launched product, not the primary launch driver.

KEY ARCHITECTURAL CHANGE FROM EARLIER IN SESSION:
  APPtrioc is no longer "a marketing fork of the web prototype."
  It is a SNAPSHOT of the Four Tasks Godot project at tile 4.14a
  (basic sticker picker complete, before the long-press context
  menu adds theme depth). Exported via Godot HTML5, deployed to
  Cloudflare Pages, configured for unilateral pairing + atrioc
  theming + three-button onboarding. This means APPtrioc is the
  SAME polish and quality as Four Tasks, just with the theme-depth
  features cut off and the unilateral pairing configured. The
  gift is real, not a janky prototype.

  Tile 4.14 has been split in the todo (4.14a pre-fork, 4.14b
  post-fork) to make the fork point a structural feature of the
  development plan. See `four_tasks_godot_todo.txt` "APPTRIOC FORK
  ARCHITECTURE" section at top.

Target launch window: shaped by family priorities (Morgan's father
unwell) and the dependency chain — Four Tasks must be Phase 5 ready
before APPtrioc can be built (tile 4.14a must close first), then
Phase 5b adds the APPtrioc fork+ship cycle. LAUNCH READINESS, NOT
CALENDAR, is the optimisation target. Earliest plausible Four Tasks
ship is late Q3 2026; APPtrioc ships shortly after. The atrioc beat
is not time-sensitive (the video can quote atrioc's original moment
regardless of when he showed his app).

A high-effort YouTube video bridges the two: tells the story (Morgan,
beam orb, sticky steve, FIFO partner connection, seeing atrioc's vibe-
coded calendar, iterating it into Four Tasks, gifting the community
APPtrioc, mentioning the full version on stores). Reddit post on
r/atrioc same day, timed to hit the 100-upvote threshold for inclusion
in the monthly Reddit recap.

Tip jar: NONE. APPtrioc is a clean gift. Redirect to the paid app is
the only ask.

Long-tail strategy is community-driven content + collaborations,
deferred channel decisions (Discord vs subreddit), in-app news for
theme/music/artist drops, NYE keepsake and monthly theme-seal moments
as natural marketing beats.

Subreddit subculture seeding is paired with TARGETED PROMO CODES
offering 3-month free trials + soft landing into founders pricing.
See Section 6 below for the locked promo code model (added session 7).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SECTION 1 — APPTRIOC LAUNCH PLAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

THE IMPETUS:
  Atrioc — ex-Twitch employee, full-time streamer, builds in a vibe-
  coding hobbyist mode. Recently showed his audience an app he made
  for himself: a calendar where you track 3-4 daily things, à la
  Seinfeld's "don't break the chain." His line: "this fucking exact
  thing doesn't exist so I just made it." That line is the narrative
  spine of the entire launch.

  Four Tasks predates atrioc's calendar (or is independent of it —
  the partner mechanic is the real innovation), but the parallel is
  the hook. Morgan saw the same gap and went further: a partner
  calendar, not just a personal one. That's the story.

THE COMMUNITY CONTEXT:
  Atrioc's audience is a self-aware, pun-loving, in-joke-heavy crowd.
  Examples of community linguistic play: "spoontrioc," "down with
  the atriarchy." The community has its own Reddit which atrioc
  features in monthly recap videos. The recap auto-includes posts
  that hit roughly 100 upvotes — a known threshold and a real
  mechanism for community OC to reach atrioc's main audience.

  Audience reach: main videos ~1M views, gaming content ~500k,
  politics content ~250k. Reddit recap is consistently popular but
  exact view count uncertain.

THE GIFT:
  APPtrioc is the gift. Specifications:
    - A SNAPSHOT of the Four Tasks Godot project at tile 4.14a,
      exported via Godot HTML5 target. Same polish, same quality,
      same engine as Four Tasks proper — just with the theme depth
      features deliberately absent and unilateral pairing
      configured. This is the structural shift from the earlier
      framing of "fork the prototype." The prototype is barebones
      and buggy; shipping a buggy gift would have undercut the "by
      the way, the full version is on stores" pitch.
    - Web-only, hosted at apptrioc.thepicklemoon.com (or similar
      subdomain). Deployed to Cloudflare Pages.
    - NOT Capacitor-wrapped. PWA manifest + "add to home screen"
      support is the install ceiling. Keeps operational burden
      minimal — no app store listings, no review queues, no
      iOS/Android divergence to maintain. Browser limitations (push
      notifications on iOS specifically) accepted as the cost of
      the lower-burden path. The daily-check-in mechanic mostly
      works without push.
    - Community-themed. Hand-painted stickers riffing on in-jokes
      (spoontrioc, atriarchy, etc — exact set TBD). SMALL, focused
      sticker catalogue — likely 5-8 stickers total, mix of inside-
      joke specific and palette-only fillers. Deliberately limited
      vs the full Four Tasks system, both because the painting
      budget is finite and to create paid-app pull for users who
      want the deeper catalogue.
    - Long-press on stickers is DELIBERATELY INERT. The picker
      shows themed-sticker markers (glints/badges) hinting at
      depth, but APPtrioc users can't access it — long-press
      either does nothing or shows a small tooltip pointing to
      Four Tasks. This is the conversion mechanic working as
      designed, not a missing feature.
    - Task label placeholders are inside jokes: "interact-rioc every
      day" (drop a like / prime / sub / general engagement), other
      community-flavoured labels TBD.
    - Default partner is atrioc. UNILATERAL pairing model — see
      onboarding section below.
    - Free. No tip jar. No ads. No data harvesting. Pure gift.
    - Privacy policy applies (THE PICKLE MOON's standard policy
      from pickle-moon-public, possibly with a small APPtrioc-
      specific note about the password-gate mechanic).

THE UNILATERAL PAIRING MODEL — STRUCTURAL DIFFERENCE FROM FOUR TASKS:
  In Four Tasks proper, pairing is bilateral. Both users see each
  other's calendars. The pair-key v2 system enforces this — six-
  value identity tuple, both sides commit.

  In APPtrioc, pairing is one-way. Every regular user has atrioc as
  their fixed partner. They see his calendar. He does NOT see each
  of theirs — that would be unscalable (thousands of partner
  calendars on one screen) and unwanted (atrioc isn't running
  customer support for every user's daily habits).

  This is a real architectural divergence, not a config flag. The
  data model needs to support "user X follows atrioc; atrioc does
  not have user X as a tracked partner." The Four Tasks pair-key
  model doesn't natively express this asymmetry. APPtrioc's
  identity layer needs its own design.

  Sketch (to be refined in its own design pass):
    - atrioc has a single calendar that all APPtrioc users read.
      Public-ish — they see his progress, not his identity tuple.
    - Each regular user has a private calendar that only they see.
      No partner reads their data. They consume atrioc's data
      but produce data for themselves only.
    - Ari is the exception. She and atrioc have bilateral pairing
      between each other, similar to Four Tasks proper. Their
      calendars are mutually visible to each other only.
    - Implementation likely: APPtrioc has its own simpler schema,
      not a fork of the Four Tasks D1 schema. A `users` table
      keyed by username + password hash. A single hardcoded
      "atrioc" user whose calendar is readable by all. A
      special-case bilateral relationship between atrioc and Ari.

  Customer service implications: atrioc has zero CS burden. He
  doesn't see anyone's data except Ari's. If a user has a bug or
  question, it routes to Morgan via standard support channels,
  not to atrioc. Worth being explicit in the video and in the
  app: "atrioc isn't reading your calendar; this is your private
  space with him as a fixed motivational presence." That framing
  is honest and protective on both sides.

ONBOARDING FLOW — THE PASSWORD HOOK:
  When the app loads for the first time, user is prompted with a
  small set of identity buttons. Working draft:
    - "I'm atrioc"        (requires password)
    - "I'm Ari"           (requires password — Ari is atrioc's wife)
    - "I'm everyone else" (normal pair flow)

  Atrioc's password is hidden in his Twitch chat or a YouTube video
  description in the 1-2 weeks leading up to launch. Community
  members find it, share it, build pre-launch buzz organically.
  Morgan doesn't need to be online for any of this to happen.

  Ari's password handled separately. Could be hidden somewhere else
  to make it a fun moment of "wait who's that" when she sees it.

  If atrioc logs in with the correct password, his partner is auto-
  set to Ari. If Ari logs in, her partner is auto-set to atrioc.
  Everyone else falls through to the standard pair-key flow with
  their own chosen partner.

  Edge case: what if both atrioc and Ari hit "I'm atrioc"? What if
  one hits the right button and the other never installs it? These
  are real failure modes. Mitigations TBD but they exist as
  questions, not blockers.

THE VIDEO:
  10 minutes target. Structure:
    1. Morgan introduces himself. FIFO worker, indie dev, THE PICKLE
       MOON. Brief — under 90 seconds.
    2. Beam orb and sticky steve — first games, first contact with
       vibe-coding, what was learned, what worked, what didn't. The
       "I can actually make software" moment.
    3. Saw atrioc's calendar on stream. "This exact thing doesn't
       exist." Decided to build something similar for the wife.
       FIFO context: gone for weeks at a time, needed something to
       stay in each other's days while apart.
    4. Iterated into Four Tasks — partner calendar, two real users,
       morning sequence, sticker pool, theme system. Brief tour.
    5. The gift moment. APPtrioc unveiled. Walk through the inside
       jokes. Atrioc as default partner. Live URL on screen.
    6. Soft thank-you. Acknowledge atrioc and community for years
       of entertainment. APPtrioc is the gift. No catch.
    7. The redirect. "By the way, if you liked this and want the
       full system — pair with anyone, hand-painted theme library,
       all the depth — Four Tasks is on the stores right now. Two
       real users, your partner / mate / family / nemesis, whoever."
    8. Sign-off. THE PICKLE MOON. Discord/subreddit link. Done.

  Tone: warm, slightly self-deprecating, no marketing voice. Plain
  language. Honest about being a solo dev. The story sells; the
  pitch is incidental.

  RECORDING TIMING (locked late session 5):
  Video is recorded AFTER Four Tasks is on stores and APPtrioc is
  live and tested. The demo footage in points 4 and 5 is real-
  product, not placeholder. This is a Phase 5b tile (5b.10) — the
  whole launch sequence is: ship Four Tasks → build APPtrioc →
  test APPtrioc → record video → drop everything. NOT: record
  video against placeholder, then race to ship the products to
  match. Authenticity of the footage is part of the pitch.

THE CHAIN-IMPORT SWEETENER:
  Build a one-time import path so atrioc, when he first opens
  APPtrioc, sees his existing don't-break-the-chain calendar
  already populated. Pulled from the data visible on his stream
  during the original reveal. Engineering cost: low (one-off
  import script reading from a manually-entered date list). User
  experience cost: a small touch that signals "I watched what you
  built, I respected it, I'm building on it not replacing it."
  Increases his probability of actually adopting APPtrioc as a
  daily tool — which in turn dramatically increases the chance of
  follow-up moments in his streams where he references it
  organically, which is worth more than any paid sponsorship.

CONTINGENCY — IF ATRIOC DOESN'T ENGAGE:
  REDUCED RISK PROFILE due to sequenced launch.

  In the original simultaneous-launch plan, atrioc's reaction was
  a real load-bearing dependency. With the sequenced launch, Four
  Tasks is ALREADY ON STORES by the time the atrioc beat drops.
  The atrioc beat is a force multiplier on an already-shipped
  product, not a launch driver.

  Best case: atrioc sees the video, downloads APPtrioc, mentions it
  on stream, the Reddit post hits the recap, the community floods
  in. Four Tasks gets a secondary surge of exposure as the natural
  upgrade.

  Middle case (most likely): atrioc never sees it, or sees it
  cooly. APPtrioc lives at its URL anyway as a community gift,
  generating its own slow trickle of users. Four Tasks continues
  whatever growth it had from the Phase 5 quiet launch. The
  cherry-on-top didn't land but the cake is already on the table.

  Worst case: atrioc reacts negatively. Probability low given the
  genuine framing and gift structure, but non-zero. If it happens,
  APPtrioc comes down quietly, the video gets pulled or re-cut,
  and Four Tasks continues to find its audience through subculture
  seeding. Reputational damage is minimal because Four Tasks
  exists independently — no "this launch was a flop" narrative,
  just "that one piece of marketing didn't land."

  Bottom line: the sequenced launch structure makes atrioc's
  reaction ENTIRELY upside. The risk of relying on it as a
  primary driver is eliminated.

THE PASSWORD-HUNT BUZZ:
  Hiding atrioc's password in chat 1-2 weeks before launch creates
  its own pre-launch moment. People will:
    - Notice the password being said / typed
    - Speculate about what it's for
    - Share theories in the Discord / subreddit
    - Build anticipation organically
  No marketing budget required. The cost is one carefully-timed chat
  message. Morgan doesn't need to be present or active during the
  buildup. Worth a thought: leak a *second* password later for Ari,
  in a different community space, so it becomes a small treasure
  hunt rather than a single one-shot.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SECTION 2 — SUBCULTURE SEEDING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Four Tasks is a multi-purpose primitive — "two people commit to
small daily things together" — that maps onto many communities. Each
needs its own framing. The product is the same; the pitch is per-
audience.

Seven primary subcultures plus two added in session 5 conversation:

ADHD COMMUNITIES.
  Angles: body doubling, externalised accountability, fiddly
  customisable UI satisfies the dopamine-on-tweaking trait, low-
  stakes daily check-ins reduce executive function burden vs full
  habit trackers that demand setup ceremony.
  Channels: r/ADHD, r/ADHDmemes, ADHD coach Twitter/Bluesky, ADHD
  TikTok if Morgan ever does video there. Pinned posts in
  community Discords if invited in.
  Voice: lived-experience, non-clinical, no productivity-shaming.

GYM / WELLNESS.
  Angles: streak mechanic for daily habits, partner accountability
  (gym buddy / training partner), MOTD system can be re-framed as
  daily motivation. The "rest day" mechanic respects rest as a
  legitimate state, not failure — aligns with current wellness
  discourse.
  Channels: r/Fitness, r/loseit, r/xxfitness, r/yoga, indie wellness
  Discords. Coach / PT crossovers possible.
  Voice: practical, low-key, anti-grindset.

PRODUCTIVITY COMMUNITIES.
  Angles: minimal cognitive load, no productivity-theatre, no
  notifications, the partner mechanic differentiates from every
  other habit tracker. Anti-pattern: don't lean too hard into
  "productivity" branding — Four Tasks is warmer than that.
  Channels: r/productivity, r/getmotivated, r/decidingtobebetter,
  Notion/Obsidian-adjacent communities (different aesthetic but
  overlapping audience).
  Voice: skeptical-of-productivity-culture, real-tools-not-systems.

INDIE DEV.
  Angles: hand-drawn pix art, all music indie, solo dev story,
  Godot port content. Indie devs love seeing other indie devs ship.
  Channels: r/IndieGaming, r/IndieDev, r/godot, indie game Discords,
  Bluesky #gamedev, devlog YouTube.
  Voice: "I built this thing, here's how, here's where it failed."
  This is where the devlog content lives.

FIFO / DISTANCE WORK.
  Angles: explicit origin story — Morgan built this because he
  works away from home for weeks at a time. Partner connection
  across distance is the headline feature for this group, not a
  side benefit.
  Channels: r/FIFOAustralia, r/mining (Australian sub), r/auswa,
  Facebook FIFO groups (older demographic but huge). Word-of-mouth
  on rosters is real.
  Voice: lived-experience. Morgan IS this audience. No translation
  needed.

COUPLES (GENERAL).
  Angles: the obvious one. A small daily ritual you do together,
  whatever shape that takes. The shared calendar, the MOTD, the
  morning sequence, the partner panel — all designed around two
  people sharing something small daily. (Partner reactions deepen
  this but land in v1.x — don't lead launch copy with them.)
  Channels: r/relationships, r/marriage, r/longdistance, couples
  Discord servers (small and scattered). Wedding/anniversary blogs.
  Voice: warm, no schmaltz, no normative assumptions about what
  "couple" means.

LGBT.
  Angles: safe space, colourful, no normative coupling assumptions
  in the UI ("partner" is identity-neutral), pride-themed sticker
  drops over time, community engagement matters.
  Channels: r/lgbt, r/actuallesbians, r/asktransgender, specific
  community Discords if invited. NOT a one-off launch post —
  ongoing presence matters more than a single drop here.
  Voice: not performative, not allyship-coded. Just present and
  inclusive in the actual product.

RECOVERY / SOBRIETY (added session 5).
  Angles: the daily check-in + accountability partner mechanic IS
  the sober-buddies pattern. Many recovery tools are AA-affiliated
  or evangelical; Four Tasks is neither. Day-counter / streak
  semantic is culturally legible. Privacy is critical (the partner
  is the only person who sees the data — no public profile, no
  social graph).
  Channels: r/stopdrinking, r/leaves (cannabis cessation),
  r/loseit (overlapping with gym/wellness), recovery Discords.
  Caveats: high-vulnerability audience. Marketing voice must be
  careful — never imply Four Tasks is treatment, never imply it
  can replace professional support. Consider promotional offers
  for long-term users in recovery contexts. Approach respectfully
  or not at all.
  Voice: lived-experience-adjacent, non-clinical, never
  evangelical.

LONG-DISTANCE FRIENDSHIPS (added session 5).
  Angles: the partner mechanic doesn't have to be romantic. "My
  best mate moved interstate and we use this to stay in each
  other's day." Lower commitment than a relationship app, higher
  warmth than a generic productivity tool. Particularly hits the
  group that *want* to stay close to friends but find existing
  tools (group chats, social media) too noisy or performative.
  Channels: r/longdistance (overlap with couples but distinct sub-
  audience), r/MakeNewFriendsHere, friendship-focused Discord
  servers.
  Voice: warm, low-key, "for the people who matter to you, romantic
  or not."

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SECTION 3 — STREAMER OUTREACH
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

PHASE 1 (LAUNCH) — ATRIOC, INDIRECT.
  No direct outreach. APPtrioc + video + Reddit post are the
  mechanism. Atrioc's engagement is upside, not dependency. Cost:
  $0. Risk: low (the video stands without his reaction).

PHASE 2 (REVENUE-FUNDED) — PAID STREAMER READS.
  Candidates identified:

  GinoMachino. Health-conscious dark souls streamer. Audience:
  productivity-curious gamers who appreciate craft. Pitch angle:
  "the daily habit app with a Souls-inspired rerollable message of
  the day." Cultural fit is exact — Souls players understand the
  death-and-message-from-strangers mechanic at a bone level. The
  MOTD reroll is a Soulslike feature in a habit tracker. That's
  the read.

  IronPineapple. Indie Souls-like coverage. Adjacent audience.
  Hand-drawn pixel art + Souls-aesthetic + indie-dev origin story
  all land here. Possibly a higher conversion rate than GinoMachino
  per-impression but smaller total audience.

ECONOMICS REALITY CHECK:
  Twitch sponsorship reads vary widely. Rough ranges (subject to
  market shifts):
    Small streamer (<5k avg viewers): $200-500 per read
    Mid-tier (10-50k avg): $1k-5k per read
    Upper-tier (50k+): $5k+ per read, often much more
  Atrioc himself sits at upper-tier and would be cost-prohibitive
  even if available. The "atrioc engages organically" play is the
  ONLY economically viable atrioc outcome.

  Break-even math example: a $1,500 read needs to net X new
  subscribers at $Y/month minus Apple/Google's 15-30% cut. The math
  often disappoints — sponsorship reads convert at lower rates
  than people expect, especially for a product that requires the
  viewer to *also recruit a partner* to use it. The partner
  requirement is a structural conversion drag worth pricing into
  the math.

  STRUCTURAL ADVANTAGE OF ATRIOC PLAY: $0 cost, story-driven
  conversion. The video does the conversion work. Paid reads are a
  Phase 2 supplement, not a primary channel. Don't overinvest in
  them early.

OUTREACH APPROACH WHEN PHASE 2 LANDS:
  - Reach via streamer business email (usually in channel about /
    panel, sometimes via management agency).
  - One-page pitch: what Four Tasks is, why this streamer's
    audience specifically, a sample read script the streamer can
    adapt to their voice.
  - Send the streamer a free pair-code or a free month so they can
    actually use the app before reading copy about it. Reads land
    better when the streamer has genuine usage to reference.
  - Negotiate a rate. Don't pay the first quote — there's typically
    flex.
  - Track conversion. If the first read pays back, fund a second.
    If it doesn't, learn what failed (creative? audience fit?
    timing?) before spending again.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SECTION 4 — LONG-TAIL CONTENT + COMMUNITY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The launch is a moment. The long tail is the work. Most indie
apps die in the trough between launch buzz and product-market fit
because the founder underestimated how much sustained content
work the long tail requires. This section is the plan to not die
in that trough.

NATURAL MARKETING BEATS BAKED INTO THE PRODUCT:
  - NYE keepsake (Jan 1 each year). When a year seals, the app
    offers a downloadable artefact of that completed year. This is
    a *cultural moment* if framed right. "Screenshot the year you
    built with your partner" — users will share these organically
    on social media. Plan a content push around the first NYE
    post-launch.
  - Monthly theme-seal. Each month gets locked when it ends, the
    theme it was sealed with is preserved. Visible monthly
    progress for the user; potential monthly "month in review"
    content prompts ("post your favourite sealed month").
  - New themed sticker drops. Each new theme is a content moment.
    Devlog video showing the painting process, the sticker reveal,
    the theme art transformation. The folder-per-sticker model
    makes "ship a new theme" a contained operation — sustainable
    cadence over months / years.
  - New theme music drops (if pursued). Same shape as sticker
    drops — content moment + collaboration opportunity.
  - Birthday anchors (potential feature). If implemented, partner's
    birthday becomes an in-app moment. Worth considering for v1.x.

OPERATING MODEL — HUB + ONE CHANNEL (locked session 20):
  The follow-through plan (content drops + subreddit + Discord +
  localisation + street teams) is a second full-time job that starts at
  launch and never ends, on a FIFO roster, competing for the same hours
  as the partner the whole venture exists to protect. Running channels
  BADLY is worse than not running them — a dead Discord signals
  abandonment to the exact people being converted. The model is built
  around what survives a roster, not what is theoretically optimal.

  HUB — thepicklemoon.com. The owned centre (not rented land: survives
  Reddit/Discord deplatforming and algorithm shifts). Permanent, cosy,
  devlogs, community, and the in-app link destination. Lowest marginal
  effort — domain already registered, Cloudflare Pages already known
  from the APPtrioc plan, devlog content already written in the public
  design-notes repo. This is the piece Morgan is most excited about, and
  excitement is the only fuel that survives a roster — so it leads.

  ONE CONVERSATION CHANNEL AT LAUNCH, NOT TWO. Broadcast (YouTube) plus
  one conversation channel is the ceiling; two conversation channels is
  not. The tension: Morgan likes Discord most, but the strategic read
  favours Reddit — better discovery, the atrioc post lives there anyway,
  and it is ASYNC so it forgives a week-on/week-off roster. Discord
  demands real-time presence a FIFO schedule cannot give; a founder
  vanishing fortnightly reads as abandonment. Lean Reddit at launch;
  Discord deferred until volunteers or proven traction can keep it warm
  (see CONTENT CHANNELS triage below). Decide this consciously — do not
  drift into running both.

  CONTENT BUFFER SURVIVES THE WORST REALISTIC MONTH — not merely "does
  not hit zero". A buffer that reaches zero was too shallow to begin
  with. Paint the stockpile pre-launch while build-energy still exists;
  the dopamine that funds it drains after launch. (Supersedes the
  "skip-this-fortnight acceptable if buffer hits zero" framing in the
  todo — right in spirit, too shallow in practice.)

  FIRST HIRE MAY PRECEDE THE 10k RUNG. The load from ~3k to 10k users
  may exceed one rostered person without wheels coming off at home. The
  case is for a part-time community manager funded by 3k-quit income —
  to make scaling SURVIVABLE, not as a reward earned at scale. (Ladder
  numbers are aspirational, not gospel.)

CONTENT CHANNELS — PICK FEWER, DO THEM WELL:
  Running Discord + Reddit + RSS + YouTube + social media + reply-
  to-every-comment is a full-time job. Solo dev who's also FIFO
  cannot sustain all of it. Honest channel triage:

  YOUTUBE — KEEP, PRIMARY.
    One-direction broadcast, asynchronous, FIFO-friendly to film
    in bursts when home off-roster. Devlog videos, feature reveals,
    sticker drops, behind-the-scenes paint timelapses, occasional
    longer-form essays about the design philosophy. Format that
    rewards depth and pays back for years (Soulslike SEO benefits
    discoverability long after upload).

  REDDIT — KEEP IF AUDIENCE IS REDDIT-NATIVE, OTHERWISE DROP.
    The atrioc launch is Reddit-native. The ADHD / productivity /
    indie dev subcultures are also Reddit-heavy. Subculture seeding
    plan IS a Reddit plan. Worth running.
    Decision: don't run an OFFICIAL r/fourtasks subreddit yet —
    nothing dies faster than an empty official sub. Wait until
    organic Four Tasks discussion appears in other subs, THEN
    launch the official sub and migrate the active users.

  DISCORD — DEFER UNTIL DEMAND IS PROVEN.
    Discord requires constant attention to feel alive. An empty
    Discord is worse than no Discord — visitors leave with the
    impression that the project is dead. If a Discord launches,
    it needs daily presence from Morgan for at least the first
    few months. Defer until: (a) Discord users are asking where
    the server is, AND (b) Morgan has a sustainable cadence to
    show up there.

  RSS — DROP.
    Niche audience, low engagement, high maintenance. YouTube
    serves the same "subscribers get notified of new content"
    function with a much larger reach.

  SOCIAL MEDIA (TWITTER/BLUESKY/TIKTOK) — OPPORTUNISTIC.
    Don't commit to a posting schedule. Post when there's
    genuinely something to share — a new theme, a milestone, a
    funny user story. The pressure of "I haven't posted in a week"
    is a content-quality killer.

COMMUNITY ENGAGEMENT PROMPTS — RUNNING BACKLOG:
  Save these for use when the community channel exists. Low-
  friction UGC that generates content without Morgan producing
  each piece.
    - What's the weirdest task label your partner added?
    - What's the most horrifying MOTD you've kept?
    - Screenshot your app — what's your current theme?
    - How have your four tasks changed since you started?
    - Post your sealed month — what was the vibe?
    - Show your partner's calendar (with their permission).
    - First check-in story — how'd you convince your partner to
      join?
    - Which sticker is your "this means I'm having a good day"?
    - What's a task you wanted to give up on but didn't because
      your partner could see?
  More to come. Saved here for the running list.

COLLABORATION TEMPLATE — DRAFT WHEN FIRST COLLAB HAPPENS:
  When the first artist or musician collaboration lands, capture a
  reusable template covering: commission rate (flat / per-asset /
  royalty?), exclusivity terms (can the artist sell the same work
  elsewhere?), attribution credit format (in-app artist credit
  page? splash screen?), promotion split (who announces, when,
  where), payment timeline, IP ownership clarification.
  Get this right ONCE, re-use it for every subsequent collab.
  Specifically for guest-artist sticker months / theme music drops
  — these have the multi-party benefit structure (artist gets
  paid + exposure, users get new content, Four Tasks gets
  marketing) and should become a recurring feature.

THE FOUNDER VISIBILITY PROBLEM:
  Morgan wants to be a visible, responsive solo dev. This is the
  right instinct — it differentiates from faceless app-startup
  competitors and matches the indie dev / community-driven ethos.
  But it also has limits.

  Sustainable visibility levels:
    HIGH: respond to substantive feature requests, bug reports,
    interesting user stories. These deserve real engagement.
    MEDIUM: post devlog content on a comfortable cadence. Don't
    promise weekly devlogs that turn into stress.
    LOW: don't try to respond to every comment on every video /
    post. Burnout is real. "Read everything, respond to what
    deserves response" is a sustainable posture.

  Be explicit with the community about this when the time comes.
  "I read everything but I can't respond to everything — please
  don't take silence as dismissal" said once, on a pinned post,
  buys a lot of grace.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SECTION 5 — OPEN QUESTIONS / DECISIONS TO MAKE LATER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Things on the table that need more thinking, listed for revisit:

1. APPtrioc atrioc/Ari onboarding edge cases. LIKELY THREE BUTTONS
   (atrioc / Ari / everyone else) with two hidden passwords —
   atrioc's password leaked in his chat 1-2 weeks pre-launch,
   Ari's leaked separately elsewhere to create a small treasure-
   hunt feel. Three buttons is slightly less elegant than two but
   eliminates the "both partners hit the wrong button" failure
   mode. Final UX decision deferred to APPtrioc implementation
   session. Recovery flow not needed if three-button is locked.

2. APPtrioc sticker catalogue scope. How many themed stickers?
   Which inside jokes are the headliners? Is the "spoontrioc"
   sticker its own theme or a one-off completion sticker? Worth
   a brainstorming pass with someone who's a deeper member of the
   community than Morgan — possibly an audience-recruited co-
   designer if anyone obvious volunteers.

3. APPtrioc data lifecycle. LOCKED: APPtrioc data is preserved
   indefinitely even if the bit dies. Cloudflare Pages + a separate
   D1 instance is cheap to keep alive indefinitely. Users keep
   their calendars regardless of atrioc-meme-relevance trajectory.

4. Migration path from APPtrioc to Four Tasks. If a user starts in
   APPtrioc, falls in love, and wants the paid native app — what's
   the import story? Probably "we don't import, you start fresh in
   Four Tasks." Honest framing: APPtrioc was the gift, Four Tasks
   is the product, they're not the same identity. But worth
   thinking about whether a partial export (calendar history as a
   keepsake image, not as importable data) would be valuable.

5. Video production logistics. Self-shot. Voice-over with screen
   capture / app demos. Morgan has a nice mic, script can be
   written at work, performance is not a concern. Music: minimal
   to none — Morgan has original hummable material if needed but
   the video is not aiming for essay-tier production. NO licensed
   music, NO royalty-free anything that could trip copyright.
   Honest-not-polished is the brand-aligned target.
   Hard editing time budget: ~20 hours total. Ship at the budget
   even with rough edges. "Perfection is the enemy of good" is the
   known failure mode to fight on this. The audience values
   authenticity over polish; over-editing actively hurts the
   message.

6. Launch-day playbook. The hour-by-hour: what goes live when?
   Reddit post timing relative to video drop, Discord/Twitter
   announcements, monitoring tools to catch the first surge of
   users, support inbox setup, etc. Draft a checklist before
   launch day so nothing gets forgotten in the rush.

7. Press / journalist outreach. Worth approaching indie game press
   (Rock Paper Shotgun, PC Gamer indie corner, smaller outlets),
   Australian tech press, productivity-focused blogs? Probably
   yes for some, no for others. Decide which subset matters and
   draft pitch emails. Belongs in its own session.

8. ASO (app store optimisation). Keywords, screenshots, store
   listing copy, app description format, video preview if doing
   one. Belongs alongside store submission tiles (5.6-5.11).
   Worth its own focused pass.

9. Naming for APPtrioc. "APPtrioc" is the working name. Is it
   THE name? Worth a sanity check against atrioc's own
   trademarks (if any), Twitch ToS around derivative naming, and
   the broader vibe — "is this name funny enough to land?"

10. Pricing for Four Tasks. RESOLVED in monetisation v2.0
    (session 6 part 2). Subscription with bilateral library
    sharing + small stacking coin bonus. Founders pricing tier
    for launch cohort. Year-2 standard pricing +50% above
    founders. See four_tasks_monetisation_position.md.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SECTION 6 — PROMO CODES + SUBCULTURE TARGETED LAUNCH
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Added session 7 (mobile). Pairs the subculture seeding plan in
Section 2 with a concrete promo code mechanism for acquisition.

CORE MODEL: 3-MONTH FREE TRIAL → FOUNDERS PRICING SOFT LANDING.

  Each promo code grants:
    1. A 3-month free trial extension on top of the standard 30-day
       trial. During this period, the user has subscription_active
       = true server-side. Library sharing is INCLUDED — they get
       the signature feature, not a peephole.
    2. Choice of one TIER 1 starter pack from the existing launch
       catalogue, attached to their account at redemption.
    3. On trial expiry, the user lands automatically on the
       FOUNDERS PRICING tier (locked rate until end of year 2).
       Not a price cliff — a soft landing into the lowest rate the
       studio commits to charging.

  This stacks THREE psychological mechanics:
    - Free period as acquisition lever (3 months is enough to
      live multiple pack drops, build streak history, develop
      habit dependency).
    - Founders price lock as loyalty reward (permanent until
      year 2 sunset).
    - Future price hike for non-founders (+50% at year 2+)
      activates as retroactive validation of early adoption.

  All three reinforce the buddy-ware tone. No predatory pattern.
  User feels rewarded for trusting the app early.

WHY 3 MONTHS, NOT 12:
  12 months was the initial sketch. Walked back because:
    - 12 months commits us to losing a full year of revenue per
      redemption.
    - The founders pricing lock already does the "you got in
      early" work — the trial extension doesn't need to be that
      generous to land.
    - 3 months is long enough for the user to fully internalise
      the product. The "soft landing into founders rate" is the
      sticky thing, not the trial length itself.

WHY LIBRARY SHARING IS INCLUDED IN TRIAL:
  Considered: "promo code gives you 90% of a sub but library
  sharing is the paywall." Rejected. Library sharing is the
  SIGNATURE feature of the v2.0 monetisation model. A trial that
  withholds it isn't a trial of the product — it's a trial of the
  unmonetised parts. The user can't develop attachment to the
  feature they'd be paying for. We give them the real thing, then
  ask them to pay for it after they've lived with it. Honest
  funnel.

CODE STRUCTURE PER SUBCULTURE:

  Each subreddit gets its own code: FIFO50, ADHD100, INDIEDEV50,
  RECOVERY50, etc. Naming convention: subculture identifier +
  cap number.

  Per-code constraints:
    - Hard redemption cap (50-200 per code depending on sub size
      / projected demand).
    - Short expiry (14 days from posting).
    - One redemption per user per code (server-enforced).
    - Each code attaches a contextually-relevant starter pack
      offer (FIFO → choice of TIER 1 pack with a marketing
      nickname "FIFO pack" but underlying it's the existing
      catalogue, e.g. "Swamp" branded for the campaign).
    - Code is invalid once cap is hit OR expiry passes —
      whichever first.

  Cap is critical. Without it, codes leak to deal aggregator
  subs and exposure is unbounded.

RETARGETING MODEL — WAVE-BASED, NO WAITLIST:

  When demand exceeds cap (tracked via attempted-redemption
  logging), the same sub can be retargeted in subsequent waves:

    Wave 1: SUB-50 (or whatever initial cap). 14-day expiry.
            Trial extension offer.
    Wave 2: SUB-50-WAVE2. Same offer. Posted 30+ days after
            wave 1 if demand was over-cap. Acknowledges
            previous overwhelm: "we saw last month's response,
            here are more spots."
    Wave 3: SUB-FOUNDERS. Larger pool. NO trial extension —
            direct access to founders pricing instead. Frames
            as: "this sub has been so responsive we want to
            offer the cheapest possible long-term price to
            everyone, not just first-N."

  Wave 3 is the high-volume option — lower per-user cost to us
  (no trial losses), still genuinely generous (founders pricing
  is a lifetime perk), and respects the sub's interest without
  picking favourites.

  Waitlist mechanism was considered and REJECTED — psychologically
  it's a queue, which is a control mechanism, which clashes with
  the buddy-ware tone. Over-cap redeemers see "this code is
  closed, watch this space for the next wave" instead of "you're
  in line."

OVER-CAP ATTEMPT TRACKING:

  Every redemption attempt — successful or rejected — logs to a
  `redemption_attempts` table. Schema sketch:
    code TEXT, user_id TEXT, success BOOLEAN, timestamp INTEGER

  Aggregate query per code gives the demand signal that informs
  whether/when wave 2 is justified. Also useful for:
    - Measuring code-level conversion 3 months post-redemption
      (which subs convert into long-term subscribers best).
    - Detecting bot/abuse patterns (same IP hammering codes).
    - Attribution for retention analytics over time.

STARTER PACK AT REDEMPTION:

  Initial sketch was bespoke packs per cohort ("FIFO mining pack",
  "ADHD fidget spinner pack"). Walked back — bespoke art is a
  weekend's painting per pack and gating launch on cohort-bespoke
  art doesn't scale.

  Locked: redeemers get CHOICE of one existing TIER 1 pack from
  the launch catalogue. The marketing copy can frame it
  contextually ("FIFO redeemers get a free starter pack — try the
  cooking pack or the swamp pack") but underlying it's the same
  set. The interest hook ("free starter sticker pack") lands
  without committing to per-cohort art.

  Bespoke packs reserved for RETENTION drops, not acquisition.
  E.g. a "1 year of FIFO redeemers" anniversary pack, after we've
  confirmed FIFO is a sustained subscriber segment, becomes
  worth the painting time.

REDEMPTION SURFACE:

  Onboarding: "Have a promo code?" text link on the welcome
  screen (screen 1 of standard onboarding flow per the onboarding
  doc). Goes to a small input screen, validates server-side,
  applies extension + pack on success, falls back to standard
  flow on failure.

  Existing users: settings → "redeem code" entry. Same endpoint.
  Same one-redemption-per-user enforcement.

SERVER ENDPOINTS NEEDED (Phase 5 work):

  POST /redeem
    Body: { code: string }
    Caller identified per standard auth model.
    Logic:
      1. Look up code in promo_codes table. 404 if not found.
      2. Check redemptions_used < cap. 409 if cap hit.
      3. Check expiry > now. 409 if expired.
      4. Check this user hasn't redeemed this code already.
         409 if duplicate.
      5. Log attempt (regardless of outcome) to redemption_attempts.
      6. On success: increment redemptions_used, set
         users.trial_extension_days += 90, attach chosen starter
         pack to user's owned packs, set
         users.founders_rate_eligible = true (locks in founders
         rate at trial expiry).
      7. Return new state.

  Schema additions (Phase 5 prep, NOT v1.0-blocker):
    promo_codes table:
      code TEXT PRIMARY KEY
      cap INTEGER NOT NULL
      redemptions_used INTEGER DEFAULT 0
      trial_extension_days INTEGER
      starter_pack_options TEXT (JSON array of pack IDs)
      expires_at INTEGER (unix timestamp)
      active BOOLEAN DEFAULT 1
    redemption_attempts table:
      code TEXT
      user_id TEXT
      success BOOLEAN
      timestamp INTEGER
    users additions:
      trial_extension_days INTEGER DEFAULT 0
      founders_rate_eligible BOOLEAN DEFAULT 0

  These tables + columns already exist in the v1.0 schema
  (shipped session 12, inert). What's pending is the `/redeem`
  endpoint, at Phase 5 entry.

APPLE / GOOGLE POLICY COMPLIANCE:

  Server-side trial extension does NOT touch Apple/Google's
  subscription state. Users still progress through the store's
  standard 30-day free trial flow at the store level. Our server
  treats them as entitled for additional N days beyond the store
  trial. Apple/Google get paid when the user converts to real
  billing post-extension. No store-level policy violation.

  The user's app-store subscription state is what
  RevenueCat/store webhooks report. Our trial extension is an
  ADDITIONAL flag we honour on top of the store state. If the
  user's store trial ends and they don't convert, our server
  still marks them as entitled for the remainder of their
  extension period.

ANTI-PATTERN: DON'T BLANKET-ISSUE CODES.

  The codes are intentionally scarce + targeted. Don't:
    - Issue a "WELCOME50" generic code that anyone can use.
    - Share the codes in Four Tasks' own marketing channels
      (defeats the per-subculture targeting).
    - Run codes longer than 14 days (defeats the
      scarcity-and-attention urgency).
    - Skip the cap (defeats the bounded exposure rule).
    - Make redemption easier than account creation (defeats
      the "this is a real product" framing).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
JOURNAL — RAW ENTRIES AS THEY ARRIVE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Timestamped entries below. Raw thoughts. Shaping happens when
critical mass accumulates and a thread justifies its own section.

[2026-05-12, session 5] Doc started. First major entry covering
APPtrioc plan + subculture targeting + streamer outreach + long-
tail strategy. Sections above are the structured form of what
Morgan dumped in conversation. Open questions in section 5 are
the actionable list of things to decide later. Journal entries
go here from now on.

[2026-05-12, session 5 follow-up] Iterated on the 10 structured
questions. Locked decisions:
  - APPtrioc stays web-only, NO Capacitor wrap. PWA install
    ceiling. Decision: lower operational burden over feature
    parity with Four Tasks.
  - APPtrioc onboarding: three buttons (atrioc / Ari / everyone
    else) with two hidden passwords. Eliminates "both partners
    hit wrong button" failure.
  - APPtrioc pairing model: UNILATERAL. Every regular user reads
    atrioc's calendar; atrioc doesn't read theirs. Ari is the
    bilateral exception. Architectural divergence from Four Tasks
    proper — needs its own design pass at APPtrioc implementation
    time.
  - APPtrioc sticker scope: 5-8 stickers total. Mix of inside-joke
    specific + palette-only fillers. Small catalogue creates
    paid-app pull.
  - Chain-import sweetener: build a one-off import for atrioc's
    existing don't-break-the-chain calendar so APPtrioc launches
    populated for him. Low engineering cost, high signal value,
    increases probability of organic stream mentions.
  - Video: self-shot, voice-over, minimal music (Morgan's own if
    any), hard 20-hour editing budget, ship at budget.
  - Target launch: July 2026 ideal, slips to Sep/Oct acceptable,
    past Nov forfeits first-NYE story. LAUNCH READINESS is the
    optimisation target, not the calendar — family takes
    precedence (father unwell).
  - Streamer outreach: scout pricing pre-revenue without
    commitment. Honest framing — "I'm building, not buying yet,
    want to understand your sponsorship process for later."
  - Founder visibility: hoping front-loaded bug work with Claude
    in Phases 1-4 reduces post-launch firefighting, freeing time
    for community presence. Bet on careful work now → easier time
    later.
  - Subculture pre-engagement: join subs Morgan would genuinely
    belong in (FIFO, indie dev, possibly others). Not duplicitous
    if genuine. Strategic karma-farming would be — line is clear
    internally.

Open questions remaining in section 5: 2 (exact sticker
catalogue), 4 (migration path from APPtrioc to Four Tasks), 6
(launch-day playbook), 7 (press / journalist outreach), 8 (ASO),
9 (APPtrioc naming sanity check), 10 (Four Tasks pricing —
references monetisation position doc).

[2026-05-12, session 5 architectural reframe — late session]
Major restructure of the APPtrioc launch architecture. The change:

  PRE-REFRAME:
    - APPtrioc is a fork of the buggy web prototype
    - Frontend cloned from prototype, backend either Firebase
      (Morgan's initial assumption) or Cloudflare
    - Launch is SIMULTANEOUS — Four Tasks store approval gates
      both launches landing the same day
    - APPtrioc is a "marketing stunt" sized opportunistically

  POST-REFRAME:
    - APPtrioc is a SNAPSHOT of the Four Tasks Godot project
      at tile 4.14a (basic picker, before context menu adds
      theme depth)
    - Exported via Godot HTML5 to Cloudflare Pages
    - Launch is SEQUENCED — Four Tasks ships to stores first
      (Phase 5), APPtrioc gets built+deployed afterward
      (Phase 5b), atrioc video drops once both are live
    - APPtrioc is a structural part of the product roadmap,
      not a marketing fork

  WHY THE REFRAME LANDED:
    Morgan's question: "should we not just be using the native
    build to better showcase the full builds potential? the
    proto is barebones and has bugs tbh." Pointed at the
    structural problem that had been glossed over — shipping a
    janky prototype as the community gift would have undercut
    the conversion pitch ("the full version is on stores"
    sounds like a bait-and-switch if the free version was a
    rough demo).

    Morgan's reframe: "build the full build in godot, and as
    we reach the point where we would be adding functionality
    to the icon picker to reskin the app, we stop there and
    call it a cutoff point for apptrioc iteration. we ship
    that and work out the backend, then once its stable go
    back into godot and keep pushing to the finish line."

    This is structurally cleaner than anything previously
    proposed. One codebase, one engine, one stack. The "fork
    point" becomes a real waypoint in Phase 4 development that
    determines what APPtrioc gets vs what's Four Tasks
    exclusive. Tile 4.14 split into 4.14a (pre-fork) + 4.14b
    (post-fork). New Phase 5b created for the fork+ship work.

  IMPLICATIONS:
    - Four Tasks launches on its own merits via subculture
      seeding. No atrioc dependency.
    - APPtrioc launches AS A SECOND WAVE, force-multiplying an
      already-shipped product. Risk profile of atrioc-not-
      engaging drops from "real concern" to "no big deal."
    - APPtrioc is the same polish and engine as Four Tasks,
      just with theme depth cut off and unilateral pairing
      configured. The gift is REAL, not janky.
    - Long-press on stickers in APPtrioc is DELIBERATELY
      INERT. Picker shows themed-sticker markers (glints/
      badges) hinting at depth users can't access. Built-in
      conversion mechanic.
    - Cloudflare-for-both confirmed. Different Workers, different
      D1 instances, same account. Firebase considered for
      APPtrioc and rejected — Spark tier caps fast against
      atrioc-audience scale, Blaze is unbounded billing,
      Cloudflare $5/month covers any plausible APPtrioc
      traffic with a hard cap.

  FILES TO UPDATE (committed in session-5 close batch):
    - This marketing doc (top-of-doc snapshot + gift spec +
      contingency + video timing + this journal entry).
    - Todo: new APPTRIOC FORK ARCHITECTURE section at top,
      tile 4.14 split into a/b, new Phase 5b, session 5
      ledger entries.
    - Devlog: session 5 entry covering theme + marketing +
      architectural reframe.

  STILL OPEN (carried forward):
    - APPtrioc-specific design pass once Phase 5b is the
      next active phase (schema, password gates, endpoint
      surface, rate limiting).
    - APPtrioc-to-Four-Tasks keepsake export (soft migration
      path) — captured as ACTIVE-tier todo.
    - APPtrioc naming sanity check (legal/community gut-check
      before deploy) — captured as tile 5b.0.

[2026-05-13, session 5 extended — morning sequence design closeout]
Tonight at work, slow shift between calls, used the time for
design conversation rather than marketing thinking. Eight design
questions on the morning sequence got locked into a new design doc
(four_tasks_morning_sequence_design_notes.md). No marketing
implications directly — it's pure product design — but worth noting
here because the morning sequence is the daily ceremony that
defines what Four Tasks FEELS like to use, and that feeling is
what the marketing has to communicate.

Three implications for future marketing copy:

  1. The video should show the morning sequence playing as part of
     its product demo. It's the most cinematic moment in the app
     and the most differentiated. Most habit trackers don't have a
     daily ceremony. Capturing 20-30 seconds of the sequence on
     screen in the launch video will sell more than any feature
     bullet point.

  2. The "private ceremony" framing of the morning sequence (Q4
     locked) — your partner doesn't see your stamp until you've
     claimed it — is a connection mechanic worth surfacing in
     marketing copy. "Watch your partner's morning land live"
     reads as warmer than "real-time sync." The partner panel
     waking up as your partner engages with their app is the
     feature; sync is the implementation.

  3. The rest day's affirmation-tone purple stamp (Q6) is
     marketing-friendly. Most productivity apps treat skipped
     days as failure. Four Tasks treats deliberately-rested days
     as their own kind of completion, with their own ceremony.
     "Rest is part of the work" is a copy direction. Affirmation
     of rest is a real differentiator against the wellness-shame
     pattern in the productivity-app category.

No action items from tonight that affect marketing doc structure.
Just noting the morning sequence doc exists and these three
copy-direction implications are worth folding into the eventual
video script + store-listing copy work.

[2026-05-14, session 7 — promo codes + subreddit targeting]
Morgan joined a bunch of relevant subreddits at work today and
started thinking about acquisition mechanics. Conversation
resolved into Section 6 above as a locked design.

Key thread: how to give subreddit cohorts a real welcome without
either (a) committing to bespoke art per cohort (unsustainable),
(b) leaking exposure via uncapped codes, or (c) gating the
signature feature behind paywall (defeats the trial).

Locked positions:
  - 3-month free trial via promo codes (not 12 — would commit
    a year of revenue per redeemer).
  - Soft landing into founders pricing on expiry, not a price
    cliff.
  - Year-2+ standard pricing +50% above founders (Morgan's
    initial instinct of 100% recognised as "latent greed" and
    walked back).
  - Caps small (50-200 per sub), expiry short (14 days).
  - Over-cap attempts tracked, retargeted via wave 2 (same
    offer, +30 days) and wave 3 (founders-only direct access,
    larger pool).
  - Waitlist mechanism REJECTED — psychologically a queue,
    clashes with buddy-ware tone. Code-closed users see "next
    wave coming" instead of "you're in line."
  - Library sharing INCLUDED in trial. Not gated. The signature
    feature must be lived during the trial or the trial isn't
    real.
  - Starter pack = choice of existing TIER 1 pack, not
    bespoke. Bespoke packs reserved for retention drops, not
    acquisition.
  - Three psychological mechanics stacked: free trial (urgency)
    + founders lock (loyalty) + future hike (retroactive
    validation). All reinforce buddy-ware tone, none predatory.

Server work needed for promo codes: the `/redeem` endpoint. The
`promo_codes` + `redemption_attempts` tables and the
`trial_extension_days` / `founders_rate_eligible` user columns
already SHIPPED in the v1.0 schema (session 12) — they sit inert,
no read/write path touches them yet. So the remaining work is the
endpoint plus a promo-codes design doc, at Phase 5 entry. NOT a
v1.0 store-ship blocker — but it IS a blocker for the first
targeted post: no promo code can be issued until the redeem
endpoint exists.

Marketing pre-engagement note: Morgan joining subs today is the
"belong before pitching" pattern. Genuine community membership
before any code drops. Strategic posture is sound — observe tone,
absorb pain points, eventually contribute, only then promote.
The promo code becomes a way to thank a community you've already
spent time in, not a cold pitch.

Open from this conversation:
  - Per-sub code naming conventions and starter pack defaults
    (e.g. FIFO50 → defaults to swamp pack but redeemer can
    choose? or no choice, FIFO redeemers get swamp specifically?).
    Probably "choice" for flexibility. Lock at Phase 5 code
    authoring.
  - Wave timing rules — exactly when does wave 2 land? 30+
    days is the floor; the ceiling depends on whether wave 1
    sold out and how recently.
  - Wave 3 mechanics — does "founders-only" mean a code that
    skips the trial extension entirely, or a code that grants
    a shorter trial (e.g. 30 days) plus founders rate? Probably
    the former for cleanliness. Confirm at Phase 5.
  - Attribution analytics dashboard — at what scale does this
    matter? Probably wave 3+ before it's worth building UI.
    Until then, raw queries against redemption_attempts.
