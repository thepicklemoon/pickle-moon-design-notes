# Envelope — Design Doc

*Working name. Likely to change once feature scope settles.*

**Author:** Morgan / The Pickle Moon (ABN 92 231 512 645)
**Status:** Concept. Post-Four Tasks app.
**Last updated:** May 2026

---

## One-line pitch

A private family messaging app built around **forced acknowledgement** — important messages can't be ignored, and everything else disappears.

## The problem

Group chats are broken for families:

- Important messages get lost in chatter ("did you see Mum's text about Saturday?")
- Some family members silently read but never react, which hurts the senders who care about that
- Meta owns the dominant family group chat (Messenger/WhatsApp) and harvests the data
- Important info and small talk live in the same scrollable soup

## The solution

Two message types, one app:

1. **Envelopes** — high-signal messages that require an acknowledgement reaction before the recipient can return to the app. Visible delivery and acknowledgement status to sender.
2. **Regular messages** — ephemeral by default, vanish after a disclosed window.

Plus a tight set of family-utility features layered on top once the core works.

---

## Principles

- **No ads, ever.** Ads suck the life out of the human race.
- **Privacy is the product.** No data sold, no algorithmic feed, no engagement traps.
- **Friction where it matters, frictionless everywhere else.** Sending an Envelope is deliberate; sending a regular message is instant.
- **Ephemeral by default, with disclosure.** People accept Snapchat's rules because Snapchat is loud about them. Same here.
- **One feature done well beats five half-done.** Envelope mechanic ships alone in 1.0. Everything else is roadmap.

---

## Core feature: The Envelope

### Sender flow
- Write message → choose audience (any subset of your connections, not bound to a fixed "group") → mark as Envelope → send.
- Optional: save audience as a preset for reuse. Minimal UI footprint — a small "Saved groups" dropdown, no dedicated screen.

### Recipient flow
- Push notification arrives. App icon shows a wiggling envelope.
- Opening the app: large animated envelope takes over the screen. Cannot be dismissed, backgrounded out of, or scrolled past.
- Tap to open → message displays → user must select one acknowledgement reaction before returning to the rest of the app.
- Acknowledgement is recorded and visible to sender in real time.

### Reaction set
- A curated set of **custom emojis from the sister app's collection** — large, proud, expressive. Designed by Morgan.
- v1.x: families can create and contribute their own custom emojis to their family pool.
- The set should be small enough that choosing one feels meaningful, not like dismissing a modal. Probably 6–10 options covering: acknowledge, agree, will-do, can't-right-now, need-to-talk, love, plus a few expressive vibes.
- **Open question:** do we split "acknowledgement" reactions from "vibe" reactions, or trust one set to cover both? Leaning toward one set — splitting it adds UI complexity and feels like a checkbox at the doctor's office.

### Sender visibility
- Real-time per-recipient status: Sent → Delivered → Opened → Acknowledged (with which reaction).
- For unopened Envelopes after a threshold (24h? configurable?), sender sees a clear "unopened by: Dad, Aunt Jen" status. Decide later whether to nag-notify the recipient.

### Cannot be muted
- Envelopes bypass Do Not Disturb-style suppression within the app.
- Users can **snooze** an individual Envelope for up to 2 hours. After that, the wiggling envelope returns.
- OS-level notification muting is out of our control, but the in-app wiggle persists until acknowledged.

### Persistence vs ephemerality
- Envelopes disappear like regular messages but on a longer fuse (default: 30 days vs 24h-7d for regular).
- Either user can **star** an Envelope or regular message to keep it locally. Starred messages live on the device, not the server — keeps the backend clean.
- **Export** option for sentimental archive (PDF or plain text dump). Better for backend hygiene than letting users hoard everything server-side.

---

## Connections, not groups

- Users connect to other users (phone number or invite code — TBD).
- For each message/Envelope, sender picks audience from their connections list.
- No persistent "group" entity required. Presets exist for convenience only.
- This keeps the model simple and avoids "who's the admin" politics that wreck WhatsApp groups.

---

## Ephemerality

