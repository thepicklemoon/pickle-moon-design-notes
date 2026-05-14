# Four Tasks — Hidden Achievements Brainstorm

Status: BRAINSTORM, NOT LOCKED. Prompt-as-document.

Started session 7 mobile. Morgan tired, looking to capture a wild
thought experiment about hidden achievements rewarding the seekers
who find them with lifetime subscriptions for global-first unlocks.
Intent is to document the idea while fresh, not to commit to
implementation. This doc is a starting point for future Morgan to
react against — agree, disagree, prune, expand, add.

Implementation status: SPECULATIVE. May ship at v1.x, may never
ship, may ship in heavily modified form. Capture-it-while-fresh
discipline applies.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE CORE IDEA
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Achievements that are:

  1. PURE EASTER EGG. Not listed in any in-app achievement menu.
     Not surfaced via staggered disclosure. Not hinted at in
     onboarding. The only way to know they exist is to find them.

  2. UNLOCKED BY WEIRD BEHAVIOUR. Not "use the app for 30 days."
     Conditions that are creative, lateral, sometimes silly, often
     requiring deliberate sustained effort. Things the average
     user will never accidentally do.

  3. REWARDED WITH A LIFETIME SUBSCRIPTION FOR THE GLOBAL FIRST
     UNLOCKER. Server-side adjudication: the FIRST user worldwide
     to satisfy the condition gets the reward. Subsequent
     unlockers get the achievement itself (a private notification
     and maybe a hidden cosmetic) but not the lifetime sub.

  4. SEMI-VIRAL BY DESIGN. The reward is disproportionate enough
     that the lucky recipient will want to share. Reddit posts,
     screenshots, "I just got a free lifetime sub from this app
     I'd been using for a month" stories. The disproportion is
     the marketing.

  5. INDEPENDENTLY ANNOUNCEABLE BY THE STUDIO. Every global-first
     trigger pings the studio. If the recipient doesn't post
     publicly, THE PICKLE MOON can announce it themselves through
     official channels: "the first 'Embrace the Chaos' achievement
     was just unlocked. Lifetime sub awarded."

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHY THIS WORKS (AND WHY IT MIGHT NOT)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Why it works:
  - The hidden-Easter-egg framing matches the buddy-ware tone.
    The app is being generous in secret, not loud in marketing.
    Users discover surprises rather than being pitched on them.
  - Global firsts create cultural events. "Someone just unlocked
    The X achievement" becomes a community moment when it
    happens.
  - The reward is permanent for the recipient and zero ongoing
    cost vs the lifetime sub revenue we wouldn't have collected
    from a non-believer anyway.
  - Average global-first unlocks per achievement = 1. Worst case
    if we ship ten achievements: ten lifetime subs. Combined with
    the founders flag (~20-30) and any other free-for-life
    allocations, we're talking <0.1% of the user base at 10k+
    scale. Real but bounded cost.
  - Achievements that REQUIRE engagement-with-app over weeks or
    months are a retention mechanic disguised as a treasure hunt.
    A user pursuing an achievement is a user who's coming back
    every day.

Why it might not:
  - Adjudication has to be airtight. Once a lifetime sub is
    awarded, you can't pull it back. Concurrent unlocks, timezone
    edge cases, replay attacks, server clock drift — all real.
  - Cheating becomes economically motivated. Coin cheating was
    OK because the upside was decorative; lifetime-sub cheating
    is real revenue lost. Conditions that are EASY to forge
    locally (manipulate device clock, edit local state) are
    vulnerable.
  - Late adopters may feel locked out of "the cool stuff." If
    all ten globals are claimed by week one, year-two users have
    a known class of rewards they can never earn.
  - Detection logic must be RELIABLE. False positives award
    rewards we didn't mean. False negatives mean the first
    deserved recipient doesn't get credit; later genuine
    unlockers might trigger before them due to a logic bug.
  - The "global first" concept biases play. Users who know
    an achievement exists will try to engineer it. The hidden
    requirement is meant to prevent this — but seekers will
    share theories and the discovery race becomes a meta-game.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ADJUDICATION ARCHITECTURE — SKETCH
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

