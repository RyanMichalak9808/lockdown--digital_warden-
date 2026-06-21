# Lockdown — GrapheneOS device-owner app blocker

A personal device-owner app for GrapheneOS that can:
- One consolidated Apps list where every installed app gets exactly one mode: Allow, Block
  (suspended permanently, cannot launch at all — instant, OS-level, not a kill-after-launch
  hack), or Time Limit (allowed N minutes/day, adjustable inline, then suspended until the next
  2am reset). Block always takes priority if there's ever a conflict.
- Apply Settings restrictions (block VPN config changes, USB debugging, factory reset, etc.)
- Enforce a Private DNS hostname with DISALLOW_CONFIG_PRIVATE_DNS so the Settings screen for
  changing it is gone entirely — there's nothing to revert because there's no write path left
- Show every app's screen time today (Usage tab), pulled directly from UsageStatsManager
- Schedule Block-mode app suspension by day/time window
- Require the PIN every time the app is opened or resumed (not just once), with a Change PIN
  flow that re-verifies the current PIN first, plus a "remove everything" escape hatch

## 1. Build

Open this folder in Android Studio (or run from the command line if you have the Android SDK + Gradle installed):

```
./gradlew assembleDebug
```

The APK will be at `app/build/outputs/apk/debug/app-debug.apk`.

## 2. Install

With the phone connected via USB and USB debugging enabled:

```
adb install app/build/outputs/apk/debug/app-debug.apk
```

## 3. Provision as device owner

This MUST be done before adding any accounts to the device, and before any other
device admin app is active. On a freshly wiped GrapheneOS profile (owner profile,
before completing setup wizard account steps):

```
adb shell dpm set-device-owner com.digitalwarden.app/.AdminReceiver
```

Confirm it took:

```
adb shell dpm list-owners
```

## 4. First launch

Open the app. You'll be asked to set a PIN (used to gate the control panel, the PIN change
screen, and the escape-hatch "remove all restrictions" button). The PIN is required every time
the app is opened or resumed from the background, not just once per install. After setup you
land on four tabs:

- **Apps** — one list of every installed app. Each app has three mutually exclusive modes:
  **Allow** (no restriction), **Block** (suspended permanently, cannot launch at all), or
  **Time Limit** (allowed up to a minutes/day allowance you set inline, then suspended until
  the next 2am reset). Block always wins: switching an app to Block clears any time-limit
  tracking on it, and an app can never be both blocked and quota-tracked at once.
- **Settings** — toggle OS-level restrictions, set/enforce a Private DNS hostname, change your
  PIN, and the escape hatch.
- **Schedule** — add time windows (e.g. "10pm–7am, every day") during which the Block-mode
  apps are suspended; if no schedule exists, Block-mode apps are blocked all the time.
- **Usage** — every app's screen time today, sorted by most used, pulled directly from
  Android's own UsageStatsManager (the same data Digital Wellbeing would show, surfaced here
  since GrapheneOS doesn't ship Digital Wellbeing).

## 5. Test before you trust it

In order:
1. In the Apps tab, set one app to Block, confirm tapping its icon shows "app suspended"
2. Set a different app to Time Limit with a 1-2 minute quota, use it past the limit, and
   confirm it gets suspended within a few seconds (not minutes) of going over
3. Set an app to Block, then try also setting a time limit on it - confirm it stays in Block
   mode and the time-limit minutes field doesn't apply (Block has priority)
4. Toggle one restriction (start with something low-risk like DISALLOW_CONFIG_WIFI, not
   DISALLOW_DEBUGGING_FEATURES or DISALLOW_FACTORY_RESET) and confirm it actually takes
   effect in system Settings
5. Set a Private DNS hostname with enforcement on, then go to Settings → Network & internet →
   Private DNS and confirm the screen is now disabled/inaccessible rather than just reverting
   after you change it
6. Test "Change PIN" in the Settings tab: confirm it asks for your current PIN first, then lets
   you set a new one, and that you land back in the app without being asked again
7. Close the app fully (swipe from recents) and reopen it - confirm the PIN prompt appears
   again rather than dropping you straight into the Apps tab
8. Reboot the device and confirm everything re-applies without you touching the app, including
   the 2am reset alarm being re-armed
