# Screen Time on iOS: what the API lets you do, and what it does not

Prayer Lock blocks apps with Apple's Screen Time frameworks: `FamilyControls`, `ManagedSettings`, `ManagedSettingsUI` and `DeviceActivity`. They are powerful and thinly documented. These are the things that cost us the most time.

## 1. The shield button cannot open your app

The shield is the full-screen overlay iOS draws on top of a blocked app. You customize it with a `ShieldConfiguration` extension and handle its buttons in a `ShieldAction` extension. The natural design is a button that says "I'm ready to pray" and opens your app.

**That is not possible.** `ShieldActionDelegate` runs in a system extension process with no `UIApplication` and no injected `NSExtensionContext`. The only valid responses are `.none`, `.close` and `.defer`. This holds on iOS 16 through 26 (Apple Developer Forums thread 719905, Feedback FB17261679 still open).

Things we tried that do not work and that you should not ship:

- Key-value-coding your way to an `extensionContext` and calling `open(_:)`. Never fired, and it is a private API path that risks App Store rejection.
- Deep links from the shield. No API to open a URL from the action extension.

What shipped instead:

1. The button handler writes a `shieldRequestedPrayer` flag with a timestamp into the App Group and posts an **immediate local notification** ("Tap to start your prayer"). Then it returns `.close`.
2. The notification is `.timeSensitive`, which needs the `com.apple.developer.usernotifications.time-sensitive` entitlement on both the app and the action extension.
3. The user taps the notification. That is the one way iOS lets you launch the app from this flow.
4. On `AppState` becoming `active`, the root layout calls a native `consumeShieldPrayerRequest()`. If the flag is fresher than five minutes, it starts a prayer session and routes to the focus screen.

The shield subtitle tells users to tap the notification, because a button that only closes the shield feels broken otherwise.

Two app-side bugs made this look even more broken, and both are worth checking in your own app:

- **Notification permission was never requested** at the moment blocking was enabled; only reminder screens asked. Guests who skipped reminders had no permission, so the notification never appeared and the tap looked dead. Fix: request permission when Screen Time authorization is granted, and show a banner with a Settings link on the blocked-apps screen when it is denied.
- **Cold-start race.** The auth store set `user` before `isLoading` turned false. The shield consumer pushed the focus screen, then the index route saw a logged-in user and replaced it with home. Fix: the consumer waits for `isLoading === false`.

For diagnosing, the action extension writes to an `extensionLogs` array in the App Group (authorization status, notification `add()` result). A hidden screen in the app, opened by tapping the version label five times, shows and shares those logs. Ask for that dump before guessing.

## 2. Shield colors are tints, not fills

`ShieldConfiguration.backgroundColor` is **not painted opaquely**. iOS applies it as a tint over a system material. A saturated orange `#F2804A` renders as pale peach on a real device. The simulator shows it saturated, which is why we shipped a build with white text on what turned out to be a cream background.

Rules that held up:

- Test shields on a physical device. The simulator lies about shield appearance.
- Assume a light base. Use dark ink text, a solid-color button, and `backgroundBlurStyle: .systemUltraThinMaterialLight` so the base stays light in dark mode too.
- Do not put white text, light rays or cream buttons on the shield expecting a strong background behind them.

## 3. Extension point identifiers Apple actually accepts

Wrong `NSExtensionPointIdentifier` values pass local builds and fail at upload with an opaque validation 409. The correct ones:

| Extension | `NSExtensionPointIdentifier` |
|---|---|
| DeviceActivityMonitor | `com.apple.deviceactivity.monitor-extension` |
| ShieldAction | `com.apple.ManagedSettings.shield-action-service` |
| ShieldConfiguration | `com.apple.ManagedSettingsUI.shield-configuration-service` |

Note the asymmetry: ShieldAction is under `ManagedSettings`, ShieldConfiguration under `ManagedSettingsUI`.

## 4. Family Controls distribution entitlement

The `com.apple.developer.family-controls` entitlement needs a one-time request to Apple for distribution builds. Once granted it applies account-wide; you do not re-request per target. Every target that touches the frameworks (app, monitor, shield config, shield action) needs the entitlement in its provisioning profile.

## 5. Config plugins cannot create extension targets

Expo config plugins can edit entitlements, Info.plists and the App Group, and copy source files into `ios/`. They cannot add Xcode targets. The three Screen Time extensions and the widget target were created once in Xcode, and `ios/` is committed. Keep the Swift sources in a top-level `extensions/` folder and have the plugin copy them in, so changes stay reviewable in Git.

## 6. Localizing the shield

The extensions run outside the JS runtime, so i18n has to be duplicated in Swift. Prayer Lock ships the shield copy in ten languages inside the extension, with a daily rotating line, and the app writes the user's chosen language into the App Group so the extension can pick it.

## Summary for anyone starting out

- Plan for the shield to close, not to open your app. Design the notification path from day one.
- Request notification permission at the same moment you request Screen Time permission.
- Test on device. Shield colors, time-sensitive notifications and extension logs all behave differently there.
- Put the identifiers above in your plists before your first upload.
