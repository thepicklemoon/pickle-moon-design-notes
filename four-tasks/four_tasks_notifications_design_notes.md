# Four Tasks — Notifications Design Notes (tile 4.26 + reliability)

Status: LOCKED. Original crude/local pass LOCKED s33 (Morgan decisions 1-8);
**RELIABILITY layer LOCKED s47/48** — and it REVERSES s33 decision 7 (see the
banner below). This is the single authority for the notification permission +
scheduling + delivery model. Server push / FCM remains v1.x (DEFERRED).

This doc supersedes the two prior files it was merged from (the s33 notifications
doc + the s47 reliability draft). Everything about WHAT is sent and WHEN — copy,
times, suppression, decay, ids — is unchanged from s33 and still authoritative.
What changed is HOW a scheduled nudge survives to fire on the devices real users
carry.

---

## ★ DECISION REVERSAL (s47/48) — exact alarms are now REQUIRED

The s33 doc decided: *"we never request `SCHEDULE_EXACT_ALARM` — inexact ±15min
is fine for these,"* and leaned on the plugin's automatic fallback to inexact
alarms. **That decision is now WRONG and is reversed.** It is the root cause of
Ayshe's 8pm reminder never firing on her Samsung.

Why it was wrong: inexact alarms are the FIRST thing Samsung's One UI defers or
drops, and on stock Android 13+ an app that wants reliable timed delivery while
closed needs the exact-alarm permission it was never requesting. "±15min is
fine" assumed the alarm fires at all — on Samsung, idle, it doesn't.

Every passage in the old doc implying inexact-is-sufficient or
"no exact-alarm permission requested" is DEAD. The corrected model is below
(Permission + the four-layer strategy). The scheduling LOGIC (decay, ids,
suppression, fingerprint) is untouched by this reversal — only the alarm TYPE
and the permissions change.

---

## Why reliability is a launch blocker (the problem, plainly)

Ayshe's 8pm reminder did not fire on her Samsung. That is not an edge case — it
is the core loop failing for the majority of the install base. For a habit
tracker the reminder IS the product: a daily nudge that doesn't arrive is a
calendar nobody opens. If ~60% of installs are Samsung (Morgan's field estimate,
consistent with AU Android share) and default One UI silently kills scheduled
alarms, the product is broken for most people who install it.

Two distinct Android mechanisms conspire, and they are NOT the same fix:

1. **Exact-alarm gating (stock Android 13+).** Since Android 13 a normal app
   doesn't get exact alarms for free; since Android 14 `SCHEDULE_EXACT_ALARM` is
   *denied by default* on fresh installs targeting API 33+. Schedule an exact
   alarm without the right permission and it's refused (and the app can be
   flagged at Play submission). This is almost certainly the gap on Ayshe's
   phone — the app requests POST_NOTIFICATIONS (confirmed granted) but never
   requested the exact-alarm grant. "Can't remember if the right permission was
   given at boot" = it wasn't wired.

2. **OEM battery killing (Samsung One UI, the worst offender).** Samsung's own
   developer docs confirm: an app unused for ~3 days drops into a restricted App
   Standby bucket where **Jobs, Alarms, and foreground services are throttled**;
   ~16 days → "deep sleep" (runs only when opened). "Put unused apps to sleep" is
   ON by default and re-sleeps apps after OS updates/reboots even when manually
   excused. A deliberate divergence from stock Android. No API lets the app
   exempt itself silently.

The hard constraint shaping everything: **Google Play prohibits most apps from
directly requesting battery-optimisation exemption.** Apps get rejected for
declaring `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` and firing
`ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` (Syncthing is the canonical
rejection). So the strategy can't be "silently whitelist ourselves." It must be:
hold the right install-granted permission, request the user-grant permissions
through *allowed* intents, guide the user through OEM settings when needed, and —
critically — design so a dropped nudge is never a lost day.

---

## Permission model (REVERSED from s33 decision 3+7)

- **POST_NOTIFICATIONS** — requested at app boot (Android 13+; older auto-grant).
  Denied → subsystem dormant for the session, retried next boot; re-grant via
  system settings is picked up next boot. UNCHANGED from s33. (This is the only
  part of the original permission decision that survives.)
- **Exact alarms — NOW REQUESTED.** See the USE vs SCHEDULE decision below.
- **Battery exemption — requested via the POLICY-ALLOWED intent only.** Never the
  direct whitelist intent on the Play build.