9. Test the escape hatch (PIN-gated "Remove all restrictions" button) BEFORE you turn on
   DISALLOW_FACTORY_RESET or DISALLOW_DEBUGGING_FEATURES, since those narrow your
   recovery options if something goes wrong

## Known limitations / things worth knowing

- **VPN bypass**: Private DNS has no authority over VPN-provided DNS. If you want DNS
  enforcement to be unconditional, also enable the "Block VPN configuration changes"
  restriction in the Settings tab.
- **Time limit polling granularity**: usage is checked every 5 seconds, so worst-case overshoot
  past a quota is a few seconds, not minutes. This trades a small amount of battery for that
  precision; if you don't need it that tight, raise POLL_INTERVAL_MS in UsageTrackerService.
- **Usage access permission**: the app silently grants itself PACKAGE_USAGE_STATS via
  DevicePolicyManager.setPermissionGrantState() since it's device owner. If that ever fails on
  a given OS build, the Time Limits feature won't see foreground app data — verify against
  step 4 above and grant Usage Access manually as a fallback if needed.
- **2am reset assumes the device is "on" at 2am or boots before the next poll**: if the phone
  is powered off across 2am, UsageTrackerService detects the overdue reset on the next tick
  after boot rather than waiting for the following night.
- **Package visibility**: app suspension relies on `getInstalledApplications()` seeing
  all installed apps. Device owner apps are exempt from Android's package visibility
  filtering, so this should work without extra permissions, but hasn't been verified
  against a real GrapheneOS build's behavior end-to-end — verify against step 5 above.
- **Exact alarms on Android 12+**: the schedule feature requests `SCHEDULE_EXACT_ALARM`
  in the manifest. If `canScheduleExactAlarms()` returns false at runtime (some OEMs/
  configurations restrict this), the code falls back to `setAndAllowWhileIdle`, which
  is not guaranteed to fire at the exact minute — Doze mode can delay it. The 2am reset alarm
  has the same fallback behavior.
- **Restrictions persist at the OS level** once added, independent of whether the app
  is still installed. The escape hatch removes them via `clearUserRestriction`/
  `clearAllUserRestrictions`, but if you ever uninstall this app without using the
  escape hatch first, restrictions may remain stuck until you re-provision device
  owner or factory reset.
- **No remote recovery**: GrapheneOS doesn't have a Google-account-based remote wipe/
  unlock fallback. If you lock yourself out (e.g. via DISALLOW_FACTORY_RESET combined
  with losing your PIN), a factory reset from recovery may be your only way back in,
  with full data loss. Keep the PIN written down somewhere safe.

## Project structure

```
AdminReceiver.java        - device admin entry point
ServiceStarter.java       - small helper for starting the foreground service
BootReceiver.java         - restarts service + re-arms schedule/reset alarms after reboot
LockdownService.java      - core enforcement: app suspension, restrictions, DNS lock, usage tracker
ScheduleManager.java      - AlarmManager scheduling logic (time-of-day app blocking windows)
ScheduleReceiver.java     - fires at window start/end, re-arms next week
UsageTrackerService.java  - polls UsageStatsManager, accumulates per-app usage, suspends on quota hit
UsageResetReceiver.java   - fires daily at 2am, resets usage counters and un-suspends timed apps
                              (never un-suspends a package that's also permanently blocked)
AppUsageStats.java        - read-only helper for the Usage tab's "screen time today" numbers
AppMode.java              - the three-way Allow/Block/Time Limit enum used by the Apps tab
TimedAppQuota.java        - per-app daily minute limit + today's usage + blocked flag
PrefsRepository.java      - JSON-backed persistence (SharedPreferences + Gson)
LockdownConfig.java       - the persisted state shape
TimeWindow.java           - a single schedule entry
PinUtil.java              - salted SHA-256 PIN hashing
SessionState.java         - in-memory-only "is the app currently unlocked" flag, never persisted
PinAuthActivity.java      - PIN gate / first-run setup / PIN reset flow, app's launcher activity
MainActivity.java         - four-tab control panel host; re-checks the PIN on every onResume()
AppsFragment.java / AppRuleAdapter.java       - consolidated Allow/Block/Time-Limit list, one row per app
RestrictionsFragment.java                     - DISALLOW_* checklist, DNS field, Change PIN, escape hatch
ScheduleFragment.java / ScheduleListAdapter.java - schedule UI
UsageFragment.java / AppUsageAdapter.java     - per-app screen time today
```