This is sketch, not lock. Real design pass needed if achievements
ship.

PER-ACHIEVEMENT TABLE:
  Server-side `achievements` table:
    achievement_id  TEXT PRIMARY KEY
    name            TEXT  (internal name — never user-facing)
    public_name     TEXT  (cute name shown in achievement notification)
    description     TEXT  (cute description shown in notification)
    condition       TEXT  (machine-readable spec of unlock condition)
    global_first    TEXT  (user_id of the first user to unlock, NULL if unclaimed)
    global_first_at INTEGER (timestamp of global first)
    total_unlocks   INTEGER (count for analytics — increments each unlock)
    active          BOOLEAN (allows enabling/disabling without deleting)

PER-USER UNLOCK RECORDS:
  `user_achievements` table:
    user_id        TEXT
    achievement_id TEXT
    unlocked_at    INTEGER
    PRIMARY KEY (user_id, achievement_id)

DETECTION:
  Most achievements detected server-side at relevant write
  endpoints (claim_morning, day-tick, coin-spend, etc). Each
  endpoint, after its main logic, runs the relevant achievement
  checks for that user.

  Some achievements need batch detection (e.g. "haven't spent
  coins in X days") — run as part of the morning sequence claim
  flow which fires once per user-day.

GLOBAL-FIRST RACE HANDLING:
  When a user qualifies, server runs an atomic UPDATE on the
  achievements table:
    UPDATE achievements
    SET global_first = ?, global_first_at = ?, total_unlocks = total_unlocks + 1
    WHERE achievement_id = ? AND global_first IS NULL

  If the UPDATE returns 1 row affected, this user is the global
  first. Award lifetime sub.
  If 0 rows affected (someone got there in a concurrent request),
  award standard achievement only, increment total_unlocks via
  separate UPDATE.

  No race possible because the SQL atomicity guarantees exactly
  one winner.

LIFETIME SUB AWARD:
  Server sets users.lifetime_sub = true. Similar to founders flag
  — perpetually treated as subscription_active. Doesn't go through
  Apple/Google billing (it's a studio-issued grant, not a store
  product).

  Pings the studio (Discord webhook, email, or internal admin
  panel notification) so Morgan knows it's happened and can
  announce.

ANTI-CHEAT POSTURE:
  Server-side adjudication only. Client never asserts achievement
  unlock — server detects from actual write history. Local clock
  manipulation can't fake conditions because the server tracks
  the timestamps it received writes on, not what the client
  claims happened.

  Some achievements (the "embrace chaos for a month" one) check
  conditions over time windows where the server has its own
  record of when each day was sealed. Client timezone can be
  spoofed but server-recorded write timestamps can't (well,
  beyond plausibility validation already in place).

  Honest limit: a sufficiently determined attacker could write
  a bot that hits the server's endpoints with plausible
  timestamps. We accept this. The reward isn't valuable enough
  to justify forging at scale — one lifetime sub at $4/month
  is ~$3000 of present-value revenue at best, and the effort
  to forge convincingly across many achievement conditions
  exceeds that for all but extremely persistent attackers. Not
  a meaningful adversary class for an indie habit tracker.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THE ACHIEVEMENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Ten starting candidates. Some are weird, some are wild, all are
the kind of thing only a seeker would find. Names are placeholder
where noted — they should ideally be derived from MOTD pool
vocabulary or written in that voice. Don't take any as final.

═════════════════════════════════════════════════════════
1. EMBRACE THE CHAOS
═════════════════════════════════════════════════════════

Condition: Leave random-reroll on the sticker picker turned on
every day for one calendar month AND don't spend any coins
during that period.

The user is letting the universe pick their sticker pool daily
and refusing to consciously curate. They're trusting the system.
Anti-control as a virtue.

Cute alt-names:
  "The Universe Provides"
  "Take What You're Given"
  "Whatever the Day Brings"

What it tells us about the user: they're playful, generous,
non-controlling. Probably exactly the kind of person who'd
post about getting a lifetime sub for being chill.

