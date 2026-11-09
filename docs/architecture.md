# Architecture

Prayer Lock is a React Native app (Expo SDK 55, React Native 0.83, Expo Router, TypeScript) with native modules on both platforms. This note explains how the pieces fit, so the other notes make sense.

## One rule: the app never branches on platform for blocking

All blocking goes through a `BlockingService` that talks to a `BlockingProvider` interface:

```
BlockingService
  └─ BlockingProvider
       ├─ IOSBlockingProvider      → native ScreenTimeApi Expo Module (Swift)
       ├─ AndroidBlockingProvider  → native AppBlocker Expo Module (Kotlin)
       └─ MockBlockingProvider     → in-memory simulation (Expo Go, tests, web)
```

Hooks, stores and the root layout call the service. They do not know which platform they run on. The provider decides what "start blocking these apps until further notice" means on its OS.

Every call into a native module is wrapped in a lazy `require` with a try/catch. If the module is missing (Expo Go, an old dev client, a unit test), the app falls back to the mock provider instead of crashing at startup. The same pattern is used for Google Sign-In, Screen Time, store review and RevenueCat.

## Module layout

```
app/                     Expo Router screens
src/
  components/ hooks/ stores/ (Zustand) lib/ services/ i18n/ theme/
  lib/offline/           snapshot cache, outbox, sync orchestrator
  services/blocking/     BlockingService + providers
modules/
  screen-time-api/       Expo Module, Swift: FamilyControls, ManagedSettings, DeviceActivity
  app-blocker-android/   Expo Module, Kotlin: UsageStats, overlay shield, AlarmManager
extensions/              source copies of the iOS app extensions (synced into ios/ by a config plugin)
ios/                     bare, committed Xcode project (three Screen Time extensions + widget target)
android/                 bare, committed Gradle project
plugins/                 Expo config plugins (Screen Time entitlements, App Group, extension Info.plists)
scripts/                 App Store Connect API tooling (listing, screenshots, custom product pages, A/B tests)
supabase/                schema, migrations, edge functions
```

Both native modules are imported by relative path from `src/` rather than from `node_modules`. They are `file:` dependencies, so Yarn copies them into `node_modules`; that copy goes stale and TypeScript would otherwise resolve types against the old version while autolinking builds the new one.

## iOS: how JS reaches the Screen Time extensions

Apple's Screen Time stack runs partly in the app and partly in separate extension processes that the app cannot call directly:

| Piece | Runs in | What it does for Prayer Lock |
|---|---|---|
| `FamilyControls` authorization and `FamilyActivityPicker` | app | Ask permission, let the user choose apps and categories |
| `ManagedSettingsStore` | app | Apply or clear shields on the chosen apps |
| `DeviceActivityMonitor` extension | system process | Wake up at scheduled prayer time, apply shields |
| `ShieldConfiguration` extension | system process | Draw the shield the user sees over a blocked app |
| `ShieldAction` extension | system process | Handle the shield's button taps |

The only shared memory between these processes is an **App Group** (`UserDefaults(suiteName:)`). Prayer Lock uses it for the selection token, the current session state, a "the user asked to pray from the shield" flag, localized shield copy, and extension logs. A native Expo Module exposes read and write functions to JS.

The extension targets have to be created in Xcode by hand; config plugins cannot add targets. What the plugin does do: write entitlements (Family Controls, App Group, time-sensitive notifications), fix the extension Info.plists, and copy the Swift sources from `extensions/` into `ios/` so they stay version-controlled alongside the JS.

The `ios/` folder is committed and EAS builds it as-is, without `expo prebuild`. That is a deliberate trade: manual target setup survives, at the cost of applying some config changes by hand.

## Android: how JS reaches the blocker

The Kotlin module owns a foreground service that polls `UsageStatsManager` for the foreground package, a `WindowManager` overlay that acts as the shield, an `AlarmManager` scheduler for the daily prayer time, and a `SharedPreferences` file that plays the same role as the iOS App Group. See [App blocking on Android](android-blocking.md).

## Data

Supabase holds profiles, prayer sessions, moods, notification settings and onboarding answers. Row-level security scopes everything to the authenticated user, including anonymous guests. Streak and total-prayer counts are recomputed server-side by a trigger so offline sessions that arrive late still count for the right day. See [Offline-first sync](offline-first.md).

## Observability

Sentry for crashes and permanent sync failures. PostHog for product events (session started, cancelled with elapsed seconds and progress, completed, paywall shown, purchase) and for feature flags that drive paywall experiments. Source maps are uploaded from EAS builds.