### The anchor decision: USE_EXACT_ALARM vs SCHEDULE_EXACT_ALARM

Two ways to be allowed to schedule an exact alarm. The pick is load-bearing — it
sets the permission model AND the Play submission risk.

| | `USE_EXACT_ALARM` | `SCHEDULE_EXACT_ALARM` |
|---|---|---|
| Grant | Install-time, normal. No prompt. | User must grant; *denied by default* on Android 14 fresh installs. |
| Revocable | Only by uninstall | User or system can revoke (and Samsung sleep effectively does) |
| Play policy | Publishable only if the app qualifies as "calendar or alarm clock" sending "calendar reminders, wake-up alarms, or alerts when the app is no longer running." **Reviewed at submission.** | Allowed for any app, but needs an in-app rationale + grant flow. |
| Risk | Submission rejection if Google decides a habit tracker isn't "alarm/calendar." | No submission risk; ongoing reliability risk (revocation + default-denied friction). |

**DECISION — declare BOTH, layered, lead with USE_EXACT_ALARM.**

- Declare `USE_EXACT_ALARM`. A fixed daily habit reminder fired "when the app is
  no longer running" is a good-faith fit for the policy's "alerts when the app is
  no longer running." Install-granted, so it's the ONLY option giving a fresh
  Samsung install reliable exact alarms with zero friction and no revocation
  surface. Primary mechanism.
- ALSO declare `SCHEDULE_EXACT_ALARM` as the fallback if Google rejects
  `USE_EXACT_ALARM` at review. With both declared: if USE is granted it wins and
  SCHEDULE is moot; if USE is ever stripped, the app falls back to the
  request-grant flow.

**THE SUBMISSION RISK, stated honestly (re-read at submission):** "habit tracker
daily reminder" is borderline for `USE_EXACT_ALARM` — Google names
calendar/alarm-clock apps explicitly. Some reminder apps ship it; some get
bounced. The Console shows a Declaration prompt. Mitigation:
  - In the declaration, frame the use as user-set, time-of-day reminders fired
    while the app is closed (true).
  - Keep the `SCHEDULE_EXACT_ALARM` + battery-guidance path ready (it's built
    anyway, Layer 2), so rejection costs a re-submission, not a redesign.
  - If rejected: drop `USE_EXACT_ALARM`, keep `SCHEDULE_EXACT_ALARM` + the
    first-run flow. App still works; Samsung reliability leans harder on the
    battery card + the open-anyway fallback.

**The plugin already supports all of this.** `godot-mobile-plugins/
godot-notification-scheduler` exposes `request_schedule_exact_alarm_permission()`,
the `schedule_exact_alarm_permission_granted` / `_denied` signals, a
battery-optimisation request with a `battery_optimizations_permission_granted`
signal. Its docs note: **"Obtaining the IGNORE_BATTERY_OPTIMIZATIONS permission
will also allow scheduling of exact alarms — additionally requesting
SCHEDULE_EXACT_ALARM permission is not necessary."** The hooks exist; the gap is
that `Notifications.gd` calls only `request_post_notifications_permission()`. This
is wiring + a flow + a manifest change, not new plumbing.

---

## The four-layer reliability strategy

Each layer is independently valuable; they stack. Build top-down — Layer 1 alone
probably fixes Ayshe.

### Layer 1 — Hold the permission, schedule with `setExactAndAllowWhileIdle`

- **Manifest** (Android export → permissions): add `USE_EXACT_ALARM` +
  `SCHEDULE_EXACT_ALARM`. Confirm `RECEIVE_BOOT_COMPLETED` (reboot survival — the
  plugin re-schedules on restart but needs the receiver) and INTERNET stay
  ticked after preset edits.
- Alarms must use the "and allow while idle" variant
  (`setExactAndAllowWhileIdle`) so Doze doesn't defer them. Plugin's job —
  VERIFY the plugin's exact path uses it (read its Android source; don't assume).
- **Boot flow** (`Notifications.gd`): after POST_NOTIFICATIONS, check the
  plugin's exact-alarm state / `canScheduleExactAlarms()` equivalent. Not held →
  Layer 2.

### Layer 2 — A first-run permission sequence (allowed intents only)