Detection: requires the daily-random-reroll feature to ship
(not yet designed — see "Implementation prerequisites" below).
Easy server-side detection once the feature exists.

═════════════════════════════════════════════════════════
2. ESPRESSO HEART
═════════════════════════════════════════════════════════

Condition: Complete the morning sequence within 60 seconds of
the user's local 5am for seven consecutive days.

The user is genuinely up at dawn, every day, opening the app
immediately. This is FIFO worker territory, early-shift workers,
parents of newborns, dedicated runners, monks. A specific
lifestyle commitment manifesting as a usage pattern.

Cute alt-names:
  "Before The Birds"
  "First Light"
  "Workman's Hour"

What it tells us: dedicated user with a real morning routine.
Probably testifies to the app's utility unprompted.

Detection: server checks claim_morning timestamps in user's
local timezone. Seven consecutive days within the 5:00-5:01
window. Tight tolerance is intentional — accidentally satisfying
is nearly impossible.

═════════════════════════════════════════════════════════
3. THE LONGEST SHORTEST DAY
═════════════════════════════════════════════════════════

Condition: Both partners complete all four tasks AND post their
MOTD on June 21 (or December 21 for southern hemisphere — the
solstice in user's local zone).

The solstice is the longest or shortest day depending on
hemisphere. A pair who're paying enough attention to the calendar
to sync up on this specific date is a pair with cultural
literacy and intentional play.

Cute alt-names:
  "Standing Still Sun"
  "The Pivot"
  "Midwinter / Midsummer"

What it tells us: intentional, ritual-aware users. The kind of
people who post seasonally on social media.

Detection: server checks date against user's location (already
roughly known via timezone) and confirms both partners' day
records show 4T + MOTD set. Limited to two unlock windows per
calendar year.

═════════════════════════════════════════════════════════
4. CONFESS NOTHING
═════════════════════════════════════════════════════════

Condition: Write 100 MOTDs that are EXACTLY one character long.

The user is using the MOTD slot but refusing to actually confess
anything. A wall of single characters. Refusing to play the
diary game while still playing the app. There's something
meditative or rebellious about it.

Cute alt-names:
  "Single Sigil"
  "One-Character Diary"
  "The Hermit's Page"

What it tells us: probably an artist, a minimalist, or someone
processing something private. Loves the app enough to use it
daily but doesn't want to share. Respect.

Detection: count of days where the user's diary field has
exactly length 1. Easy server-side.

═════════════════════════════════════════════════════════
5. THE FIRST AND LAST CHILLI
═════════════════════════════════════════════════════════

Condition: On a single day, complete all four tasks where each
task's text contains a chilli emoji (🌶️) and the user's active
sticker pool consists of exactly the chilli sticker (no other
stickers in pool, just the chilli).

Pure devotion to a single sticker theme. The user has built their
day around the chilli pack. This is identity-as-cosplay through
the medium of a habit tracker.

Cute alt-names:
  "Chilli Day"
  "All Heat"
  "Pepper Pilgrim"

What it tells us: theme superfan, willing to perform the bit.
Probably will post a chilli-themed screenshot the moment they
unlock.

Detection: server reads day's task labels + user's active
sticker pool. Strict match required.

NOTE: this is sticker-specific. A version of this achievement
exists for EVERY pack — frog day, vampire day, fairy day, etc.
The "global first" applies per-sticker, so the first chilli
day, the first frog day, the first wizard day, all award
lifetime subs independently. Or — alternatively — there's only
ONE such achievement and it's pack-agnostic (any sticker's
pack-day counts). Decide at implementation.

═════════════════════════════════════════════════════════
6. THE BUDDY EXCHANGE
═════════════════════════════════════════════════════════

Condition: Two users who are paired with each other both
SWAP their identity values (name, username, icon) on the same
day — i.e., user A becomes the values that user B used to be,
and vice versa.

This is impossible to do by accident. The pair has to
deliberately coordinate. They're playing with the identity
system as a couple's joke or a "let's see what happens" lark.
The achievement rewards genuinely creative engagement with the
underlying mechanic.

