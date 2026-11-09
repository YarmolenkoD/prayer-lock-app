# Shipping to the App Store with EAS

A React Native app with four native targets (app, three Screen Time extensions) plus a widget target hits problems a single-target app never sees. In order of time lost:

## `** ARCHIVE FAILED **` with no `error:` line

The default EAS image for Expo SDK 55 shipped Xcode 26.2 with Swift 6.2.3, which segfaults in the `SendNonSendable` SIL pass while compiling `expo-modules-core` (`SharedObject.emit(event:arguments:)`). A compiler crash, not a code bug: the log ends in `ARCHIVE FAILED` with no error line.

Fix: pin the image in `eas.json`:

```json
"build": { "production": { "ios": { "image": "macos-sequoia-15.6-xcode-26.1" } } }
```

## ITMS-90018: "bundle is invalid" on upload

The Xcode project had manual "Embed" shell-script phases that copied each `.appex` into `PlugIns/` with `cp -R`. In archive builds the `.appex` products in `BUILT_PRODUCTS_DIR` are **symlinks** into `IntermediateBuildFilesPath/UninstalledProducts/`. `cp -R` copied dangling symlinks into the IPA.

Fix: `cp -RL` (dereference). To diagnose, download the build's `.ipa`, unzip, and `ls -la Payload/*.app/PlugIns/`: real directories versus symlinks.

## "Doesn't include signing certificate"

Five targets had provisioning profiles split across two distribution certificates. Delete the old certificate in `eas credentials`, then "Set up all the required credentials" to regenerate every profile against one certificate. Family Controls, once granted by Apple, is account-wide; regenerate the shield-action profile to pick it up.

## Transporter 409 "is not a valid ID for this relationship"

Transporter sometimes rejects a valid app id. Skip it: `eas submit -p ios --profile production --latest` with an App Store Connect API key in `eas.json` works.

## Wrong extension point identifiers

Validation 409 at upload. See the table in [Screen Time on iOS](screen-time-ios.md#3-extension-point-identifiers-apple-actually-accepts).

## Do not bump build numbers by hand

With `autoIncrement: true` and `appVersionSource: local`, EAS CLI bumps `app.json` and every target's `Info.plist` right before upload. Bumping manually too skips a number. Commit the bump **after** the build from the files EAS changed.

## Listing text is locked during review

While a version is Waiting for Review or In Review, both version localizations and the app-info record are read-only through the API. Keep the listing in a JSON file and apply it with a script once a new editable version exists. The dry run validates character limits.

## Sentry source maps

Set the real org and project in `app.json` and in `ios/sentry.properties` and `android/sentry.properties`. Put the DSN in the EAS environment for every profile, and the org auth token as an EAS secret, so builds upload source maps. A placeholder org name gives "Project not found" at build time, and a missing DSN means Sentry silently never starts in production.

## Getting build logs without clicking

The EAS GraphQL API (`https://api.expo.dev/graphql`) accepts the CLI's session secret in an `expo-session` header. `builds { byId(buildId) { logFiles error { message errorCode } } }` returns signed log URLs you can download and grep.

## Store tooling we ended up writing

All against the App Store Connect API, all in the private repo's `scripts/`:

- apply the localized listing from JSON, with limit validation;
- upload screenshot sets per locale and device from an HTML render pipeline;
- create and populate custom product pages;
- create a product page optimization (A/B) test from those same screenshot sets, submit it, start it, and read its state.

The lesson: everything in App Store Connect that you will touch more than twice should be a script. The UI is slow, locks at the wrong moments, and does not diff.