A short staged sequence at the RIGHT moment — not the cold-open, not buried. Per
staggered-disclosure, ask when the value is obvious: right after onboarding /
task-set, when the user has just expressed intent to build a habit.

Each step gated on the prior:
1. **POST_NOTIFICATIONS** (already wired). Denied → no notifications, app still
   works (Layer 4); offer a re-ask entry point in settings, don't nag.
2. **Exact alarm**: if `USE_EXACT_ALARM` granted at install, skip. Else
   `request_schedule_exact_alarm_permission()` → the system **Alarms &
   reminders** page (`ACTION_REQUEST_SCHEDULE_EXACT_ALARM`, the allowed intent).
   One toggle.
3. **Battery exemption** — allowed path only:
   - DO use `ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS` (opens the battery
     settings list — allowed for any app), or the plugin's battery request ONLY
     IF it routes through a policy-safe intent.
   - DO NOT fire `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` (direct one-tap
     whitelist) on the public Play build — that's the Syncthing rejection. (Fine
     for a sideloaded/internal test build; must not ship.)
   - Frame as benefit: "Samsung and some phones pause apps to save battery, which
     can silence your reminders. Tap below to keep Four Tasks awake." Deep-link
     to the right screen.

Each grant fires a plugin signal → `Notifications.gd` re-runs its rebuild (add
the permission-changed signals as rebuild triggers — see below). Sequence is
RE-ENTRANT + idempotent (house rule): a half-done sequence resumes; a granted
step no-ops; never destructive.

### Layer 3 — Samsung-specific guidance (the honest can't-fully-fix-in-code layer)

Even with Layers 1-2, One UI's sleep can re-restrict the app after a few days
idle, and re-sleep it after an OS update. No silent fix. The grade-A move is a
short, dismissible, OEM-aware help card:

- Detect `Build.MANUFACTURER` and, only on Samsung (et al.), surface a one-time
  card: "On Samsung phones, add Four Tasks to 'Never sleeping apps' so your
  reminders always arrive," with the exact path (Settings → Battery → Background
  usage limits → Never sleeping apps) and a deep-link where possible.
- OPT-IN and dismissible — guidance, not a wall. Don't block the app on it.
- This is the difference between "go figure out your battery settings yourself"
  (unacceptable, what we have now) and "here's the one tap that fixes it, on your
  specific phone." Per-OEM paths: dontkillmyapp.com. Samsung first;
  Xiaomi/Oppo/Vivo/Huawei have their own killers if the base broadens.

### Layer 4 — The fallback that makes a dropped nudge survivable (defensive design)

The most important architectural point, and it depends on NO permission landing.
Android alarms are *best-effort by contract* — even a correctly-permissioned
exact alarm can be dropped by an OEM. So the app must never rely SOLELY on the
notification to keep the habit intact.

Four Tasks already has the right shape: server is source of truth; the client
polls + re-derives on open; the morning claim walks back and seals lazily, so
opening the app reconstructs correct state regardless of whether a nudge fired.

**Design rule (already mostly true — make it explicit and VERIFY it holds): a
missed notification costs at most a missed nudge, never a missed day or a broken
streak.** The moment the user opens the app, state is correct and streak math is
intact (the economy doc's rescue/grace mechanics cover the human cost of a truly
missed day). The notification is the IDEAL path back, not the ONLY one. This caps
the blast radius of every OEM that defeats Layers 1-3: worst case is "she didn't
get the 8pm ping but her streak was fine next morning." Tolerable. A broken
streak from a silenced reminder is not.

---

## The two notifications (UNCHANGED from s33)

Times fixed v1.0. The v1.x customisation seam is a single pair of constants in
`Notifications.gd` (`MORNING_HOUR/MINUTE`, `EVENING_HOUR/MINUTE`) — every read
goes through them, so in-app customisation later means swapping the constants for
Prefs-backed values and nothing else. Copy strings live as constants next to the
time constants, same seam.

**Morning — 6:30am local.**
- title: `your coins are waiting`
- body:  `come collect yesterday.`
- alt: title `good morning` / body `your coins are waiting.`

**Evening — 8:00pm local.**
- title: `before bed?`
- body:  `today's tasks are still unticked.`
- alt: title `still time` / body `you haven't ticked your tasks today. want to do
  that before bed?`

Copy ships lowercase to match the app's register.

