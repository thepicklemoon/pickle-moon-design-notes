# Four Tasks — Notifications Design Notes (tile 4.26)

Status: LOCKED (s33, Morgan decisions 1-8) — crude/local pass for the
daily-driver APK. Server push / FCM is v1.x (DEFERRED list).

## Scope

Two daily local notifications, Android only for v1.0 (Ayshe + Morgan both
Android). Tap = plain app open; the lazy morning claim fires naturally on
open, so no deeplink. No in-app toggle v1.0 (system settings cover it).
iOS plugs in at Phase 5 — the chosen plugin's GDScript interface is
unified, so the client code does not change.

## Plugin

`godot-mobile-plugins/godot-notification-scheduler` (cengiz-pz lineage —
the Godot-4 successor every archived Godot-3 plugin points at; on the
Godot Asset Store as "Notification Scheduler Plugin").

Why: Godot 4.x, unified Android/iOS GDScript interface, schedule/cancel
by integer id, permission request + grant/deny signals, ships a default
small icon (`ic_default_notification`), falls back to inexact alarms
automatically when `SCHEDULE_EXACT_ALARM` is absent (we never request
it — inexact ±15min is fine for these), and scheduled notifications
survive device reboots (plugin registers a boot receiver). MIT.

Requires **Use Gradle Build** in the Android export preset + the Android
Build Template installed (AAR injection via export plugin). One-time
setup; see install checklist below.

## The two notifications

Times are fixed v1.0. The customisation seam for v1.x is a single pair
of constants in `Notifications.gd` (`MORNING_HOUR/MINUTE`,
`EVENING_HOUR/MINUTE`) — every read goes through them, so in-app
customisation later means swapping the constants for Prefs-backed values
and nothing else.

**Morning — 6:30am local.**
- title: `your coins are waiting`
- body:  `come collect yesterday.`
- alt (if the primary reads wrong on a real lockscreen):
  title `good morning` / body `your coins are waiting.`

**Evening — 8:00pm local.**
- title: `before bed?`
- body:  `today's tasks are still unticked.`
- alt: title `still time` / body `you haven't ticked your tasks today.
  want to do that before bed?` (the s32 doc phrasing verbatim)

Copy ships lowercase to match the app's register. Strings live as
constants next to the time constants — same v1.x seam.

## Suppression model (re-derive, never remember)

There is no stored notification state. On every relevant change the
client cancels the full deterministic id range and reschedules the whole
forward window from current `State`. Suppression is therefore structural
— a suppressed notification is simply never in the schedule — and the
system self-heals from any crash, kill, or missed event at the next
rebuild.

- **Morning, day 0: never scheduled.** Any rebuild implies the app is
  open today, and an app open IS the claim (lazy claim on open/focus).
  So "cancel the morning notification once the ceremony has run" is
  satisfied structurally — today's morning is never in the schedule.
  Future mornings are always legit (new coins accrue daily).
- **Evening, day 0: conditional.** Skipped when today's row is a rest
  day, when `tasks_done` is 4/4, or when 8pm has already passed. An
  untick back below 4/4 before 8pm re-adds it (rebuild on `day_changed`).
- **Evening, future days:** skipped only if that date's row already
  carries `rest_day = true` (cheap, covers pre-set rests if they ever
  exist). Otherwise scheduled unconditionally — if the user opens the
  app that day, the rebuild makes it day 0 and the conditional logic
  takes over.
- **Mock clock (DevKit):** when `State.mock_today()` is set, the
  rebuild cancels everything and schedules nothing. Clock-game sessions
  stay notification-silent; clearing the mock restores on next rebuild.

## Lapse decay (Morgan decision 6)

Day 0 = the most recent rebuild (i.e. last app open — opening the app
resets the curve to daily). Notification days by offset:

- days 1–7: daily
- then the gap grows by one day per notification day: 9, 12, 16, 21,
  27, 34
- then weekly: 41, 48, 55 (horizon 56 days)

