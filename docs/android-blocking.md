# App blocking on Android

Android has no Screen Time API. Prayer Lock implements blocking itself in a Kotlin Expo Module. Two problems shaped the design.

## Problem 1: you cannot start an activity from the background

The first version detected a blocked app in the foreground and called `startActivity()` to bounce the user back into Prayer Lock. Detection worked. Enforcement did nothing.

Since Android 10 (API 29), an app cannot start an activity from the background. With `targetSdk 36` the call is dropped silently: no exception, no log line a casual reader notices. Blocked apps were never interrupted.

**What shipped:** a full-screen **overlay shield**. When the foreground service sees a blocked package, it draws a `WindowManager` view of type `TYPE_APPLICATION_OVERLAY` on top of it. Holding `SYSTEM_ALERT_WINDOW` both allows the overlay and exempts the app from the background-activity restriction, so the shield's button can open Prayer Lock directly. (This is the one place Android is friendlier than iOS; see [Screen Time on iOS](screen-time-ios.md).)

The overlay is styled to match the in-app focus screen: background photo, dark vignette, a lock header, a status line and a glass card with the brand button. Its strings are passed from JS at `startBlocking(packages, { title, message, button })`, written to `SharedPreferences`, and read by the overlay, so the shield is localized like the rest of the app.

Two runtime permissions are required and are both gated on the blocked-apps screen:

- `PACKAGE_USAGE_STATS` (Usage Access): foreground app detection via `UsageStatsManager`, polled every 700 ms while a session is active.
- `SYSTEM_ALERT_WINDOW` (Display over other apps): the overlay shield.

Permissions never block starting a prayer. A missing permission shows a one-time "Set up" prompt and the prayer still starts, because the first prayer is where new users drop off (see [Growth](growth.md)).

## Problem 2: scheduling under Play policy

Daily prayer time has to fire without the app open. Prayer Lock uses `AlarmManager`:

- exact alarm when `SCHEDULE_EXACT_ALARM` is granted, inexact (±10 minutes) otherwise;
- `USE_EXACT_ALARM` is deliberately **not** requested. Play reserves it for alarm clocks and calendars, and a prayer reminder does not qualify.
- The foreground service declares `specialUse` as its type with an explanation for Play review.

## Shared state

`SharedPreferences` (`app_blocker_prefs`) plays the role of the iOS App Group: current session, shield strings, the `shield_requested_prayer` flag. The root layout consumes that flag through the same `BlockingService.consumeShieldPrayerRequest()` call used on iOS, so the JS side is identical.

## Widgets

Voltra's `updateWidget` is iOS-only. Android widgets use `voltra/android/client` and `VoltraAndroid` primitives with dp-size variants (110×110, 250×110, 250×250). They are separate component trees from the iOS ones. See [Home-screen widgets](widgets.md).

## Testing on the emulator

- Grant both permissions without the UI:
  ```
  adb shell appops set <package> GET_USAGE_STATS allow
  adb shell appops set <package> SYSTEM_ALERT_WINDOW allow
  ```
- Confirm the overlay is up: `adb shell dumpsys window windows | grep appop=SYSTEM_ALERT_WINDOW`
- A debug APK for all ABIs is ~250 MB and fills the default emulator image. Build for `arm64-v8a` only.
- Guest mode cannot start a prayer from the main button and billing is unavailable on emulators, so to exercise native blocking we call the Expo module directly from the Hermes inspector (`globalThis.expo.modules.AppBlockerModule.startBlockingWithTimeout([...])`) over Metro's CDP websocket.
- Environment variables are inlined at transform time. Changing `EXPO_PUBLIC_*` needs `--clear` on Metro.

## Status

Android reached feature parity with iOS in November 2026: scheduled blocking, shield-to-prayer routing, widgets, Play assets. Release is planned alongside the iOS 1.0.5 cycle; the waitlist is on [prayer-lock.com](https://prayer-lock.com/).