## Suppression model (UNCHANGED — re-derive, never remember)

No stored notification state. On every relevant change the client cancels the
full deterministic id range and reschedules the whole forward window from current
`State`. Suppression is structural — a suppressed notification is simply never in
the schedule — and self-heals from any crash/kill/missed event at the next
rebuild.

- **Morning, day 0: never scheduled.** Any rebuild implies the app is open today,
  and an app open IS the claim. Future mornings are always legit (new coins daily).
- **Evening, day 0: conditional.** Skipped when today is a rest day, `tasks_done`
  is 4/4, or 8pm has passed. An untick back below 4/4 before 8pm re-adds it
  (rebuild on `day_changed`).
- **Evening, future days:** skipped only if that date's row already carries
  `rest_day = true`. Otherwise scheduled unconditionally — opening the app that
  day makes it day 0 and the conditional logic takes over.
- **Mock clock (DevKit):** when `State.mock_today()` is set, rebuild cancels
  everything and schedules nothing; clearing the mock restores on next rebuild.

KNOWN ACCEPTED EDGE (bug sweep s47): a FUTURE-dated rest-day set does not
immediately reschedule — `_on_day_changed` filters its rebuild trigger to today
only. Self-heals on next app open. Acceptable; do NOT "fix" into a per-future-day
reschedule (the open re-anchor already covers it).

## Lapse decay (UNCHANGED — Morgan decision 6)

Day 0 = most recent rebuild (last app open — opening resets the curve to daily).
Notification days by offset: days 1-7 daily; then the gap grows one day per nudge
(9, 12, 16, 21, 27, 34); then weekly (41, 48, 55; horizon 56).

[VETO — interpretation, retained] Morgan's spec was "every day for a week, then
every second day, then every third until every 7th day." The above reads that as
gap-grows-by-one-per-nudge. The alternative (a full week at each gap size)
stretches to ~3 months; rejected as nag-heavier for no benefit. Both morning and
evening fire on lapse days.

Because the app rebuilds on open, the decay only plays out on a device genuinely
not opening the app — exactly the audience for it. Reboot persistence means the
curve survives restarts.

## Rebuild triggers + fingerprint (UNCHANGED + one addition)

Triggers: plugin init/permission grant, `NOTIFICATION_APPLICATION_FOCUS_IN`,
`State.self_changed`, `State.cleared`, `State.day_changed` for (self, today).
**ADD (s47/48):** the plugin's exact-alarm + battery permission-changed signals
as rebuild triggers, so a mid-session grant re-arms the schedule immediately.

Rebuilds gated by a fingerprint (`loaded | mock | today_iso |
evening-suppressed-today`) so repeated focus-ins are free; absolute alarms need
no same-day refresh. `morning_claimed` / `ceremony_changed` deliberately NOT
connected (day-0 morning is structurally absent).

## Id scheme (UNCHANGED)

`MORNING_ID_BASE = 1000`, `EVENING_ID_BASE = 2000`, id = base + day offset. Cancel
sweep covers offsets 0-60 for both bases (122 cancel calls per rebuild;
cancelling a non-existent id is a no-op). Deterministic ids = no persisted
bookkeeping; the sweep is the re-derive.

## Onboarding gate (UNCHANGED)

No identity / `State.is_loaded()` false → rebuild produces an empty schedule
(sweep still runs, so logout via `State.cleared` clears all pending). First load
after onboarding fires `self_changed` → first real schedule.

---

## Build tiles (what this doc authorises)

Server-before-client doesn't apply (pure client + manifest); order is risk-first.

- **N1 — Manifest + Layer 1.** Add USE_EXACT_ALARM + SCHEDULE_EXACT_ALARM (+
  confirm RECEIVE_BOOT_COMPLETED/INTERNET). Wire `Notifications.gd` to request
  the exact-alarm permission after POST_NOTIFICATIONS; add the plugin's
  exact-alarm + battery signals as rebuild triggers; verify the plugin schedules
  with exact-and-allow-while-idle. **This tile alone most likely fixes Ayshe** —
  ship + verify before building the rest.
- **N2 — Layer 2 first-run sequence.** The staged permission flow at
  end-of-onboarding, re-entrant, with a settings re-entry point. Allowed intents
  only — NO direct battery-whitelist intent on the Play build.