[VETO — interpretation] Morgan's spec was "every day for a week, then
every second day, then every third until every 7th day". The above
reads that as gap-grows-by-one-per-nudge. The alternative reading
(a full week at each gap size) stretches the curve to ~3 months;
rejected as nag-heavier for no benefit. Both morning and evening fire
on lapse days — also a call; flag if either should change.

Because the app rebuilds on open, the decay only ever plays out on a
device that genuinely isn't opening the app — which is exactly the
audience for it. Reboot persistence means the curve survives restarts
without the app running.

## Rebuild triggers + fingerprint

Triggers: plugin init/permission grant, `NOTIFICATION_APPLICATION_FOCUS_IN`,
`State.self_changed`, `State.cleared`, and `State.day_changed` for
(self, today). Rebuilds are gated by a fingerprint
(`loaded | mock | today_iso | evening-suppressed-today`) so repeated
focus-ins are free; scheduled alarms are absolute once set and need no
same-day refresh.

`morning_claimed` / `ceremony_changed` are deliberately NOT connected:
the claim changes nothing in the schedule (day-0 morning is structurally
absent).

## Id scheme

`MORNING_ID_BASE = 1000`, `EVENING_ID_BASE = 2000`, id = base + day
offset. Cancel sweep covers offsets 0–60 for both bases (122 cancel
calls per actual rebuild; cancelling a non-existent id is a no-op).
Deterministic ids mean no persisted bookkeeping — the sweep is the
re-derive.

## Permission (Morgan decision 3)

`POST_NOTIFICATIONS` requested straight at app boot (Android 13+; older
versions auto-grant). Denied → subsystem goes dormant for the session,
retried next boot. Granted later via system settings → picked up at next
boot. No exact-alarm or battery-optimization permissions requested.

## Onboarding gate

No identity / `State.is_loaded()` false → rebuild produces an empty
schedule (sweep still runs, so logout via `State.cleared` also clears
all pending notifications). First load after onboarding fires
`self_changed` → first real schedule.

## Install checklist (one-time, laptop or home PC)

1. Godot: **Project → Install Android Build Template** (creates
   `android/build/`). Commit it per upstream guidance.
2. Android export preset: tick **Use Gradle Build**. First export after
   this is slow (gradle bootstrap); subsequent exports are normal.
   Verify **INTERNET permission still ticked** after preset edits.
3. AssetLib in-editor: search `NotificationScheduler` → Download →
   Install to project root, **Ignore asset root** enabled.
4. **Project → Project Settings → Plugins**: enable
   `NotificationScheduler`. (Install the plugin BEFORE adding the
   autoload — `Notifications.gd` references the plugin's class names
   and won't parse without it.)
5. **Project → Project Settings → Autoload**: add
   `res://scripts/Notifications.gd` as `Notifications`, ordered AFTER
   `State` (it connects to State signals in `_ready`).
6. Export, `adb install -r`, device pass (below).

## Device pass (gates tile close)

- D-N1: fresh boot → permission dialog appears once; grant.
- D-N2: with tasks <4/4, set device clock to 7:59pm, background the app
  → 8:00pm evening nudge fires; tap opens app.
- D-N3: tick 4/4 before 8pm → no evening nudge.
- D-N4: rest-day today → no evening nudge.
- D-N5: background the app before 6:30am next day → morning fires;
  open app (ceremony runs) → no stale morning after.
- D-N6: deny permission → app behaves normally, zero notifications;
  re-grant in system settings + relaunch → notifications return.
- D-N7: reboot device with app closed → next scheduled notification
  still fires (boot receiver).
- (Lapse curve beyond day 1 is impractical to device-test; verified by
  code inspection of `_decay_offsets()` + one real multi-day gap during
  daily-driver use.)

## Deferred (v1.x lines)

- In-app time customisation + per-notification toggle (seam: the
  constants block in `Notifications.gd`).
- Server push / FCM (already on the DEFERRED list).
- iOS wiring at Phase 5 (plugin export setting + icon; client code
  unchanged).
- Copy variation per lapse depth — single fixed strings v1.0.
- DST-correct scheduling for non-fixed-offset timezones — same family
  as the IANA timezone v1.x item in the timezone doc; irrelevant in WA.