- **Regular messages:** default lifespan TBD. Candidates: 24h, 72h, 7d. Lean toward 7d — long enough to feel like a conversation, short enough that nothing lingers.
- **Envelopes:** 30d default.
- **Disclosed and obvious.** Countdown visible on each message ("disappears in 4d 2h"). Onboarding makes this loud.
- **Star to keep** (local-only). **Export** for archival.
- No edit history, no "deleted message" placeholders. Gone is gone.

---

## Roadmap (1.x, not launch)

These are family-utility features Morgan's family currently uses third-party apps for. Each one needs the core Envelope mechanic to be proven first.

- **Shared family calendar.** Clean, uncluttered UI. Family events only, not a Google Calendar competitor.
- **Shopping list.** Shared, real-time. "Who's bringing what" mode for events.
- **Secret Santa.** Annual, low-effort.
- **Date polls.** Structured Envelope variant — multi-choice, everyone must respond before it closes. Same core primitive, different config.
- **Custom family emoji pool.** Family members upload/draw emojis that become part of their group's reaction set.

Anything that doesn't directly serve family communication or coordination is out of scope.

---

## Open decisions

### Pricing
Three options, none obviously right:

1. **99c one-time per user** — simplest, but every family member has to pay, which creates evangelism friction.
2. **$5–10 one-time per family** — host pays, invites are free. Better adoption curve, harder to enforce technically.
3. **Free passion project** — eat the costs, ship for love.

Per-message pricing rejected: friction on sending kills volume, volume is the whole game.

Ads are off the table permanently.

### Encryption
**Decision:** committed to end-to-end encryption in principle, but acknowledge the cost.

**What E2EE buys us:** the privacy pitch is real, not marketing. Matches Morgan's values. Strong differentiator vs Meta.

**What E2EE costs us:**
- No server-side search across message history
- No password recovery — lose key, lose messages
- More complex multi-device sync
- Push notifications harder (can't include message preview server-side)
- Cannot help a locked-out family member recover

**Honest threat model for "family picnic data":** approximately zero. Realistic alternative is server-side encryption at rest + TLS in transit + clean privacy policy + no data sales. Delivers "we don't read your messages" without the operational pain.

**Lean:** still do E2EE because the brand demands it, but eyes open. Decide before architecture is locked.

### Platform & stack
- **Both platforms day one.** iOS and Android.
- **Cross-platform confidence needed.** Not Godot — wrong tool for real-time sync, push notifications, and E2EE plumbing.
- **Candidates:** Flutter, React Native, or native iOS + native Android with a shared backend.
- Decide closer to build start. Four Tasks app ships first.

---

## Non-goals (explicit)

- Not a Messenger/WhatsApp replacement for the wider world. Family-scale only.
- No public profiles, discovery, follower counts, or anything resembling social media.
- No AI features unless they serve the core (e.g. nothing generative for its own sake).
- No business/team use case. If that demand appears, it's a separate product.
- No web client at launch. Mobile-first, mobile-only initially.

---

## Risk register

- **Adoption:** every family member needs the app or it's useless. Single biggest risk. Mitigation: nail the Envelope value prop so one person can sell it to the rest.
- **Forced acknowledgement could feel hostile** if mis-tuned. Mitigation: the snooze valve, the reaction options including "can't right now."
- **E2EE recovery scenarios** will produce angry support emails. Mitigation: extremely clear onboarding, optional encrypted cloud backup with user-held key.
- **Solo dev bandwidth** — this is post-Four Tasks. Don't start until that's shipped and stable.
- **Pixel art / custom emoji volume** is already a known bottleneck on the other app. This compounds it.

---

## Decisions still to make before build

1. Pricing model (three candidates above)
2. E2EE vs server-side encryption — final call
3. Stack choice (Flutter / React Native / dual native)
4. Regular message default lifespan
5. Reaction set: one merged set or split acknowledgement/vibe?
6. Connection mechanism: phone number, invite code, both?
7. Backend: roll own, Firebase again, or something purpose-built for messaging (e.g. Matrix protocol, Stream Chat)?