- **N3 — Layer 3 Samsung card.** Manufacturer-detected, dismissible, deep-linked
  guidance card. Samsung first.
- **N4 — Layer 4 audit.** VERIFY (don't assume) that opening the app after a
  missed nudge yields correct streak/seal state on a real lapsed-day scenario. A
  TEST tile against existing behaviour, but the one that makes the whole thing
  safe — it gates "notifications done."

## Install checklist (one-time, laptop or home PC — UNCHANGED + manifest note)

1. Godot: **Project → Install Android Build Template** (creates `android/build/`).
   Commit per upstream guidance.
2. Android export preset: tick **Use Gradle Build**. First export after this is
   slow (gradle bootstrap). Verify **INTERNET still ticked** after preset edits.
   **(s47/48) Add USE_EXACT_ALARM + SCHEDULE_EXACT_ALARM to the permission list;
   confirm RECEIVE_BOOT_COMPLETED.**
3. AssetLib in-editor: search `NotificationScheduler` → Download → Install to
   project root, **Ignore asset root** enabled.
4. **Project Settings → Plugins**: enable `NotificationScheduler`. (Install the
   plugin BEFORE adding the autoload — `Notifications.gd` references the plugin's
   class names and won't parse without it.)
5. **Project Settings → Autoload**: add `res://scripts/Notifications.gd` as
   `Notifications`, ordered AFTER `State`.
6. Export, `adb install -r`, device pass (below).

## Device pass — logic gates (UNCHANGED from s33, D-N1..D-N7)

- D-N1: fresh boot → permission dialog appears once; grant.
- D-N2: tasks <4/4, device clock 7:59pm, background → 8:00pm evening fires; tap
  opens app.
- D-N3: tick 4/4 before 8pm → no evening nudge.
- D-N4: rest-day today → no evening nudge.
- D-N5: background before 6:30am next day → morning fires; open (ceremony runs) →
  no stale morning after.
- D-N6: deny permission → app normal, zero notifications; re-grant in settings +
  relaunch → notifications return.
- D-N7: reboot with app closed → next scheduled notification still fires (boot
  receiver).
- (Lapse curve beyond day 1 impractical to device-test; verified by inspection of
  `_decay_offsets()` + one real multi-day gap during daily-driver use.)

## Device pass — DELIVERY matrix (NEW s47/48, gates LAUNCH)

The D-N pass above tests scheduling LOGIC. This tests DELIVERY on real hardware —
a named launch gate, not "Morgan's phone + Ayshe's" as an afterthought:

- **D-D1 Pixel (stock Android):** exact alarm fires at the minute, app closed,
  after reboot. Baseline, roughly proven.
- **D-D2 Samsung One UI (Ayshe's + ideally one more One UI version):** the real
  gate. Fires with the app (a) freshly used, (b) idle 3+ days (the sleep bucket —
  the actual failure mode), (c) after reboot, (d) with the battery card's "Never
  sleeping" exemption applied vs not. Her D-N1 flow rides the new sequence.
- **D-D3 permission-denied paths:** POST_NOTIFICATIONS denied → app works, no
  notifications, re-ask entry present. Exact-alarm denied → graceful fallback,
  never crashes, never blocks.
- **D-D4 the Layer-4 net:** on each device, force a missed nudge (deny/sleep),
  open next morning → streak + seal correct. Must pass everywhere even when
  delivery fails.

## Open questions (decide at build time)

- Does the plugin's battery request route through the policy-SAFE intent
  (`ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS`) or the prohibited direct one?
  READ the plugin's Android source before calling its battery method on the Play
  build. If it fires `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`, don't use it
  — deep-link the settings screen manually.
- Wording of the three permission prompts + the Samsung card — plain-spoken,
  benefit-framed, no corporate language. Write at build time.

## Deferred (v1.x)

- In-app time customisation + per-notification toggle (seam: the constants block).
- Server push / FCM (already DEFERRED).
- iOS wiring at Phase 5 — iOS local notifications have NONE of this OEM-killing
  problem and a simpler permission model; the plugin's GDScript interface is
  unified so client code is largely unchanged. The whole four-layer strategy
  above is Android-only.
- Copy variation per lapse depth — single fixed strings v1.0.
- DST-correct scheduling for non-fixed-offset timezones — same family as the IANA
  timezone v1.x item; irrelevant in WA.