Cute alt-names:
  "Body Swap"
  "Identity Migration Day"
  "We're Each Other Now"

What it tells us: deeply engaged paired users with a sense of
humour and the bandwidth to coordinate a bit. Marketing-perfect
audience.

Detection: server detects pair-key migration events where both
users' new identity tuples are exactly the previous values of
their partner. Within a 24-hour window.

Subtle implementation note: name is immutable per pair-key v2.
So the achievement actually only works on username + icon swap
(since name can't change). Adjust accordingly. The "buddy
exchange" framing still lands even at partial swap.

═════════════════════════════════════════════════════════
7. ONE HUNDRED REROLLS
═════════════════════════════════════════════════════════

Condition: Spend 100 coins on MOTD rerolls in a single day.

The user has built up a coin balance and is burning it
specifically on rerolling their MOTD over and over, hunting
for the perfect message. They're treating the spinner as
slot machine. They want the universe to give them a SPECIFIC
message, not just any message. Beautiful.

Cute alt-names:
  "Hunting The Message"
  "Wheel of Fortune"
  "Until It's Right"

What it tells us: user who attaches genuine meaning to the
MOTD content. Probably has a strong sense of personal narrative
or self-talk. Would absolutely write about why they rerolled
100 times.

Detection: server tracks coin spending events; sum reroll-event
coins per user-day. 100 in a single day.

═════════════════════════════════════════════════════════
8. EQUAL AND OPPOSITE
═════════════════════════════════════════════════════════

Condition: For an entire calendar month, the pair has the
property that whenever one user 4Ts a day, the other rests, and
vice versa. Perfect complementary play.

The pair has internalised the asymmetric play pattern — when
one's on, the other's recovering. This is the philosophical
opposite of the 1.5x both-completed bonus everyone else chases.
This pair is playing a different game.

Cute alt-names:
  "Yin Yang"
  "Counterweight"
  "When You Go, I Stay"

What it tells us: thoughtful pair who've reframed the
mechanics intentionally. Probably explicitly philosophising
about the app's design with each other. Would write a Medium
post.

Detection: server checks each day of the calendar month — for
every day, exactly one user 4Ts and exactly the other rests.
No day where both 4T, no day where neither does, no day where
both rest.

═════════════════════════════════════════════════════════
9. NINE NINES
═════════════════════════════════════════════════════════

Condition: Have your streak hit exactly 999 days.

Three years of unbroken use. The kind of milestone that's
plainly visible but psychologically loaded. The user has been
with the app for the entire post-launch period and is staring
at the four-digit cliff.

The hidden Easter egg: the achievement fires AT 999 specifically,
not at 1000. The "almost" is the joke. The user will be expecting
something at 1000, get nothing, and quietly wonder. Then years
later — or via Reddit — they find out it was at 999 and feel
strangely seen.

Cute alt-names:
  "One Less Than Forever"
  "Almost There"
  "The Threshold"

What it tells us: a user who has built their life partly around
this app. A founder of the long-tail.

Detection: trivial — server checks streak count after each
day-tick.

NOTE: this only becomes claimable years after launch. The
"global first" race is delayed. Different vibe than the early-
unlock ones. Worth its own design pass at the time it becomes
imminent (e.g. announce the achievement existed once someone
hits 998).

═════════════════════════════════════════════════════════
10. THE GHOST CALENDAR
═════════════════════════════════════════════════════════

Condition: Maintain a 14-day streak entirely on REST DAYS.
Every day for two weeks: rest day, paid in coins, claimed.
No tasks completed at all.

The user is paying coins to deliberately not perform tasks for
two weeks. They're using the app to mark a period of intentional
inactivity — maybe sick, maybe on holiday, maybe processing
something. The app is bearing witness to their rest.

This achievement honours one of the most unexpected ways someone
might use the app: as a permission slip to rest.

Cute alt-names:
  "Sabbatical"
  "Permission Slip"
  "Witness My Rest"
  "Recovering Out Loud"

What it tells us: someone using the app for emotional support
rather than productivity. Likely the most heart-tugging story
of any unlocker. Would absolutely cry-post about getting a
lifetime sub.

Detection: server checks 14 consecutive sealed days where
every single one is rest_day = true and tasks_done = 0.

NOTE: this is the most emotionally loaded achievement on the
list. The framing in the unlock notification needs care. NOT
"you did nothing for 14 days!" but something like "the app saw
you, and rested with you."

═════════════════════════════════════════════════════════

(I wonder what other achievements there are with similar
rewards for global first?)

The list above is ten. There are dozens more shapes possible.
The catalogue should grow over time — new achievements seeded
quietly via app updates, with no announcement, waiting to be
discovered.

Some seed ideas for the future, undeveloped:
  - An achievement tied to a specific date with cultural meaning
    (Pi Day, leap day, app's own anniversary)
  - An achievement involving the MOTD vocabulary intersecting
    with the user's task labels in a specific way
  - An achievement requiring the user to use a SPECIFIC sticker
    on a SPECIFIC date that the sticker thematically connects to
    (the chilli sticker on Cinco de Mayo, the frog sticker on
    Leap Day, the wizard sticker on a solstice)
  - An achievement that requires three users in a chain (A
    paired with B; B later re-pairs with C; A re-pairs with D —
    a network of past connections in a "small world" pattern)
  - An achievement that ONLY UNLOCKS during a specific window
    each year (e.g., the week of NYE, or the equinoxes), making
    the global-first race seasonal
  - An achievement that requires the user to have NEVER changed
    their initial four task labels — committed to the original
    intent for a full year. The "iron will" achievement.
  - An achievement that fires when a user's MOTD content
    contains specific magical phrases from the MOTD vocabulary
    pool itself (eating the dragon, the snake in the garden,
    etc — whatever the actual vocabulary ends up being)
  - An achievement that requires the user to have used EVERY
    sticker in their library as calendar_leader at least once
    across the year
  - An achievement that requires the user to have RECEIVED a
    partner reaction on every single sealed day for a month
    (the partner has been deeply attentive)
  - An achievement involving the user's username — if it forms
    a palindrome, or if it spells something specific, or if it
    matches their partner's username in some way

The list should grow organically as more app behaviour is
designed. The constraint is just: every achievement should be
hidden, weird, only findable by seekers, and emotionally
satisfying when discovered.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
IMPLEMENTATION PREREQUISITES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Features that must exist BEFORE these achievements can be
detected. Most aren't implemented yet:

  - Daily-random-reroll for sticker pool (for #1 Embrace The
    Chaos). New feature, doesn't exist in current design.
    Worth its own design pass if pursued — automated pool reroll
    on day boundary with optional toggle.
  - Server-side coin spending event log (for #7 One Hundred
    Rerolls). Spending events need to be auditable, not just
    summed.
  - Pair-key migration event log (for #6 The Buddy Exchange).
    Migrations are atomic transactions already; just need their
    before/after states logged.
  - Server-stored streak length per user (for #9 Nine Nines).
    Probably already in the schema; confirm.
  - All other detections work with current planned schema +
    write rules.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DISCOVERY MECHANICS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Hidden Easter eggs need to be findable, eventually, or they
serve no purpose. Three possible discovery paths:

  1. ORGANIC. Users stumble into them by living the app. Some
     achievements (like #4 Confess Nothing or #10 Ghost Calendar)
     will be discovered by people who happen to have used the app
     that way for personal reasons. The achievement notification
     when it fires is their first hint.

  2. SHARED. Once one user finds and posts about an achievement,
     others can replicate. The community discovers the catalogue
     over time. The first finder of each achievement gets the
     lifetime sub; everyone after gets the achievement only.

  3. SEEDED. Studio drops cryptic hints in marketing channels —
     a single line in a YouTube devlog video, an in-app news
     post that says only "rest can be its own routine," etc.
     Patient seekers connect the dots. This is the most
     deliberate strategy.

ALL THREE CAN COEXIST. Discovery posture is a marketing decision,
not an architectural one. The achievements themselves work the
same regardless.

NOTIFICATION ON UNLOCK:
  When an achievement fires for any user (global first or not),
  the user gets a soft in-app notification:

    "You found something."
    [achievement public_name]
    [achievement description]
    [if global first: "You're the first to do this."
                       "Lifetime subscription unlocked."]

  Not loud. No fireworks. The vibe is "the app noticed." The
  global-first language is what triggers the screenshot impulse.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STUDIO ANNOUNCEMENT FLOW
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

When a global-first fires:

  1. Server detects, awards lifetime sub, sends webhook ping to
     studio Discord/email/admin panel.
  2. Studio is notified within minutes. Sees: which achievement,
     which user (username), local time.
  3. Decision: announce immediately, or wait to see if user
     posts publicly first?
     - Honest path: wait 24-48 hours for user to share, then
       announce regardless on official channels.
     - Quicker path: announce same-day with permission asked
       privately.
  4. Announcement style: cryptic. "Achievement #3 unlocked
     today. Solstice pair found." Don't reveal the condition —
     keep the discovery game alive for others.

This makes EVERY global-first unlock a marketing event without
spamming. Spaced naturally over time. Each one a small
cultural moment.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WHAT THIS DOC LEAVES OPEN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Everything, basically. This is a brainstorm.

Specific questions for future-Morgan to react against:

  - Lifetime sub or step down? (12-month founders rate is the
    alternative for less-load-bearing achievements.) Mixed
    rewards by achievement difficulty is an option.
  - Per-pack achievements or pack-agnostic? (#5 The First And
    Last Chilli specifically — does each pack get its own
    "all-day devotion" achievement?)
  - How visible is the achievement framework itself? Fully
    hidden, or is there a single secret menu somewhere that
    lists how many a user has unlocked (without revealing
    conditions)?
  - Anti-cheat tolerance. The honest position is "we don't
    bother defending against determined attackers." Is that
    OK at the lifetime-sub stake level? Probably yes, but
    worth confirming before shipping.
  - Achievement copy. Every notification text needs writing
    in the same voice as the MOTD pool. Voice consistency is
    part of the brand. Hire a copy pass when ready, or write
    by Morgan, or both.
  - Notification UX. Soft in-app banner? Modal? Lock-screen
    push (if push is on)? Worth experimenting at implementation
    time.
  - Are there REGIONAL or DEVICE-LOCAL achievements rather than
    only globals? E.g., the first user in Australia to unlock
    something, vs the global first. Adds a second tier of
    rewards (e.g. founders rate instead of lifetime). Defer.
  - Do achievements expire? Could "global firsts" for some
    achievements be available only during a specific season
    or month, creating a finite window? Defer.
  - Do partners share achievements? If user A unlocks one in a
    pair, does user B also get credit? Or is it per-user only?
    Probably per-user, but worth considering pair-shared
    versions specifically for relationship-themed achievements
    (#6 The Buddy Exchange, #8 Equal and Opposite).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELATED DOCS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  - four_tasks_monetisation_position.md — lifetime sub
    mechanism extends the founders flag concept. The "studio-
    issued perpetual subscription" pattern needs to be
    consistent across founders flag and achievement awards.
  - four_tasks_staggered_disclosure_design_notes.md — these
    achievements DO NOT use staggered disclosure. They are
    explicitly hidden, not revealed. Counter-example to the
    disclosure pattern.
  - four_tasks_morning_sequence_design_notes.md — MOTD
    vocabulary is the natural pool for achievement names.
  - four_tasks_pair_key_v2_design_notes.md — #6 The Buddy
    Exchange depends on pair-key migration semantics. Name
    immutability constrains the swap to username + icon only.
  - four_tasks_write_rules_design_notes.md — server-side
    detection points live in the write endpoints. Each
    relevant endpoint runs achievement checks after its main
    logic.
  - four_tasks_sticker_pack_brainstorm.md — pack-specific
    achievements scale with the catalogue. The brainstorm is
    where pack-specific "devotion day" achievements would be
    seeded.
